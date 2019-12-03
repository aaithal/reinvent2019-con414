# Running an AWS Fargate task with ASM
## 1 Register the template task definition
```
ECR_IMAGE=$(aws --region us-west-2 cloudformation describe-stacks --stack-name con414-cluster-fargate-public-vpc | jq '.Stacks[0].Outputs | map(select(.OutputKey=="ECRContainerImageRepository"))' | jq -r '.[].OutputValue')
ECR_IMAGE="${ECR_IMAGE}:v2"
taskdef=$(cat runtask/01-taskdefinition.json | awk -v img="${ECR_IMAGE}" '{gsub("ECRIMAGE", img, $0); print}')
ASM_SECRET=$(aws --region us-west-2 cloudformation describe-stacks --stack-name con414-cluster-fargate-public-vpc | jq '.Stacks[0].Outputs | map(select(.OutputKey=="ApplicationSecret"))' | jq -r '.[].OutputValue')
taskdef=$(echo $taskdef | awk -v secret="${ASM_SECRET}" '{gsub("SECRET", secret, $0); print}')
TASK_DEFINITION=$(aws --region us-west-2 ecs register-task-definition --cli-input-json "${taskdef}" --query "taskDefinition.taskDefinitionArn" --output text)
```
## 2 Prepare run-task parameters
```
CLUSTER=$(aws --region us-west-2 cloudformation describe-stacks --stack-name con414-cluster-fargate-public-vpc | jq '.Stacks[0].Outputs | map(select(.OutputKey=="ClusterName"))' | jq -r '.[].OutputValue')
ECS_EXECUTION_ROLE=$(aws --region us-west-2 cloudformation describe-stacks --stack-name con414-cluster-fargate-public-vpc | jq '.Stacks[0].Outputs | map(select(.OutputKey=="ECSTaskExecutionRole"))' | jq -r '.[].OutputValue')
SECURITY_GROUP=$(aws --region us-west-2 cloudformation describe-stacks --stack-name con414-cluster-fargate-public-vpc | jq '.Stacks[0].Outputs | map(select(.OutputKey=="ContainerSecurityGroup"))' | jq -r '.[].OutputValue')
SUBNET=$(aws --region us-west-2 cloudformation describe-stacks --stack-name con414-cluster-fargate-public-vpc | jq '.Stacks[0].Outputs | map(select(.OutputKey=="PublicSubnetOne"))' | jq -r '.[].OutputValue')
```
## 3 Run the task
```
TASK_ARN=$(aws --region us-west-2 ecs run-task --cluster ${CLUSTER} \
	--task-definition ${TASK_DEFINITION} \
	--launch-type FARGATE \
	--overrides "{\"executionRoleArn\": \"${ECS_EXECUTION_ROLE}\",\"containerOverrides\":[{\"name\": \"httpd-auth\",\"environment\":[{\"name\":\"HTPASSWD\",\"value\":\"${ASM_SECRET}\"}]}]}" \
	--network-configuration "awsvpcConfiguration={subnets=[$SUBNET],securityGroups=[$SECURITY_GROUP],assignPublicIp=ENABLED}" \
	--query "tasks[0].taskArn" \
	--output text)
```
## 4 Check if task is RUNNING
```
aws --region us-west-2 ecs describe-tasks --cluster ${CLUSTER} --task ${TASK_ARN} \
	--query "tasks[0].lastStatus" \
	--output text
```
## 5 Get the task public IPv4 address
```
## Get the task ENI id
TASK_ENI=$(aws --region us-west-2 ecs describe-tasks --cluster ${CLUSTER} --task ${TASK_ARN} | jq '.tasks[0].attachments[0].details | map(select(.name=="networkInterfaceId"))' | jq -r '.[].value')
## Print the task ip address
aws --region us-west-2 ec2 describe-network-interfaces --network-interface-ids ${TASK_ENI} --query "NetworkInterfaces[0].PrivateIpAddresses[0].Association.PublicIp" --output text
```
## 6 Access the task IPv4 address
1. Navigate to the task IPv4 address using a browser. You'll be asked for a username and a password
2. Enter 'fargate' as the username and 'swordfish' as the password. You should be able to see the nginx landing page
3. Try running the command `ssh ${TASK_IP}`. There should be no prompt of any sort since access to this port is blocked by security group rules.
