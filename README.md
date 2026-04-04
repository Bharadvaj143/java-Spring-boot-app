# 🚀 Ultimate CI/CD Pipeline: Jenkins + ArgoCD + Kubernetes

> End-to-End CI/CD pipeline using Jenkins, SonarQube, Docker, ArgoCD and Kubernetes — based on [Abhishek Veeramalla's](https://www.youtube.com/@AbhishekVeeramalla) project.

---

## 📋 Table of Contents

- [Architecture Overview](#-architecture-overview)
- [Tech Stack](#-tech-stack)
- [Prerequisites](#-prerequisites)
- [Part 1: AWS EC2 Setup](#-part-1-aws-ec2-setup)
- [Part 2: Jenkins Installation](#-part-2-jenkins-installation)
- [Part 3: Docker Installation](#-part-3-docker-installation)
- [Part 4: SonarQube Installation](#-part-4-sonarqube-installation)
- [Part 5: Jenkins Configuration](#-part-5-jenkins-configuration)
- [Part 6: Minikube Setup (WSL2)](#-part-6-minikube-setup-wsl2)
- [Part 7: ArgoCD Installation](#-part-7-argocd-installation)
- [Part 8: Pipeline Flow](#-part-8-pipeline-flow)

---

## 🏗️ Architecture Overview

```
Developer pushes code to GitHub
            │
            ▼
    ┌───────────────┐
    │    Jenkins    │  ◄── Webhook trigger
    │   (CI Server) │
    └──────┬────────┘
           │
    ┌──────▼────────┐
    │  Maven Build  │  ◄── Compile + Unit Test
    └──────┬────────┘
           │
    ┌──────▼────────┐
    │  SonarQube    │  ◄── Static Code Analysis
    └──────┬────────┘
           │
    ┌──────▼────────┐
    │ Docker Build  │  ◄── Build + Push to DockerHub
    │   & Push      │
    └──────┬────────┘
           │
    ┌──────▼────────┐
    │  Update Git   │  ◄── Update image tag in deployment.yaml
    │  Manifest     │
    └──────┬────────┘
           │
    ┌──────▼────────┐
    │    ArgoCD     │  ◄── Detects Git change (GitOps)
    │  (CD Server)  │
    └──────┬────────┘
           │
    ┌──────▼────────┐
    │  Kubernetes   │  ◄── Deploys updated application
    │   Cluster     │
    └───────────────┘
```

---

## 🛠️ Tech Stack

| Tool | Purpose |
|------|---------|
| **Jenkins** | CI Server — Build, Test, Push |
| **Maven** | Build tool for Java |
| **SonarQube** | Static Code Analysis |
| **Docker** | Containerization |
| **DockerHub** | Container Registry |
| **ArgoCD** | GitOps CD tool |
| **Kubernetes / Minikube** | Container Orchestration |
| **AWS EC2** | Cloud hosting |

---

## ✅ Prerequisites

- AWS Account with EC2 access
- DockerHub account
- GitHub account (fork the repo)
- WSL2 installed on Windows (for local Kubernetes)
- Minimum EC2: **t2.large** (2 CPU, 8GB RAM)

---

## ☁️ Part 1: AWS EC2 Setup

### Step 1: Launch EC2 Instance

1. Go to **AWS Console → EC2 → Launch Instance**
2. Choose **Ubuntu 24.04 LTS**
3. Instance type: **t2.large**
4. Create or select a key pair
5. Click **Launch**

### Step 2: Configure Security Group — Open these ports

| Port | Service |
|------|---------|
| 22 | SSH |
| 8080 | Jenkins |
| 9000 | SonarQube |
| 443 | HTTPS |

Go to **EC2 → Security Groups → Inbound Rules → Add Rule** for each port above.

### Step 3: SSH into your EC2 instance

```bash
ssh -i your-key.pem ubuntu@<EC2-PUBLIC-IP>
```

---

## 🔧 Part 2: Jenkins Installation

### Step 1: Install Java (required by Jenkins)

```bash
sudo apt update
sudo apt install openjdk-17-jre -y
```

Verify:
```bash
java -version
```

Expected output:
```
openjdk version "17.x.x"
```

### Step 2: Add Jenkins Repository & Install

```bash
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update
sudo apt-get install jenkins -y
```

### Step 3: Start Jenkins

```bash
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins
```

### Step 4: Access Jenkins

Open browser: `http://<EC2-PUBLIC-IP>:8080`

Get the initial admin password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### Step 5: Complete Jenkins Setup

1. Paste the admin password
2. Click **Install suggested plugins**
3. Wait for plugins to install
4. Create your admin user
5. Jenkins is ready ✅

---

## 🐳 Part 3: Docker Installation

> ⚠️ Do NOT use `sudo apt install docker.io` on Ubuntu 24.04 — it is broken. Use the official Docker method below.

### Step 1: Remove any old/broken Docker

```bash
sudo apt remove docker docker.io -y
sudo apt autoremove -y
```

### Step 2: Install Dependencies

```bash
sudo apt install ca-certificates curl gnupg -y
```

### Step 3: Add Docker's Official GPG Key

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

### Step 4: Add Docker Repository

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Step 5: Install Docker CE

```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io -y
```

### Step 6: Verify Docker

```bash
docker --version
```

Expected:
```
Docker version 26.x.x, build xxxxxxx
```

### Step 7: Add Jenkins & Ubuntu user to Docker group

```bash
sudo usermod -aG docker jenkins
sudo usermod -aG docker ubuntu
sudo systemctl restart docker
```

### Step 8: Fix Docker socket permissions

```bash
sudo chmod 666 /var/run/docker.sock
```

### Step 9: Restart Jenkins

```bash
sudo systemctl restart jenkins
```

Or via browser: `http://<EC2-PUBLIC-IP>:8080/restart`

### Step 10: Install Docker Pipeline Plugin in Jenkins

1. Jenkins → **Manage Jenkins** → **Manage Plugins**
2. Click **Available** tab
3. Search: `Docker Pipeline`
4. Check the box → Click **Install**
5. Restart Jenkins

---

## 📊 Part 4: SonarQube Installation

### Step 1: Create SonarQube User

```bash
sudo adduser sonarqube
```

### Step 2: Switch to SonarQube User

```bash
sudo su - sonarqube
```

### Step 3: Download SonarQube

```bash
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.4.0.54424.zip
```

### Step 4: Install & Extract

```bash
sudo apt install unzip -y
unzip sonarqube-9.4.0.54424.zip
chmod -R 755 /home/sonarqube/sonarqube-9.4.0.54424
```

### Step 5: Start SonarQube

```bash
cd sonarqube-9.4.0.54424/bin/linux-x86-64/
./sonar.sh start
```

Check status:
```bash
./sonar.sh status
```

Expected:
```
SonarQube is running (PID).
```

### Step 6: Access SonarQube

Open browser: `http://<EC2-PUBLIC-IP>:9000`

```
Default Username: admin
Default Password: admin
```

> ⚠️ You will be prompted to change the password on first login.

### Step 7: Generate SonarQube Token

1. Top right → **My Account** → **Security** tab
2. Enter token name: `jenkins-sonar-token`
3. Click **Generate**
4. **Copy the token immediately** — you won't see it again!

```
Example: squ_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

---

## ⚙️ Part 5: Jenkins Configuration

### Step 1: Install SonarQube Scanner Plugin

1. Jenkins → **Manage Jenkins** → **Manage Plugins**
2. Search: `SonarQube Scanner`
3. Install → Restart Jenkins

### Step 2: Add Credentials to Jenkins

Go to: **Jenkins → Manage Jenkins → Credentials → (global) → Add Credentials**

#### Add SonarQube Token:
```
Kind:        Secret text
Secret:      <your sonarqube token>
ID:          sonarqube
Description: SonarQube Token
```

#### Add DockerHub Credentials:
```
Kind:        Username with password
Username:    <your dockerhub username>
Password:    <your dockerhub password>
ID:          docker-cred
Description: DockerHub Credentials
```

#### Add GitHub Token:
```
Kind:        Secret text
Secret:      <your github personal access token>
ID:          github
Description: GitHub Token
```

### Step 3: Configure SonarQube Server

Go to: **Jenkins → Manage Jenkins → Configure System → SonarQube Servers**

Click **Add SonarQube**:
```
Name:               sonar-server
Server URL:         http://<EC2-PUBLIC-IP>:9000
Server auth token:  sonarqube   ← (select the credential)
```

Click **Save**

### Step 4: Configure SonarQube Scanner Tool

Go to: **Jenkins → Manage Jenkins → Global Tool Configuration → SonarQube Scanner**

Click **Add SonarQube Scanner**:
```
Name:                  sonar-scanner
Install automatically: ✅ checked
Version:               SonarQube Scanner 4.8 (latest)
```

Click **Save**

### Step 5: Create Jenkins Pipeline Job

1. Jenkins → **New Item**
2. Name: `spring-app-ci`
3. Select **Pipeline** → Click **OK**
4. Under **Pipeline**:
   - Definition: `Pipeline script from SCM`
   - SCM: `Git`
   - Repository URL: `https://github.com/<your-username>/java-Spring-boot-app`
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`
5. Click **Save**

### Step 6: Configure GitHub Webhook

1. Go to your GitHub repo → **Settings → Webhooks → Add webhook**
2. Payload URL: `http://<EC2-PUBLIC-IP>:8080/github-webhook/`
3. Content type: `application/json`
4. Trigger: **Just the push event**
5. Click **Add webhook**

---

## 🖥️ Part 6: Minikube Setup (WSL2)

### Step 1: Enable WSL2 on Windows

Open PowerShell as Administrator:
```powershell
wsl --set-default-version 2
wsl --status
```

### Step 2: Install Docker Desktop on Windows

1. Download from: https://www.docker.com/products/docker-desktop
2. Install on Windows
3. Open Docker Desktop → **Settings → Resources → WSL Integration**
4. Enable for your Ubuntu distro
5. Click **Apply & Restart**

Verify in WSL2:
```bash
docker --version
```

### Step 3: Install kubectl in WSL2

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
  https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

kubectl version --client
```

### Step 4: Install Minikube in WSL2

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version
```

### Step 5: Start Minikube

```bash
minikube start --driver=docker
```

Expected output:
```
😄  minikube v1.x.x on Ubuntu 24.04
✨  Using the docker driver
👍  Starting control plane node
✅  Done! kubectl is now configured
```

### Step 6: Verify Cluster

```bash
minikube status
kubectl get nodes
```

Expected:
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   1m    v1.x.x
```

> ⚠️ **Troubleshooting WSL2 Docker Socket Error**
> If you see `dial unix /var/run/docker.sock: no such file or directory`:
> ```bash
> sudo service docker start
> sudo chmod 666 /var/run/docker.sock
> ```

---

## 🔄 Part 7: ArgoCD Installation

### Step 1: Create ArgoCD Namespace

```bash
kubectl create namespace argocd
```

### Step 2: Install ArgoCD

```bash
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Step 3: Watch Pods Come Up

```bash
kubectl get pods -n argocd -w
```

Wait until all pods show `Running`:
```
argocd-server-xxxxxxxxx        1/1   Running   0   2m
argocd-repo-server-xxxxxxxxx   1/1   Running   0   2m
argocd-application-controller  1/1   Running   0   2m
```

### Step 4: Get ArgoCD Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

Copy this password.

### Step 5: Access ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open browser: `https://localhost:8080`

```
Username: admin
Password: <output from Step 4>
```

### Step 6: Create ArgoCD Application

In ArgoCD UI → **New App**:

```
Application Name:  spring-boot-app
Project:           default
Sync Policy:       Automatic ✅
Repository URL:    https://github.com/<your-username>/java-Spring-boot-app
Revision:          HEAD
Path:              spring-boot-app-manifests/
Cluster URL:       https://kubernetes.default.svc
Namespace:         default
```

Click **Create**

ArgoCD will now automatically sync and deploy whenever the Git manifest changes ✅

---

## 🔁 Part 8: Pipeline Flow

Once everything is set up, the full automated flow is:

```
1. You push code to GitHub
        ↓
2. GitHub Webhook triggers Jenkins
        ↓
3. Jenkins: Maven Build & Unit Tests
        ↓
4. Jenkins: SonarQube Code Analysis
        ↓
5. Jenkins: Docker Build & Push to DockerHub
        ↓
6. Jenkins: Update image tag in deployment.yaml & push to GitHub
        ↓
7. ArgoCD detects Git change (polls every 3 minutes)
        ↓
8. ArgoCD deploys updated image to Kubernetes
        ↓
9. Application is live with new version ✅
```

---

## 📚 References

- 📺 [Theory Video - Abhishek Veeramalla](https://www.youtube.com/watch?v=jNPGo6A4VHc)
- 📺 [Implementation Video - Abhishek Veeramalla](https://www.youtube.com/watch?v=JGQI5pkK82w)
- 📦 [Original GitHub Repo](https://github.com/iam-veeramalla/Jenkins-Zero-To-Hero)
- 📖 [ArgoCD Documentation](https://argo-cd.readthedocs.io/en/stable/)
- 📖 [Jenkins Documentation](https://www.jenkins.io/doc/)
- 📖 [SonarQube Documentation](https://docs.sonarqube.org/)

---

## 👨‍💻 Author

Project replicated from **Abhishek Veeramalla's** DevOps tutorial series.

⭐ If this helped you, please star the repository!
