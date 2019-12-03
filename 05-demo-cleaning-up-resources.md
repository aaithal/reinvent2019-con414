## Cleaning up
* Stop all of your running tasks:
```
## Get cluster arn
CLUSTER=$(aws --region us-west-2 cloudformation describe-stacks --stack-name con414-cluster-fargate-public-vpc | jq '.Stacks[0].Outputs | map(select(.OutputKey=="ClusterName"))' | jq -r '.[].OutputValue')
## Stop all tasks
for t in $(aws --region us-west-2 ecs list-tasks --cluster ${CLUSTER} | jq -r '.taskArns[]'); do aws --region us-west-2 ecs stop-task --cluster ${CLUSTER} --task $t > /dev/null; done
```
* Delete all images in the ECR repository:
```
## Get the ECR repo
ECR_REPO_NAME=$(aws --region us-west-2 cloudformation describe-stacks --stack-name con414-cluster-fargate-public-vpc | jq '.Stacks[0].Outputs | map(select(.OutputKey=="ECRContainerImageRepository"))' | jq -r '.[].OutputValue' | cut -f2 -d/)
## Delete all images
for digest in $(aws --region us-west-2 ecr list-images --repository-name ${ECR_REPO_NAME} | jq -r '.imageIds[].imageDigest'); do aws --region us-west-2 ecr batch-delete-image  --repository-name ${ECR_REPO_NAME} --image-ids imageDigest=$digest; done
```
* Delete the cloudformation stack
```
## Delete stack
aws --region us-west-2 cloudformation delete-stack --stack-name con414-cluster-fargate-public-vpc
## Wait for deletion
aws --region us-west-2 cloudformation wait stack-delete-complete --stack-name con414-cluster-fargate-public-vpc
```
