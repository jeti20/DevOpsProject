![image](https://github.com/jeti20/DevOpsProject/assets/61649661/28ba9c6e-ec49-44af-90a0-c7c2e2920c20)# DevOpsProject
Architcture: 
![image](https://github.com/jeti20/DevOpsProject/assets/61649661/5b4c0690-6251-4d87-9ca6-712c2087c049)

Technologies: Jenkins, Maven, SonarQube, AquaTrivy, NexusRepository, Docker, Kubernete, Kubeaudit, Prometheus, Grafana, Email, Jira, Github

====================================================AWS SetUp=======================================

 On AWS check if you have defoult VPC Available
 ![image](https://github.com/jeti20/DevOpsProject/assets/61649661/a0459836-fe59-4756-b932-e31a11e5dd4b)

Beofre Creating EC2 Instances create SecurityGroup and edit Inbound rules
![image](https://github.com/jeti20/DevOpsProject/assets/61649661/05b42ef1-0cda-475e-a611-6269dabb9495)

30000 - 32767 ->  because we will use many VM in klaster
465 -> want t sent email notification to gmail address
6443 -> needed to set up Kubernetes Kluster
22 -> SSH
443 -> SMTPS https
80 -> http
3000-10000 -> aplikacje będę tego wymagały
25 -> SMTP, nie używamy tego

Create 3 EC2 Instances (MasterNode must be at least medium instance, because it must have 2cp and some extra ram, if not "sudo kubeadm init --pod-network-cidr=10.244.0.0/16" this command cannot be executed). Ubuntu, free trail, select SecurityGroup which you created before + KeyPair for SSH connection. Select 25GiB
![image](https://github.com/jeti20/DevOpsProject/assets/61649661/c7af3765-6555-4698-870a-19733a24e086)

![image](https://github.com/jeti20/DevOpsProject/assets/61649661/4d87c24b-e146-4b4a-8447-33293de98f11)


In MobaXTerm add created Instances
![image](https://github.com/jeti20/DevOpsProject/assets/61649661/f2514cd3-5fc3-4b73-bb53-49139b8dd576)

Go into settings in Moba and check this box to not get logged out after some time of afk 
![image](https://github.com/jeti20/DevOpsProject/assets/61649661/71c2590d-699a-44d6-a910-40461e5d27a5)

======================================================================================Creating/Set up the Kubernetes Cluster==============================================
Log in Master server, create new file in which u reate script for all these commands

```
vim 1.sh
```

```
sudo apt-get update

sudo apt install docker.io -y
sudo chmod 666 /var/run/docker.sock

sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
sudo mkdir -p -m 755 /etc/apt/keyrings

sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
sudo mkdir -p -m 755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

![image](https://github.com/jeti20/DevOpsProject/assets/61649661/3ce9d634-7911-44bb-a19a-6da8a8688427)

Make this file executable ```chmod +x 1.sh```, and execute it ```./1.sh``` -> do this on Mater and Slaves. You can use "MultiExec" option in mobaXTerm to write the same thing in multiple terminals 
![image](https://github.com/jeti20/DevOpsProject/assets/61649661/9b3ad117-d36e-4bf3-a444-fbb6d49316d2)
![image](https://github.com/jeti20/DevOpsProject/assets/61649661/d07f9fd5-5e1b-4eab-aea7-82fbe057cdd2)

Next step is to Update Package List

```
sudo apt update
```

Install on Master and Slaves:
1. Kubeadm is a tool used for bootstrapping Kubernetes clusters, simplifying tasks like provisioning master and worker nodes and configuring networking. Is goign to set up Kubernetes Cluster
2. kubelet: Kubelet is the primary "node agent" in a Kubernetes cluster responsible for managing container lifecycle, including starting, stopping, and monitoring containers, as well as communicating with the Kubernetes master node. Creating ports to which we are deploying application
3. kubectl: Kubectl is a command-line tool used for interacting with Kubernetes clusters. It allows users to deploy and manage applications, inspect cluster resources, and troubleshoot issues from the command line interface. CLI to interact with Cluster

```
sudo apt install -y kubeadm=1.28.1-1.1 kubelet=1.28.1-1.1 kubectl=1.28.1-1.1
```

**Initialize Kubernetes Master Node**
Now focus onlu on the Master node - the command "sudo kubeadm init --pod-network-cidr=10.244.0.0/16" initializes the Kubernetes cluster setup while configuring the pod network to use the specified CIDR range. CIDR stands for Classless Inter-Domain Routing. It's a method used for specifying IP addresses and their routing properties. In the context of Kubernetes, specifying a CIDR range with --pod-network-cidr option during cluster initialization defines the range of IP addresses that Kubernetes will assign to pods within the cluster. This is crucial for networking configuration within the Kubernetes cluster.

```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
You should see token and ha256
![image](https://github.com/jeti20/DevOpsProject/assets/61649661/877dbc84-5afa-4d2b-9375-0c88d562442a)
```
kubeadm join YourIP:6443 --token YourToken \
        --discovery-token-ca-cert-hash sha256:Yoursha
```
Copy and past it into Slaves nodes to connect them to Master node. You should see 
![image](https://github.com/jeti20/DevOpsProject/assets/61649661/74181416-4c7a-4f37-b52f-5c62809b226f)

So bassicly, you just created Cluster - Master node and 2 Slaves node which are conected to Master Nodes by the kubeadm

Now on Master Node execute these comands. This folder will contain config file for Kubernetes. 
These commands create the ~/.kube directory in the user's home directory, copy the Kubernetes cluster configuration file from /etc/kubernetes/admin.conf to ~/.kube/config, and then change the ownership of the config file to the current user and group. This enables the user to interact with the Kubernetes cluster locally using kubectl without needing to use sudo, simplifying Kubernetes cluster management.
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Deploy Networking Solution (Calico) [On MasterNode]
```
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```
Deploy Ingress Controller (NGINX) [On MasterNode]
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.49.0/deploy/static/provider/baremetal/deploy.yaml
```
The first command kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml applies the Calico network configuration to the Kubernetes cluster, providing network functionality to the cluster.

The second command kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.49.0/deploy/static/provider/baremetal/deploy.yaml installs the Ingress Nginx controller for managing HTTP and HTTPS traffic within the Kubernetes cluster, including handling incoming requests and routing them to the appropriate services.


```
Setup K8-Cluster using kubeadm [K8 Version-->1.28.1]
1. Update System Packages [On Master & Worker Node]
sudo apt-get update
2. Install Docker[On Master & Worker Node]
sudo apt install docker.io -y
sudo chmod 666 /var/run/docker.sock
3. Install Required Dependencies for Kubernetes[On Master & Worker Node]
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
sudo mkdir -p -m 755 /etc/apt/keyrings
4. Add Kubernetes Repository and GPG Key[On Master & Worker Node]
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
5. Update Package List[On Master & Worker Node]
sudo apt update
6. Install Kubernetes Components[On Master & Worker Node]
sudo apt install -y kubeadm=1.28.1-1.1 kubelet=1.28.1-1.1 kubectl=1.28.1-1.1
7. Initialize Kubernetes Master Node [On MasterNode]
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
8. Configure Kubernetes Cluster [On MasterNode]
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
9. Deploy Networking Solution (Calico) [On MasterNode]
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
10. Deploy Ingress Controller (NGINX) [On MasterNode]
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.49.0/deploy/static/provider/baremetal/deploy.yaml
```

