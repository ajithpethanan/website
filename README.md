# Project Title: End-to-End CI/CD Automation Using Terraform, Jenkins, Docker, and Kubernetes

# Overview

# This project demonstrates complete automation of infrastructure provisioning and CI/CD deployment pipeline using Terraform, Jenkins, Docker, and Kubernetes on AWS. It provisions three EC2 Ubuntu servers (1master, 2 slaves), installs Kubernetes, deploys a sample web app from GitHub, builds a Docker image, pushes it to Docker Hub, and deploys it to Kubernetes.

# Tools Used

1.Terraform

2.Jenkins

3.Docker

4.Kubernetes

5.AWS EC2 (Ubuntu 22.04)

6.GitHub

7.Docker Hub

# Step-by-Step Implementation

# Step 1: Create an Ubuntu EC2 Instance

Created an instance with:
                   OS: Ubuntu
                   Key Pair Name: ubuntu-ajith
                   
# Step 2: Install Terraform on the Instance

Installed Terraform using the following commands:

       wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
       echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee 
       /etc/apt/sources.list.d/hashicorp.list
       sudo apt update && sudo apt install terraform

# Step 3: Create Terraform Configuration File (main.tf):

Created a file named main.tf with the following content:

                             terraform {
                                   required_providers {
                                                aws = {
                                               source  = "hashicorp/aws"
                                                }
                                             }
                                          }

                             provider "aws" {
                              region     = "ap-south-1"
                              access_key = ""
                              secret_key = ""
                            }

                            resource "aws_instance" "example" {
                                 ami           = "ami-0f5ee92e2d63afc18"
                                 count         = 2
                                 instance_type = "t2.medium"
                                 key_name      = "ubuntu"
                                 tags = {
                                    Name = "kub-slave"
                                    }
                           }

                           resource "aws_instance" "main" {
                                  ami           = "ami-0f5ee92e2d63afc18"
                                  count         = 1
                                  instance_type = "t2.medium"
                                  key_name      = "ubuntu"
                                 tags = {
                                     Name = "kub1-master"
                                    }
                           }

# Step 4: Initialize and Deploy Infrastructure Using Terraform

After creating the main.tf file, ran the following commands to provision the infrastructure:

Initialize Terraform:
             terraform init
Create an Execution Plan:
             terraform plan
Apply the Plan and Create the Resources:
             terraform apply


This created 3 EC2 instances:

2 instances with the tag kub-slave

1 instance with the tag kub1-master


# Step 5: Set Up Docker and Kubernetes on Master and Slave Nodes

On each node (master and both slaves), created a bash script named install.sh to install Docker and Kubernetes.

Content of install.sh:
!/bin/bash
      
# Update the system
sudo apt update -y

# Install Docker
sudo apt install docker.io -y

# Start and Enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Install Kubernetes Components
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list


To Run the Script:
               bash install.sh
               
This script installed:

                Docker
                Kubeadm
                Kubectl
                Kubelet
                
on both the master and slave instances.

# Step 6: Initialize Kubernetes Cluster

Switched to root user on the Master node:
               sudo -i
Initialized the Kubernetes cluster using:
               kubeadm init
After running kubeadm init, Kubernetes provided a kubeadm join token command.

# Step 7: Join Slave Nodes to the Master:

Copied the kubeadm join command generated after the master initialization.

On both Slave nodes, ran the copied kubeadm join command to connect the nodes to the Master.

Example:
         kubeadm join <Master-IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>

This successfully added both Slave nodes to the Kubernetes cluster.

# Step 8: Configure kubectl on Master Node

After initializing the cluster, configured kubectl to manage the Kubernetes cluster from the Master node:

Ran the following commands on the Master:
                mkdir -p $HOME/.kube
                sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
                sudo chown $(id -u):$(id -g) $HOME/.kube/config

These commands set up the kubeconfig file properly, allowing the use of kubectl without needing sudo.

Verified the nodes with:
                kubectl get nodes
                
# Step 9: Install CNI (Container Network Interface) Plugin

To enable networking between pods and nodes, installed the Weave Net CNI plugin.

Ran the following command on the Master node:
                  kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
This deployed the Weave Net networking solution across the Kubernetes cluster.

Checked the node status using:
                 kubectl get nodes 
After a few minutes, all nodes showed status as Ready.

# Step 10: Install Java (OpenJDK 17) on All Machines

Installed OpenJDK 17 on the Master (Controller) and Slave nodes:
                 sudo apt update
                 sudo apt install openjdk-17-jdk -y
Java installation was necessary for running Jenkins and other automation tools.

# Step 11: Install Jenkins on Controller Machine

On the Controller machine (where Terraform is installed), installed Jenkins by creating a script named jenkins.sh.

Content of jenkins.sh:
            #!/bin/bash
            sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
            https://pkg.jenkins.io/debian/jenkins.io-2023.key
            echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] \
            https://pkg.jenkins.io/debian binary/" | sudo tee \
            /etc/apt/sources.list.d/jenkins.list > /dev/null
            sudo apt-get update
            sudo apt-get install jenkins -y
            
To Execute the Jenkins Installation:
            bash jenkins.sh
This installed Jenkins successfully on the Controller machine.

# Step 12: Access Jenkins Dashboard

After installing Jenkins on the Controller machine, accessed the Jenkins UI via a web browser:
            http://<controller-machine-public-ip>:8080
            
Initial Setup:
1. Unlocked Jenkins using the initial admin password:
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
2.Installed suggested plugins.
3.Created the first admin user.
4.Reached the Jenkins Dashboard successfully.

Jenkins was now ready for automating Kubernetes or Docker tasks.

# Step 13: Add Jenkins Agent Node (kub-master)

To automate builds and deployments from Jenkins to the Kubernetes environment, added a new agent node in Jenkins.

Steps Followed:
1.Navigated to:
   Jenkins Dashboard > Manage Jenkins > Manage Nodes and Clouds > New Node
2.Node Configuration:
    *Node Name: kub-master
    *Type: Permanent Agent
    *Remote root directory: /home/ubuntu/jenkins
    *Labels: Kub-master (used to target jobs)
    *Usage: Use this node as much as possible
    *Launch Method: Launch agents via SSH
               Host: IP address of the node
               Credentials: Added using the .pem key file (EC2 key pair)
                              Username: ubuntu
                              Private key: pasted contents of the .pem file directly
               Host Key Verification Strategy: Non verifying verification strategy
3.Clicked Save — the agent node was successfully added and came online.

# Step 14: Create a Test Jenkins Pipeline Job

To verify the agent node and the connection, created a simple test pipeline.

Steps Followed:
1.Dashboard > New Item
2.Item Name: pipeline
3.Project Type: Pipeline
4.Under the Pipeline section, added the following script:
                       pipeline {
                           agent none
                              stages {
                                  stage('Hello') {
                                  agent {
                                  label 'Kub-master'
                                     }
                             steps {
                                 echo 'Hello World'
                                  }
                                 }
                               }
                             }
 5.Clicked Save, then Build Now.
 Pipeline executed successfully and printed "Hello World" — confirming the Jenkins Agent node was working.

#  Step 15: DockerHub Integration in Jenkins
 To enable Docker image pushes to DockerHub from Jenkins, DockerHub credentials were added under Jenkins Global Credentials.
 
 Steps:
 1.Go to Jenkins > Manage Jenkins > Credentials > (global) > Add Credentials.
 2.Chose:
      Kind: Username with password
      Username: <your_dockerhub_username>
      Password: <your_dockerhub_password>
3.Saved it — Jenkins generated a Credentials ID:
      Example: e814f99d-8cc0-425d-840e-0c10c489f570
This ID is used in the Jenkins pipeline script for DockerHub authentication.

 # Step 16: Jenkins Pipeline Script (Docker + Kubernetes Deployment)
 
 A new Jenkins pipeline was created with the following script:
                              pipeline {
                                  agent none
                                  environment {
                                           DOCKERHUB_CREDENTIALS = credentials('e814f99d-8cc0-425d-840e-0c10c489f570')
                                            }

                                   stages {
                                      stage('Hello') {
                                            agent {
                                                label 'Kub-master'
                                             }
                                            steps {
                                               echo 'Hello World'
                                              }
                                            }

                                     stage('Git') {
                                          agent {
                                          label 'Kub-master'
                                          }
                                          steps {
                                              git 'https://github.com/intellipaat2/website.git'
                                        }
                                      }

                                 stage('Docker Build & Push') {
                                     agent {
                                         label 'Kub-master'
                                     }
                                     steps {
                                          sh 'sudo docker build /home/ubuntu/jenkins/workspace/pipeline -t ritikdevoper123/demo1'
                                          sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                                         sh 'docker push ritikdevoper123/demo1'
                                      }
                                   }

                               stage('Kubernetes Deployment') {
                                  agent {
                                       label 'Kub-master'
                                 }
                                 steps {
                                     sh 'kubectl create -f deploy.yml'
                                     sh 'kubectl create -f svc.yml'
                                  }
                                 }
                              }
                            }

      This pipeline:
             Clones the GitHub repository
             Builds the Docker image
             Pushes it to DockerHub
             Deploys the container into Kubernetes using YAML files

 # Step 17: Kubernetes Deployment Configuration
 
 Two YAML files were used for deploying the Docker container on the Kubernetes cluster.

deploy.yml

         apiVersion: apps/v1
         kind: Deployment
         metadata:
            name: webdeployment
         spec:
           replicas: 1
           selector:
             matchLabels:
               app: webapp
           template:
             metadata:
               labels:
                 app: webapp
            spec:
                containers:
               - name: webcontainer
                 image: ritikdevoper123/demo1
                 ports:
                  - containerPort: 80


   svc.yml
   
          apiVersion: v1
          kind: Service
          metadata:
             name: webappsvc
          spec:
            type: NodePort
            selector:
               app: webapp
            ports:
              - protocol: TCP
                port: 80
                targetPort: 80
                nodePort: 30007

  These files create a deployment and NodePort service which expose the application on port 30007.

# Step 18: Verify the Deployment

  Commands Used:
  
              kubectl get pods
              kubectl get svc
        
  You should see the pod running and the service exposing port 30007.
  
  Access the application using:
  
              http://<NodePublicIP>:30007
         
  The application successfully ran inside a Kubernetes pod, deployed through Jenkins pipeline automation.











  












             

       


                   


 



