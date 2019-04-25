# EKS Deployment on AWS
The purpose of this simple workshop is to provide guide as a starting of your Kubernates Journey on top of AWS.
After setting up your EKS Cluster, we will deploy a microservices e-Commerce contributed by WeaveWorks to our EKS Cluster 
and try out AutoScaling features (Manual and Horizontal Pods AutoScale).

## STEP1: EKS CLUSTER CREATION
#### 1. Create EKS Service Role
#### 2. Create VPC using CloudFormation Template
  ```
  https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-02-11/amazon-eks-vpc-sample.yaml
  ```
#### 3. Go to EKS Console and Create EKS Cluster

Next step onwards is to prepare a client machine to install kubectl and manage your EKS Cluster/WorkerNodes, 
you can use your laptop, or using EC2 or simplest one is Cloud9

IMPORTANT! If you Cloud9 by default STEP2 will failed because Cloud9 rotate credentials (secret/access key) 
and this is not supported by kubectl, because it detect/match the exact access key that represent 
IAM User that is used to initially create the cluster.
If you use Cloud9, then ensure you go to Preferences -> AWS Settings -> Turn off "AWS managed temporary credentials

#### 4. install kubectl
   ```bash
   mkdir $HOME/bin
   curl -o kubectl https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/linux/amd64/kubectl
   chmod +x ./kubectl
   cp ./kubectl $HOME/bin/kubectl
   export PATH=$HOME/bin:$PATH
   echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
   kubectl version --client
   ```
#### 5. Install IAM Authenticator -> this is to allow us to manage EKS Cluster using our IAM identity
   ```bash
   curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/linux/amd64/aws-iam-authenticator
   chmod +x ./aws-iam-authenticator
   cp ./aws-iam-authenticator $HOME/bin
   export PATH=$HOME/bin:$PATH
   echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
   aws-iam-authenticator help
   ```
#### 6. Configure kubectl for EKS
   ```bash
   aws eks update-kubeconfig --name <EKS CLuster Name>
   ```
  Test your kubectl command
   ```bash
   kubectl cluster-info
   ```

## STEP2: DEPLOYING WORKER NODES INTO CLUSTER
#### 1. Worker Nodes CloudFormation Template
  ```
  https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2018-11-07/amazon-eks-nodegroup.yaml
  ```
  Specify:
  ```
    StackName: e.g. EKS-WorkerNodes
    ClusterName: e.g. EKSCluster
    ClusterControlPlaneSG: <find EKS-VPC-ControlPlaneSecurityGroup>
    NodeGroupName: e.g. NodeGroup
    Observe AutoScalingGroup etc and change if required
    AMI ID, get latest from below
      Latest AMI List:
      https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html
    VPCID: <Select VPC created using CloudFormation Template in STEP1.2>
    Subnets: <Select all 3 "EKS-VPC" available subnets created using CloudFormation Template in STEP1.2>
    KeyName: EC2KeyPair to SSH to the node
  ```
  Proceed with Stack creation, once completed, go to "Output" and get "NodeInstanceRole" ARN for next step
#### 2. Worker Nodes to join EKS Cluster
  In order for the worker nodes to join EKS Cluster, we need to use "AWS Authenticator Configuration Map"
  Download
   ```bash
    curl -O https:/amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2018-11-07/aws-auth-cm.yaml
   ```
  Edit the downloaded yaml file, add in NodeInstanceRole from previous Step 2.1
  ```bash
   kubectl apply -f aws-auth-cm.yaml
   kubectl get nodes --watch
  ```

## STEP3: DEPLOY YOUR MICROSERVICES E-COMMERCE & WATCH IT IN ACTION!
#### 1. Install Git
  ```
  sudo yum install git
  ```
#### 2. Switch to home directory and clone Microservice repo
  ```
  git clone https://github.com/antoniuslee/microservices-demo
  ```
#### 3. Install Microservice Application to the Cluster
  Create namespace for the application
  ```
  kubectl create namespace sock-shop
  ```
  Install microservice application under "sock-shop" namespace
  ```
   kubectl -n sock-shop create -f microservices-demo/deploy/kubernetes/complete-demo.yaml
  ```
  List pods for newly created application
  ``` 
  kubectl get pods -n sock-shop 
  ```
  or watch them in real time
  ```
  kubectl get pods -n sock-shop -w
  ```
#### 4. Access Microservice Application
  List services in the application
  ```
   kubectl get services -n sock-shop
  ```
  Get information about the NodePort service:
  ```
  kubectl describe services -n sock-shop front-end
  ```
  Access the application using the IP of one of the cluster nodes and the port from the "NodePort" service
  ``` http://<IP_ADDRESS>:30080
  ```
  
  If you can't access, please check your security group for worker nodes to ensure port 30080 accessible from the internet

#### 5. Try out your MicroService Application (e.g. Create an account, login, browse products, add product to cart, view cart and checkout etc)


## OPTIONAL
### MANUAL SCALING MICROSERVICES
#### 1. List deployments
  ``` 
  kubectl get deployments -n sock-shop
  ```
#### 2. List deployment and pods
  ```bash
  kubectl get deployments front-end -n sock-shop
  kubectl get pods -n sock-shop
  ```
#### 3. Get additional information about deployments
 ```
 kubectl describe deployments front-end -n sock-shop
 ```
#### 4. Scale directly from command line
  ```
  kubectl scale deployment/front-end --replicas=3 -n sock-shop
  ```
#### 5. List deployment and pods
  ```bash 
  kubectl get deployments front-end -n sock-shop
  kubectl get pods -n sock-shop
  ```

## AUTO SCALING MICROSERVICES using HPA (Horizontal Pod AutoScaler)
#### 1. List current deployments
  ```
  kubectl get deployments front-end -n sock-shop
  ```
#### 2. Create Horizontal Pod Autoscaler for a deployment
  ```
  kubectl autoscale deployment front-end -n sock-shop --min 2 --max 6 --cpu-percent 65
  ```
#### 3. List Horizontal Pod Autoscaler
  ```
  kubectl get hpa -n sock-shop
  ```
#### 4. Delete HPA
  ```
  kubectl delete hpa -n sock-shop <HPA>
  ```
  
## ADDITIONAL NOTES;
Explore eksctl, a much easier way to deploy EKS Cluster in single command on AWS!
> https://eksctl.io
