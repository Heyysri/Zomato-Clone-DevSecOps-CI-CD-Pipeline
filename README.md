# 🍽️ Zomato Clone — DevSecOps CI/CD Pipeline on AWS EKS

A Zomato Clone web application deployed using a fully automated DevSecOps CI/CD pipeline on AWS EKS with GitOps-based continuous delivery using Argo CD.

---

# 🏗️ Architecture & Project Structure

```
Code Push → GitHub → Jenkins →
SonarQube (Code Quality) → Quality Gate →
Trivy (File Scan) → Docker Build →
Trivy (Image Scan) → Docker Hub →
K8s Manifest Update → Argo CD →
AWS EKS Deployment ✅
```

---

# 🛠️ Tech Stack

| Layer | Technology |
|-------|------------|
| Application | React.js |
| CI/CD | Jenkins |
| Code Quality | SonarQube |
| Security Scanning | Trivy |
| Containerization | Docker |
| Container Registry | Docker Hub |
| Orchestration | Kubernetes, Amazon EKS |
| GitOps | Argo CD |
| Cloud | AWS (EC2, EKS, IAM, VPC) |
| OS | Ubuntu 26.04 LTS |

---

# ✨ Features

- Automated CI/CD pipeline triggered on every code push
- Static code analysis with SonarQube quality gate enforcement
- Container vulnerability scanning with Trivy (filesystem + image)
- Docker image auto-tagged with Jenkins build number
- GitOps-based deployment — Argo CD auto-syncs on manifest update
- Zero-downtime deployments on AWS EKS
- Kubernetes LoadBalancer service for public app access

---

# 🔒 Security Groups

| SG Name | Inbound Rules |
|---------|--------------|
| Boom | Port 22 (SSH), Port 8080 (Jenkins), Port 9000 (SonarQube), Port 443 (HTTPS), Port 80 (HTTP) |

---

# 🚀 Deployment Steps

## Prerequisites

- EC2 Key Pair
- GitHub Account with Personal Access Token
- DockerHub Account with Read/Write Token

---

## 1️⃣ Launch AWS EC2 Instance

- AMI: Ubuntu 26.04 LTS
- Instance Type: c7i-flex.large
- Storage: 30 GB
- Open Security Group ports: 22, 8080, 9000, 443, 80

---

## 2️⃣ Install Java 17

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk
java -version
```

---

## 3️⃣ Install Jenkins 

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt install jenkins
systemctl start jenkins
systemctl enable jenkins
systemctl status jenkins
```

Get admin password:

```bash
cat /var/jenkins_home/secrets/initialAdminPassword
```

Access Jenkins: `http://<EC2-PUBLIC-IP>:8080`

---

## 4️⃣ Install Docker

```bash
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
sudo usermod -aG docker jenkins
sudo chmod 777 /var/run/docker.sock
```

---

## 5️⃣ Install SonarQube

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

Access SonarQube: `http://<EC2-PUBLIC-IP>:9000`
- Default login: `admin / admin`
- Go to **My Account → Security → Generate Token** → copy token

---

## 6️⃣ Install Trivy

```bash
sudo apt-get install wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

---

## 7️⃣ Configure Jenkins

### Plugins to Install
- Stage View
- NodeJS
- SonarQube Scanner
- Docker & Docker Pipeline
- CloudBees Docker Build and Publish
- Eclipse Temurin Installer

### Tools (Manage Jenkins → Tools)

| Tool | Name |
|------|------|
| JDK 17 | `jdk17` |
| NodeJS 18 | `node18` |
| SonarQube Scanner | `sonar-scanner` |
| Docker | `docker` |

### Credentials (Manage Jenkins → Credentials → Global)

| ID | Kind | Purpose |
|----|------|---------|
| `docker-cred` | Username & Password | DockerHub Read/Write Token |
| `git-cred` | Username & Password | GitHub Personal Access Token |
| `sonar-token` | Secret Text | SonarQube Token |

### SonarQube Server (Manage Jenkins → System)
- Name: `sonar-server`
- URL: `http://<EC2-IP>:9000`
- Token: `sonar-token`

### SonarQube Webhook
- Go to SonarQube → **Administration → Webhooks → Create**
- Name: `jenkins`
- URL: `http://<EC2-IP>:8080/sonarqube-webhook/`

---

## 8️⃣ Clone Repo & Run Pipeline

```bash
git clone https://github.com/Heyysri/Zomato-Clone-DevSecOps-CI-CD-Pipeline.git
```

In Jenkins:
- New Item → Pipeline
- Pipeline script from SCM → Git
- Repo URL: `https://github.com/Heyysri/Zomato-Clone-DevSecOps-CI-CD-Pipeline.git`
- Branch: `main`
- Script Path: `Jenkinsfile`
- Click **Build Now**

### Pipeline Stages

```
✅ Clean Workspace
✅ Checkout Code
✅ SonarQube Analysis
✅ Quality Gate
✅ Install Dependencies
✅ Trivy File Scan
✅ Build Docker Image
✅ Trivy Image Scan
✅ Push Docker Image
✅ Update K8s Manifest
```

---

## 9️⃣ Create AWS EKS Cluster through AWS Terminal

```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
chmod +x kubectl
mkdir -p ~/.local/bin
mv ./kubectl ~/.local/bin/kubectl
kubectl version --client

# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version

# Install AWS CLI
sudo apt install unzip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure AWS CLI
aws configure

# Create cluster using eksctl
eksctl create cluster \
  --name eks-zomato \
  --region eu-north-1 \
  --version 1.31 \
  --nodegroup-name linux-nodes \
  --node-type c7i-flex.large  \
  --nodes 2

# Log in to Cluster
aws eks update-kubeconfig --name eks-zomato

```

---

## 🔟 Install & Configure Argo CD

```bash
# Create namespace
kubectl create namespace argocd

# Install Argo CD
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods
kubectl get pods -n argocd

# Expose UI
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "LoadBalancer"}}'

# Get external IP
kubectl get svc -n argocd

# Get admin password
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

Access Argo CD: `http://<ARGOCD-EXTERNAL-IP>`

### Argo CD Application Config

| Field | Value |
|-------|-------|
| App Name | `zomato-clone` |
| Project | `default` |
| Sync Policy | Automatic |
| Repo URL | `https://github.com/Heyysri/Zomato-Clone-DevSecOps-CI-CD-Pipeline.git` |
| Path | `k8s` |
| Cluster | `https://kubernetes.default.svc` |
| Namespace | `default` |

### Verify Deployment

```bash
kubectl get pods
kubectl get svc
```

Access app: `http://<EXTERNAL-IP>` ✅

---

# 📸 Screenshots

## EC2 Instance
> *(Add screenshot here)*

## Security Group
> *(Add screenshot here)*

## Jenkins Pipeline Success
> *(Add screenshot here)*

## Jenkins Console logs Success
> *(Add screenshot here)*

## SonarQube Quality Gate Passed
> *(Add screenshot here)*

## DockerHub Image Pushed
> *(Add screenshot here)*

## Argo CD Synced & Healthy
> *(Add screenshot here)*

## Live Application
> *(Add screenshot here)*

---

# 📂 Project Structure

```
Zomato-Clone-DevSecOps-CI-CD-Pipeline/
├── src/
├── public/
├── k8s/
├── Dockerfile
├── Jenkinsfile
├── sonar-project.properties
├── package.json
├── package-lock.json
├── .gitignore
├── README.md
```

---

# 🔮 Future Improvements

- Add Datadog monitoring on EKS
- Configure Terraform for full infrastructure provisioning
- Enable HTTPS using AWS ACM + Load Balancer
- Add email/Slack notifications on pipeline failure
- Implement Blue-Green deployment strategy
- Add Kubernetes HPA for auto-scaling

---

# 👤 Author

## Srikanth Sanjay Pawar

- LinkedIn: https://linkedin.com/in/srikanth-pawar
- GitHub: https://github.com/Heyysri
- Email: sreekanthsanjay5@gmail.com

---

# ⭐ Project Highlights

- End-to-end DevSecOps CI/CD pipeline
- Security-first approach with SonarQube + Trivy
- GitOps deployment via Argo CD
- AWS EKS container orchestration
- Docker image versioning with build tags
- Kubernetes LoadBalancer service
- Production-style cloud deployment
