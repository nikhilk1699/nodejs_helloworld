*STEP 1.Launch an Ubuntu(22.04) T3 medium Instance*
Launch an AWS T3 medium Instance. Use the image as Ubuntu. You can create a new key pair or use an existing one. 
Enable HTTP and HTTPS settings in the Security Group and open all ports (not best case to open all ports but just for learning purposes it’s okay).

*Step 2 — Install Jenkins, Docker and Trivy*
To Install Jenkins
vi jenkins.sh #make sure run in Root (or) add at userdata while ec2 launch
```
#!/bin/bash
sudo apt update -y
#sudo apt upgrade -y
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
/usr/bin/java --version
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
                  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
                  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
                              /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl status jenkins 
```
```
sudo chmod 777 jenkins.sh
./jenkins.sh    # this will installl jenkins
```
Once Jenkins is installed, you will need to go to your AWS EC2 Security Group and open Inbound Port 8080, since Jenkins works on Port 8080.

Now, grab your Public IP Address
```
<EC2 Public IP Address:8080>
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
![image](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/02fe2b9a-b772-4a96-911a-a7821dea1f93)

Unlock Jenkins using an administrative password and install the suggested plugins.Jenkins will now get installed and install all the libraries.
Create a user click on save and continue. Jenkins Getting Started Screen.
![image](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/c2958897-d285-4944-8654-ecfc356ca53d)

Install Docker
```
sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER   #my case is ubuntu
newgrp docker
sudo chmod 777 /var/run/docker.sock
```
After the docker installation, we create a sonarqube container (Remember to add 9000 ports in the security group).
```
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```
Now our sonarqube is up and running
copy instance public ip run port number 9000
![image](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/2146fb51-744d-4d41-b675-9b18911d74c0)
Enter username and password, click on login and change password. Update New password, This is Sonar Dashboard.

Install Trivy

```
vi trivy.sh
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y
```

*step 3 Install Prometheus and Grafana On the new Server*

First of all, let’s create a dedicated Linux user sometimes called a system account for Prometheus. Having individual users for each service serves two main purposes:
It is a security measure to reduce the impact in case of an incident with the service. It simplifies administration as it becomes easier to track down what resources belong to which service. To create a system user or system account, run the following command:
```
sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false prometheus
```
– system – Will create a system account.
– no-create-home – We don’t need a home directory for Prometheus or any other system accounts in our case.
– shell /bin/false – It prevents logging in as a Prometheus user.
- Prometheus – Will create a Prometheus user and a group with the same name.
```
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
```
```
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
```
Usually, you would have a disk mounted to the data directory. For this tutorial, I will simply create a /data directory. Also, you need a folder for Prometheus configuration files.
```
sudo mkdir -p /data /etc/prometheus
```
Now, let’s change the directory to Prometheus and move some files.
```
cd prometheus-2.47.1.linux-amd64/
```
First of all, let’s move the Prometheus binary and a promtool to the /usr/local/bin/. promtool is used to check configuration files and Prometheus rules.
```
sudo mv prometheus promtool /usr/local/bin/
```
Optionally, we can move console libraries to the Prometheus configuration directory. Console templates allow for the creation of arbitrary consoles using the Go templating language. 
```
sudo mv consoles/ console_libraries/ /etc/prometheus/
```
the main Prometheus configuration file.
```
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
```
To avoid permission issues, you need to set the correct ownership for the /etc/prometheus/ and data directory.
```
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
```
You can delete the archive and a Prometheus folder when you are done.
```
cd
rm -rf prometheus-2.47.1.linux-amd64.tar.gz
```
Verify that you can execute the Prometheus binary by running the following command:
```
prometheus --version
```
We’re going to use Systemd, which is a system and service manager for Linux operating systems. 
For that, we need to create a Systemd unit configuration file.
```
sudo vim /etc/systemd/system/prometheus.service
```
**Prometheus.service**
```
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
```
the most important options related to Systemd and Prometheus. 
Restart – Configures whether the service shall be restarted when the service process exits, is killed, or a timeout is reached.
RestartSec – Configures the time to sleep before restarting a service.
User and Group – Are Linux user and a group to start a Prometheus process.
–config.file=/etc/prometheus/prometheus.yml – Path to the main Prometheus configuration file.
–storage.tsdb.path=/data – Location to store Prometheus data.
–web.listen-address=0.0.0.0:9090 – Configure to listen on all network interfaces. 
In some situations, you may have a proxy such as nginx to redirect requests to Prometheus. 
In that case, you would configure Prometheus to listen only on localhost.
–web.enable-lifecycle — Allows to manage Prometheus, for example, to reload configuration without restarting the service.
To automatically start the Prometheus after reboot, run enable.
```
sudo systemctl enable prometheus
```
Then just start the Prometheus.
```
sudo systemctl start prometheus
```
Install Node Exporter on Ubuntu
Next, we’re going to set up and configure Node Exporter to collect Linux system metrics like CPU load and disk I/O. Node Exporter will expose these as Prometheus-style metrics. Since the installation process is very similar, I’m not going to cover as deep as Prometheus.
First, let’s create a system user for Node Exporter by running the following command:
```
sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false node_exporter
```
Use the wget command to download the binary.
```
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
```
Extract the node exporter from the archive.
````
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
```
Move binary to the /usr/local/bin.
```
sudo mv \
  node_exporter-1.6.1.linux-amd64/node_exporter \
  /usr/local/bin/
```
Clean up, and delete node_exporter archive and a folder.
```
rm -rf node_exporter*
```
Verify that you can run the binary.
```
node_exporter --version
```
Next, create a similar systemd unit file.
```
sudo vim /etc/systemd/system/node_exporter.service
````
***node_exporter.service**
```
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
ExecStart=/usr/local/bin/node_exporter \
    --collector.logind
[Install]
WantedBy=multi-user.target
```
Replace Prometheus user and group to node_exporter, and update the ExecStart command.
To automatically start the Node Exporter after reboot, enable the service.
```
sudo systemctl enable node_exporter
```
then start the Node Exporter.
```
sudo systemctl start node_exporter
```
To create a static target, you need to add job_name with static_configs
```
sudo vim /etc/prometheus/prometheus.yml
```
**prometheus.yml**
```
- job_name: node_export
    static_configs:
      - targets: ["localhost:9100"]
```
![image](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/c20d2453-d4b9-4fc6-8d2b-699c88420552)

Since we enabled lifecycle management via API calls, we can reload the Prometheus config without restarting the service and causing downtime.
Before, restarting check if the config is valid.
```
promtool check config /etc/prometheus/prometheus.yml
```
Then, you can use a POST request to reload the config.
```
curl -X POST http://localhost:9090/-/reload
```
![image](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/e37ef3e9-bb9b-42e9-afbe-4d56423810d8)

*Install Grafana on Ubuntu*
To visualize metrics we can use Grafana. There are many different data sources that Grafana supports, one of them is Prometheus. First, let’s make sure that all the dependencies are installed.
```
sudo apt-get install -y apt-transport-https software-properties-common
```
Next, add the GPG key.
```
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
```
Add this repository for stable releases.
```
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
```
After you add the repository, update and install Garafana.
```
sudo apt-get update
```
```
sudo apt-get -y install grafana
```
To automatically start the Grafana after reboot, enable the service.
```
sudo systemctl enable grafana-server
```
Then start the Grafana.
```
sudo systemctl start grafana-server
```
Go to http://<ip>:3000 and log in to the Grafana using default credentials. The username is admin, and the password is admin as well.
username admin
password admin
To visualize metrics, you need to add a data source first.
![Screenshot 2024-03-13 181229](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/f4dd39c0-12a9-4b93-aad4-4b1c4c2b48af)

Click Add data source and select Prometheus.
or the URL, enter localhost:9090 and click Save and test. You can see Data source is working.
![Screenshot 2024-03-13 181345](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/5484bf2e-3f82-4060-ab41-24688de82bea)
Click on Save and Test.



















