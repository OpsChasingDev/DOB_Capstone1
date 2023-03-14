# DOB_Capstone1
Increment and build an app version artifact to a private ECR repository and deploy the app to AWS EKS cluster.

CloudFormation template used for EKS VPC:
https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml

## Pre-Deployment Steps - EKS Cluster Configuration
1. Create IAMs role to give AWS Service trusted entity type the ability to manage EKS Cluster on our behalf.  This should have the "AmazonEKSClusterPolicy" AWS managed policy attached to it by default during setup.
2. Create the VPC the worker nodes will use by leveraging CloudFormation part of AWS.  Use the [AWS managed template](https://docs.aws.amazon.com/eks/latest/userguide/creating-a-vpc.html) yaml for two private and two public subnets found at [this url](https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml).