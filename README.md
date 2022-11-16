
# GCP with Terraform and Jenkins (ITI Final Project)

Creating infrastructure and deployment process to implement and configue secure Google Container Cluster (GKE) on Google Cloud Platform (GCP) using Terraform (IaC).

I used a private VM Instance to connect to an all-private GKE Cluster. 

## Illustration
### Tools
- Terraform
- Google Cloud Platform (GCP)
- Jenkins
Each tool implementations are written in its section.

All work is applied on a single GCP project and region: us-central1.
Variables can be changed.

### Terraform & GCP
Backend bucket to store the state file as an object that can be accessed by the users working on the same project.

Network module is created that setups VPC Network with two subnets and the rest necessary configurations as shown below:
- Management subnet (Instance) / Cluster subnet (Cluster)
- NAT service
- Firewall that allows SSH and HTTP

Private VM Instance with a service account that has Kubernetes Cluster Admin permission attached with the Management subnet so I can use it to manage the cluster GKE.
Adding a start-up script that installs:
- Docker --> to build/push images.
- kubectl --> to apply kubernetes commands on the cluster.
- gcloud authentication plugin --> to extend kubectl’s authentication to support GKE.

Standard Google Container Cluster service with private control plane and working nodes, having a service account that has the Storage Admin permission. Giving the authorization to the created VM instance only.  

### Jenkins
In order to let Jenkins connect to the cluster, I have to give the app the required credentials and use the gcloud command.
So I created a Dockerfile with the jenkins image and added gcloud installation commands. Then I built the image and pushed it to Dockerhub so I can pull it from the deployment file.

Dockerhub jenkins image repository: https://hub.docker.com/repository/docker/shassem/jenkinsgcp 

Deployment file:
- Created a new namespace called "jenkins" for the jenkins app.
- Init Containers to install docker and kubernetes so I can use docker cli and kubectl in the pipeline.
- Mounting the docker.sock path so I can execute docker commands inside the application.
- Mounting jenkins_home to save all the configurations and details done in the jenkins app.
- Load balancer that listens on port 8080.

You may get encountered by an error when you open the jenkins app for the first time:
"HTTP ERROR 403 No valid crumb was included in the request".
This error may prevent the app to function properly. In order to fix this issue:
Skip admin creation (optional: if you are stuck) --> Enter "Manage Jenkins" --> "Configure Global Security" --> "Enable proxy compatibility".
Reference: https://stackoverflow.com/questions/44711696/jenkins-403-no-valid-crumb-was-included-in-the-request 

Credentials Configurations:
- Create a credential for your Dockerhub account.
- Secret file credentials that contains the VM service account key pair file in order to have access to the cluster.


## Setup
### Commands

#### Creating resources with Terraform

```bash
    cd Terraform
    terraform init                     #initializes a working directory and install plugins for google provider
    terraform plan                     #to check the changes
    terraform apply -auto-approve      #creating the resources on GCP
```
#### The VM Startup Script

Installing kubectl: 
```bash
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

    sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
Installing dockercli
```bash
    sudo apt-get update && sudo apt-get install ca-certificates curl gnupg lsb-release && sudo mkdir -p /etc/apt/keyrings

    curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

    echo   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
    $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    sudo apt-get update && sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```
Installing the gcloud authentication plugin
```bash
    sudo apt-get install google-cloud-sdk-gke-gcloud-auth-plugin
```

Installation References:
- https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
- https://docs.docker.com/engine/install/ubuntu/
- https://cloud.google.com/blog/products/containers-kubernetes/kubectl-auth-changes-in-gke

#### SSH into the VM instance
From the GUI --> ssh to the instance and then connect to the cluster
```bash
    gcloud container clusters get-credentials {cluster-id} --region {region} --project {project-id}
```
cluster-id = my-gke-cluster <br />
region = us-central1 <br />
project = neat-talent-367811

#### Building the Dockerfile for jenkins and pushing to Dockerhub
Create the Dockerfile: <br />
Then:
```bash
    sudo docker build -t shassem/jenkinsgcp
    sudo docker login
    sudo docker push shassem/jenkinsgcp
```
#### Deploying the jenkins app
Create the deployment.yml file <br />
```bash
    kubectl apply -f deployment.yml
```
Checking on the pods and getting the load balancer external IP
```bash
    kubectl get all -n jenkins
```
Output:

### Now you are ready to use Jenkins on a GKE cluster!🚀

## Connected Repository:

CI/CD using Jenkins --> https://github.com/shassem/ITI-FinalProject-Jenkins 