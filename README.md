☸️ Deploy Amazon Prime Clone Application AWS using DevSecOps Approach 🐳
image

Project Blogs
Your New Video Title Here	Deploy Amazon Prime Video Clone On EKS Cluster
Install all prerequisites
Install AWS CLI
sudo apt install unzip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
Install Jenkins on Ubuntu:
#!/bin/bash
sudo apt update -y
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | sudo tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | sudo tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
/usr/bin/java --version
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl status jenkins
Install Docker on Ubuntu:
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo usermod -aG docker ubuntu
sudo chmod 777 /var/run/docker.sock
newgrp docker
sudo systemctl status docker
Install Trivy on Ubuntu:
Reference Doc: https://aquasecurity.github.io/trivy/v0.55/getting-started/installation/

sudo apt-get install wget apt-transport-https gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
Sonarqube Setup On Docker
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
Deployment Stages:
image

Jenkins Stage-View:
stage

Jenkins Complete pipeline
pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16' // Ensure 'node16' is configured in Jenkins under Manage Jenkins > Global Tool Configuration
    }
    environment {
        SCANNER_HOME = tool 'sonar'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }


        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/harshitsahu2311/Amazon-Prime-Clone-App.git'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=prime-clone \
                        -Dsonar.projectKey=prime-clone
                    '''
                }
            }
        }
        stage("quality gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-cred'
                }
            }
        }
        stage ("Trivy File Scan") {
            steps {
                sh "trivy fs . > trivy.txt"
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    // Build the Docker image tagged as 'prime-clone:latest'
                    sh "docker build -t prime-clone:latest ."
                }
            }
        }
        stage ("Trivy Image Scan") {
            steps {
                sh "trivy image prime-clone:latest"
            }
        }
        stage('Tag & Push to DockerHub') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker tag prime-clone:latest harshitsahu2311/prime-clone:latest"
                        sh "docker push harshitsahu2311/prime-clone:latest"
                    }
                }
            }
        }
        stage ("Deploy to Kubernetes") {
            steps {
                script {
                    withCredentials([file(credentialsId: 'kuber-cred', variable: 'KUBECONFIG_FILE')]) {
                        sh """
                            # Set the KUBECONFIG environment variable
                            export KUBECONFIG=${KUBECONFIG_FILE}

                            # Navigate to the Kubernetes manifests directory
                            cd Kubernetes

                            # Apply all Kubernetes manifests
                            kubectl apply -f .

                        """
                    }
                }
            }

        }
    }

}
post {
    success {
            echo '✅ Deployment successful!'
            // Optional: Add notifications (e.g., email, Slack) here
    }
    failure {
            echo '❌ Deployment failed.'
            // Optional: Add notifications (e.g., email, Slack) here
    }
    }


Phase 4: Monitoring

Install Prometheus and Grafana:

Set up Prometheus and Grafana to monitor your application.

Installing Prometheus:

First, create a dedicated Linux user for Prometheus and download Prometheus:

sudo useradd --system --no-create-home --shell /bin/false prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
Extract Prometheus files, move them, and create directories:

tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
cd prometheus-2.47.1.linux-amd64/
sudo mkdir -p /data /etc/prometheus
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
Set ownership for directories:

sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
Create a systemd unit configuration file for Prometheus:

sudo nano /etc/systemd/system/prometheus.service
Add the following content to the prometheus.service file:

[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
Here's a brief explanation of the key parts in this prometheus.service file:

User and Group specify the Linux user and group under which Prometheus will run.

ExecStart is where you specify the Prometheus binary path, the location of the configuration file (prometheus.yml), the storage directory, and other settings.

web.listen-address configures Prometheus to listen on all network interfaces on port 9090.

web.enable-lifecycle allows for management of Prometheus through API calls.

Enable and start Prometheus:

sudo systemctl enable prometheus
sudo systemctl start prometheus
Verify Prometheus's status:

sudo systemctl status prometheus
You can access Prometheus in a web browser using your server's IP and port 9090:

http://<your-server-ip>:9090

Installing Node Exporter:

Create a system user for Node Exporter and download Node Exporter:

sudo useradd --system --no-create-home --shell /bin/false node_exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
Extract Node Exporter files, move the binary, and clean up:

tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter*
Create a systemd unit configuration file for Node Exporter:

sudo nano /etc/systemd/system/node_exporter.service
Add the following content to the node_exporter.service file:

[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
Replace --collector.logind with any additional flags as needed.

Enable and start Node Exporter:

sudo systemctl enable node_exporter
sudo systemctl start node_exporter
Verify the Node Exporter's status:

sudo systemctl status node_exporter
You can access Node Exporter metrics in Prometheus.

Configure Prometheus Plugin Integration:

Integrate Jenkins with Prometheus to monitor the CI/CD pipeline.

Prometheus Configuration:

To configure Prometheus to scrape metrics from Node Exporter and Jenkins, you need to modify the prometheus.yml file. Here is an example prometheus.yml configuration for your setup:

global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<your-jenkins-ip>:<your-jenkins-port>']
Make sure to replace <your-jenkins-ip> and <your-jenkins-port> with the appropriate values for your Jenkins setup.

Check the validity of the configuration file:

promtool check config /etc/prometheus/prometheus.yml
Reload the Prometheus configuration without restarting:

curl -X POST http://localhost:9090/-/reload
You can access Prometheus targets at:

http://<your-prometheus-ip>:9090/targets

Grafana
Install Grafana on Ubuntu 22.04 and Set it up to Work with Prometheus

Step 1: Install Dependencies:

First, ensure that all necessary dependencies are installed:

sudo apt-get update
sudo apt-get install -y apt-transport-https software-properties-common
Step 2: Add the GPG Key:

Add the GPG key for Grafana:

wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
Step 3: Add Grafana Repository:

Add the repository for Grafana stable releases:

echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
Step 4: Update and Install Grafana:

Update the package list and install Grafana:

sudo apt-get update
sudo apt-get -y install grafana
Step 5: Enable and Start Grafana Service:

To automatically start Grafana after a reboot, enable the service:

sudo systemctl enable grafana-server
Then, start Grafana:

sudo systemctl start grafana-server
Step 6: Check Grafana Status:

Verify the status of the Grafana service to ensure it's running correctly:

sudo systemctl status grafana-server
Step 7: Access Grafana Web Interface:

Open a web browser and navigate to Grafana using your server's IP address. The default port for Grafana is 3000. For example:

http://<your-server-ip>:3000

Monitor Kubernetes with Prometheus
Prometheus is a powerful monitoring and alerting toolkit, and you'll use it to monitor your Kubernetes cluster. Additionally, you'll install the node exporter using Helm to collect metrics from your cluster nodes.

Install Node Exporter using Helm
To begin monitoring your Kubernetes cluster, you'll install the Prometheus Node Exporter. This component allows you to collect system-level metrics from your cluster nodes. Here are the steps to install the Node Exporter using Helm:

Add the Prometheus Community Helm repository:

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
Create a Kubernetes namespace for the Node Exporter:

kubectl create namespace prometheus-node-exporter
Install the Node Exporter using Helm:

helm install prometheus-node-exporter prometheus-community/prometheus-node-exporter --namespace prometheus-node-exporter
Add a Job to Scrape Metrics on nodeip:9001/metrics in prometheus.yml:

Update your Prometheus configuration (prometheus.yml) to add a new job for scraping metrics from nodeip:9001/metrics. You can do this by adding the following configuration to your prometheus.yml file:

  - job_name: 'k8s'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['node1Ip:9100']
Replace 'your-job-name' with a descriptive name for your job. The static_configs section specifies the targets to scrape metrics from, and in this case, it's set to nodeip:9001.

Don't forget to reload or restart Prometheus to apply these changes to your configuration.

To deploy an application with ArgoCD, you can follow these steps, which I'll outline in Markdown format:

Deploy Application with ArgoCD
Install ArgoCD:

You can install ArgoCD on your Kubernetes cluster by following the instructions provided in the EKS Workshop documentation.

Set Your GitHub Repository as a Source:

After installing ArgoCD, you need to set up your GitHub repository as a source for your application deployment. This typically involves configuring the connection to your repository and defining the source for your ArgoCD application. The specific steps will depend on your setup and requirements.

Create an ArgoCD Application:

name: Set the name for your application.
destination: Define the destination where your application should be deployed.
project: Specify the project the application belongs to.
source: Set the source of your application, including the GitHub repository URL, revision, and the path to the application within the repository.
syncPolicy: Configure the sync policy, including automatic syncing, pruning, and self-healing.
Access your Application

To Access the app make sure port 30007 is open in your security group and then open a new tab paste your NodeIP:30007, your app should be running.
Phase 7: Cleanup

Cleanup AWS EC2 Instances:
Terminate AWS EC2 instances that are no longer needed.
About
No description, website, or topics provided.
Resources
 Readme
 Activity
Stars
 0 stars
Watchers
 0 watching
Forks
 0 forks
Releases
No releases published
Create a new release
Packages
No packages published
Publish your first package
Languages
CSS
77.8%
 
JavaScript
22.0%
 
Other
0.2%
Suggested workflows
Based on your tech stack
Grunt logo
Grunt
Build a NodeJS project with npm and grunt.
Deno logo
Deno
Test your Deno project
Node.js logo
Node.js
Build and test a Node.js project with npm.
More workflows
Footer
