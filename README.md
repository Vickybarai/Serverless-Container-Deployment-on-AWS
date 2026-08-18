
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
cd node-todo-cicd

# Build the Docker image
docker build -t node-todo-app .
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
1. Go to the AWS Console ➔ **ECS (Elastic Container Service)**.
2. **Create Cluster:**
   * Click "Create Cluster" ➔ Select "Networking only" (Fargate) ➔ Name it `todo-cluster` ➔ Create.
3. **Create Task Definition:**
   * Click "Task Definitions" ➔ "Create new Task Definition" ➔ **Fargate**.
   * Task Definition Name: `node-todo-task`
   * Task Role: None
   * Task Execution Role: `ecsTaskExecutionRole` (AWS creates this automatically)
   * Task Memory: `0.5 GB (512)`
   * Task CPU: `0.25 vCPU (256)`
   * **Add Container:**
     * Container name: `node-todo-container`
     * Image URI: Paste your ECR image URI (`<AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/node-todo-app:latest`)
     * Port mappings: **8000** tcp
     * Scroll down to **LOGS** ➔ Select "Auto-configure CloudWatch Logs" ➔ Log prefix: `ecs-todo`
   * Click **Create**.
4. **Run the Task:**
   * Go to your `todo-cluster` ➔ Click **Tasks** ➔ **Run new Task**.
   * Select the Task Definition you just created (`node-todo-task`).
   * **Networking:** Select your default VPC and two Subnets.
   * **Security Group:** Create a new one, and **ADD A RULE allowing All Traffic / TCP / Port 8000 from Anywhere (0.0.0.0/0)**.
   * Click **Run Task**.

### Step 7: Verify & Check Logs
1. Wait a minute for the task status to change to **Running**.
2. Click on the Task ID, scroll down to the **Networking** section, and copy the **Public IP**.
3. Open a new browser tab and type: `http://<PUBLIC_IP>:8000`
4. **Check Logs:** Go to AWS CloudWatch ➔ Logs ➔ Log groups ➔ `/ecs/ecs-todo` to see your application logs.

---

## 🆘 Common Troubleshooting
* **Page won't load (Timeout):** You forgot to open Port `8000` in the ECS Task Security Group.
* **Task keeps crashing/stopping:** Check CloudWatch logs. Usually means the app crashed on startup or the port mapping in the Task Definition is wrong.
* **Docker push fails:** Ensure your `aws configure` credentials are correct and the ECR image URI has no typos.

---


## 📝 Resume Bullet Point
> "Containerized a Node.js application using Docker, pushed images to AWS ECR, and deployed the application on serverless AWS ECS Fargate. Configured VPC networking, security groups, and integrated CloudWatch for centralized log monitoring."
