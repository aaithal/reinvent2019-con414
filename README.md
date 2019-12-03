# reinvent2019-con414
This repository contains the sample code and artifacts used in the CON414 builder session at AWS re:invent 2019.

## Learning objectives
1. Building container images and storing them securely in an Elastic Container Registry(ECR) repository
2. Creating resources required to run a container securely in AWS Fargate
3. Securing access to containers running in AWS Fargate using security groups
4. Using AWS Secrets Manager(ASM) to create store secrets
5. Using ASM and Identity and Access Management (IAM) roles to securely bootstrap containers with secrets

## Demo
### Prerequisites
The following set of tools/software are required to run the commands listed in the next section:
1. git
2. Docker
3. jq
4. aws cli (with output format set to JSON)
5. ssh client 
6. This repository has been cloned/downloaded on the target machine

### 1. Setting up the infrastructure
The goal of this section is to create all of the resources needed to run a contianer in AWS Fargate using an Amazon Elastic Container Service (ECS) task. You will use the [cluster-fargate-vpc](cloudformation/00-cluster-fargate-vpc.yml) cloudformation template to create the following resources:
1. An Amazon Virtual Private Cloud (VPC) with public subnets and a security group
2. An ECS cluster
3. An IAM role used by ECS when starting the container (also referred to as [Task Execution Role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html))
4. An ECR repository

Once these are created, you will build and push the `httpd-login` container image to the ECR repository that was created using the cloudformation template.

The container image runs an `nginx` proxy fronting an apache webserver using `httpd`. It also starts an `ssh` daemon in the background. Subsequent sections in the demo show how we can secure these components. 
   
The instructions for setting up the infrastructure can be found at [here](00-demo-setting-up-infra.md).

At this stage, you have successfully pushed your code into a secure container repository that can only be accessed if you have the relevant AWS permissions. This makes sure that malicious actors cannot override your build artifacts as ECR requires valid AWS IAM permissions to push container images. You can selectively grant permissions to this repository using [these instructions](https://docs.aws.amazon.com/AmazonECR/latest/userguide/RepositoryPolicyExamples.html).
   
### 2. Running a Fargate task
The goal of this section is to run a container in Fargate as an ECS Task. In order to do this, you will need to:
1. Register the container in ECS using an ECS task definition
2. Run the container using the task definition and the `run-task` ECS api

The instructions for setting up the infrastructure can be found [here](01-demo-running-task-stage-1.md).

At this stage, you are running your container in AWS Fargate in one of the VPC subnets, and are securing the access to this task using a security group as well. Fargate enforces a number of security best practices by default and you have utilized many of these:
1. By running your container in a VPC, using the [`awsvpc`](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-networking.html) networking mode, you have ensured that each copy of your application gets its own distinct network stack (distinct IPv4 address, distincy Elastic Networking Interface(ENI) etc). You can further set up anamoly detection, alarming etc on suspcious network access patterns to your containers using VPC flow logs. Here's an example of analyzing [VPC flow logs using AWS Lambda](https://aws.amazon.com/blogs/mt/analyzing-vpc-flow-logs-got-easier-with-support-for-s3-as-a-destination/).
2. By using Task Execution Role permissions, you have ensured that your container image was downloaded after an authentication & authorization scheme enforced by ECR. You have also ensured that your container logs will flow securely to Amazon Cloudwatch Logs using IAM role credentials over a secure TLS connection.
3. If your webserver gets incredibly popular and you have to scale out fast without causing an availability event, you can flexibly scale out your container as an [ECS service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_services.html). Some exmaples of the same can be found in this [github repository](https://github.com/nathanpeck/ecs-cloudformation) as well.

However, your container still has some security vulnerabilities: 
* The container security group allows access to your task on ports `80`(HTTP) and `22`(SSH) for any IP address on the internet
* Allowing any IP address on the internet SSH access to your container is a critical security risk as malicious actors could exploit it to
 	* Break into your container
 	* Cause availability events by flooding requests using multiple SSH clients (denial of service attack)

You might still want to retain the SSH daemon for when you need to debug your container. But you can block access to the same for everyone on the internet and instead only allow trusted IP addresses or private IP addresses to connect over SSH to your containers.

### 3. Securing access to the Fargate task   
The goal of this section is to restrict the access to the container using security groups. 

The instructions for the same can be found [here](02-demo-security-group-restrictions.md).

At this stage, you are running your container more securely in AWS Fargate. However, your container is still vulnerable to malicious actors. The login password for your webserver has been hardcoded into your container image. If you check the [Dockerfile](dockerfiles/httpd-login/00-Dockerfile) to a source code repository, you will leak/expose the password to your webserver to any one who has read access to the repository. We can remediate this threat by using the Amazon Secrets Manager to store this secret and using the [ECS integration](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/specifying-sensitive-data.html) to securely bootstrap our container with secrets.

Before taking a look at the instructions for doing this, you can optionally stop this task: `aws --region us-west-2 ecs stop-task --cluster ${CLUSTER} --task ${TASK_ARN}`

### 4. Hardening the security posture of the Fargate task using ASM
The goal of this section is to not hard code the application to your container image during build time, but to use ASM for the same. 

The instructions for the same can be found [here](03-demo-updating-infra-with-asm.md).

At this stage:
1. You have rebuilt and pushed the container image so that the secret bootstrap happens during runtime, rather than at build time
2. You have also created an ASM secret and updated the task execution role with permissions to access this secret
3. Your container security group is also updated to only allow HTTP (port 80) traffic and block everything else

### 5. Running an AWS Fargate task with ASM
The goal of this section is to use the ASM integration with AWS Fargate, where the secret is passed as an environment variable to the container at run time using the ECS task definition.

The instructions for the same can be found [here](04-demo-running-task-stage-2.md).

At this stage you have achieved all of the security objects we set out to achieve with this demo. You can stop the task using the command `aws ecs stop-task --cluster ${CLUSTER} --task ${TASK_ARN}`

### 6. Cleaning up
You can follow the instructions in [here](05-demo-cleaning-up-resources.md) to clean up resources that were used for this demo.

### 7. Further exercise
1. Set up AWS Cloudtrail alarms for your ECS resources [hint](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudwatch-alarms-for-cloudtrail.html)
2. Set up VPC Flow log monitors for your VPC [hint](https://aws.amazon.com/blogs/mt/analyzing-vpc-flow-logs-got-easier-with-support-for-s3-as-a-destination/)
3. Use KMS to encrypt your secret with ASM [hint](https://docs.aws.amazon.com/kms/latest/developerguide/services-secrets-manager.html) 
4. Get this container running behind a load balancer as an ECS service [hint](https://github.com/nathanpeck/ecs-cloudformation)
