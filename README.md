# DevOps Pipeline Task: Automating Deployment to AWS

## Overview

This guide provides a step-by-step solution** to containerize an application that uses **Nginx and a database**, and deploy it to **AWS**, ensuring scalability and automation using **Terraform**, **Docker**, and **Jenkins**.

---

## 📦 Step 1: Containerize the Application

### 1.1 Project Structure

```
project/
├── backend/
│   ├── app.py / index.js
│   ├── Dockerfile
├── nginx/
│   ├── nginx.conf
│   └── Dockerfile
├── docker-compose.yml
└── .env
```

### 1.2 Dockerfile for Backend

**backend/Dockerfile**

```Dockerfile
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 3000
CMD ["npm", "start"]
```

### 1.3 Dockerfile for Nginx

**nginx/Dockerfile**

```Dockerfile
FROM nginx:alpine
COPY nginx.conf /etc/nginx/nginx.conf
```

### 1.4 docker-compose.yml

```yaml
version: '3'
services:
  backend:
    build: ./backend
    ports:
      - "3000:3000"
    environment:
      - DB_HOST=mongodb
  nginx:
    build: ./nginx
    ports:
      - "80:80"
    depends_on:
      - backend
  mongodb:
    image: mongo:5.0
    ports:
      - "27017:27017"
```

### 1.5 Test Locally

```bash
docker-compose up --build
```

Check: [http://localhost](http://localhost)

---

## ☁️ Step 2: Provision Infrastructure on AWS with Terraform

### 2.1 Prerequisites

* Install AWS CLI
* Install Terraform
* Create a key pair in AWS EC2 console (e.g., `devops-key`)

### 2.2 Terraform File Structure

```
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── user-data.sh
```

### 2.3 main.tf

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "app_server" {
  ami = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  key_name = "devops-key"
  user_data = file("user-data.sh")

  tags = {
    Name = "AppServer"
  }

  security_groups = [aws_security_group.sg.id]
}

resource "aws_security_group" "sg" {
  name        = "app-sg"
  description = "Allow ports 22, 80, 3000"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 3000
    to_port     = 3000
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### 2.4 user-data.sh

```bash
#!/bin/bash
sudo apt update -y
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo docker run -d -p 80:80 nginx
```

### 2.5 Deploy with Terraform

```bash
cd terraform
terraform init
terraform apply -auto-approve
```

Copy the `public_ip` output and test in the browser.

---

## ⚙️ Step 3: Automate with Jenkins

### 3.1 Install Jenkins

Install Jenkins on EC2 or locally. Ensure these plugins are installed:

* Docker Pipeline
* Git
* Pipeline

### 3.2 Jenkins Pipeline (Jenkinsfile)

```groovy
pipeline {
  agent any
  stages {
    stage('Checkout') {
      steps {
        git 'https://github.com/<your-repo>/app.git'
      }
    }
    stage('Build Image') {
      steps {
        sh 'docker build -t app-backend ./backend'
        sh 'docker build -t app-nginx ./nginx'
      }
    }
    stage('Push to ECR') {
      steps {
        // Add AWS CLI and Docker login
      }
    }
    stage('Deploy') {
      steps {
        sshagent(['your-ssh-key']) {
          sh 'ssh -o StrictHostKeyChecking=no ubuntu@<EC2-PUBLIC-IP> "docker-compose up -d"'
        }
      }
    }
  }
}
```

---

## 📈 Step 4: Scalability with AWS ASG + Load Balancer (Optional)

### Use AWS Console or Terraform to:

* Create an AMI of your instance
* Launch an **Auto Scaling Group (ASG)** with this AMI
* Attach a **Load Balancer (ALB)** in front

---

## 📄 Deliverables Checklist

* [x] Dockerfiles and docker-compose.yml
* [x] Terraform infrastructure files
* [x] Jenkinsfile for CI/CD pipeline
* [x] Setup instructions and documentation
* [x] GitHub repository link with source code

---

✅ Congratulations! You’ve automated the deployment of a scalable web application using Docker, Jenkins, Terraform, and AWS.
