# StreamingApp

Stream premium video content, host live watch parties, and manage your catalogue with a modern microservice architecture. The platform now ships with a production-ready admin portal, real-time chat, S3-backed adaptive streaming, and a redesigned cinematic frontend experience.

## Architecture

| Service | Port | Description |
| --- | --- | --- |
| `authService` | 3001 | User authentication, registration, JWT issuance |
| `streamingService` | 3002 | Video catalogue, S3 playback endpoints, public APIs |
| `adminService` | 3003 | Dedicated admin microservice for asset management and uploads |
| `chatService` | 3004 | Websocket + REST chat for live watch parties |
| `frontend` | 3000 | React SPA with revamped UI and integrated chat |
| `mongo` | 27017 | Shared MongoDB instance |

All backend services share common database models and utilities through `backend/common`.

## Environment Configuration

Create an `.env` for each service (or export variables before running). All services accept the standard AWS credentials for S3 access.

### Auth Service (`backend/authService/.env`)
```ini
PORT=3001
MONGO_URI=mongodb://localhost:27017/streamingapp
JWT_SECRET=changeme
CLIENT_URLS=http://localhost:3000
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=ap-south-1
AWS_S3_BUCKET=
```

### Streaming Service (`backend/streamingService/.env`)
```ini
PORT=3002
MONGO_URI=mongodb://localhost:27017/streamingapp
JWT_SECRET=changeme
CLIENT_URLS=http://localhost:3000
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=ap-south-1
AWS_S3_BUCKET=
AWS_CDN_URL=
STREAMING_PUBLIC_URL=http://localhost:3002
```

### Admin Service (`backend/adminService/.env`)
```ini
PORT=3003
MONGO_URI=mongodb://localhost:27017/streamingapp
JWT_SECRET=changeme
CLIENT_URLS=http://localhost:3000
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=ap-south-1
AWS_S3_BUCKET=
```

### Chat Service (`backend/chatService/.env`)
```ini
PORT=3004
MONGO_URI=mongodb://localhost:27017/streamingapp
JWT_SECRET=changeme
CLIENT_URLS=http://localhost:3000
```

### Frontend build variables (`frontend/.env` or Docker build args)
```ini
REACT_APP_AUTH_API_URL=http://localhost:3001/api
REACT_APP_STREAMING_API_URL=http://localhost:3002/api
REACT_APP_STREAMING_PUBLIC_URL=http://localhost:3002
REACT_APP_ADMIN_API_URL=http://localhost:3003/api/admin
REACT_APP_CHAT_API_URL=http://localhost:3004/api/chat
REACT_APP_CHAT_SOCKET_URL=http://localhost:3004
```

## Running with Docker Compose

1. Populate the environment variables above (or rely on the defaults baked into `docker-compose.yml`).
2. Build and start the stack:
   ```bash
   docker-compose up --build
   ```
3. Navigate to `http://localhost:3000` for the web app.

The compose file provisions MongoDB plus all four Node.js microservices. S3 credentials are optional for local testing—you can still browse seeded metadata, but streaming requires valid S3 objects.

## Local Development

Install dependencies for each service:

```bash
# auth service
cd backend/authService && npm install

# streaming service
cd ../streamingService && npm install

# admin service
cd ../adminService && npm install

# chat service
cd ../chatService && npm install

# frontend
cd ../../frontend && npm install
```

Run the services (in separate terminals) after starting MongoDB:

```bash
cd backend/authService && npm run dev
cd backend/streamingService && npm run dev
cd backend/adminService && npm run dev
cd backend/chatService && npm run dev
cd frontend && npm start
```

## Feature Highlights

- **S3-backed adaptive streaming** with secure signed uploads for admins.
- **Dedicated admin microservice** for video ingestion, metadata management, and featured curation.
- **Real-time chat** overlay in the player (Socket.IO + persistent message history).
- **Modern React experience** featuring cinematic hero sections, dynamic carousels, and responsive design.
- **Role-aware access control** across frontend routes and backend microservices.

## Testing

Automated tests are not yet included. Recommended smoke checks:

1. Register and log in through the web UI.
2. Upload a small video + thumbnail via the admin dashboard (requires valid S3 credentials).
3. Confirm playback from the browse page and verify that chat messages broadcast between multiple browser tabs.

Here is a clean, well-structured **`README.md` Setup Guide** based exactly on what you wrote, formatted and ready to drop into your repository.

---

# Streaming App – Setup Guide

This document describes the **exact steps I followed to set up the entire project from scratch**, including local tooling, AWS infrastructure, CI with Jenkins, and deployment to Kubernetes using Helm.

---

## Tools Installed

* **Git** (already installed)
* **Docker Desktop** – for building container images
* **AWS CLI v2** – to interact with Amazon Web Services from the terminal
* **kubectl** – to manage the Kubernetes cluster
* **eksctl** – simplifies creation and management of EKS clusters
* **Helm** – for packaging and deploying Kubernetes applications
---

## Getting the Code

```bash
git clone https://github.com/sannnn1234/StreamingApp.git
cd StreamingApp
git remote add upstream https://github.com/UnpredictablePrashant/StreamingApp.git
```
The `upstream` remote allows pulling updates from the original repository later:

```bash
git fetch upstream
git merge upstream/main
```

---

## AWS Configuration

Configured AWS credentials:

```bash
aws configure
```

Values used:

* Region: `ap-south-1` (Mumbai)
* Output format: `json`

Verified configuration:

```bash
aws sts get-caller-identity
```

---

## Creating ECR Repositories

Each microservice uses its own **ECR repository**:

```bash
aws ecr create-repository --repository-name streaming-frontend --region ap-south-1
aws ecr create-repository --repository-name streaming-auth --region ap-south-1
aws ecr create-repository --repository-name streaming-streaming --region ap-south-1
aws ecr create-repository --repository-name streaming-admin --region ap-south-1
aws ecr create-repository --repository-name streaming-chat --region ap-south-1
```

## AWS ECR
![Local Setup](https://raw.githubusercontent.com/sannnn1234/StreamingApp/main/document/ecr.png)

---

---

## Building and Pushing Docker Images

### Login to ECR

```bash
aws ecr get-login-password --region ap-south-1 \
| docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com
```

### Build Images

Dockerfiles were already present in the repository.

```bash
docker build -t streaming-frontend ./frontend
docker build -t streaming-auth ./backend/authService
docker build -t streaming-streaming -f ./backend/streamingService/Dockerfile ./backend
docker build -t streaming-admin -f ./backend/adminService/Dockerfile ./backend
docker build -t streaming-chat -f ./backend/chatService/Dockerfile ./backend
```

> **Note:** Streaming, Admin, and Chat services require `./backend` as the build context because their Dockerfiles reference shared backend files.

### Tag and Push Images

```bash
ECR=<ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com

@services = {
   'frontend',
   'auth',
   'streaming',
   'admin',
   'chat'
}
foreach ($svc in $services) {
    Write-Host "Processing service: $svc"
    $localImage  = "streaming-$svc:latest"
    $remoteImage = "$ECR/streaming-$svc:latest"
    docker tag $localImage $remoteImage
    docker push $remoteImage
}

```

---

## Jenkins Setup

I launched a **t2.medium EC2 instance** using **Amazon Linux 2023**.

![Local Setup](https://raw.githubusercontent.com/sannnn1234/StreamingApp/main/document/ec2.png)

---
---
### Security Group

* Opened **port 8080** for Jenkins UI access

### Install Dependencies

```bash
sudo yum install -y java-21-amazon-corretto docker git
sudo systemctl start docker
sudo systemctl enable docker
```

### Install Jenkins

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum install -y jenkins
sudo usermod -aG docker jenkins
sudo systemctl start jenkins
```

> **Important:** Jenkins `2.555` requires **Java 21**. Jenkins failed to start with Java 17 until Java 21 was installed and `JAVA_HOME` was set correctly.

### Install AWS CLI on Jenkins EC2

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### Jenkins Configuration

In Jenkins UI, I installed these plugins:

* Pipeline
* Git
* Docker Pipeline
* GitHub Integration

Added AWS credentials:

* **Type:** Secret Text
* **IDs:**

  * `aws-access-key-id`
  * `aws-secret-access-key`

---

## EKS Cluster Setup

Created the Kubernetes cluster using `eksctl`:

```bash
eksctl create cluster \
  --name streaming-app-cluster \
  --region ap-south-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```

## EKS CLUSTER
![Local Setup](https://raw.githubusercontent.com/sannnn1234/StreamingApp/main/document/cluster.png)

---
### Configure kubectl

```bash
aws eks update-kubeconfig --name streaming-app-cluster --region ap-south-1
kubectl get nodes
```
---

## Deploying with Helm

Deployed the application using Helm:

```bash
helm install streaming-app ./helm/streaming-app
```

Verify deployment:

```bash
kubectl get pods
kubectl get svc
```
## PODS
![Local Setup](https://raw.githubusercontent.com/sannnn1234/StreamingApp/main/document/pods.png)

* The **frontend-service** exposes an **EXTERNAL-IP**
* This LoadBalancer URL is used to access the application

---

# Monitoring and Logging
I set up monitoring using Amazon CloudWatch since the app runs on Amazon Web Services

### Container Insights
I enabled the CloudWatch Observability addon on the EKS cluster. This automatically collects metrics like:

* CPU usage
* Memory usage
* Network traffic
* Disk usage
* Pod restart counts

from all the nodes and pods.

To view these metrics:

```bash
AWS Console -> CloudWatch -> Container Insights -> Select the cluster
```
---

![Local Setup](https://raw.githubusercontent.com/sannnn1234/StreamingApp/main/document/cloudwatch.png)


### Control Plane Logs
I enabled logging for the EKS control plane. This helps capture cluster-level activities such as:
* API requests
* Authentication events
* Scheduler decisions
* Controller manager actions

These logs appear in CloudWatch under the log group:
```bash
/aws/eks/streaming-app-cluster/cluster
```
I enabled all five control plane log types:
* API server logs
* Audit logs
* Authenticator logs
* Controller manager logs
* Scheduler logs
---

### Alarms
I configured two CloudWatch alarms:
#### EKS-HighCPU-StreamingApp
Triggers when average CPU usage exceeds **80% for 10 minutes**.

#### EKS-HighMemory-StreamingApp
Triggers when average memory usage exceeds **80% for 10 minutes**.
These alarms can later be integrated with:
* Email notifications using SNS
* Slack alerts
* PagerDuty or other incident tools

---
![Local Setup](https://raw.githubusercontent.com/sannnn1234/StreamingApp/main/document/alarm.png)

## SNS Notification alert
![Local Setup](https://raw.githubusercontent.com/sannnn1234/StreamingApp/main/document/sns.png)

## Checking Logs from Terminal
Used `kubectl` commands for quick debugging.

### View logs from a pod
```bash
kubectl logs <pod-name>
```

### Stream logs live

```bash
kubectl logs -f <pod-name>
```

### View logs for all frontend pods
```bash
kubectl logs -l app=frontend
```

---

## Checking Resource Usage

### Node-level CPU and memory usage
```bash
kubectl top nodes
```

### Pod-level resource usage
```bash
kubectl top pods
```

### View recent cluster events
Helpful when pods fail to start.

```bash
kubectl get events --sort-by='.lastTimestamp'
```

---

## Troubleshooting

### Pod not starting

Check pod events and detailed error messages:

```bash
kubectl describe pod <pod-name>
```

---

### Application returning errors
Check container logs:

```bash
kubectl logs <pod-name>
```

---

### Unable to access the application
Verify the service has an external LoadBalancer IP:

```bash
kubectl get svc
```

---
### Image pull errors
Usually caused by:
* Expired ECR login session
* Incorrect image URI in `values.yaml`
* Missing image pull permissions
----

## Running Application
![Local Setup](https://raw.githubusercontent.com/sannnn1234/StreamingApp/main/document/application.png)

## Cleanup (Important)

To avoid unnecessary AWS charges, delete all resources after use:

```bash
helm uninstall streaming-app
eksctl delete cluster --name streaming-app-cluster --region ap-south-1
```

