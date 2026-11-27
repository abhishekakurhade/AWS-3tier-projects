# 3-Tier Student Management Application: AWS Deployment Guide
 
A 3-tier application (Frontend, Backend, Database) deployed on AWS using **Jenkins CI/CD**, **Docker**, **Terraform**, **Amazon EKS (Kubernetes)** and **Amazon RDS (MySQL)**.
 
---
 
## Table of Contents
 
1. [Architecture](#architecture)
2. [Tech Stack](#tech-stack)
3. [Prerequisites](#prerequisites)
4. [Step 1: Create the RDS Database](#step-1-create-the-rds-database)
5. [Step 2: Set Up the Jenkins Server](#step-2-set-up-the-jenkins-server)
6. [Step 3: Set Up the k8s-agent Node](#step-3-set-up-the-k8s-agent-node)
7. [Step 4: Initialise the Database Schema](#step-4-initialise-the-database-schema)
8. [Step 5: Configure Jenkins Credentials and Agent](#step-5-configure-jenkins-credentials-and-agent)
9. [Step 6: Jenkins Pipelines](#step-6-jenkins-pipelines)
10. [Step 7: Verify the Deployment](#step-7-verify-the-deployment)
11. [Known Issues and Fixes](#known-issues-and-fixes)
12. [Security Notes](#security-notes)
13. [Troubleshooting](#troubleshooting)
---
 
## Architecture
 
```
                 GitHub (source code)
                         |
                         v
        +----------------------------------+
        |  Jenkins Server (port 8081)      |
        |  - Controller / UI               |
        +----------------+-----------------+
                         | SSH
                         v
        +----------------------------------+
        |  k8s-agent (Jenkins agent node)  |
        |  Docker | kubectl | aws-cli      |
        |  eksctl | terraform              |
        +--------+-----------------+-------+
                 |                 |
     Terraform   |                 | docker push
                 v                 v
        +----------------+   +--------------+
        |  Amazon EKS    |   |  Docker Hub  |
        |  Cluster       |<--+  (images)    |
        |  +----------+  |   +--------------+
        |  | Frontend |  |
        |  +----+-----+  |
        |       v        |
        |  +----------+  |
        |  | Backend  |--+--------> Amazon RDS (MySQL)
        |  +----------+  |           student_db
        +----------------+
```
 
**Flow:**
1. Jenkins pipeline runs on the `k8s-agent` node.
2. Terraform provisions the EKS cluster.
3. Backend and frontend pipelines build Docker images, push them to Docker Hub, and deploy them to EKS.
4. The backend connects to the RDS MySQL database.
---
 
## Tech Stack
 
| Layer / Tool     | Technology                       |
|------------------|----------------------------------|
| Frontend         | Containerised web app (Docker)   |
| Backend          | Spring Boot (Docker)             |
| Database         | Amazon RDS (MySQL)               |
| CI/CD            | Jenkins (LTS)                    |
| Container Registry | Docker Hub                     |
| Orchestration    | Amazon EKS (Kubernetes)          |
| Infrastructure   | Terraform, eksctl                |
| Cloud            | AWS (region: `ap-south-1`)       |
 
---
 
## Prerequisites
 
- An AWS account with permissions for EKS, EC2, RDS, IAM and VPC
- A Docker Hub account
- Two Ubuntu EC2 instances:
  - **Jenkins server**
  - **k8s-agent** (build/deploy agent)
- Security groups that allow:
  - Jenkins server: inbound `22` (SSH) and `8081` (Jenkins UI)
  - RDS: inbound `3306` from the k8s-agent and from the EKS worker nodes
- Source repository: `https://github.com/abhishekakurhade/AWS-3tier-projects.git`
---
 
## Step 1: Create the RDS Database
 
Create an **Amazon RDS MySQL** instance from the AWS Console:
 
- Engine: MySQL
- Public access: only if needed for setup; prefer private access inside the VPC
- Note down the **endpoint**, **master username** and **password**
- Ensure the RDS security group allows port `3306` from the k8s-agent instance
---
 
## Step 2: Set Up the Jenkins Server
 
### 2.1 Install Java
 
```bash
sudo apt update
sudo apt install fontconfig openjdk-21-jre -y
java -version
```
 
### 2.2 Install Jenkins (LTS)
 
```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
 
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
 
sudo apt update
sudo apt install jenkins -y
```
 
### 2.3 Change the Jenkins port from 8080 to 8081
 
```bash
sudo nano /lib/systemd/system/jenkins.service
```
 
Find and update:
 
```ini
Environment="JENKINS_PORT=8081"
```
 
> **Tip:** A cleaner approach that survives package upgrades is
> `sudo systemctl edit jenkins` and adding the same `[Service]` / `Environment` line.
 
### 2.4 Reload systemd and restart Jenkins
 
```bash
sudo systemctl daemon-reload
sudo systemctl restart jenkins
sudo systemctl status jenkins
```
 
### 2.5 Unlock Jenkins
 
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
 
Open `http://<jenkins-public-ip>:8081`, paste the password, install suggested plugins and create the admin user.
 
---
 
## Step 3: Set Up the k8s-agent Node
 
### 3.1 Install Java
 
```bash
sudo apt update
sudo apt install fontconfig openjdk-21-jre -y
java -version
```
 
### 3.2 Install Docker
 
```bash
sudo apt install docker.io -y
```
 
### 3.3 Give the agent user Docker permissions
 
```bash
sudo usermod -aG docker $USER && newgrp docker
```
 
### 3.4 Install kubectl, AWS CLI, eksctl and Terraform
 
**kubectl**
 
```bash
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --client
```
 
> The 1.19.6 binary is very old. For a current EKS version, download the matching kubectl from the
> [Amazon EKS docs](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html).
> kubectl should be within one minor version of your cluster.
 
**AWS CLI**
 
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install
aws configure
```
 
Alternative:
 
```bash
sudo snap install aws-cli --classic
```
 
**eksctl**
 
```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```
 
**Terraform**
 
```bash
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
 
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list
 
sudo apt update && sudo apt install terraform -y
terraform -version
```
 
---
 
## Step 4: Initialise the Database Schema
 
Run these from the **k8s-agent** instance.
 
### 4.1 Install the MySQL/MariaDB client
 
```bash
sudo apt update && sudo apt install mariadb-client -y
```
 
> Only the client is needed because the database runs on RDS. The full `mariadb-server` package is not required.
 
### 4.2 Connect to RDS
 
```bash
mysql -h <RDS-ENDPOINT> -u <username> -p
```
 
### 4.3 Create the database, user and table
 
```sql
CREATE DATABASE student_db;
```
```sql
-- Create an application user (replace the password)
GRANT ALL PRIVILEGES ON springbackend.* TO 'username'@'localhost' IDENTIFIED BY 'your_password';
```
```sql
USE student_db;
 ```
```sql
CREATE TABLE `students` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) DEFAULT NULL,
  `email` varchar(255) DEFAULT NULL,
  `course` varchar(255) DEFAULT NULL,
  `student_class` varchar(255) DEFAULT NULL,
  `percentage` double DEFAULT NULL,
  `branch` varchar(255) DEFAULT NULL,
  `mobile_number` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=80 DEFAULT CHARSET=latin1 COLLATE=latin1_swedish_ci;
```
```sql
EXIT;
```
 
Make sure the database name, username and password match what the backend application is configured to use (for example `application.properties` or Kubernetes env vars/Secrets).
 
---
 
## Step 5: Configure Jenkins Credentials and Agent
 
### 5.1 Add credentials
 
Go to **Manage Jenkins → Credentials → System → Global credentials**:
 
| ID             | Type                         | Purpose                         |
|----------------|------------------------------|---------------------------------|
| `docker_login` | Username with password       | Docker Hub login                |
| `aws-login`    | AWS Credentials              | AWS access key / secret key     |
| `k8s-agent-ssh`| SSH Username with private key| SSH access to the k8s-agent node|
 
> The `aws-login` type requires the **AWS Credentials** plugin (part of *CloudBees AWS Credentials*).
 
### 5.2 Add the k8s-agent node
 
Go to **Manage Jenkins → Nodes → New Node**:
 
- Name: `k8s-agent`
- Type: Permanent Agent
- Remote root directory: `/home/ubuntu/`
- Labels: `k8s-agent`
- Launch method: *Launch agents via SSH*
- Host: `<k8s-agent-ip>`
- Credentials: the SSH credential created above
Confirm that the node shows **online**.
 
---
 
## Step 6: Jenkins Pipelines
 
Create three **Pipeline** jobs. Run them in this order.
 
### 6.1 K8s infrastructure pipeline
 
Provisions the EKS cluster using Terraform.
 
```groovy
pipeline {
    agent { label "k8s-agent" }
 
    stages {
        stage('clone code') {
            steps {
                git url: "https://github.com/abhishekakurhade/AWS-3tier-projects.git", branch: "main"
            }
        }
        stage('create k8s') {
            steps {
                withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-login', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                        terraform init
                        terraform apply -auto-approve
                    '''
                }
            }
        }
    }
}
```
 
### 6.2 Backend pipeline
 
Builds the backend image, pushes it to Docker Hub and deploys it to EKS.
 
```groovy
pipeline {
    agent { label "k8s-agent" }
    environment {
        IMAGE = "abhishekakurhade/3tier-backend"
        TAG   = "${BUILD_NUMBER}"
    }
    stages {
        stage('clone') {
            steps {
                git url: "https://github.com/abhishekakurhade/AWS-3tier-projects.git", branch: "main"
            }
        }
        stage('build image') {
            steps {
                sh '''docker build -f ./backend/dockerfile -t $IMAGE:$TAG .'''
            }
        }
        stage('push on docker') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker_login', passwordVariable: 'docker_pass', usernameVariable: 'docker_user')]) {
                    sh '''
                        echo "$docker_pass" | docker login -u "$docker_user" --password-stdin
                        docker push $IMAGE:$TAG
                    '''
                }
            }
        }
        stage('deploy on eks') {
            steps {
                withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-login', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                        aws eks update-kubeconfig --region ap-south-1 --name my-cluster
                        sed -i "s|image: .*|image: ${IMAGE}:${TAG}|g" backend/backend-pod.yaml
                        kubectl apply -f backend/backend-pod.yaml -n default
                        kubectl apply -f backend/backend-svc.yaml -n default
                        kubectl rollout status deployment -n default --timeout=120s
                    '''
                }
            }
        }
    }
}
```
 
### 6.3 Frontend pipeline
 
Same as the backend pipeline, with different image name and manifest files.
 
```groovy
pipeline {
    agent { label "k8s-agent" }
    environment {
        IMAGE = "abhishekakurhade/3tier-frontend"
        TAG   = "${BUILD_NUMBER}"
    }
    stages {
        stage('clone') {
            steps {
                git url: "https://github.com/abhishekakurhade/AWS-3tier-projects.git", branch: "main"
            }
        }
        stage('build image') {
            steps {
                sh '''docker build -f ./frontend/dockerfile -t $IMAGE:$TAG .'''
            }
        }
        stage('push on docker') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker_login', passwordVariable: 'docker_pass', usernameVariable: 'docker_user')]) {
                    sh '''
                        echo "$docker_pass" | docker login -u "$docker_user" --password-stdin
                        docker push $IMAGE:$TAG
                    '''
                }
            }
        }
        stage('deploy on eks') {
            steps {
                withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-login', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                        aws eks update-kubeconfig --region ap-south-1 --name my-cluster
                        sed -i "s|image: .*|image: ${IMAGE}:${TAG}|g" frontend/frontend-deploy.yml
                        kubectl apply -f frontend/frontend-deploy.yml -n default
                        kubectl apply -f frontend/frontend-svc.yml -n default
                        kubectl rollout status deployment -n default --timeout=120s
                    '''
                }
            }
        }
    }
}
```
 
### Pipeline summary
 
| Pipeline        | Image                                   | Manifests                                                |
|-----------------|-----------------------------------------|----------------------------------------------------------|
| K8s infra       | n/a                                     | Terraform files in repo root                             |
| Backend         | `abhishekakurhade/3tier-backend`        | `backend/backend-pod.yaml`, `backend/backend-svc.yaml`   |
| Frontend        | `abhishekakurhade/3tier-frontend`       | `frontend/frontend-deploy.yml`, `frontend/frontend-svc.yml` |
 
---
 
## Step 7: Verify the Deployment
 
On the k8s-agent (or any machine with AWS access):
 
```bash
# 1. Configure AWS
aws configure
 
# 2. Point kubectl at the EKS cluster
aws eks update-kubeconfig --region <region-name> --name <cluster-name>
 
# 3. List pods, deployments and services
kubectl get all
```
 
To get the public URL of the frontend (if exposed via a LoadBalancer service):
 
```bash
kubectl get svc
```
 

 
## Author
 
**Abhishek Kurhade**
Repository: <https://github.com/abhishekakurhade/AWS-3tier-projects>
 
