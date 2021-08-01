## S3 Cli

### Create bucket, add file
    aws s3api create-bucket --bucket my-033212455158-bucket --acl public-read-write --region us-east-1 --profile udacity
    aws s3api put-object --bucket my-033212455158-bucket --key sample.html --body sample.html --profile udacity

### Delete file, delete bucket
    aws s3api delete-object --bucket my-033212455158-bucket --key sample.html --profile udacity
    aws s3api delete-bucket --bucket my-033212455158-bucket --profile udacity

## Get caller identity
    
    aws sts get-caller-identity --profile udacity

`
{
    "UserId": "AIDAUOBHCOE2TP75G4RHQ",
    "Account": "305024626997",
    "Arn": "arn:aws:iam::305024626997:user/udacity"
}
`
# Roles

We create a role along with the trust file that sais which services can assume that role. Then we create a policy for the role and assign it.

### Trust file
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "AWS": "arn:aws:iam::305024626997:root"
                },
                "Action": "sts:AssumeRole"
            }
        ]
    }
### Create role command
    aws iam create-role --role-name UdacityFlaskDeployCBKubectlRole --assume-role-policy-document file://trust.json --output text --query 'Role.Arn' --profile udacity

### Policy file

    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "eks:Describe*",
                    "ssm:GetParameters"
                ],
                "Resource": "*"
            }
        ]
    }

##
    

### Put policy to the role
    aws iam put-role-policy --role-name UdacityFlaskDeployCBKubectlRole --policy-name eks-describe --policy-document file://iam-role-policy.json --profile udacity

# Cluster creation tool `eksctl`
You can go to th CloudFormation console to see the state of the creations in progress. It takes some time for the cluster to create. Time to grab a coffee.

    eksctl create cluster --name myCluster --nodes=4 --profile udacity

    eksctl create cluster --config-file=<path> --profile udacity

    eksctl utils describe-stacks --region=us-east-1 --cluster=eksctl-demo --profile udacity

    eksctl get cluster --name=eksctl-demo --region=us-east-2 --profile udacity

    eksctl delete cluster eksctl-demo  --profile udacity

# K8S control tool `kubectl`
Cheatsheet: https://kubernetes.io/docs/reference/kubectl/cheatsheet/

```
kubectl get nodes
```

## Kubernetes RBAC

When you create a new cluster, only you will have the sole permission to administer it. No other user/AWS service will be able to perform any action. Later, if you want the CodeBuild service to interact with your cluster, you will have to make an entry into the ConfigMap. Just make sure:
- The IAM role assumed by the CodeBuild must have the necessary permissions.
- Make an entry of that IAM role into the ConfigMap within Kubernetes.

Download the configmap:

    kubectl get -n kube-system configmap/aws-auth -o yaml > /tmp/aws-auth-patch.yml

Add this part:

    mapRoles: |
    - groups:
    - system:masters
    rolearn: arn:aws:iam::305024626997:role/UdacityFlaskDeployCBKubectlRole
    username: build

Patch the original file:
    
    kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"

Expected output is `configmap/aws-auth patched`

## Deployment

    kubectl apply -f deployment.yml
    # Verify the deployment
    kubectl get deployments
    # Check the rollout status
    kubectl rollout status deployment/simple-flask-deployment
    # Show the pods in the cluster
    kubectl get pods
    # Show the services in the cluster
    kubectl describe services
    # Display information about the cluster
    kubectl cluster-info

troubleshoot

    # List all namespaces, all pods
    kubectl get all -A
    # Show all events
    kubectl get events -w
    # Show component status
    kubectl get componentstatuses

clean up

    # Delete your deployment
    kubectl delete deployments/simple-flask-deployment
    # Tear down your cluster
    eksctl delete cluster eksctl-demo --profile udacity

# Docker hub

    docker build -t <YOUR DOCKER LOGIN>/simple-flask .
    docker image ls
    docker push  <DockerHub username>/simple-flask:latest
    # Or, if different tag for pushing:
    docker tag local-image:<tag-name> <Repo-name>:<tag-name>
    # If logged in in docker cli
    docker push <Repo-name>:<tag-name>

# Cloudformation

Create a stack
    
    aws cloudformation create-stack  --stack-name myFirstTest --region us-east-1 --template-body file://myFirstTemplate.yml

    aws cloudformation update-stack  --stack-name myFirstTest --region us-east-1 --template-body file://myFirstTemplate.yml

    aws cloudformation describe-stacks --stack-name myFirstTest

    aws cloudformation delete-stack --stack-name myFirstTest

myFirstTemplate.yml

    AWSTemplateFormatVersion: 2010-09-09
    Description: Udacity - This template deploys a VPC
    Resources:
    myUdacityVPC:
        Type: 'AWS::EC2::VPC'
        Properties:
        CidrBlock: 10.0.0.0/16
        EnableDnsHostnames: 'true'

