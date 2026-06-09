# 🍽️ Zomato Clone — DevSecOps CI/CD Pipeline

A React-based Zomato Clone application built from scratch and deployed using a fully automated DevSecOps CI/CD pipeline on AWS EC2. Implements security-first principles using Jenkins, SonarQube, Trivy, and Docker.

---

## 🏗️ Project Architecture

```
Developer Push → GitHub → Jenkins Pipeline →
SonarQube Analysis → Quality Gate →
Trivy File Scan → Docker Build →
Trivy Image Scan → Docker Push →
Update K8s Manifest ✅
```

---

## 🛠️ Tech Stack

| Category | Tools |
|----------|-------|
| Frontend | React.js |
| CI/CD | Jenkins |
| Code Quality | SonarQube |
| Security Scan | Trivy |
| Containerization | Docker |
| Container Registry | Docker Hub |
| Orchestration | Kubernetes (K8s Manifests) |
| Cloud | AWS EC2 (Ubuntu 26.04) |
| Version Control | Git, GitHub |

---

## 📁 Project Structure

```
Zomato-Clone/
├── public/
├── src/
│   ├── components/
│   │   ├── Navbar.js
│   │   ├── Navbar.css
│   │   ├── Hero.js
│   │   ├── Hero.css
│   │   ├── RestaurantList.js
│   │   ├── RestaurantList.css
│   │   ├── RestaurantCard.js
│   │   └── RestaurantCard.css
│   ├── App.js
│   ├── App.css
│   └── index.js
├── k8s/
│   └── deployment.yml
├── Dockerfile
├── Jenkinsfile
├── sonar-project.properties
├── package.json
└── README.md
```

---

## 🚀 Implementation Steps

### Phase 1 — AWS EC2 Setup

- Launched EC2 instance (Ubuntu 22.04, c7i-flex.large)
- Configured Security Group with the following ports:

| Port | Service |
|------|---------|
| 22 | SSH |
| 8080 | Jenkins |
| 9000 | SonarQube |
| 3000 | React App |
| 80 | HTTP |

---

### Phase 2 — Build React App from Scratch

Built the entire Zomato Clone React app from scratch on EC2:

```bash
npx create-react-app zomato-clone
cd zomato-clone
mkdir src/components
```

**Components Created:**
- `Navbar` — top navigation with Zomato branding
- `Hero` — search bar section with red background
- `RestaurantList` — filter buttons + restaurant grid
- `RestaurantCard` — individual restaurant card with emoji, rating, time, price

**Features:**
- Filter restaurants by category (All, Fast Food, Pizza, Indian, Healthy)
- Responsive 3-column grid layout
- Hover animation on cards
- Tested on `http://<EC2-IP>:3000` ✅

---

### Phase 3 — Install Tools on EC2

**Java 17:**
```bash
sudo apt update
sudo apt install -y openjdk-17-jdk
java -version
```

**Jenkins (via WAR file):**
```bash
wget https://get.jenkins.io/war-stable/latest/jenkins.war
java -jar jenkins.war --httpPort=8080 &
```

**Docker:**
```bash
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
sudo usermod -aG docker jenkins
sudo chmod 777 /var/run/docker.sock
```

**SonarQube (via Docker):**
```bash
docker run -d --name sonar \
  -p 9000:9000 \
  sonarqube:lts-community
```

**Trivy:**
```bash
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key \
  | sudo gpg --dearmor \
  | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] \
  https://aquasecurity.github.io/trivy-repo/deb generic main" \
  | sudo tee /etc/apt/sources.list.d/trivy.list

sudo apt update
sudo apt install trivy -y
trivy --version
```

---

### Phase 4 — Configure Jenkins

**Plugins Installed:**
- NodeJS
- SonarQube Scanner
- Docker
- Docker Pipeline
- CloudBees Docker Build and Publish
- Eclipse Temurin Installer

**Tools Configured (Manage Jenkins → Tools):**

| Tool | Name |
|------|------|
| JDK 17 | `jdk17` |
| NodeJS 18 | `node18` |
| SonarQube Scanner | `sonar-scanner` |
| Docker | `docker` |

**Credentials Added (Manage Jenkins → Credentials):**

| Credential | ID |
|------------|-----|
| DockerHub Username & Password | `docker-cred` |
| GitHub Personal Access Token | `git-cred` |
| SonarQube Token | `sonar-token` |

**SonarQube Server (Manage Jenkins → System):**
- Name: `sonar-server`
- URL: `http://<EC2-IP>:9000`
- Token: `sonar-token`

**SonarQube Webhook:**
- Go to SonarQube → Administration → Webhooks → Create
- URL: `http://<EC2-IP>:8080/sonarqube-webhook/`

---

### Phase 5 — Jenkins Pipeline

**Jenkinsfile Stages:**

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

**Pipeline Flow:**
1. Cleans Jenkins workspace before every build
2. Checks out latest code from GitHub
3. Runs SonarQube static code analysis on React source
4. Waits for SonarQube Quality Gate result via webhook
5. Installs npm dependencies
6. Trivy scans filesystem for vulnerabilities
7. Builds Docker image with build number tag
8. Trivy scans Docker image for HIGH/CRITICAL CVEs
9. Pushes verified image to Docker Hub
10. Updates K8s deployment manifest with new image tag and pushes to GitHub

---

### Phase 6 — Dockerfile

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
RUN npm install -g serve
EXPOSE 3000
CMD ["serve", "-s", "build"]
```

---

### Phase 7 — Kubernetes Manifests

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zomato-clone
spec:
  replicas: 2
  selector:
    matchLabels:
      app: zomato-clone
  template:
    metadata:
      labels:
        app: zomato-clone
    spec:
      containers:
      - name: zomato-clone
        image: <DOCKERHUB-USERNAME>/zomato-clone:latest
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: zomato-clone-svc
spec:
  type: LoadBalancer
  selector:
    app: zomato-clone
  ports:
  - port: 80
    targetPort: 3000
```

---

## 🔒 Security Highlights

- SonarQube enforces code quality gates before every deployment
- Trivy scans both filesystem and Docker images for CVEs
- Pipeline automatically fails on HIGH/CRITICAL vulnerabilities
- Docker images tagged with Jenkins build number for full traceability
- Security Groups restrict EC2 access to required ports only
- DockerHub and GitHub credentials managed via Jenkins Credentials Store

---

## 📸 Screenshots

> *(Add Jenkins pipeline success screenshot here)*
> *(Add SonarQube analysis result screenshot here)*
> *(Add Trivy scan report screenshot here)*

---

## 👨‍💻 Author

**Srikanth Sanjay Pawar**
AWS Certified Solutions Architect – Associate

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue)](https://linkedin.com/in/srikanth-pawar)
[![GitHub](https://img.shields.io/badge/GitHub-Follow-black)](https://github.com/Heyysri)
