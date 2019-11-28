# reinvent2019-con414
Repository with sample code and cloudformation templates for reinvent2019 con414 session

## Using resources in this repository
### Prerequisites
The following set of tools/software are required to run the commands listed in the next section:
1. git
2. Docker
3. jq
4. aws cli (with output format set to JSON)
5. ssh client 
6. This repository has been cloned/downloaded

### 1. Setting up the infrastructure
1. Create resources required to run Amazon Elastic Container Serviecs(ECS) tasks in AWS Fargate using the [cluster-fargate-public-vpc](cloudformation/cluster-fargate-public-vpc.yml) cloudformation template: `aws cloudformation create-stack --stack-name con414-cluster-fargate-public-vpc --template-body file://./cloudformation/cluster-fargate-public-vpc.yml --capabilities CAPABILITY_IAM`
2. Wait for stack creation to complete: `aws cloudformation wait stack-create-complete --stack-name con414-cluster-fargate-public-vpc`
3. Login to Elastic Container Registry (ECR) repository that was created in the previous step: `$(aws ecr get-login --no-include-email --region us-west-2)`
4. Get the ECR repository URI: `ECR_REPO=$(aws cloudformation describe-stacks --stack-name con414-cluster-fargate-public-vpc | jq '.Stacks[0].Outputs | map(select(.OutputKey=="ECRContainerImageRepository"))' | jq -r '.[].OutputValue')`
5. Build the `httpd-login` container using the [dockerfile](dockerfiles/httpd-login/Dockerfile): `cd dockerfiles/httpd-login; docker build -t ${ECR_REPO}:v1 .; cd -`
6. Push the container image to the ECR repository: `docker push ${ECR_REPO}:v1`

### 2. Running a Fargate task
#### 2.1 Register the template task definition
```
ECR_IMAGE=$(aws cloudformation describe-stacks --stack-name con414-cluster-fargate-public-vpc | jq '.Stacks[0].Outputs | map(select(.OutputKey=="ECRContainerImageRepository"))' | jq -r '.[].OutputValue')
ECR_IMAGE="${ECR_IMAGE}:v1"
taskdef=$(cat runtask/taskdefinition.json | awk -v img="${ECR_IMAGE}" '{gsub("ECRIMAGE", img, $0); print}')
TASK_DEFINITION=$(aws ecs register-task-definition --cli-input-json "${taskdef}" --query "taskDefinition.taskDefinitionArn" --output text)
```
#### 2.2 Prepare run-task parameters
```
CLUSTER=$(aws cloudformation describe-stacks --stack-name con414-cluster-fargate-public-vpc | jq '.Stacks[0].Outputs | map(select(.OutputKey=="ClusterName"))' | jq -r '.[].OutputValue')
ECS_EXECUTION_ROLE=$(aws cloudformation describe-stacks --stack-name con414-cluster-fargate-public-vpc | jq '.Stacks[0].Outputs | map(select(.OutputKey=="ECSTaskExecutionRole"))' | jq -r '.[].OutputValue')
SECURITY_GROUP=$(aws cloudformation describe-stacks --stack-name con414-cluster-fargate-public-vpc | jq '.Stacks[0].Outputs | map(select(.OutputKey=="ContainerSecurityGroup"))' | jq -r '.[].OutputValue')
SUBNET=$(aws cloudformation describe-stacks --stack-name con414-cluster-fargate-public-vpc | jq '.Stacks[0].Outputs | map(select(.OutputKey=="PublicSubnetOne"))' | jq -r '.[].OutputValue')
```
#### 2.3 Run the task
```
TASK_ARN=$(aws ecs run-task --cluster ${CLUSTER} \
	--task-definition ${TASK_DEFINITION} \
	--launch-type FARGATE \
	--overrides "{\"executionRoleArn\": \"${ECS_EXECUTION_ROLE}\"}" \
	--network-configuration "awsvpcConfiguration={subnets=[$SUBNET],securityGroups=[$SECURITY_GROUP],assignPublicIp=ENABLED}" \
	--query "tasks[0].taskArn" \
	--output text)
```
#### 2.4 Check if task is RUNNING
```
aws ecs describe-tasks --cluster ${CLUSTER} --task ${TASK_ARN} \
	--query "tasks[0].lastStatus" \
	--output text
```
#### 2.5 Get the task public IPv4 address
```
TASK_ENI=$(aws ecs describe-tasks --cluster ${CLUSTER} --task ${TASK_ARN} | jq '.tasks[0].attachments[0].details | map(select(.name=="networkInterfaceId"))' | jq -r '.[].value')
TASK_IP=$(aws ec2 describe-network-interfaces --network-interface-ids ${TASK_ENI} --query "NetworkInterfaces[0].PrivateIpAddresses[0].Association.PublicIp" --output text)
```

#### 2.6 Access the task IPv4 address
1. Navigate to the task IPv4 address using a browser. You'll be asked for a username and a password
2. Enter 'fargate' as the username and 'swordfish' as the password. You should be able to see the nginx landing page
3. Try running the command `ssh ${TASK_IP}`. You should see a password prompt. We haven't set any ssh keys for the task. So, you will not be able to login
4. Press CTRL-C to cancel

#### 2.7 Fix security group rules for the task
```
### Drop the allow all rule
aws ec2 revoke-security-group-ingress --group-id ${SECURITY_GROUP} --cidr 0.0.0.0/0 --protocol all
### Add the rule to all HTTP access to the task
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP} --protocol tcp --port 80 --cidr 0.0.0.0/0
```
Repeat 2.6. You should still be able to access the IPv4 address via port 80 (on your browser). But, you will not be able to connect to the task using SSH since we have blocked it.

#### 2.8 Stop the task
```
aws ecs stop-task --cluster ${CLUSTER} --task ${TASK_ARN}
```
