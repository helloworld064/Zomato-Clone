# 🍽️ Zomato Clone App with DevSecOps CI/CD 🚀

A full-fledged DevSecOps pipeline for a Zomato clone application using **Jenkins**, **Docker**, **SonarQube**, **OWASP Dependency Check**, and **Trivy**. This project sets up automated CI/CD with integrated security tools on an AWS EC2 instance.

---

## 📸 Project Architecture

![Zomato CI/CD Pipeline](https://your-image-url.com/image.png) <!-- Replace with actual image link -->

---

## 🛠️ Tech Stack

- **Jenkins** for CI/CD orchestration  
- **Docker** for containerization  
- **SonarQube** for static code analysis  
- **OWASP Dependency Check** for vulnerability scanning  
- **Trivy** for container and filesystem scanning  
- **Node.js 16**  
- **Java 17 (Temurin)**  
- **AWS EC2 (Ubuntu 22.04)**

---

## 🚀 Pipeline Workflow

### Step 1 — Launch AWS EC2 Instance  
- OS: Ubuntu 22.04  
- Instance Type: t2.large  
- Open inbound ports: `22`, `8080`, `9000`, `3000`

---

### Step 2 — Install Jenkins, Docker, and Trivy  
```bash
vi jenkins.sh

#!/bin/bash
sudo apt update -y
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins


sudo chmod +x jenkins.sh
./jenkins.sh

Access Jenkins: http://<EC2-Public-IP>:8080

Unlock Jenkins:


sudo cat /var/lib/jenkins/secrets/initialAdminPassword



Step 3 — Install Tools & Plugins in Jenkins
Go to: Manage Jenkins → Plugins
Install the following:

✅ Eclipse Temurin Installer
✅ SonarQube Scanner
✅ NodeJs Plugin
✅ OWASP Dependency Check
✅ Docker Pipeline Plugins (docker, docker commons, docker pipeline, etc.)


Step 4 — Jenkins Global Tool Configuration
Go to: Manage Jenkins → Global Tool Configuration

Install:

JDK 17 (temurin)

Node.js 16

SonarQube Scanner

OWASP Dependency Check (DP-Check)


Step 5 — Setup SonarQube with Docker


sudo apt-get install docker.io -y
sudo usermod -aG docker $USER
newgrp docker
sudo chmod 777 /var/run/docker.sock

docker run -d --name sonar -p 9000:9000 sonarqube:lts-community


Access SonarQube: http://<EC2-Public-IP>:9000

Create a token via Administration → Security → Users → Tokens

Add webhook in SonarQube:
URL: http://<JENKINS-IP>:8080/sonarqube-webhook/


Step 6 — Install Trivy
vi trivy.sh

sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y


🧪 Jenkins Pipeline Script

pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/Aj7Ay/Zomato-Clone.git'
            }
        }
        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=zomato \
                    -Dsonar.projectKey=zomato '''
                }
            }
        }
        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }
        stage("Docker Build & Push") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh 'docker build -t zomato .'
                        sh 'docker tag zomato sevenajay/zomato:latest'
                        sh 'docker push sevenajay/zomato:latest'
                    }
                }
            }
        }
        stage("Trivy Image Scan") {
            steps {
                sh 'trivy image sevenajay/zomato:latest > trivy.txt'
            }
        }
        stage("Deploy to Container") {
            steps {
                sh 'docker run -d --name zomato -p 3000:3000 sevenajay/zomato:latest'
            }
        }
    }
}

🧹 Final Step — Terminate AWS Instances
After completing the pipeline, don’t forget to terminate your EC2 instances to avoid unnecessary billing.

