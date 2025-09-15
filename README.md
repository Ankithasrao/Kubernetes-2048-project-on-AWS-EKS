## 📌 Project Overview
- Deploys the 2048 game as a Kubernetes application.
- Uses AWS EKS as the Kubernetes cluster.
- Includes **Deployment**, **Service**, and **Ingress** configurations.

# Prerequisites
Before starting, install the following tools:
kubectl - A command line tool for working with Kubernetes clusters.
eksctl - A command line tool for working with EKS clusters that automates many individual tasks. 
AWS CLI - A command line tool for working with AWS services, including Amazon EKS.

# Configuring the AWS CLI and kubectl
  * Installing the AWS CLI : Download and install the AWS CLI on your local machine. You can find installation instructions for various operating systems [![here]

## ☁️ Step 1: Configure AWS CLI
```python
aws configure
```
Provide your AWS Access Key, Secret Key, region, and output format.

## 🔧 Step 2 : Install EKS
### Install using Fargate
```
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```
##### ✅ Why Fargate?
AWS Fargate provides serverless compute for containers, so you don't need to manage EC2 nodes. It simplifies cluster management and auto-scales based on workload needs.
It creates:
✅ A dedicated VPC
✅ Subnets (public and/or private)
✅ Route Tables
✅ Internet Gateway (IGW)
✅ NAT Gateway (optional, depends on config)
✅ Security Groups
✅ Elastic IPs (for NAT)
✅ IAM Roles and Policies
✅ CloudFormation stack (manages all of the above)

## ☸️ Step 3 : Configuring kubectl for EKS:
* Use the AWS CLI to update the kubeconfig file:
```
aws eks update-kubeconfig --name your-cluster-name
```
* Verify the configuration by running a kubectl command against your EKS cluster:
```
kubectl get nodes
```
## 🧩 Step 4: Create Fargate Profile 
```
eksctl create fargateprofile \
    --cluster <cluster-name> \
    --region <region-name> \
    --name alb-sample-app \
    --namespace game-2048
```
Check the profile in the AWS EKS Console under Cluster → Compute → Fargate Profiles.

## 🚀 Step 5: Deploy 2048 Game + Service + Ingress

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```
Verify:
```
kubectl get pods -n game-2048
```
###### ℹ️ Note: Ingress will be created, but without an ALB address unless the AWS Load Balancer Controller is installed.

## 🔐 Step 6: Commands to configure IAM OIDC providerr
 ### ✅ Why IAM OIDC?
This allows EKS to securely assume IAM roles for Kubernetes service accounts.
```
export cluster_name=demo-cluster
```
```
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5) 
```
### Check if there is an IAM OIDC provider configured already
* ###### aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4\n
If not, run the below command
```
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```
## 🛡️ Step 7: Set Up ALB Add-On
### Download IAM Policy
```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```
### Create IAM Policy
```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```
### Create IAM Role
```
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```
## 📦 Step 8: Install AWS Load Balancer Controller (via Helm)
##### Add Helm Repo
```
helm repo add eks https://aws.github.io/eks-charts
```
##### Update Helm Repo
```
helm repo update eks
```
##### Install Controller
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=<cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region-name> \
  --set vpcId=<vpc-id>
```
🔍 Find VPC ID from AWS Console → EKS → Cluster → Networking

## ✅ Step 9: Verify Deployment

##### Check Controller Deployment

```
kubectl get deployment -n kube-system aws-load-balancer-controller
```
##### Check Running Pods
```
kubectl get pods -n kube-system
```
## 🌐 Step 10: Verify Ingress and Access the Game
```
kubectl get ingress -n game-2048
```
##### 🟢 Check the ALB DNS in browser once it Active:

##### Open < alb-dns > to play the 2048 game.

## 🧹 Step-by-Step: Delete All Resources
##### ✅ 1. Delete Kubernetes Resources (Ingress, Service, Deployments)
```
kubectl delete -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```
##### ✅ 2. Delete Fargate Profile
```
eksctl delete fargateprofile \
  --cluster <cluster-name> \
  --name alb-sample-app \
  --region <region-name>
  ```
##### ✅ 3. Delete Load Balancer Controller
If installed with Helm:
```
helm uninstall aws-load-balancer-controller -n kube-system
```
Then, delete the service account:
```
eksctl delete iamserviceaccount \
  --name aws-load-balancer-controller \
  --namespace kube-system \
  --cluster <cluster-name> \
  --region <region-name>
  ```
  ##### ✅ 4. Delete IAM Policy
  ```
   aws iam delete-policy --policy-arn arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy
  ```
  ###### Replace <your-aws-account-id> accordingly.
  
  ##### ✅ 5. Delete EKS Cluster
  This will delete the cluster and all associated Fargate profiles automatically:
  ```
  eksctl delete cluster --name <cluster-name> --region <region-name>
  ```
  ##### ✅ 6. Delete ALB (If Leftover)
  In rare cases, if the ALB created by the ingress is still active:
* Go to the AWS Console → EC2 → Load Balancers
* Manually find and delete the ALB associated with your cluster

 ##### ✅ 7. Clean Up Remaining Resources (VPC, Subnets, etc.)
