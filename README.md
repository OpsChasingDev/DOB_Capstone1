# DOB_Capstone1
Increment and build an app version artifact to a private ECR repository and deploy the app to AWS EKS cluster.

CloudFormation template used for EKS VPC:
https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml

## Pre-Deployment Steps - EKS Cluster Configuration
1. Create IAMs role to give AWS Service trusted entity type the ability to manage EKS Cluster on our behalf.  This should have the "AmazonEKSClusterPolicy" AWS managed policy attached to it by default during setup.
2. Create the VPC the worker nodes will use by leveraging CloudFormation part of AWS.  Use the [AWS managed template](https://docs.aws.amazon.com/eks/latest/userguide/creating-a-vpc.html) yaml for two private and two public subnets found at [this url](https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml).

*Note here that the information provided in the "Outputs" section of CloudFormation once the VPC is created will be needed later for telling the control plane where to connect to for managing the worker nodes in the cluster.*

3. Create the EKS cluster (manual option)
   1. Specify the role we created in step 1 as the Cluster Service Role option.
   2. Specify the ID of the new VPC we created in step 2 with CloudFormation.
   3. Specify the security group of the new VPC we created in step 2.
   4. Choose an endpoint access option (this determines from where your worker nodes access the managed VPC running the master nodes managed by AWS automatically with the EKS service).  Ideal choice is "Public and Private".
4. Create EKS Cluster using eksctl (automated - skip step 5 if doing this method)
   1. Install eksctl (use [chocolatey](https://chocolatey.org/install) for Windows)
      ```PowerShell
      choco install eksctl
      ```
   2. Configure AWS creds for eksctl - the below should list a ~/.aws/config file present
      ```Bash
      aws configure list
      ```
   3. Run command to create cluster using any default overrides (or run yaml file containing this information)
      ```PowerShell
      eksctl create cluster `
      --name demo-cluster `
      --version 1.25 `
      --region us-east-1 `
      --nodegroup-name demo-nodes `
      --node-type t2.micro `
      --nodes 2 `
      --nodes-min 1 `
      --nodes-max 3
      ```
   4. After some time, EKS will be available to connect to automatically using the newly generated kubeconfig file.  Verify the cluster is running with worker nodes by running the below:
      ```Bash
      kubectl.exe get node
      ```
5. Connect to the EKS cluster
   1. From a terminal, run the below to verify the value for the "region" field matches where the cluster was created (if not, this will need to be changed in the AWS CLI with "aws configure")
      ```Bash
      aws configure list
      ```
   2. Create new kubeconfig file by running the below command, providing the name of the created cluster
      ```Bash
      aws eks update-kubeconfig --name {{ eks_cluster_name }}
      ```
   3. Kubeconfig file can be viewed at output location ~/.kube/config
   4. Run the below command to verify kubectl is now connected to the EKS cluster
      ```Bash
      kubectl cluster-info
      ```

## Pre-Deployment Steps - Jenkins Server Requirements
1. Install kubectl on Jenkins server
   ```Bash
   curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl; chmod +x ./kubectl; mv ./kubectl /usr/local/bin/kubectl
   ```
2. Install AWS IAM Authenticator (specific to AWS)
   ```Bash
   curl -o aws-iam-authenticator https://amazon-eks.s3.us-west-2.amazonaws.com/1.15.10.2020-02-22/bin/linux/amd64/aws-iam-authenticator
   ```
3. Add execute permissions to aws-iam-authenticator
   ```Bash
   chmod +x ./aws-iam-authenticator
   ```
4. Move aws-iam-authenticator
   ```Bash
   mv ./aws-iam-authenticator /usr/local/bin
   ```
5. Prepare kubeconfig file for Jenkins
   1. Create manually in Docker droplet and then copy to container (for easier editing)
   2. Create file called config with the below template (yaml format, so indentation matters)
      ```Yaml
      apiVersion: v1
      clusters:
      - cluster:
          server: <endpoint-url>
          certificate-authority-data: <base64-encoded-ca-cert>
        name: kubernetes
      contexts:
      - context:
          cluster: kubernetes
          user: aws
        name: aws
      current-context: aws
      kind: Config
      preferences: {}
      users:
      - name: aws
        user:
          exec:
            apiVersion: client.authentication.k8s.io/v1alpha1
            command: aws-iam-authenticator
            args:
              - "token"
              - "-i"
              - "<cluster-name>"
      ```
   3. Replace the values appropriately for the <endpoint-url>, <base64-encoded-ca-cert>, and <cluster-name>.  
      1. The <endpoint-url> will come from the public access url for the cluster API given to us by AWS after creating the EKS cluster.
      2. The <base64-encoded-ca-cert> we can obtain from our local machine's kubeconfig file after connecting to the newly created cluster.
      3. The <cluster-name> we defined ourselves when creating the EKS cluster.
   4. Make new config file availble for Jenkins container at the default kubeconfig location.
      1. In Jenkins container, create directory ".kube" in the Jenkins user's home directory.
      2. Back out to the droplet and copy the config file to the container location
         ```Bash
         docker cp config {{ container_id }}:/var/jenkins_home/.kube/
         ```
6. Prepare credentials for AWS User in Jenkins by creating two "secret text" type credentials in Jenkins server UI; one for the AWS Access Key ID, and another for the AWS Secret Access Key.