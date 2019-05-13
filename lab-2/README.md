## From Monolith To Microservices

In this lab we take our monolithic application deployed on ECS and split it up into microservices.

![Reference architecture of microservices on EC2 Container Service](/images/microservice-containers.png)

## Why Microservices?

__Isolation of crashes:__ Even the best engineering organizations can and do have fatal crashes in production. In addition to following all the standard best practices for handling crashes gracefully, one approach that can limit the impact of such crashes is building microservices. Good microservice architecture means that if one micro piece of your service is crashing then only that part of your service will go down. The rest of your service can continue to work properly.

__Isolation for security:__ In a monolithic application if one feature of the application has a security breach, for example a vulnerability that allows remote code execution then you must assume that an attacker could have gained access to every other feature of the system as well. This can be dangerous if for example your avatar upload feature has a security issue which ends up compromising your database with user passwords. Separating out your features into micorservices using EC2 Container Service allows you to lock down access to AWS resources by giving each service its own IAM role. When microservice best practices are followed the result is that if an attacker compromises one service they only gain access to the resources of that service, and can't horizontally access other resources from other services without breaking into those services as well.

__Independent scaling:__ When features are broken out into microservices then the amount of infrastructure and number of instances of each microservice class can be scaled up and down independently. This makes it easier to measure the infrastructure cost of particular feature, identify features that may need to be optimized first, as well as keep performance reliable for other features if one particular feature is going out of control on its resource needs.

__Development velocity__: Microservices can enable a team to build faster by lowering the risk of development. In a monolith adding a new feature can potentially impact every other feature that the monolith contains. Developers must carefully consider the impact of any code they add, and ensure that they don't break anything. On the other hand a proper microservice architecture has new code for a new feature going into a new service. Developers can be confident that any code they write will actually not be able to impact the existing code at all unless they explictly write a connection between two microservices.

## Application Changes for Microsevices

__Define microservice boundaries:__ Defining the boundaries for services is specific to your application's design, but for this REST API one fairly clear approach to breaking it up is to make one service for each of the top level classes of objects that the API serves:

```
/api/users/* -> A service for all user related REST paths
/api/posts/* -> A service for all post related REST paths
/api/threads/* -> A service for all thread related REST paths
```

So each service will only serve one particular class of REST object, and nothing else. This will give us some significant advantages in our ability to independently monitor and independently scale each service.

__Stitching microservices together:__ Once we have created three separate microservices we need a way to stitch these separate services back together into one API that we can expose to clients. This is where Amazon Application Load Balancer (ALB) comes in. We can create rules on the ALB that direct requests that match a specific path to a specific service. The ALB looks like one API to clients and they don't need to even know that there are multiple microservices working together behind the scenes.

__Chipping away slowly:__ It is not always possible to fully break apart a monolithic service in one go as it is with this simple example. If our monolith was too complicated to break apart all at once we can still use ALB to redirect just a subset of the traffic from the monolithic service out to a microservice. The rest of the traffic would continue on to the monolith exactly as it did before.

Once we have verified this new microservice works we can remove the old code paths that are no longer being executed in the monolith. Whenever ready repeat the process by splitting another small portion of the code out into a new service. In this way even very complicated monoliths can be gradually broken apart in a safe manner that will not risk existing features.

## Instructions:

1. Create ECR Repositories for all microservices

   <pre>
   aws ecr create-repository \
	--region us-east-1 \
	--repository-name "<b>SERVICE_NAME</b>" \
	--query "repository.repositoryUri" \
	--output text
	</pre>
	
	**Note: Make note of the repositoryUri**
	
2. Build the container, and assign a tag to it for versioning

   <pre>
   docker build -t <b>SERVICE_NAME</b> ./services/<b>SERVICE_NAME</b>
	docker tag <b>SERVICE_NAME</b>:latest <b>repositoryUri</b>:latest
	</pre>
	
3. Push the tag up so we can make a task definition for deploying it

   <pre>
   docker push <b>repositoryUri</b>:latest
   </pre>

4. Create json input files for registering task definitions for each service. File name - fargate-task-def-**SERVICE_NAME**.json

   <pre>
      {
      "requiresCompatibilities": [
           "FARGATE"
      ],
      "containerDefinitions": [
           {
               "name": "<b>SERVICE_NAME</b>-cntr",
               "image": "<b>012345678912</b>.dkr.ecr.us-east-1.amazonaws.com/users:latest",
               "memoryReservation": 128,
               "essential": true,
               "portMappings": [
                   {
                       "containerPort": 3000,
                        "protocol": "tcp"
                  }
               ],
               "logConfiguration": {
                  "logDriver": "awslogs",
                  "options": {
                     "awslogs-group": "/ecs/<b>SERVICE_NAME</b>",
                     "awslogs-region": "us-east-1",
                     "awslogs-stream-prefix": "ecs"
                  }
               }
         }
      ],
      "volumes": [],
      "networkMode": "awsvpc",
      "memory": "512",
      "cpu": "256",
      "executionRoleArn": "arn:aws:iam::<b>012345678912</b>:role/ecsTaskExecutionRole",
      "taskRoleArn": "arn:aws:iam::<b>012345678912</b>:role/ECSTaskRole",
      "family": "<b>SERVICE_NAME</b>-task-def"
   }
   </pre>

5. Create CloudWatch log groups for each service

   <pre>
   aws logs create-log-group --log-group-name "/ecs/<b>SERVICE_NAME</b>" --region us-east-1
   </pre>

6. Register task definitions for each service

   <pre>
   aws ecs register-task-definition --cli-input-json file://fargate-task-def-<b>SERVICE_NAME</b>.json
   </pre>
   
7. Create new Target Groups for each service

   <pre>
   aws elbv2 create-target-group \
	--region us-east-1 \
	--name <b>SERVICE_NAME</b>-tg \
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

8. Get the listener for the existing load balancer

   <pre>
   aws elbv2 describe-listeners \
    --region us-east-1 \
    --query "Listeners[0].ListenerArn" \
    --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:<b>012345678912</b>:loadbalancer/app/alb-container-labs/86a05a2486126aa0/0e0cffc93cec3218 \
    --output text
   </pre>

9. Now lets add rules to the existing listener, this way the existing monolith can continue to serve requests i.e. 0 downtime.

   <pre>
   aws elbv2 create-rule \
   --region us-east-1 \
   --listener-arn arn:aws:elasticloadbalancing:us-east-1:776055576349:listener/app/alb-container-labs/86a05a2486126aa0/0e0cffc93cec3218 \
   --priority 1 \
   --conditions Field=path-pattern,Values='/api/<b>SERVICE_NAME</b>*' \
   --actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:<b>012345678912</b>:targetgroup/<b>SERVICE_NAME</b>-tg/73e2d6bc24d8a067
   </pre>

10. Create a new service using the json input file for each service. File name - ecs-service-**SERVICE_NAME**.json

   <pre>
      {
      "cluster": "my_first_ecs_cluster", 
      "serviceName": "<b>SERVICE_NAME</b>", 
      "taskDefinition": "<b>SERVICE_NAME</b>-task-def:1", 
      "loadBalancers": [
         {
               "targetGroupArn": "arn:aws:elasticloadbalancing:us-east-1:<b>012345678912</b>:targetgroup/<b>SERVICE_NAME</b>-tg/566b90ffcc10985e", 
               "containerName": "<b>SERVICE_NAME</b>-cntr", 
               "containerPort": 3000
         }
      ], 
      "desiredCount": 1, 
      "clientToken": "", 
      "launchType": "FARGATE", 
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
         "type": "ECS"
      }
   }
   </pre>

11. Create the service

   <pre>
   aws ecs create-service --cli-input-json file://ecs-service-<b>SERVICE_NAME</b>.json
   </pre>

12. Drain the existing service by changing the desired count to 0.
13. Test the load balancer endpoint, the users, posts & threads should be different.
