
# 🚀 Node.js Todo App on AWS ECS Fargate

A simple Node.js Todo application containerized with Docker and deployed on AWS ECS (Fargate) using EC2 as a build server. This project uses ECR for image storage and CloudWatch for logging.

---

## 🏗️ Project Architecture
```text
GitHub → EC2 (Build Machine) → Docker Image → ECR (Image Storage) → ECS Fargate (Hosting) → CloudWatch (Logs)
```

---

## 🛠️ Step-by-Step Setup Guide

### Step 1: Launch an EC2 Instance (Ubuntu)
1. Go to the AWS Console and navigate to **EC2**.
2. Click **Launch Instance**.
3. Select **Ubuntu** as the OS.
4. Choose an instance type (e.g., `t2.micro` for free tier).
5. Create or select a Security Group that allows **SSH (Port 22)** from your IP.
6. Launch the instance and connect to it using your `.pem` key:
   ```bash
   ssh -i "your-key.pem" ubuntu@<your-ec2-public-ip>
   ```

### Step 2: Install Docker & AWS CLI on EC2
Once connected to your EC2 terminal, run the following commands:
```bash
# Update packages
sudo apt-get update -y

# Install Docker
sudo apt-get install docker.io -y

# Start Docker and enable it to start on boot
sudo systemctl start docker
sudo systemctl enable docker

# Add ubuntu user to docker group (so you don't need 'sudo' for docker commands)
sudo usermod -aG docker ubuntu

# Install AWS CLI
sudo apt-get install awscli -y
```
⚠️ **Important:** Log out and log back into SSH for the Docker group changes to take effect.

### Step 3: Configure AWS Credentials
Your EC2 instance needs permission to talk to AWS (to push images to ECR).
1. Go to AWS IAM and create a User with **Programmatic Access**.
2. Attach the `AmazonEC2ContainerRegistryPowerUser` policy to this user.
3. Take note of the **Access Key ID** and **Secret Access Key**.
4. Run this command on your EC2 terminal:
   ```bash
   aws configure
   ```
   * Paste your Access Key ID
   * Paste your Secret Access Key
   * Region: `us-east-1` (or your chosen region)
   * Output format: `json`

### Step 4: Clone Code & Build Docker Image
```bash
# Clone the repository
git clone https://github.com/Vickybarai/serverless-ecs-ecr.git

# Go into the project folder
cd serverless-ecs-ecr

# Build the Docker image
```

### Step 5: Create ECR Repository & Push Image
1. Go to the AWS Console ➔ **Elastic Container Registry (ECR)**.
2. Click **Create repository**.
3. Name it `node-todo-app` and click Create.

Now, go back to your EC2 terminal to push the image (Replace `<AWS_ACCOUNT_ID>` and `<REGION>` with your actual details, e.g., `123456789012` and `us-east-1`):
```bash
# Log into ECR
aws ecr get-login-password --region <REGION> | docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com

# Tag your image with the ECR URI
docker tag node-todo-app:latest <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/node-todo-app:latest

# Push the image to ECR
docker push <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/node-todo-app:latest
```

### Step 6: Deploy to ECS Fargate

#### 6.1 Create ECS Cluster
1. Go to the AWS Console ➔ **ECS (Elastic Container Service)**.
2. Click **Create Cluster**.
3. Select **Networking only** (Fargate) as the cluster template.
4. Configure cluster settings:
   - **Cluster name**: `node-app-cluster`
   - **Infrastructure**: AWS Fargate (serverless)
   - **Monitoring**: Enable Container Insights for CloudWatch logs
5. Click **Create**.

#### 6.2 Create Task Definition
1. In the ECS console, click **Task Definitions** → **Create new Task Definition**.
2. Select **Fargate** as the launch type compatibility.
3. Configure task settings:
   - **Task Definition Name**: `node-to-do-app-task-definition`
   - **Infrastructure**: AWS Fargate
   - **Architecture**: Linux/X86_64
   - **Task Size**: 
     - CPU: `2 vCPU` (2048 CPU units)
     - Memory: `8 GB` (8192 MB)
   - **Task Role**: Select `ecsTaskExecutionRole`【turn0search0】【turn0search11】【turn0search13】
   - **Network Mode**: `awsvpc` (required for Fargate)【turn0search0】【turn0search5】

4. Add container definition:
   - **Container name**: `node-container`
   - **Image URI**: Paste your ECR image URI (`<AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/node-todo-app:latest`)
   - **Port mappings**: 
     - Container port: `8000`
     - Protocol: `tcp`
   - **Resource Limits**:
     - CPU: `2 vCPU` (2048 CPU units)
     - Memory: `8 GB` (8192 MB)
   - **Log Configuration**:
     - Log driver: `awslogs`
     - Options:
       - awslogs-group: `/ecs/node-todo`
       - awslogs-region: `<REGION>`
       - awslogs-stream-prefix: `ecs`

5. Click **Create**.

#### 6.3 Run the Task
1. Go to your `node-app-cluster` → **Tasks** → **Run new Task**.
2. Configure task settings:
   - **Launch Type**: Fargate
   - **Cluster**: `node-app-cluster`
   - **Task Definition**: Select `node-to-do-app-task-definition`
   - **Network Configuration**:
     - VPC: Select your default VPC
     - Subnets: Select 2 public subnets (for public IP assignment)【turn0search5】【turn0search9】
     - Security Groups: Create a new security group with:
       - Inbound rule: Allow TCP port `8000` from anywhere (0.0.0.0/0)
       - Outbound rule: Allow all traffic
   - **Auto-assign public IP**: ENABLED (required for public access)【turn0search5】【turn0search9】

3. Click **Run Task**.
4. Wait for the task status to change from `PROVISIONING` to `RUNNING` (typically 1-2 minutes).

#### 6.4 Access the Application
1. Click on the **Task ID** in the tasks list.
2. Scroll down to the **Networking** section.
3. Copy the **Public IP** address.
4. Open a web browser and navigate to: `http://<PUBLIC_IP>:8000`

### Step 7: Verify & Check Logs
1. **Access the Application**: Verify the Todo app loads correctly in your browser.
2. **Check CloudWatch Logs**:
   - Go to AWS CloudWatch → **Log groups**
   - Select `/ecs/node-todo`
   - Click on the log stream (starting with `ecs/`)
   - Verify application logs are streaming correctly

## 🧹 Resource Cleanup (Important!)

To avoid ongoing AWS charges, clean up all resources when you're done:

### 1. Stop and Delete ECS Task
```bash
# List running tasks
aws ecs list-tasks --cluster node-app-cluster --region <REGION>

# Stop the task (replace <TASK_ID> with actual task ID)
aws ecs stop-task --cluster node-app-cluster --task <TASK_ID> --region <REGION>
```

### 2. Delete ECS Service and Cluster
```bash
# Delete the ECS cluster (this will delete all services and tasks)
aws ecs delete-cluster --cluster node-app-cluster --region <REGION>
```

### 3. Deregister Task Definition
```bash
# List task definition revisions
aws ecs list-task-definitions --family node-to-do-app-task-definition --region <REGION>

# Deregister the specific revision (replace <REVISION> with actual revision number)
aws ecs deregister-task-definition --task-definition node-to-do-app-task-definition:<REVISION> --region <REGION>
```

### 4. Delete ECR Repository
```bash
# Delete the ECR repository (this will delete all images)
aws ecr delete-repository --repository-name node-todo-app --region <REGION> --force
```

### 5. Delete CloudWatch Log Group
```bash
# Delete the CloudWatch log group
aws logs delete-log-group --log-group-name /ecs/node-todo --region <REGION>
```

### 6. Delete Security Group
```bash
# Delete the security group (replace <SECURITY_GROUP_ID> with actual ID)
aws ec2 delete-security-group --group-id <SECURITY_GROUP_ID> --region <REGION>
```

### 7. Terminate EC2 Instance
```bash
# Terminate the EC2 instance (replace <INSTANCE_ID> with actual instance ID)
aws ec2 terminate-instances --instance-ids <INSTANCE_ID> --region <REGION>
```

## 📊 Resource Summary

| Resource | Name | Purpose |
|----------|------|---------|
| **EC2 Instance** | `ubuntu` | Build server for Docker image |
| **ECR Repository** | `node-todo-app` | Stores Docker container images |
| **ECS Cluster** | `node-app-cluster` | Manages Fargate tasks |
| **Task Definition** | `node-to-do-app-task-definition` | Template for container tasks |
| **Security Group** | `ecs-sg` | Controls network access |
| **CloudWatch Logs** | `/ecs/node-todo` | Centralized application logs |

## 🔧 Configuration Details

### Task Size Considerations
- **Video Configuration**: 2 vCPU, 8 GB memory (for production workloads)
- **Free Tier Alternative**: 0.25 vCPU (256 CPU units), 0.5 GB (512 MB) memory【turn0search0】【turn0search1】
- **CPU/Memory Validation**: Fargate requires specific CPU/memory combinations【turn0search0】【turn0search4】

### Network Configuration
- **Public IP Assignment**: Required for direct task access【turn0search5】【turn0search9】
- **Security Groups**: Must allow inbound traffic on port 8000
- **VPC Flow Logs**: Enabled for monitoring network traffic【turn0search5】

### IAM Roles
- **Task Execution Role**: `ecsTaskExecutionRole` (required for ECR pulls and CloudWatch logging)【turn0search0】【turn0search11】【turn0search13】
- **Task Role**: Optional, for application AWS API calls (not used in this basic setup)

## 🆘 Common Troubleshooting

<details>
<summary>🔍 Common Issues and Solutions</summary>

### **Task Fails to Start**
```bash
# Check task events
aws ecs describe-tasks --cluster node-app-cluster --tasks <TASK_ID> --region <REGION>

# Common causes:
# 1. Invalid CPU/memory configuration
# 2. Missing IAM permissions for task execution role
# 3. Security group blocking outbound traffic
```

### **Cannot Access Application**
```bash
# Verify security group allows port 8000
aws ec2 describe-security-groups --group-ids <SECURITY_GROUP_ID> --region <REGION>

# Check task public IP assignment
aws ecs describe-tasks --cluster node-app-cluster --tasks <TASK_ID> --region <REGION> --query 'tasks[0].attachments[0].details'
```

### **Image Pull Failure**
```bash
# Verify ECR permissions
aws ecr get-authorization-token --region <REGION> --output text --query 'authorizationData[].authorizationToken'

# Check if image exists in ECR
aws ecr describe-images --repository-name node-todo-app --region <REGION>
```

### **CloudWatch Logs Not Appearing**
```bash
# Verify log group exists
aws logs describe-log-groups --log-group-name-prefix /ecs/node-todo --region <REGION>

# Check task execution role permissions
aws iam get-role-policy --role-name ecsTaskExecutionRole --policy-name ECS-Logs-Policy --region <REGION>
```

</details>

## 📝 Resume Bullet Point
> "Containerized a Node.js application using Docker, pushed images to AWS ECR, and deployed the application on serverless AWS ECS Fargate. Configured VPC networking, security groups, and integrated CloudWatch for centralized log monitoring."

## 🎯 Key Learning Outcomes

1. **ECS Fargate Fundamentals**: Serverless container orchestration without managing EC2 instances
2. **Task Definitions**: Configuring CPU, memory, and container settings for Fargate tasks【turn0search0】【turn0search1】
3. **Network Configuration**: Understanding public IP assignment and security groups for Fargate tasks【turn0search5】【turn0search9】
4. **IAM Roles**: Differentiating between task execution roles and task roles【turn0search11】【turn0search13】
5. **Monitoring**: Centralized logging with CloudWatch Logs
6. **Cost Management**: Proper cleanup of AWS resources to avoid charges
