# Jenkins Pipeline for Java based application using Maven, SonarQube, Argo CD, Helm and Kubernetes

![New Microsoft PowerPoint Presentation](https://github.com/user-attachments/assets/dec01c09-8e5d-4861-9c5a-26e1114a86b7)

Here are the step-by-step details to set up an end-to-end Jenkins pipeline for a Java application using SonarQube, Argo CD, Helm, and Kubernetes:

Requirements:
Java application code hosted on a Git repository, Jenkins server, Kubernetes cluster, Helm package manager, Argo CD

Steps:
1. Create a t2.large Ubuntu Instance on AWS, this will serve Jenkins, Maven, SonarQube, Docker. (We will not be using a free teir resource as the jenkins server and sonarqube will be very resource intensive hencce we choose a t2.large instance)
![image](https://github.com/user-attachments/assets/833c7dce-6322-4a65-8c22-ddd6cbccee4d)


2. Install Java and Jenkins

### Run the below commands to install Java and Jenkins

Install Java

```
sudo su -
sudo apt update
sudo apt install openjdk-17-jre
```

Verify Java is Installed

```
java -version
```

Now, you can proceed with installing Jenkins

```
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
```

![image](https://github.com/user-attachments/assets/11af656c-f180-4bd2-9f9d-ad56b07c4a49)


**Note: ** By default, Jenkins will not be accessible to the external world due to the inbound traffic restriction by AWS. Open port 8080 in the inbound traffic rules as show below.

- EC2 > Instances > Click on <Instance-ID>
- In the bottom tabs -> Click on Security
- Security groups
- Add inbound traffic rules as shown in the image (you can just allow TCP 8080 as well, in my case, I allowed `All traffic`).

Check if jenkins is running
```
ps -ef | grep jenkins
```

Login to Jenkins, 
- Run the command to copy the Jenkins Admin Password - 
```
 sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

![image](https://github.com/user-attachments/assets/f6220134-9834-4698-8a48-3d24c07f6722)

- Enter the Administrator password
- Click on Install suggested plugins
      
3. In Jenkins:

New Item -> Pipeline (Grovy Scripting) -> Pipeline Configuration -> Pipeline script from SCM -> Add the Path

![image](https://github.com/user-attachments/assets/c292fa87-56d4-45ec-ab76-9f950c445bf2)

4. Install Sonarqube

### Install the Sonarqube Plugin in Jenkins

![image](https://github.com/user-attachments/assets/cbde73da-0dd5-4c6d-b5f7-d4a09a5b7b1c)

### Configure a Sonar Server

```
apt install unzip
adduser sonarqube
sudo su - sonarqube
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.4.0.54424.zip
unzip *
chmod -R 755 /home/sonarqube/sonarqube-9.4.0.54424
chown -R sonarqube:sonarqube /home/sonarqube/sonarqube-9.4.0.54424
cd sonarqube-9.4.0.54424/bin/linux-x86-64/
./sonar.sh start
```

Now access the `SonarQube Server` on `http://<ip-address>:9000` 

Generate a Sonarqube token through My Account -> Security -> Generate Token -> Copy -> Move to Jenkins -> Manage Credentails -> System -> Global Crdentails -> add credentails -> Secret Text -> Paste the token

5. Install Docker

Logout and go to root user and use the command:

```
sudo apt install docker.io
```
### Grant Jenkins user and Ubuntu user permission to docker deamon.

```
sudo su - 
usermod -aG docker jenkins
usermod -aG docker ubuntu
systemctl restart docker
```

Once you are done with the above steps, it is better to restart Jenkins.

```
http://<ec2-instance-public-ip>:8080/restart
```

The docker agent configuration is now successful.

**Note: ** Use Docker Containers as Agent

Reason: Traditionally, when using Jenkins, you would need multiple worker EC2 instances (agents) to handle various build and test jobs. These EC2 instances would often remain running and consume resources even when not in use. This approach leads to unnecessary costs and underutilized resources since EC2 instances are typically up and running continuously. Even if you use Jenkins auto-scaling with EC2 instances, there's still overhead involved. Each new EC2 instance would require configuration, such as installing the necessary tools, dependencies, and build environment settings. This process can be time-consuming and error-prone, especially when scaling up or down frequently. Hence using Docker as Agent is better as it ensures Dynamic provisioning, Resource efficiency, Fast Provisioning, Consistency, Scalability, Portability.

### Install Docker Plugin
![image](https://github.com/user-attachments/assets/e682cbd0-dd8d-40c1-a59b-01ac732a2633)
 
We use the abhishekf5/maven-abhishek-docker-agent:v1 image as the agent, it already contains maven

6. Create a t2.large Ubuntu Instance on AWS, this will serve MiniQube and ArgoCD.

![image](https://github.com/user-attachments/assets/69c64034-009c-4d9e-990e-79dcfbba6ad0)

7. Install Minikube(K8) on the new instance

Run the following commands:
```
sudo su -
sudo apt update
```

### Install Docker
```
sudo apt install docker.io
```

### Install Minikube
```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
```

### Setup Minikube
First Exit Root User and run these commands to install driver and give the permissions to group
```
sudo usermod -aG docker $USER && newgrp docker
minikube start --driver=docker
```
![image](https://github.com/user-attachments/assets/8e027877-6079-4045-80b9-34a4bb236111)

### Install Kubectl
```
sudo snap install kubectl --classic
```

8. Installing Kubernetes Controller(ArgoCD) through Operators

**Note: ** Installing Kubernetes Controllers through Operators is a best practice because Operators automate the management and lifecycle of Kubernetes controllers/applications, including deployment, scaling, upgrading, and recovery. Operators provide automated management of controllers, handling installation, upgrades, configuration changes, and health monitoring without manual intervention. We will use operatorhub.io 

### Install ArgoCD Operator

![image](https://github.com/user-attachments/assets/8f37780a-5be5-4b03-b39d-18a2a03d3d56)


Install Operator Lifecycle Manager (OLM), a tool to help manage the Operators running on your cluster.
```
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.28.0/install.sh | bash -s v0.28.0
```
This is one time for an instance
![image](https://github.com/user-attachments/assets/4381ccbc-7674-46ce-a15f-cf4f98137c48)

Install the operator by running the following command:
```
kubectl create -f https://operatorhub.io/install/argocd-operator.yaml
```
This Operator will be installed in the "operators" namespace and will be usable from all namespaces in the cluster. (This basically installs the ArgoCD Operator)


Verify the install using this:
```
kubectl get pods -n operators
```
![image](https://github.com/user-attachments/assets/7e71628e-3b10-45d4-be3b-4c65d8f0f51d)

9. Make the Jenkins File/Pipeline

**Note: ** Now what are the different stages in a Jenkins Pipeline, they are nothing but the different blocks that we are trying to build using the pipeline.

First, Checkout Stage (which is not required in our case as our jenkins file is already in the scm, if it was not in the scm and written in Jenkins UI then we would need the checkout stage)

Second, Build and Test, as maven is already installed in the docker container we only need to run
```
mvn clean package
```
After executing the command, a target folder is created which stores the web archive file and the docker file is just configured to take the jar file and run it on port 8080

Third, Static Code Analysis, we have configured sonar token and we will execute the static code analysis using the command:
```
mvn sonar:sonar -Dsonar.login=$Sonar_AUTH_TOKEN -Dsonar.host.url=${Sonar_URL}
```

Fourth, Docker Build and Push, we will be passing docker credentials, building the image and pushing to DockerHub using the ususal docker commands.

Fifth, Update Deployment File, in this stage a shell script will be executed to to replace the image tag in deployment.yaml with with the currecnt build number/version number of the image. It requires github credential as we are then pushing the new deployment file to the scm.
```
sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" CI-CD-PIPELINE-SETUP-WITH-JENKINS-DOCKER-AND-ARGO-CD/spring-boot-app-manifests/deployment.yml
```

### Add Docker and GitHub Credentials

Add the credentials in the same way we added the SonarQube credentials
![image](https://github.com/user-attachments/assets/191e9d61-28b0-40b3-972a-6e5b4ce1c4eb)

10. Run The CI Pipeline (Without the ArgoCD Part)

![image](https://github.com/user-attachments/assets/c80fa9d1-2252-4ad4-92c8-9f5178ca1b67)

11. Deployment thorough ArgoCD

### Setting up ArgoCD Workloads
Check Minikube status
```
minikube status
```
![image](https://github.com/user-attachments/assets/3f3f6303-3035-4291-92c6-b5f9152d9735)

Go to ArgoCD Operator Documentation
```
https://argocd-operator.readthedocs.io/en/latest/
```
Go to usage -> Basics

![image](https://github.com/user-attachments/assets/f2fca86e-c0e2-4865-90ae-8e0c5e64dc91)

Create a yaml file: argocd-basic.yaml file (We are trying to create the ArgoCD Controller)
```
vim argocd-basic.yaml
```
and copy paste the basic code:
```
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: example-argocd
  labels:
    example: basic
spec: {}
```
and apply command:
```
kubectl apply -f argocd-basic.yaml
kubectl get pods
```
![image](https://github.com/user-attachments/assets/d07f1483-2183-4d40-af58-8f50ec5f6685)

Now the ArgoCD workloads are created

### Accesing the Workloads

To access the workloads(on browser):
```
kubectl get svc
```
![image](https://github.com/user-attachments/assets/16626173-deb6-4bb3-aceb-a9684916b1a3)

The ```example-argocd-server``` is responisble for the ArgoCD UI

Now to Access this:
```
kubectl edit svc example-argocd-server
```

and edit type from ClusterIP to NodePort

![image](https://github.com/user-attachments/assets/1b1a6401-4ab0-4113-8613-1e20ffd8a1db)

Now we can see that example-argocd-server has changed to NodePort mode

![image](https://github.com/user-attachments/assets/6d8f4f48-4056-4d7a-ab30-64062e665113)

Add Inbound Rules for the exposed ports

![image](https://github.com/user-attachments/assets/d0aa5227-53c7-4062-a196-8f82b53d4149)


Execute the command:
```
minikube service example-argocd-server
minikube service list
```

![image](https://github.com/user-attachments/assets/5e918114-7634-409a-b35f-fd4ad06d30cb)

**Note: ** When running Minikube inside an EC2 instance, services deployed within Minikube are not directly accessible from outside the EC2 instance. This is because Minikube runs as a separate VM inside the EC2 instance, with its own internal network. Port forwarding is needed to expose the ArgoCD UI service running inside Minikube to the EC2 instance's public IP. Without port forwarding, the service would remain inaccessible from outside the EC2 instance.

### Access ArgoCD UI
Run the following commands on the instance to forward the traffic:
```
kubectl port-forward --address 0.0.0.0 svc/example-argocd-server 32043:80 &
kubectl port-forward --address 0.0.0.0 svc/example-argocd-server 32665:443 &

```
These commands create tunnels between the ArgoCD server service inside Minikube and the EC2 instance's public IP.

![image](https://github.com/user-attachments/assets/a6d20c43-b4a2-43dc-adfa-52d18105e00e)

After running these commands, you can access the ArgoCD UI using the EC2 instance's public IP:
```
HTTP: http://<EC2_PUBLIC_IP>:32043
HTTPS: https://<EC2_PUBLIC_IP>:32665
```
![image](https://github.com/user-attachments/assets/97b00822-7450-4651-97d8-7bbc43baf0b3)

Now Login to ArgoCD using admin as username and for password run the command:
```
kubectl get secret
kubectl edit secret example-argocd-cluster
```
Copy the admin password and use it

![image](https://github.com/user-attachments/assets/8440c9dc-9cfa-4174-bfc6-964e0097e4fc)

Now decrypt it using:
```
echo <copied admin pass> | base64 -d
```
![image](https://github.com/user-attachments/assets/44f99e50-7bf0-4c2a-86b8-d84941d7aa49)

Copy and login

![image](https://github.com/user-attachments/assets/e8975ac1-1ffb-481a-ae83-4dd6831fd736)

### Deployment

Create Application

![image](https://github.com/user-attachments/assets/affcd7bd-cab6-47c6-9566-17552b3cd6c5)

Then provide the URL

![image](https://github.com/user-attachments/assets/f6f2f76a-7878-4808-89ba-c7e39cf521ff)
(Edit: No need to write Deployment.yaml in the path)
(Edit: Under namespace add default)

And click on create

![image](https://github.com/user-attachments/assets/dec04c1a-dab6-423e-bf1b-7e1c08e59231)

Your application is created

![image](https://github.com/user-attachments/assets/a1d103bf-a3d0-4797-ab74-667c73038b69)

To Check:
```
kubectl get deploy
```

**Note: ** ArgoCD offers several key advantages, including automatic synchronization of application changes from Git repositories, ensuring Kubernetes deployments are always up-to-date with the desired state. It employs a declarative approach, simplifying application management through GitOps principles. It also protects against malicious users by enforcing strict RBAC (Role-Based Access Control) policies and integrating with security tools to ensure only authorized users can make changes, thereby securing the deployment process.

![image](https://github.com/user-attachments/assets/56a86906-54ee-4ca0-91ab-a10fedca3309)
 
