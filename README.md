****Deploy a Node.js "Hello World" Application with Docker and Kubernetes
Using Jenkins****

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
![grafana_pass](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/2f57ee54-52f0-4152-8d5e-3edf9889058c)

To visualize metrics, you need to add a data source first.
![Screenshot 2024-03-13 181229](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/f4dd39c0-12a9-4b93-aad4-4b1c4c2b48af)

Click Add data source and select Prometheus.
or the URL, enter localhost:9090 and click Save and test. You can see Data source is working.
![Screenshot 2024-03-13 181345](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/5484bf2e-3f82-4060-ab41-24688de82bea)
Click on Save and Test. Let’s add Dashboard for a better view. Click on Import Dashboard paste this code 1860 and click on load.

*Step 5 — Install the Prometheus Plugin and Integrate it with the Prometheus server*
Need Jenkins up and running machine
Goto Manage Jenkins –> Plugins –> Available Plugins
Search for Prometheus and install it
![promethsis plugin](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/a6c9ccfa-edc7-415d-94b8-2f120b595776)
To create a static target, you need to add job_name with static_configs. go to Prometheus server
```
sudo vim /etc/prometheus/prometheus.yml
```
```
- job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<jenkins-ip>:8080']
```
![promethisconfig](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/df7765ac-0a1f-4512-958c-6037f5da3790)
Before, restarting check if the config is valid.
```
promtool check config /etc/prometheus/prometheus.yml
```
Then, you can use a POST request to reload the config.
```
curl -X POST http://localhost:9090/-/reload
```
![Screenshot 2024-03-13 181951](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/ab5ca589-ed30-4886-b892-20264a513667)

Let’s add Dashboard for a better view in Grafana. Click On Dashboard –> + symbol –> Import Dashboard. Use Id 9964 and click on load. Select the data source and click on Import
![Grafna jenkins](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/acbec9a4-0327-4f58-8f57-0779b634e237)
Now you will see the Detailed overview of Jenkins
![nkingradna](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/bb3e4618-8162-490c-9f24-6b3d6bee1f86)

*Step 5 — Email Integration With Jenkins and Plugin Setup*
- Go to your Gmail and click on your profile
- Then click on Manage Your Google Account –> click on the security tab on the left side panel you will get this page(provide mail password).
- 2-step verification should be enabled.
- Search for the app in the search bar you will get app passwords like the below image
- Click on other and provide your name and click on Generate and copy the password
- In the new update, you will get a password like this
Once the plugin is installed in Jenkins, click on manage Jenkins –> configure system there under the E-mail Notification section configure the details as shown in the below image
![email](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/ad283101-a015-45fb-97bc-7d696b0ef996)
![email2](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/4dcf525b-92f0-472e-badf-5db4fdefc5ec)
![email3](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/822e823c-d268-4f5e-b088-b55b9b92dc36)
Click on Apply and save. Click on Manage Jenkins–> credentials and add your mail username and generated password
![mail4](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/416e2304-0d08-4e39-8b50-5f693aa9263b)
Now under the Extended E-mail Notification section configure the details as shown in the below images.
![email5](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/b09b07a2-ce80-4f0d-b04c-81c049da5200)
![mail6](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/753595de-b8dc-4a63-8df6-07b8b46e16f2)

*Step 6 — Install Plugins Sonarqube Scanner*
Goto Manage Jenkins →Plugins → Available Plugins →
SonarQube Scanner (Install without restart)
![onar_plugin](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/29c65abd-3a38-4091-9a46-30312e6385b4)

*Step 7 — Configure Sonar Server in Manage Jenkins*
Grab the Public IP Address of your EC2 Instance, Sonarqube works on Port 9000, so <Public IP>:9000. Goto your Sonarqube Server. Click on Administration → Security → Users → Click on Tokens and Update Token → Give it a name → and click on Generate Token
click on update Token. Create a token with a name and generate.copy Token
Goto Jenkins Dashboard → Manage Jenkins → Credentials → Add Secret Text. It should look like this
![sonar-token1](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/f4a1c4aa-c807-43a0-9f93-d94fd57f19a8)
Now, go to Dashboard → Manage Jenkins → System and Add like the below image.
![sonar-serverss](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/3ecbe059-afde-4634-98b6-c390761c22c0)
Click on Apply and Save. The Configure System option is used in Jenkins to configure different server.
Global Tool Configuration is used to configure different tools that we install using Plugins. We will install a sonar scanner in the tools.
In the Sonarqube Dashboard add a quality gate also Administration–> Configuration–>Webhooks
![sonar-webhook](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/26f38a01-9e71-4a64-8a49-4cfd3ecee102)
create sonar project
![sonar-project](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/6e4965b6-fcf8-48fb-84a7-3f0d536bb0de)

*Step 8 — Docker Image Build and Push*
We need to install the Docker tool in our system, Goto Dashboard → Manage Plugins → Available plugins → Search for Docker and install these plugins
![dockerplugin](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/f3734f51-3feb-4505-8003-134633f907bd)
Add DockerHub Username and Password under Global Credentials

*Step 9 — Kuberenetes Setup*
Take-Two Ubuntu 20.04 instances one for k8s master and the other one for worker. Install Kubectl on Jenkins machine also. Kubectl is to be installed on Jenkins also
```
sudo apt update
sudo apt install curl
curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```
Part 1 ———-Master Node————
```
sudo hostnamectl set-hostname K8s-Master
```
———-Worker Node————
```
sudo hostnamectl set-hostname K8s-Worker
```
Part 2 ————Both Master & Node ————
```
sudo apt-get update
sudo apt-get install -y docker.io
sudo usermod –aG docker Ubuntu
newgrp docker
sudo chmod 777 /var/run/docker.sock
```
```
$ sudo apt-get install -y apt-transport-https ca-certificates curl gpg
$ curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
$ echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
$ sudo apt-get update
$ sudo apt-get install -y kubelet kubeadm kubectl
$ sudo apt-mark hold kubelet kubeadm kubectl
```
Part 3 ————— Master —————
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
# in case your in root exit from it and run below commands
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
———-Worker Node————
```
sudo kubeadm join <master-node-ip>:<master-node-port> --token <token> --discovery-token-ca-cert-hash <hash>
```
Copy the config file to Jenkins master or the local file manager and save it
![kube config](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/863904c1-1d05-4e3e-98e8-461cf6c3fadb)
copy it and save it in documents or another folder save it as secret-file.txt
create a secret-file.txt in your file explorer save the config in it and use this at the kubernetes credential section.
Install Kubernetes Plugin, Once it’s installed successfully.
![kubentees_plugin](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/9433d044-037b-482d-b45f-21c6ceacbce4)
goto manage Jenkins –> manage credentials –> Click on Jenkins global –> add credentials
![kube_configs](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/da7fccb9-3ad0-4fb9-b438-b1e127425089)
Install Node_exporter on both master and worker
Let’s add Node_exporter on Master and Worker to monitor the metrics
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
```
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
```
Move binary to the /usr/local/bin.
```
sudo mv \
  node_exporter-1.6.1.linux-amd64/node_exporter \
  /usr/local/bin/
```
Clean up, and delete node_exporter archive and a folder
```
rm -rf node_exporter*
```
Next, create a similar systemd unit file.
```
sudo vim /etc/systemd/system/node_exporter.service

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
Replace Prometheus user and group to node_exporter, and update the ExecStart command. To automatically start the Node Exporter after reboot, enable the service.
```
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```
we have only a single target in our Prometheus. There are many different service discovery mechanisms built into Prometheus. For example, Prometheus can dynamically discover targets in AWS, GCP, and other clouds based on the labels. In the following tutorials, I’ll give you a few examples of deploying Prometheus in a cloud-specific environment. For this tutorial, let’s keep it simple and keep adding static targets. Also, I have a lesson on how to deploy and manage Prometheus in the Kubernetes cluster. To create a static target, you need to add job_name with static_configs. Go to Prometheus server
```
sudo vim /etc/prometheus/prometheus.yml
```
```
- job_name: node_export_masterk8s
    static_configs:
      - targets: ["<master-ip>:9100"]
  - job_name: node_export_workerk8s
    static_configs:
      - targets: ["<worker-ip>:9100"]
```
![promethiscokd](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/2150797d-a721-43c5-b134-39a6f8021263)
Since we enabled lifecycle management via API calls, we can reload the Prometheus config without restarting the service and causing downtime.
Before, restarting check if the config is valid.
```
promtool check config /etc/prometheus/prometheus.yml
```
Then, you can use a POST request to reload the config.
```
curl -X POST http://localhost:9090/-/reload
```
![pormethes](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/27835b06-8cec-45c2-889e-24487f8845d7)

create jenkins pipeline 
add configuration, check mark on github trigger for automatic build pipeline when whenever change in code. check mark on poll scm every day midnight

![jenkin time shcdeule](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/b7949cdf-0dfe-4c08-b4df-db5a11e4647f)
select scm add you gihub reposiotry. save and apply 
build the pipeline
![jenkins job](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/faee4f6c-0b4e-4fac-97a9-456873b9eeec)

run kubectl get all command on master
hello_world app success run on kubernetes cluster with sonarqube code analysis, scan docker image using trivy and recieved mail pipeline build successfully. 
![kubedelpypods](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/4f2feb6d-a6e7-4435-af70-8e74ac339f0a)
![codereviewsonar](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/b0cf7313-576b-46de-8325-b6a9b83fe9ab)
![image](https://github.com/nikhilk1699/nodejs_helloworld/assets/109533285/c941bf71-b846-440b-9cad-a096b012e763)
















