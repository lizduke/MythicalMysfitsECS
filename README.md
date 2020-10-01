# MythicalMysfitsEKS

First create the core infrastructure by using the core.yaml file as the cloudformation template.
Add the name of the loadbalancer you will create (so a DNS name you can CNAME or Alias to the one created by the ingress) in upload-site.
Then run the setup script under scripts "script/setup" to populate the DynamoDB table and the S3 bucket.

Create a repo using the cli.

aws --region eu-west-1 ecr create-repository --repository-name mythicaleks --image-scanning-configuration scanOnPush=true


Clone Olly's repo:

git clone https://github.com/ollypom/mysfits

Build the Docker image from the pulled repo
docker build -t mysfitsapi .

Login to ECR repo.

aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin <accountnumber>.dkr.ecr.eu-west-1.amazonaws.com

Tag the image and push to ECR.

docker tag mysfitsapi:latest <accountnumber>.dkr.ecr.eu-west-1.amazonaws.com/mythicaleks:1.0

docker push <accountnumber>.dkr.ecr.eu-west-1.amazonaws.com/mythicaleks:1.0

Populate the MythicalCluster.yaml with your region, vpc-id and subnet ids that were creted by the core.yaml file.

Build the cluster

eksctl create cluster -f MythicalCluster.yaml

ALB Ingress Controller :



Add tags to Subnets for EKS to enable Loadbalancer autodiscovery.
Subnets must contain these tags: 'kubernetes.io/cluster/MythicalEKS': ['shared' or 'owned'] and 'kubernetes.io/role/elb': ['' or '1'].

Create an oidc provider for ALB to use:

eksctl utils associate-iam-oidc-provider \
--region eu-west-1 \
➜ --cluster MythicalEKS \
➜ --approve

Download an IAM policy to use with ALB ingress controller

curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.8/docs/examples/iam-policy.json

Create an IAM policy using the downloaded file.

aws iam create-policy \
--policy-name alb-ingress-controller \
--policy-document file://iam-policy.json

Need policy-arn from previous
Create ALB ingress controller service account using

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.8/docs/examples/rbac-role.yaml


Create the IAM role using IRSA for the ALB ingress controller.

eksctl create iamserviceaccount \
--region eu-west-1 \
--name alb-ingress-controller \
--namespace kube-system \
--cluster MythicalEKS \
--attach-policy-arn arn:aws:iam::<accountnumber>:policy/alb-ingress-controller \
--override-existing-serviceaccounts \
--approve

Populate your ALB ingress controller with your vpc-id.

kubectl apply -f alb-ingress-controller.yaml

Now create the Namespace for the api server.

kubectl apply -f namespace.yaml

Create the DynamoDB IRSA role so the pods can access the table.


eksctl create iamserviceaccount \
--region eu-west-1 \
--name ddb-pod-read \
--namespace mythical-api \
--cluster MythicalEKS \
--attach-policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBReadOnlyAccess \
--override-existing-serviceaccounts \
--approve

Then you need to annotate deployment with that SA. This is included in deployment.yaml.

Create deployment - this pulls image from ECR

kubectl apply -f deployment.yaml

Create service - type NodePort was for ALB ingress.

kubectl apply -f service.yaml

Create ingress - this sits in the namespace.

kubectl apply -f ingress.yaml

Check LB is created and instances are passing health checks

Match the DNS name of the LB to the name you passed when populating the S3 bucket files.
This is in upload-site file.

The website on S3 should now allow you to select types of Mythical Mysfits.
