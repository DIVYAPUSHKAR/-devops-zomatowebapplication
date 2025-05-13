# steps too build project
1. Launch an Instance (Ubuntu, 24.04, t2.large, 30 GB)

2. Connect to the instance

3. Update the packages
$ switch to root user ---> sudo su
$ sudo apt update -y

4. Install AWS CLI
sudo apt install unzip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

5. Install Jenkins on Ubuntu
(Reference URL for commands: https://www.jenkins.io/doc/book/installing/linux/#debianubuntu)
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

Verifiy Jenkins installation: jenkins --version

5.1. Open Port No. 8080 for VM and access Jenkins
5.2. Setup Jenkins by following the necessary steps

6. Install Docker on Ubuntu
(Reference URL for commands: https://docs.docker.com/engine/install/ubuntu/)

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

Verifiy Docker installation: docker --version

7. Install Trivy on Ubuntu
(Reference URL for commands: https://aquasecurity.github.io/trivy/v0.55/getting-started/installation/)
sudo apt-get install wget apt-transport-https gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

Verifiy Trivy installation: trivy --version

8. Install Docker Scout
Make sure to Login to DockerHub account in browser


9. Install SonarQube using Docker
$ docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
$ docker ps (You can see SonarQube container)



10. Installation of Plugins in Jenkins



11. SonarQube configuration in Jenkins


11.1. Tools Configuration in Jenkins


11.2. Configuration of SonarQube Token in Jenkins




12. System Configuration in Jenkins


13. Create webhook in SonarQube


14. Create Pipeline Job
Before pasting the pipeline script, do the following changes in the script
1. In the stage 'Tag and Push to DockerHub', give your docker-hub username. Similar thing you should do in 'DockerScoutImage', 'Deploy to container' stages
2. In post actions stage in pipeline, make sure to give the email id you have configured in jenkins.

*********************
Pipeline Script
*********************
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node23'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage ("clean workspace") {
            steps {
                cleanWs()
            }
        }
        stage ("Git Checkout") {
            steps {
                git 'https://github.com/DIVYAPUSHKAR/-devops-zomatowebapplication.git'
            }
        }
        stage("Sonarqube Analysis"){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=zomato \
                    -Dsonar.projectKey=zomato '''
                }
            }
        }
        stage("Code Quality Gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage("Install NPM Dependencies") {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit -n', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
    }
}
        stage ("Trivy File Scan") {
            steps {
                sh "trivy fs . > trivy.txt"
            }
        }
        stage ("Build Docker Image") {
            steps {
                sh "docker build -t zomato ."
            }
        }
        stage ("Tag & Push to DockerHub") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker') {
                        sh "docker tag zomato divya554/zomato:latest "
                        sh "docker push divya554/zomato:latest "
                    }
                }
            }
        }
        stage('Docker Scout Image') {
            steps {
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh 'docker-scout quickview divya554/zomato:latest'
                       sh 'docker-scout cves divya554/zomato:latest'
                       sh 'docker-scout recommendations divya554/zomato:latest'
                   }
                }
            }
        }
        stage ("Deploy to Container") {
            steps {
                sh 'docker run -d --name zomato -p 3000:3000 divya554/zomato:latest'
            }
        }
    }
    post {
    always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: """
                <html>
                <body>
                    <div style="background-color: #FFA07A; padding: 10px; margin-bottom: 10px;">
                        <p style="color: white; font-weight: bold;">Project: ${env.JOB_NAME}</p>
                    </div>
                    <div style="background-color: #90EE90; padding: 10px; margin-bottom: 10px;">
                        <p style="color: white; font-weight: bold;">Build Number: ${env.BUILD_NUMBER}</p>
                    </div>
                    <div style="background-color: #87CEEB; padding: 10px; margin-bottom: 10px;">
                        <p style="color: white; font-weight: bold;">URL: ${env.BUILD_URL}</p>
                    </div>
                </body>
                </html>
            """,
            to: 'pushkardivya535@gmail.com',
            mimeType: 'text/html',
            attachmentsPattern: 'trivy.txt'
        }
    }
}



If the build stage of "OWASP FS SCAN" shows 'UNSTABLE BUILD' replace the below script in OWASP FS SCAN stage
stage('OWASP FS SCAN') {
    steps {
        dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit --update -n', odcInstallation: 'DP-Check'
        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
    }
}


Let the pipeline gets built. Meanwhile we will create VMs for monitoring.

------------------------------------------------------------
MONITORING OF APPLICATION
------------------------------------------------------------
15. Launch VM (Name: Monitoring Server, Ubuntu 24.04, t2.large, Select the SG created in the Step 1, EBS: 30GB)
We will install Grafana, Prometheus, Node Exporter in the above instance and then we will monitor
--------------------------------------------------


--------------------------------------------------
15.2. Installing Prometheus
--------------------------------------------------
First, create a dedicated Linux user for Prometheus and download Prometheus
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
sudo vi /etc/systemd/system/prometheus.service

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

Explanation of the key elements in the above prometheus.service file:
User and Group specify the Linux user and group under which Prometheus will run.
ExecStart is where you specify the Prometheus binary path, the location of the configuration file (prometheus.yml), the storage directory, and other settings.
web.listen-address configures Prometheus to listen on all network interfaces on port 9090.
web.enable-lifecycle allows for management of Prometheus through API calls.


Enable and start Prometheus:
sudo systemctl enable prometheus
sudo systemctl start prometheus

Verify Prometheus's status:
sudo systemctl status prometheus

Press Control+c to come out

Access Prometheus in browser using your server's IP and port 9090:
http://<your-server-ip>:9090

If it doesn't work, in the web link of browser, remove 's' in 'https'. Keep only 'http' and now you will be able to see.
You can see the Prometheus console.
Click on 'Status' dropdown ---> Click on 'Targets' ---> You can see 'Prometheus (1/1 up)'
--------------------------------------------------
15.3. Installing Node Exporter 
--------------------------------------------------
cd 
You are in ~ path now

Create a system user for Node Exporter and download Node Exporter:
sudo useradd --system --no-create-home --shell /bin/false node_exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz

Extract Node Exporter files, move the binary, and clean up:
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter*

Create a systemd unit configuration file for Node Exporter:
sudo vi /etc/systemd/system/node_exporter.service

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

Note: Replace --collector.logind with any additional flags as needed.

Enable and start Node Exporter:
sudo systemctl enable node_exporter
sudo systemctl start node_exporter

Verify the Node Exporter's status:
sudo systemctl status node_exporter
You can see "active (running)" in green colour
Press control+c to come out of the file

------------------------------------------------------------
15.4: Configure Prometheus Plugin Integration
------------------------------------------------------------
As of now we created Prometheus service, but we need to add a job in order to fetch the details by node exporter. So for that we need to create 2 jobs, one with 'node exporter' and the other with 'jenkins' as shown below;

Integrate Jenkins with Prometheus to monitor the CI/CD pipeline.

Prometheus Configuration:

To configure Prometheus to scrape metrics from Node Exporter and Jenkins, you need to modify the prometheus.yml file. 
The path of prometheus.yml is; cd /etc/prometheus/ ----> ls -l ----> You can see the "prometheus.yml" file ----> sudo vi prometheus.yml ----> You will see the content and also there is a default job called "Prometheus" Paste the below content at the end of the file;

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['<MonitoringVMip>:9100']

  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<your-jenkins-ip>:<your-jenkins-port>']

 In the above, replace <your-jenkins-ip> and <your-jenkins-port> with the appropriate IPs ----> esc ----> :wq

Check the validity of the configuration file:
promtool check config /etc/prometheus/prometheus.yml

You should see "SUCCESS" when you run the above command, it means every configuration made so far is good.

Reload the Prometheus configuration without restarting:
curl -X POST http://localhost:9090/-/reload

Access Prometheus in browser (if already opened, just reload the page):
http://<your-prometheus-ip>:9090/targets

Open Port number 9100 for Monitoring VM 

You should now see "Jenkins (1/1 up)" "node exporter (1/1 up)" and "prometheus (1/1 up)" in the prometheus browser.
Click on "showmore" next to "jenkins." You will see a link. Open the link in new tab, to see the metrics that are getting scraped

------------------------------------------------------------
15.5: Install Grafana
------------------------------------------------------------
You are currently in /etc/Prometheus path.

Install Grafana on Monitoring Server;

Step 1: Install Dependencies:
First, ensure that all necessary dependencies are installed:
sudo apt-get update
sudo apt-get install -y apt-transport-https software-properties-common

Step 2: Add the GPG Key:
cd ---> You are now in ~ path
Add the GPG key for Grafana:
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -

You should see OK when executed the above command.

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

Start Grafana:
sudo systemctl start grafana-server

Step 6: Check Grafana Status:
Verify the status of the Grafana service to ensure it's running correctly:
sudo systemctl status grafana-server

You should see "Active (running)" in green colour
Press control+c to come out

Step 7: Access Grafana Web Interface:
The default port for Grafana is 3000
http://<monitoring-server-ip>:3000

Default id and password is "admin"
You can Set new password or you can click on "skip now".
Click on "skip now" (If you want you can create the password)

You will see the Grafana dashboard

The first thing that we have to do in Grafana is to add the data source
Lets add the data source;

