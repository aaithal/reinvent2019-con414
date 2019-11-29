## Update the infrastructure
1. Update the resources to harden the security posture using the [cluster-fargate-vpc-asm](cloudformation/01-cluster-fargate-vpc-asm.yml] cloudformation template. `aws --region us-west-2 cloudformation update-stack --stack-name con414-cluster-fargate-public-vpc --template-body file://./cloudformation/01-cluster-fargate-vpc-asm.yml --capabilities CAPABILITY_IAM`
2. Wait for the stack update to complete: `aws cloudformation wait stack-update-complete --stack-name con414-cluster-fargate-public-vpc`  
3. Login to Elastic Container Registry (ECR) repository that was created in the previous step: `$(aws ecr get-login --no-include-email --region us-west-2)`
4. Get the ECR repository URI: `ECR_REPO=$(aws cloudformation describe-stacks --stack-name con414-cluster-fargate-public-vpc | jq '.Stacks[0].Outputs | map(select(.OutputKey=="ECRContainerImageRepository"))' | jq -r '.[].OutputValue')`
5. Build the `httpd-login` container using the [dockerfile](dockerfiles/httpd-login/01-Dockerfile): `cd dockerfiles/httpd-login; docker build -t ${ECR_REPO}:v2 -f 01-Dockerfile .; cd -`
6. Push the container image to the ECR repository: `docker push ${ECR_REPO}:v2`

