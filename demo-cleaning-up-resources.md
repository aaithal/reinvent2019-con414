## Cleaning up
* Stop all of your running tasks:
```
## Get cluster arn
CLUSTER=$(aws cloudformation describe-stacks --stack-name con414-cluster-fargate-public-vpc | jq '.Stacks[0].Outputs | map(select(.OutputKey=="ClusterName"))' | jq -r '.[].OutputValue')
## Stop all tasks
for t in $(aws ecs list-tasks --cluster ${CLUSTER} | jq -r '.taskArns[]'); do aws ecs stop-task --cluster ${CLUSTER} --task $t > /dev/null; done
```
* Delete all images in the ECR repository:
```
## Get the ECR repo
ECR_REPO_NAME=$(aws cloudformation describe-stacks --stack-name con414-cluster-fargate-public-vpc | jq '.Stacks[0].Outputs | map(select(.OutputKey=="ECRContainerImageRepository"))' | jq -r '.[].OutputValue' | cut -f2 -d/)
## Delete all images
for digest in $(aws ecr list-images --repository-name ${ECR_REPO_NAME} | jq -r '.imageIds[].imageDigest'); do aws ecr batch-delete-image  --repository-name ${ECR_REPO_NAME} --image-ids imageDigest=$digest; done
```
* Delete the cloudformation stack
```
## Delete stack
aws cloudformation delete-stack --stack-name con414-cluster-fargate-public-vpc
## Wait for deletion
aws cloudformation wait stack-delete-complete --stack-name con414-cluster-fargate-public-vpc
```
