# ArgoRollouts_Canary
I. Pre-requisite
=================
1. Install kubectl
curl -o kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.22.6/2022-03-09/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
kubectl version --short --client

2. Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version


3. Install kubectl argo plugin
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x ./kubectl-argo-rollouts-linux-amd64
sudo mv ./kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
kubectl argo rollouts version

4. Install git:
#Perform a quick update on your instance:
sudo yum update -y

#Install git in your EC2 instance
sudo yum install git -y

#Check git version
git version

5. Install docker
# Install Docker
sudo yum update -y
sudo yum -y install docker

# Start Docker
sudo service docker start

# Access Docker commands in ec2-user user
sudo usermod -a -G docker ec2-user
sudo chmod 666 /var/run/docker.sock
docker version

6. Install Istio:
curl -L https://git.io/getLatestIstio | sh -
sudo cp -v istio-1.12.0/bin/istioctl /usr/local/bin/

yes | istioctl install --set profile=demo


#7. Install terraform:


II. Creation of EKS cluster:
============================
#1. Configure aws account 
aws configure

#2. Create EKS cluster
eksctl create cluster --name eks-argo --version 1.21 --region us-east-2 --nodegroup-name linux-nodes --node-type m5.large --nodes 2
(or)
eksctl create cluster -f cluster.yaml //creation of cluster at PoC account.

III. Installation of Argo rollout controller in the EKS cluster
===============================================================
#Install Argo Rollout controller
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

#Install istioctl
curl -sL https://istio.io/downloadIstioctl | sh -
export PATH=$PATH:$HOME/.istioctl/bin

#Install istio
istioctl install --set profile=demo -y

IV. Deploy application image in apex namespace of EKS cluster:
==============================================================
#1. Create application namespace and enable istio injection
kubectl create namespace apex
kubectl label namespace apex istio-injection=enabled

#2. Option Step: Install istio addons and enable prometheus and grafana through a load balancer
kubectl apply -f addons/
kubectl apply -f prom-grafana.yaml //Can deploy only Prometheus services to visualize analysis step.(instead of both Prometheus and Grafana)

#3. Install gateway
kubectl apply -f gateway.yaml

#4. Install the target microservice
kubectl apply -f target-service-service.yaml
kubectl apply -f target-service-virtualservice.yaml
kubectl apply -f target-service-rule.yaml
kubectl apply -f target-service-analysis.yaml
kubectl apply -f target-service-rollout.yaml

(make sure to update the prometheus url in the analysis file)

V. Argo Rollout Analysis Steps:
===============================
#1. Testing of scenario
kubectl argo rollouts get rollout target-service-rollout -n apex --watch

#2. Change the version in the rollout file and apply again
kubectl apply -f target-service-rollout.yaml

#3. Promote canary to next step
kubectl argo rollouts promote target-service-rollout -n apex

#4. Postman call
GET http://a50e6f28f45ec4da2a4f95b338c1fa48-1446515496.us-east-2.elb.amazonaws.com/target/data //HTTP url can get from istio-ingress url from kubetcl get svc --all-namespaces
Header host = apex.pk.com

#5. Watch the changes
kubectl argo rollouts get rollout target-service-rollout -n apex --watch

#6. Command to check the analysis run data
kubectl describe analysisrun target-service-rollout-84ff7b74fb-4 -n apex
