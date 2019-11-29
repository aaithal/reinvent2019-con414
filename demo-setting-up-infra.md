## Setting up the infrastructure
1. Create resources required to run ECS tasks in AWS Fargate using the [cluster-fargate-vpc](cloudformation/00-cluster-fargate-vpc.yml) cloudformation template: `aws cloudformation create-stack --stack-name con414-cluster-fargate-public-vpc --template-body file://./cloudformation/00-cluster-fargate-vpc.yml --capabilities CAPABILITY_IAM`
2. Wait for stack creation to complete: `aws cloudformation wait stack-create-complete --stack-name con414-cluster-fargate-public-vpc`
3. Login to the ECR repository that was created in the previous step: `$(aws ecr get-login --no-include-email --region us-west-2)`
4. Get the ECR repository URI: `ECR_REPO=$(aws cloudformation describe-stacks --stack-name con414-cluster-fargate-public-vpc | jq '.Stacks[0].Outputs | map(select(.OutputKey=="ECRContainerImageRepository"))' | jq -r '.[].OutputValue')`
5. Build the `httpd-login` container using the [dockerfile](dockerfiles/httpd-login/00-Dockerfile): `cd dockerfiles/httpd-login; docker build -t ${ECR_REPO}:v1 -f 00-Dockerfile .; cd -`
6. Push the container image to the ECR repository using the `v1` tag : `docker push ${ECR_REPO}:v1`

