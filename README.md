# reinvent2019-con414
Repository with sample code and cloudformation templates for reinvent2019 con414 session

## Using resources in this repository
### Prerequisites
The following set of tools/software are required to run the commands listed in the next section:
1. git
2. Docker
3. jq 
4. This repository has been cloned/downloaded

### Setting up the infrastructure
1. Create resources required to run Amazon Elastic Container Serviecs(ECS) tasks in AWS Fargate using the [cluster-fargate-public-vpc](cloudformation/cluster-fargate-public-vpc.yml) cloudformation template: `aws cloudformation create-stack --stack-name con414-cluster-fargate-public-vpc --template-body file://./cloudformation/cluster-fargate-public-vpc.yml --capabilities CAPABILITY_IAM`
2. Wait for stack creation to complete: `aws cloudformation wait stack-create-complete --stack-name con414-cluster-fargate-public-vpc`
3. Login to Elastic Container Registry (ECR) repository that was created in the previous step: `$(aws ecr get-login --no-include-email --region us-west-2)`
4. Get the ECR repository URI: `ECR_REPO=$(aws cloudformation describe-stacks --stack-name con414-cluster-fargate-public-vpc | jq '.Stacks[0].Outputs | map(select(.OutputKey=="ECRContainerImageRepository"))' | jq -r '.[].OutputValue')`
5. Build the `httpd-login` container using the [dockerfile](dockerfiles/httpd-login/Dockerfile): `cd dockerfiles/httpd-login; docker build -t ${ECR_REPO}:v1 .; cd -`
6. Push the container image to the ECR repository: `docker push ${ECR_REPO}:v1`
