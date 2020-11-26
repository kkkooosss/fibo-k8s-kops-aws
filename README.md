# fibo-k8s-kops-aws

# **This is a repository of creation kubernetes cluster with KOPS in AWS. And deployment of ReactApp to this cluster**

You will need admin access to AWS for to create a new user with permissions for KOPS. Deployment could be done under the KOPS user credentials.

# 1. Creation kubernetes cluster with KOPS.

Original creation k8s cluster in AWS with KOPS you can find here: 

https://aws.amazon.com/blogs/compute/kubernetes-clusters-aws-kops/

https://kops.sigs.k8s.io/getting_started/aws/


Steps:
1. Install KOPS
```
curl -Lo kops https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
chmod +x ./kops
sudo mv ./kops /usr/local/bin/
```
Check KOPS version
```
kops version
```
2. Under ADMIN aws user create user kops
```
aws iam create-group --group-name kops

aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops

aws iam create-user --user-name kops

aws iam add-user-to-group --user-name kops --group-name kops

aws iam create-access-key --user-name kops
```
3.  Configure the aws client to use your new IAM user
```
aws configure           # Use your new access and secret key here
aws iam list-users      # you should see a list of all your IAM users here
```
4. Because "aws configure" doesn't export these vars for kops to use, we export them now
```
export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)
export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)
```
5. Create S3 bucket for the state-storage of cluster
```
aws s3api create-bucket \
    --bucket kops1-storage \
    --create-bucket-configuration LocationConstraint=eu-central-1 \ #if the region is NOT the <us-east-1>#
    --region eu-central-1
``` 
6. Creat a Cluster 3 STEPS

Step 1
Name & s3 Bucket
```
export NAME=my-kops-cluster.k8s.local ##XXX.XXX.XXX
export KOPS_STATE_STORE=s3://kops1-storage
```
Step 2
Create configuration of cluster
```
aws ec2 describe-availability-zones --region eu-central-1  | grep ZoneName
```
chooze AZ

Step 3

Create cluster configuration.
Before generate ssh keypair
```
ssh-keygen -b 2048 -t rsa -f .ssh/id_rsa
```
```
kops create cluster \
    --zones=eu-central-1a,eu-central-1b,eu-central-1c \
    ${NAME}   
```
Build the cluster 
```
kops update cluster ${NAME} --yes   
```
Cluster build successfully.

Check the cluster 
 ```
kops validate cluster
kops validate cluster --wait 10m
```
# Deployment.

1. Clone the REPO
```
git clone https://github.com/kkkooosss/fibo-k8s-kops-aws.git
```
2. In *ingress-srvice.yaml* change the domain name to yours.
   
3. In *deploy-tls-termination.yaml* change VPC CIDR block from you cluster VPC. And *arn* of SSL certificate
4. Also you can change password in *secret.yaml*. It's a password for postgress DB. (Note. It should be base 64 encoded)
5. Install kubectl if you don't have it yet.
6. deploy the application
```
kubectl apply -f .
```
we have get the complete deployment.

Last step. Go to Rote53 in you aws account and adjust A Record in hosted zone of your domain for cluster loadbalancer which had been created. 
DNS name of LB you can find in :
```
kubectl describe ingress ingress-service 
```

