
## Lab 4: CodeDeploy Blue/Green deployments

In AWS CodeDeploy, blue/green deployments help you minimize downtime during application updates. They allow you to launch a new version of your application alongside the old version and test the new version before you reroute traffic to it. You can also monitor the deployment process and, if there is an issue, quickly roll back.

With this new capability, you can create a new service in AWS Fargate or Amazon ECS  that uses CodeDeploy to manage the deployments, testing, and traffic cutover for you. When you make updates to your service, CodeDeploy triggers a deployment. This deployment, in coordination with Amazon ECS, deploys the new version of your service to the green target group, updates the listeners on your load balancer to allow you to test this new version, and performs the cutover if the health checks pass.

**Note:** Although not necessary, however it is recommended that you have completed lab-3 above as this lab uses examples that are relevant if you have completed lab-3. This lab can be attempted after lab-2, however be mindful of references of SERVICE_NAME = threads.

1. Setup an IAM service role for CodeDeploy

Because you will be using AWS CodeDeploy to handle the deployments of your application to Amazon ECS, AWS CodeDeploy needs permissions to call Amazon ECS APIs, modify your load balancers, invoke Lambda functions, and describe CloudWatch alarms. Before you create an Amazon ECS service that uses the blue/green deployment type, you must create the AWS CodeDeploy IAM role (ecsCodeDeployServiceRole).

* Create a file named CodeDeploy-iam-trust-policy.json

    <pre>
        {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "",
                "Effect": "Allow",
                "Principal": {
                    "Service": [
                        "codedeploy.amazonaws.com"
                    ]
                },
                "Action": "sts:AssumeRole"
            }
        ]
    }
    </pre>

* Create the role with name ecsCodeDeployServiceRole

    <pre>
    aws iam create-role \
    --role-name ecsCodeDeployServiceRole \
    --assume-role-policy-document file://CodeDeploy-iam-trust-policy.json
    </pre>
    
* Since the compute platform we'll be working with is ECS, use the managed policy AWSCodeDeployRoleForECS

    <pre>
    aws iam attach-role-policy \
    --role-name ecsCodeDeployServiceRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSCodeDeployRoleForECS
    </pre>
2. Lets pick **threads** service for this lab.

Since the services we deployed in previous labs use ECS as the deployment controller, it is not possible to change this configuration using the update-service API call. Hence, we need to either a) delete the service or, b) create a new service with a different deployment controller i.e. CODE_DEPLOY. For this lab, we'll go with option a)

* Change the desired count for this service to 0

    <pre>
    aws ecs update-service \
    --region us-east-1 \
    --cluster my_first_ecs_cluster \
    --service threads \
    --desired-count 0
    </pre>
    
    **Note:** Wait for the running count to be 0
    
    <pre>
    aws ecs describe-services \
    --region us-east-1 \
    --cluster my_first_ecs_cluster \
    --service threads \
    --query "services[*].taskSets[*].runningCount"
    </pre>
    
* Delete the service once the runningCount = 0

    <pre>
    aws ecs delete-service \
    --region us-east-1 \
    --cluster my_first_ecs_cluster \
    --service threads
    </pre>

3. Lets update the db.json file for threads microservice and add another thread to it.
4. Once done, build a new docker image with a tag of 0.1. Tag & push this image to ECR repository of threads. By now, you should be familiar with this process.
5. Re-use the taskDefinition file for threads i.e. fargate-task-def-threads.json
6. Update the ecs-service-threads.json file to reflect CODE_DEPLOY as the deployment controller

    <pre>
    {
        "cluster": "my_first_ecs_cluster", 
        "serviceName": "threads", 
        "taskDefinition": "threads-task-def:1", 
        "loadBalancers": [
            {
                "targetGroupArn": "<b>arn:aws:elasticloadbalancing:us-east-1:776055576349:targetgroup/threads-tg/b61b2b03ecb4c757</b>", 
                "containerName": "threads-cntr", 
                "containerPort": 3000
            }
        ], 
        "desiredCount": 1, 
        "clientToken": "", 
        "launchType": "FARGATE",
        "schedulingStrategy": "REPLICA",
        "networkConfiguration": {
            "awsvpcConfiguration": {
                "subnets": [
                    "<b>subnet-06437a4061211691a</b>","<b>subnet-0437c573c37bbd689</b>"
                ], 
                "securityGroups": [
                    "<b>sg-0f01c67f9a810f62a</b>"
                ], 
                "assignPublicIp": "DISABLED"
            }
        }, 
        "deploymentController": {
            "type": "<b>CODE_DEPLOY</b>"
        }
    }
    </pre>

7. Re-create the **threads** service

    <pre>
    aws ecs create-service \
    --region us-east-1 \
    --cluster my_first_ecs_cluster \
    --service-name threads \
    --cli-input-json file://ecs-service-threads.json
    </pre>

8. Create a new target group & a listener for green environment

These will be referenced in the deployment-group you'd create for CodeDeploy.

* Target Group

    <pre>
    aws elbv2 create-target-group \
	--region us-east-1 \
	--name <b>threads-tg-2</b> \
	--vpc-id <b>VPCId</b> \
	--port 3000 \
	--protocol HTTP \
	--target-type ip \
	--health-check-protocol HTTP \
	--health-check-path / \
	--health-check-interval-seconds 6 \
	--health-check-timeout-seconds 5 \
	--healthy-threshold-count 2 \
	--unhealthy-threshold-count 2 \
	--query "TargetGroups[0].TargetGroupArn" \
	--output text
   </pre>

* Listener with a different port

    <pre>
    aws elbv2 create-listener \
    --region us-east-1 \
    --load-balancer-arn <b>arn:aws:elasticloadbalancing:us-east-1:012345678912:loadbalancer/app/alb-container-labs/86a05a2486126aa0</b> \
    --port 8080 \
    --protocol HTTP \
    --default-actions Type=forward,TargetGroupArn=<b>arn:aws:elasticloadbalancing:us-east-1:012345678912:targetgroup/threads-tg-2/b0ce12f4f6957bb7</b> \
    --query "Listener[0].Listener.Arn" \
    --output text
    </pre>

**Note:** Make note of the targetGroupArn & listenerArn

9. Create CodeDeploy resources

* Create the application which is a collection of deployment groups and revisions.

    <details>
    <summary>INFO: What is an Application? </summary>
    A name that uniquely identifies the application you want to deploy. CodeDeploy uses this name, which functions as a container, to ensure the correct combination of revision, deployment configuration, and deployment group are referenced during a deployment.
    </details>

    <pre>
    aws deploy create-application \
    --region us-east-1 \
    --application-name threadsApp \
    --compute-platform ECS
    </pre>

* Create an input json file for deployment-group named deployment-group-threads.json

    <details>
    <summary>INFO: What is a Deployment Group? </summary>
    From an Amazon ECS perspective, specifies the Amazon ECS service with the containerized application to deploy as a task set, a production and optional test listener used to serve traffic to the deployed application, when to reroute traffic and terminate the deployed application's original task set, and optional trigger, alarm, and rollback settings.
    </details>

    <pre>
        {
        "applicationName": "threadsApp", 
        "deploymentGroupName": "threadsDG", 
        "deploymentConfigName": "CodeDeployDefault.ECSAllAtOnce", 
        "serviceRoleArn": "arn:aws:iam::<b>012345678912</b>:role/ecsCodeDeployServiceRole",
        "ecsServices": [
            {
                "serviceName": "threads", 
                "clusterName": "my_first_ecs_cluster"
            }
        ],
        "alarmConfiguration": {
            "enabled": false, 
            "ignorePollAlarmFailure": true, 
            "alarms": []
        }, 
        "autoRollbackConfiguration": {
            "enabled": true, 
            "events": [
                "DEPLOYMENT_FAILURE",
                "DEPLOYMENT_STOP_ON_REQUEST",
                "DEPLOYMENT_STOP_ON_ALARM"
            ]
        }, 
        "deploymentStyle": {
            "deploymentType": "BLUE_GREEN", 
            "deploymentOption": "WITH_TRAFFIC_CONTROL"
        }, 
        "blueGreenDeploymentConfiguration": {
            "terminateBlueInstancesOnDeploymentSuccess": {
                "action": "TERMINATE", 
                "terminationWaitTimeInMinutes": 5
            }, 
            "deploymentReadyOption": {
                "actionOnTimeout": "CONTINUE_DEPLOYMENT", 
                "waitTimeInMinutes": 0
            }
        }, 
        "loadBalancerInfo": {
            "targetGroupPairInfoList": [
                {
                    "targetGroups": [
                        {
                            "name": "<b>threads-tg</b>"
                        },
                        {
                            "name": "<b>threads-tg-2</b>"
                        }
                    ], 
                    "prodTrafficRoute": {
                        "listenerArns": [
                            "<b>arn:aws:elasticloadbalancing:us-east-1:012345678912:listener/app/alb-container-labs/86a05a2486126aa0/0e0cffc93cec3218</b>"
                        ]
                    }, 
                    "testTrafficRoute": {
                        "listenerArns": [
                            "<b>arn:aws:elasticloadbalancing:us-east-1:012345678912:listener/app/alb-container-labs/86a05a2486126aa0/d3336ca308561265</b>"
                        ]
                    }
                }
            ]
        }
    }
    </pre>


    <pre>
    aws deploy create-deployment-group \
    --region us-east-1 \
    --cli-input-json file://deployment-group-threads.json
    </pre>

* Creating a lifecycle hook for testing the new release. As discussed in the theory session, these are very helpful. The content in the 'hooks' section of the AppSpec file varies, depending on the compute platform for your deployment. The 'hooks' section for an EC2/On-Premises deployment contains mappings that link deployment lifecycle event hooks to one or more scripts. The 'hooks' section for an Amazon ECS deployment specifies Lambda validation functions to run during a deployment lifecycle event. If an event hook is not present, no operation is executed for that event. This section is required only if you are running scripts or Lambda validation functions as part of the deployment.
  So there are two parts to creating a hook, first an IAM role needs to be created which is used by lambda to pass back the testing results to CodeDeploy. You can either create the role using the CLI with the following policies
  
  - Managed Policy - AWSLambdaBasicExecutionRole for CloudWatch logs
- And a new policy with the following permissions
  
    <pre>
        {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": [
                    "codedeploy:PutLifecycleEventHookExecutionStatus"
                ],
                "Resource": "arn:aws:codedeploy:us-east-1:<b>012345678912</b>:deploymentgroup:*",
                "Effect": "Allow"
            }
        ]
    }
    </pre>
  
* Change directory to lab-3/hooks, do **npm install** and zip the contents.

* Create a lambda function

    <pre>
    aws lambda  create-function \
    --region us-east-1
    --function-name CodeDeployHook_pre-traffic-hook \
    --zip-file fileb://file-path/<b>file</b>.zip \
    --role <b>role-arn</b> \
    --environment Variables={TargetUrl=http://alb-container-labs-1439628024.us-east-1.elb.amazonaws.com:8080/api/threads} \
    --handler pre-traffic-hook.handler \
    --runtime nodejs8.10
    </pre>

**Note:** Feel free to review the configuration of the lambda function, its a simple check to verify the API works, however you can add other validation checks as well.

* Create an [AppSpec](https://docs.aws.amazon.com/codedeploy/latest/userguide/application-specification-files.html) file. It is used to manage each deployment as a series of lifecycle event hooks, which are defined in the file. For ECS as the compute platform, it can be either YAML or JSON formatted. For this lab, we'll use JSON.

    <pre>
    {
        "version": 0.0,
        "Resources": [
        {
        "TargetService": {
            "Type": "AWS::ECS::Service",
            "Properties": {
            "TaskDefinition": "threads-task-def:1",
            "LoadBalancerInfo": {
                "ContainerName": "threads-cntr",
                "ContainerPort": 3000
            },
            "PlatformVersion": "LATEST",
            "NetworkConfiguration": {
                "awsvpcConfiguration": {
                "subnets": [
                    "<b>subnet-06437a4061211691a</b>","<b>subnet-0437c573c37bbd689</b>"
                ],
                "securityGroups": [
                    "<b>sg-0f01c67f9a810f62a</b>"
                ],
                "assignPublicIp": "DISABLED"
                }
            }
            }
        }
        }
    ],
    "Hooks": [
        {
        "BeforeAllowTraffic": "CodeDeployHook_pre-traffic-hook"
        }
    ]
    }
    </pre>

* Start the deployment using the new aws ecs deloy CLI commands

    <pre>
    aws ecs deploy \
    --region us-east-1 \
    --cluster my\_first\_ecs\_cluster \
    --service threads \
    --task-definition fargate-task-def-threads.json \
    --codedeploy-appspec appspec.json \
    --codedeploy-application threadsApp \
    --codedeploy-deployment-group threadsDG
    </pre>

**Note:** The above will trigger a CodeDeploy Deployment, you can view the status of this deployment, at the end of it, it should look like ![CodeDeploy Status](../images/00-codedeploy-status.png)

If you now make some changes to the docker container threads, for example add another thread to db.json file and build a new container with a tag 0.1 + push to ECR. You can then create a revision of task definition and specifying the new image. During this process, selecting the existing CodeDeploy Application & Deployment-Group. This should trigger a new deployment in CodeDeploy and you can monitor the status in a similar way. Also feel free to check lambda logs in CloudWatch and add some of your own tests to this lambda.


### Checkpoint:
Congratulations, you've successfully deployed a service with blue/green deployments from CodeDeploy and with 0 downtime. If you have time, convert the other services to blue/green as well.  Otherwise, please remember to follow the steps below in the **Lab Cleanup** to make sure all assets created during the workshop are removed so you do not see unexpected charges after today.