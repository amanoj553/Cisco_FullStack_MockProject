# CI/CD Pipeline Documentation for JAR Deployment to EC2

## 1. Architecture Flow Diagram

![CI/CD Architecture](Flow_diagram.JPG)

GitHub (Source Code)
│
▼
Jenkins (Build & Orchestrate)
├── Maven Build
├── SonarQube Code Analysis
├── OWASP Dependency Check
├── Docker Build & Push to Docker Hub
├── Trivy Image Scan
└── Deploy to EC2 (App Server)

---

## 2. EC2 Servers Setup

| Purpose       | EC2 Type | OS           | Storage | Inbound Ports                 |
|---------------|----------|--------------|---------|-------------------------------|
| Jenkins       | t3.medium| Ubuntu 22.04 | 15 GB   | 8080 (Jenkins), 22 (SSH)      |
| SonarQube     | t3.medium| Ubuntu 22.04 | 15 GB   | 9000 (SonarQube), 5432 (DB), 22 |
| App Server    | t3.small | Ubuntu 22.04 | 10 GB   | 8090 (App), 22 (SSH)          |

---

## 3. Prerequisites and Configuration

### A. In Jenkins Server (Ubuntu 22.04)

#### Install below Packages on Jenkins server

**Install Java:**
```bash
sudo apt update
sudo apt install openjdk-21-jdk maven docker.io git unzip curl -y
```
**Install Jenkins:**
```bash
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins -y

# start Jenkins service
sudo systemctl start jenkins
sudo systemctl enable jenkins
```
**Install Trivy:**
```bash
sudo apt install wget -y
wget https://github.com/aquasecurity/trivy/releases/latest/download/trivy_0.50.1_Linux-64bit.deb
sudo dpkg -i trivy_0.50.1_Linux-64bit.deb
```
**Install OWASP Dependency Check:**
```bash
wget https://github.com/jeremylong/DependencyCheck/releases/download/v8.4.0/dependency-check-8.4.0-release.zip
unzip dependency-check-8.4.0-release.zip -d /opt/dependency-check
```

### B. In SonarQube Server (Ubuntu 22.04)
**Install SonarQube:**

```bash
# Update system packages
sudo apt update
sudo apt upgrade -y

# Install Java (required for SonarQube)
sudo apt install -y openjdk-17-jdk
java -version

# install PostgreSQL
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" /etc/apt/sources.list.d/pgdg.list'
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
sudo apt install postgresql postgresql-contrib -y
sudo systemctl enable postgresql
sudo systemctl start postgresql
sudo systemctl status postgresql
psql --version

# switch to Postgre user
sudo -i -u postgres

# Create user for SonarQube
createuser ddsonar

# login into PostgreSQL
psql

# Set a password for the ddsonar user. Use a strong password in place of my_strong_password.
ALTER USER [Created_user_name] WITH ENCRYPTED password 'my_strong_password';
# example
ALTER USER ddsonar WITH ENCRYPTED password 'mwd#2%#!!#%rgs';

# Create a SonarQube database and set the owner to ddsonar.
CREATE DATABASE [database_name] OWNER [Created_user_name];
# example
CREATE DATABASE ddsonarqube OWNER ddsonar;

# Grant all the privileges on the ddsonarqube database to the ddsonar user.
GRANT ALL PRIVILEGES ON DATABASE ddsonarqube to ddsonar;

# List databases and users
\l
\du

# Exit PostgreSQL and return to previous (root) user
\q
exit

# download and Install Sonarqube
sudo apt install zip -y
sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.0.0.68432.zip
sudo unzip sonarqube-10.0.0.68432.zip
sudo mv sonarqube-10.0.0.68432 sonarqube
sudo mv sonarqube /opt/

# Add SonarQube Group and User
sudo groupadd ddsonar
sudo useradd -d /opt/sonarqube -g ddsonar ddsonar
sudo chown ddsonar:ddsonar /opt/sonarqube -R
```
**Configure SonarQube:**
```bash
# Edit the SonarQube Configurationfile:
sudo vi /opt/sonarqube/conf/sonar.properties

# Update the following lines (uncomment and modify as needed) and save the file:
sonar.jdbc.username=ddsonar
sonar.jdbc.password=mwd#2%#!!#%rgs
sonar.jdbc.url=jdbc:postgresql://localhost:5432/ddsonarqube

# Edit the sonar script file.
sudo vi /opt/sonarqube/bin/linux-x86-64/sonar.sh
# Add the following line in top after bash line then save the file
RUN_AS_USER=ddsonar
```

**Setup Systemd service:**
```bash
# Create a systemd service file to start SonarQube at system boot.
sudo vi /etc/systemd/system/sonar.service

# Paste the following lines to the file and save the file.
[Unit]
Description=SonarQube service
After=syslog.target network.target
[Service]
Type=forking
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
User=ddsonar
Group=ddsonar
Restart=always
LimitNOFILE=65536
LimitNPROC=4096
[Install]
WantedBy=multi-user.target

Note: Here in the above script, make sure to change the User and Group section with the value that you have created.
# Enable the SonarQube service to run at system startup.
sudo systemctl enable sonar

# Start the SonarQube service.
sudo systemctl start sonar

#Check the service status.
sudo systemctl status sonar
```

**Modify Kernel System Limits:**
```bash
# SonarQube uses Elasticsearch to store its indices in an MMap FS directory. It requires some changes to the system defaults.
# Edit the sysctl configuration file.
sudo nano /etc/sysctl.conf

# Add the following lines and save the file.
vm.max_map_count=262144
fs.file-max=65536
ulimit -n 65536
ulimit -u 4096

# Reboot the system to apply the changes.
sudo reboot
```

**Access SonarQube Web Interface:**
```bash
# Access SonarQube in a web browser at your server’s IP address on port 9000.
For example, http://IP:9000

# Change the Old password with a New one.
Log in with username `admin` and password `admin`. In the next step, SonarQube will prompt you to change your password. `CHANGE THE PASSWORD`.
```
**Our SonarQube has been installed successfully.**

### C. App Server

**Install Docker**
```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker
```

## 4. Jenkins Configuration:

### Configure Global Tools
- Maven: `Maven3` (point to: `/usr/share/maven`)
- JDK: `java-21` (point to: `/usr/lib/jvm/java-21-openjdk-amd64`)

### Install plugins
- Docker Pipeline
- Pipeline
- Pipeline view
- OWASP Dependency-Check Plugin
- SonarQube Scanner
- SSH Agent

### Global tool configuration

**SonarQube:**
- Name: SonarQube
- Server URL: http://<SonarQube-IP>:9000
- Token: Add via Jenkins Credentials (Secret Text)

**DockerHub:**
- Jenkins Credential ID: dockerhub-creds
- Username & Password as secret credentials

**App Server SSH:**
- Add SSH private key to Jenkins Credentials
- ID: ec2-ssh-key

## 5. Dockerfile:

```bash
FROM openjdk:21-jdk-slim
WORKDIR /app
COPY target/hello-world-app-1.0.0.jar app.jar
EXPOSE 8090
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## 6. Jenkinsfile (Declarative Pipeline):

```bash
pipeline {
    agent any
    environment {
        SONAR_HOST_URL = 'http://34.224.169.37:9000'
        DOCKER_HUB_REPO = 'amanoj3452/hello-world-demo'
        DEPLOY_SERVER = 'ubuntu@3.80.124.38'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'poc-demo', 
                    url: 'https://github.com/amanoj553/hello-world-app.git'
            }
        }
        stage('Maven Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        stage('OWASP Dependency-Check') {
            steps {
                sh '''
                /opt/dependency-check/bin/dependency-check.sh \
                  --project "DemoApp" \
                  --scan . \
                  --format HTML \
                  --out dependency-report \
                  --data /opt/dependency-check/data \
                  --cveUrlBase https://github.com/jeremylong/CVE-Base \
                  --cveUrlModified https://github.com/jeremylong/CVE-Modified
                '''
            }
        }
        stage('Docker Build & Push') {
            steps {
                script {
                    def appImage = docker.build("${DOCKER_HUB_REPO}:latest")
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push $DOCKER_HUB_REPO:latest
                        '''
                    }
                }
            }
        }
        stage('Trivy Image Scan') {
            steps {
                sh '''
                trivy image --exit-code 1 --severity HIGH,CRITICAL ${DOCKER_HUB_REPO}:latest || true
                '''
            }
        }        
        stage('Deploy to EC2') {
            steps {
                sshagent (credentials: ['ec2-ssh-key']) {
                    script {
                        def imageTag = "${DOCKER_HUB_REPO}:latest"
                        sh """
                        ssh -o StrictHostKeyChecking=no $DEPLOY_SERVER '
                            docker pull ${imageTag} &&
                            docker stop demoapp || true &&
                            docker rm demoapp || true &&
                            docker run -d --name demoapp -p 8090:8090 ${imageTag}
                        '
                        """
                    }
                }
            }
        }
    }
}
```

![Build success status](Jenkins_build_success_status.JPG)

![SonarQube Report](SonarQube_report.JPG)


## 7. Application test:

**After deployment, verify the application:**
```bash
curl http://<App-Server-IP>:8090/
# Expected Output:
Hello from Demo JAR App!
```
**Goto browser and check with below URL**
- http://<IP>:8090/
- Expected Output:
  - `Hello from Demo JAR App!`
  
![ApplicationTest](deployment_confirm_status.JPG)

## 8. Troubleshooting Tips:

**SonarQube not loading?**
- Ensure `java` is version 17+ on SonarQube server
- Verify PostgreSQL is running
- check the sonarqube logs for more details
- check the server menory also some times insufficient also not running

**OWASP error about DB lock or permission denied?**
- Ensure `/opt/dependency-check/data` is writable by Jenkins
- Run dependency-check once manually as Jenkins user
