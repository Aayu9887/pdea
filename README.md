# pdea

1. Create EC2 Instance:
=======================
=> Create an EC2 instance named "jenkins-server" with Instance Type as "t3.small" and based on the latest Ubuntu Machine Image.
=> Navigate to the EC2 service and create a security group named "jenkins-server-security-group".
=> Configure the inbound rules to allow traffic from anywhere (IPv4) on:Port 8080 (Custom TCP), Port 22 (SSH for EC2 access).
=> Create a key pair named "jenkins-ssh-key".

NOTE: Add this code in the EC2 user data to Install amazon-ssm-agent in the Instance. (Before adding below script you have to shotdown the EC2 machine)

#!/bin/bash
sudo apt-get update
sudo apt-get install -y amazon-ssm-agent
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent

2. Configure IAM Role
=====================
=> Create IAM role named "ec2-eks-role" for the EC2 service. 
=> Assign a policy that grants full administrative access. 
=> Attach this IAM role to the EC2 instance.

=> Once the role is attached, use SSH or EC2 Instance Connect from the console to log into the EC2 instance. 

3. Install Required Software
============================
=> Install Java:

sudo apt-get update
sudo apt install default-jdk -y

=> Install AWS CLI and Authenticator: 

sudo apt-get install zip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" 
unzip awscliv2.zip 
sudo ./aws/install 
curl -Lo aws-iam-authenticator https://github.com/kubernetes-sigs/aws-iam-authenticator/releases/download/v0.5.9/aws-iam-authenticator_0.5.9_linux_amd64 

=> Install Docker

sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update 
sudo apt install docker-ce docker-ce-cli containerd.io  -y

=> Start Docker Service: 

Use systemctl to start the Docker.
You can check the status of the docker using "sudo systemctl status docker"

=> Install Jenkins: 

curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update 
sudo apt-get install jenkins -y

=> Start Jenkins Service: 

Use systemctl to start the jenkins.
You can check the status of the jenkins using  "sudo systemctl status jenkins"

On successful installation, the Jenkins service will be running and active

=> Unlock Jenkins:

Use Virtual machine provided to access jenkins (VM-lab)

Access the Jenkins UI at http://<public-IP>:8080.(replace <public-IP> with public ip of the created EC2 instance) 

=> Use the below command to get the admin password of the Jenkins server. 
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword 

Paste the password into the administration password field and click on Install suggested plugins. 
Create the first admin user with username "jenkins-admin-user" and password "jenkins_springboot4684$". 

4. Configure Jenkins with Docker Volumes and Networks
=====================================================
Once the Jenkins configuration is complete, navigate to the AWS EC2 instance for further configurations.

Use sudo for creation of resources wherever required.

1. Create a Docker volume for persistent Jenkins data with name 'jenkins_data'
2. Run Jenkins in a Docker container using the created volume.
3. Docker Network Setup:

Create a Docker network for Jenkins with name 'jenkins_network'
Attach the Jenkins container to the network

sudo docker volume create jenkins_data
sudo docker network create jenkins_network
sudo docker run -d --name jenkins \
  --restart=always \
  --network jenkins_network \
  -p 8088:8088 \
  -p 50000:50000 \
  -v jenkins_data:/var/jenkins_home \
  jenkins/jenkins:lts

5. Install Kubernetes Tools
===========================
=> Install kubectl:
  curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
  chmod +x ./kubectl
  sudo mv ./kubectl /usr/local/bin/kubectl

  Verify the kubectl installation with 
  kubectl --help 

=> Install eksctl: 
  ARCH=amd64 
  PLATFORM=$(uname -s)_$ARCH 
  curl -sLO "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz" 
  tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz 
  sudo mv /tmp/eksctl /usr/local/bin 

  Verify the installation of eksctl.

  eksctl --help

6. Create an ECR Repository
===========================
Go to the Amazon Elastic Container Registry(ECR) service and create a private repository named "springboot-jenkins-ecr-repo".
$ aws ecr create-repository --repository-name springboot-jenkins-ecr-repo --region us-east-2

7. Configure Jenkins for Docker
===============================
Add Jenkins User to Docker Group:

sudo usermod -a -G docker jenkins
sudo service jenkins restart
sudo systemctl daemon-reload
sudo service docker stop
sudo service docker start

8. Create EKS Cluster and Node Group
====================================
Create EKS Cluster with name "jenkins-cluster" in "us-east-2" region without a nodegroup
   eksctl create cluster --name jenkins-cluster --region us-east-2 --without-nodegroup
   eksctl get cluster --name jenkins-cluster --region us-east-2

Note: Cluster creation may take 15-20 minutes. The configuration will be stored in "/var/lib/jenkins/.kube/config".

Save Kube Configuration using the below command.

cat /var/lib/jenkins/.kube/config

Save the kube configuration by creating a text file on with name "kubecongig.txt" and save it which would be further required at Jenkins credentials configuration.

=> Create nodegroup with name "jenkins-nodegroup", node type as "t3.small" and with node count 2.

eksctl create nodegroup \
  --cluster jenkins-cluster \
  --name jenkins-nodegroup \
  --node-type t3.small \
  --nodes 2 \
  --region us-east-2

kubectl get nodes

9. Set Up GitHub Repository 
=========================
Navigate to virtual machine provided (VM-lab)

Open terminal and set the global git user to your personal GitHub user
git config --global user.name "Abraham Battu"
git config --global user.email "probattu@gmail.com"

https://github.com/settings/keys
https://github.com/settings/tokens

Create a private GitHub repository named "springboot-jenkins-eks" in your GitHub account.

Use the files provided in the project folder and upload them to the repository created. 

Replace <<REPOSITORY URI>> in the "eks-deploy-k8s.yaml" file with the ECR repository URI. 

10.Configure Maven Build Tool and Install Required Jenkins Plugins
==================================================================
In Jenkins, navigate to Manage Jenkins > Tools and add a Maven installation named Maven3, then save it.
Navigate to Manage Jenkins > Plugins.
Install the Kubernetes CLI, Amazon ECR, Docker, and Docker Pipeline plugins without restarting.

11. Configure Jenkins Credentials 
=================================
Go to Manage Jenkins > Credentials and configure the following: 
GitHub Credentials: Add your GitHub username and personal access token (PAT) with the ID "github-credentials". 
Kubernetes Credentials: Add the kubeconfig.txt file created earlier with the ID "K8S" (Kind: Secret File). 

12. Create and execute the Jenkins Pipeline 
===========================================
Create a new Jenkins pipeline named "springboot-jenkins-eks" with the description: "Automation of Spring Boot microservices builds using Jenkins pipeline and deploy it on AWS EKS Cluster." 

Use the pipeline script provided in the project folder with the name "jenkins_script.txt".
Ensure to configure the ECR repository and GitHub repo details as instructed.
Upon creating the Jenkins pipeline, trigger a build. A successful run will dockerize the Spring Boot application and deploy it on EKS.

13. Access the Deployed Application
===================================
A Classic Load Balancer will be created by the Kubernetes Service. Access the DNS of the load balancer to view the deployed Spring Boot application. 

Testcases
=========
Check creation of the Jenkins EC2 instance. [5 marks]
Checking for the creation of the role. [5 marks]
Checking for the creation and attachment of the security group. [5 marks]
Check installation and accessibility of Jenkins on the EC2 instance. [5 marks]
Check installation of plugins: Docker, Docker Pipeline and Kubernetes CLI on Jenkins[10 marks]
Check creation of the ECR repository and presence of Dockerized Spring Boot application. [10 marks]
Check Docker volume creation with the name jenkins_data. [10 marks]
Check Docker network creation with name jenkins_network. [10 marks]
Check if the Jenkins pipeline was successful. [10 marks]
Check creation of the EKS Cluster[10 marks]
Check Node group creation[10 marks]
Kubernetes deployment is successful[10 marks]
