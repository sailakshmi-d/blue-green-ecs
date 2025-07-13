#  ECS Fargate Blue/Green Deployment using AWS CodePipeline

##  Objective

Deploy a containerized Node.js application using **Amazon ECS Fargate** with **Blue/Green deployment strategy** via **AWS CodePipeline**, **CodeBuild**, and **CodeDeploy**.

---

##  AWS Resources Used

- Amazon ECS Cluster (Fargate)
- Task Definitions & ECS Services with `CODE_DEPLOY`
- Elastic Container Registry (ECR)
- Application Load Balancer (ALB) with Blue/Green Target Groups
- AWS CodePipeline (CI/CD)
- AWS CodeBuild (Build)
- AWS CodeDeploy (Deployment with Blue/Green)
- IAM Roles


---

##  How Blue/Green Deployment Works

1. ECS Service is created with deployment controller set to `CODE_DEPLOY`.
2. CodePipeline updates the ECS Task Definition with the new image.
3. CodeDeploy shifts traffic between Blue (old) and Green (new) targets via ALB.
4. ALB target groups manage routing for each version.
5. Zero-downtime deployment using weighted traffic shifting.

---

## Project Structure

```
bluegreen-ecs/
│
├── Dockerfile
├── buildspec.yml
├── appspec.yaml
├── taskdef.json
├── server.js
└── README.md
```

---

##  Step-by-Step Deployment Guide

###  Prerequisites

- An AWS Account
- An IAM user/role with admin privileges
- GitHub repo (or other Git provider) connected to AWS CodePipeline

---

### Step 1: Create an ECR Repository

1. Go to **Amazon ECR** → Click **Create Repository**
2. Name: `bluegreen`
3. Save the **ECR URI** (e.g., `123456789012.dkr.ecr.us-east-1.amazonaws.com/bluegreen`)

---

### Step 2: Build & Push Docker Image

1. SSH into your EC2 instance (or local terminal with Docker installed)

```bash
aws ecr get-login-password --region us-east-1 | \
docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

docker build -t bluegreen .
docker tag bluegreen:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/bluegreen:latest
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/bluegreen:latest
```

---

###  Step 3: Set Up the Load Balancer

1. Go to **EC2 → Target Groups**
2. Create two target groups: `blue-target` and `green-target`
3. Go to **Load Balancers → Create ALB**
   - Choose Application Load Balancer
   - Set listeners for port **8080**
   - Register the two target groups
   - Add listener rules to forward traffic accordingly

---

###  Step 4: Create ECS Cluster

- ECS → Clusters → **Create Cluster**
  - Choose **Networking only (Fargate)**
  - Name: `bluegreen-cluster`

---

###  Step 5: Register ECS Task Definition

1. ECS → Task Definitions → Create New
2. Launch type: FARGATE
3. Set CPU: `256`, Memory: `512`
4. Container:
   - Image: `<your_ecr_uri>:latest`
   - Port: `8080`
5. Save task definition (e.g., `bluegreen-taskdef:1`)

---

###  Step 6: Create ECS Service with Blue/Green

- ECS → Clusters → Select Cluster → Create Service
  - Launch Type: FARGATE
  - Task Definition: `bluegreen-taskdef`
  - Service Name: `bluegreen-service`
  - Deployment Type: **Blue/Green (CodeDeploy)**
  - Select ALB and both target groups

---

###  Step 7: Create CodeBuild Project

- CodeBuild → Create build project:
  - Connect GitHub repo
  - Use `buildspec.yml` (see below)
  - Set environment to use Docker runtime
  - Grant necessary IAM permissions



###  Step 8: Create CodeDeploy ECS Application

1. Go to **CodeDeploy → Applications → Create**
   - Platform: Amazon ECS
   - Name: `bluegreen-app`

2. Create Deployment Group:
   - ECS Cluster: `bluegreen-cluster`
   - Service Name: `bluegreen-service`
   - Target Groups: `blue`, `green`
   - Load Balancer: Select ALB created earlier
   - Traffic shifting: Enable Blue/Green with automatic rollout

---

###  Step 9: Create CodePipeline

1. CodePipeline → Create New Pipeline
   - Source: GitHub repo
   - Build: CodeBuild project
   - Deploy: CodeDeploy ECS application
2. Make sure pipeline includes artifacts:
   - `imagedefinitions.json`
   - `appspec.yaml` (see below)


```

---

##  Redeployment Flow

1. Change the source code in `server.js`
2. Commit and push to GitHub
3. Pipeline triggers automatically:
   - Builds new Docker image
   - Pushes to ECR
   - Updates ECS Task Definition
   - Triggers CodeDeploy with Blue/Green rollout

---

##  Validate Deployment

- Get the **DNS name of the Load Balancer**
- Open it in the browser to access the deployed app
- On new commits, check version gets updated via CodeDeploy

---

##  IAM Roles Required

- **CodeBuild Role**: ECR push, ECS task definition access
- **CodeDeploy Role**: ECS service update and ALB access
- **ECS Execution Role**: Pull image from ECR, write logs

---


---

##  References

- [Amazon ECS Blue/Green Deployments](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployments-create-blue-green.html)
- [ECS with Fargate](https://docs.aws.amazon.com/AmazonECS/latest/userguide/what-is-fargate.html)
- [CodePipeline with ECS](https://docs.aws.amazon.com/codepipeline/latest/userguide/ecs-cd-pipeline.html)
