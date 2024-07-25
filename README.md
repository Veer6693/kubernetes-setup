# Deployment of JupyterHub on Kubernetes

This document provides a step-by-step guide for setting up a Kubernetes cluster on a virtual machine and deploying JupyterHub on it. It is intended for system administrators and DevOps engineers.

## Prerequisites

### Hardware Requirements

**Master Node(s):**
- CPU: At least 2 CPUs (4 CPUs recommended)
- Memory: 2 GB RAM (4 GB or more recommended)
- Disk: 20 GB of free disk space

**Worker Node(s):**
- CPU: At least 1 CPU (2 CPUs recommended)
- Memory: 1 GB RAM (2 GB or more recommended)
- Disk: 20 GB of free disk space

### Software Requirements

**Kubernetes:**
- Operating System: Linux (Ubuntu 20.04 LTS)
- Container Runtime: Docker, containerd, or CRI-O
- Kubernetes Components: kubeadm, kubectl, and kubelet
- Network Plugin (CNI): Flannel, Calico, Weave, or Cilium


### Kubernetes:
- Use official website to install all the tools like , kubectl , kubeadm etc. : [Link](https://kubernetes.io/docs/tasks/tools/)

- use official website to install helm : [Setup Helm](https://z2jh.jupyter.org/en/stable/kubernetes/setup-helm.html)

### Network Requirements

| Node | IP Address  | Hostname     |
|------|-------------|--------------|
| Master | 10.10.20.36 | k8s-master   |
| Worker | 10.10.20.32 | k8s-worker1  |

## Cluster Setup

### Set Hostname and Update Hosts File

- Run in master node
```bash
sudo hostnamectl set-hostname "k8s-master" 
```
- Run in worker node
```bash
sudo hostnamectl set-hostname "k8s-worker1"
```

### Next we update the file /etc/hosts of all nodes.
```bash
sudo apt install -y nano
sudo nano /etc/hosts
```
### Then add the following content below the file: 
![image](https://github.com/user-attachments/assets/153a6fd3-f726-4e6f-8c80-5b59d4af2790)

### Now, Disable swap and update kernel : 

- Now run this command to execute on all nodes.
```bash
sudo swapoff -a
```

- Then check if swap is disabled ot not from this command and see all the swap are 0 B ot not.
```bash
free -h
```
![image](https://github.com/user-attachments/assets/6eb94daa-b2a9-46c6-87d8-b4a0007e1a9e)

- Next, disable swap in /etc/fstab, then find (/swap.img none swap sw 0 0 ) this line and comment it.
```bash
sudo nano /etc/fstab
```
![image](https://github.com/user-attachments/assets/10320ab6-667a-4ef2-b2f0-929db17c05e3)

- Next mount that from this command and then check swap.
```bash
sudo mount -a
free -h
```

- Load the following kernel modules on all the nodes:
```bash
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
```
```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

- Set the following Kernel parameters for Kubernetes.
```bash
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```
- Then reload sysctl
```bash
sudo sysctl --system
```

### Install containerd run time : 

- Run the following commands on all nodes :
```bash
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install -y containerd.io
```

- Once installed, we add the containerd configuration.
```bash
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### Install Kubernetes :
*(Note : install all the dependencies/tools from the official documentation only. )*

**These instructions are for Kubernetes v1.30.**

1.	Update the apt package index and install packages needed to use the Kubernetes apt repository:
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

2.  Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

3. Add the appropriate Kubernetes apt repository. Please note that this repository have packages only for Kubernetes 1.30; for other Kubernetes minor versions, you need to change the Kubernetes minor version in the URL to match your desired minor version (you should also check that you are reading the documentation for the version of Kubernetes that you plan to install).
```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

4. Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version:
```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

5. (Optional) Enable the kubelet service before running kubeadm:
```bash
sudo systemctl enable --now kubelet
```

****Note : till here do all the steps on both node(master and worker***

### After installing all tools now initialize cluster using kubeadm.

- Run the following command below on master node
```bash
sudo kubeadm init --pod-network-cidr=10.10.0.0/16 
```
`In which 10.10.0.0/16 is the CIDR of the pod network, you can change it as needed.`

- If the run is successful, the result will be as follows :


 You’ll saw this type of command : 
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Run this command to start the cluster,
 
Now you also get kubeadm join command which is look like this
```bash
sudo kubeadm join k8s-master.nvtienanh.local:6443 --token daii9y.g4dq24u6irkz4pt0 \
    	--discovery-token-ca-cert-hash sha256:58b9cc96ed57a5797fddea653756dbda830efbff55b720a10cffb3948d489148 \
    	--control-plane
```
Copy this command and paste it on your worker so ur worker node is also join in the cluster.

- To check the cluster status run following command : 
```bash
kubectl cluster-info
kubectl get nodes
```

- In this you’ll see the nodes but all are in not ready stage, to get ready we have to add calico.yaml file, and run following command to apply that calico 
https://github.com/projectcalico/calico/blob/master/manifests/calico.yaml
```bash
kubectl apply -f calico.yaml
```

## For load balancing Deploy metalllb in k8’s : 

```bash
sudo apt install sipcalc
```
![image](https://github.com/user-attachments/assets/60492ee7-1d9a-4ef2-9d03-c8e3461cd5d3)
```bash
sipcalc 10.10.20.0/24
```
***Note: choose your available IPs***

*Note: If you’re using kube-proxy in IPVS mode, since Kubernetes v1.14.2 you have to enable strict ARP mode.*

you don’t need this if you’re using kube-router as service-proxy because it is enabling strict ARP by default.
- You can achieve this by editing kube-proxy config in current cluster:
```bash
kubectl edit configmap -n kube-system kube-proxy
```
- and set:
```
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
 strictARP: true
```

#### First , go to the official website and install 

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.7/config/manifests/metallb-native.yaml
```
```bash
nano metallb.yaml
```
`(create file and copy this)`
```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
- 10.10.20.240-10.10.20.250
```
- set the ip range for allocating to different apps

*(check the available range from your ips by using sipcalc)*

```bash
kubectl apply -f metallb.yaml
```

## 5.	Now, setting up dynamic NFS provisioning in k8’s

```bash
sudo apt install nfs-common
sudo apt install nfs-kernel-server
```
```bash
sudo mkdir /srv/nfs/kubedata -p
sudo chown nobody: /srv/nfs/kubedata
```
```bash
sudo nano /etc/exports
```
- changes are done in this file so that we can expose the location to everybody.

![image](https://github.com/user-attachments/assets/c48e94d5-1900-419b-9834-bf54a3e5cb58)

```bash
sudo exportfs -rav
```
![image](https://github.com/user-attachments/assets/369573cf-2b0a-45b7-bdc5-6b4b5bf51c1c)
- output should be displayed like this.

Nfs provisioner Link From Github:[Link](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/v4.0.2/deploy)
- All the required  yaml files are present in the above repository.
```bash
nano rbac.yaml
```
```bash
kubectl apply -f rbac.yaml
```
![image](https://github.com/user-attachments/assets/dfc65ca6-05e5-4def-a0fe-159362f5bd0a)
```bash
nano deployment.yaml
```
```bash
kubectl apply -f deployment.yaml
```
![image](https://github.com/user-attachments/assets/db516853-96d2-4640-860d-1b87a9591867)

```bash
nano 4-pvc-nfs.yaml 
```
```bash
kubectl apply -f 4-pvc-nfs.yaml
```
![image](https://github.com/user-attachments/assets/7ea78926-3534-467e-95bd-f6e1465e81d6)

## Jupyterhub Deployments :

### Check all the nodes are running fine, 
```bash
kubectl cluster info
kubectl get nodes
```

### Check all the pods(system) that are running or not, 
```bash
kubectl get pods -A
```

### Check metalllb is running fine  and also check dynamic NFS provisioning is running properly, which we did earlier
```bash
kubectl get all
```
 ### Also check storage class ,
```bash
kubectl get storageclass
```

### Now check helm is installed or not 
```bash
which helm
```

- if helm is not not present install using command below
```bash
curl https://raw.githubusercontent.com/helm/helm/HEAD/scripts/get-helm-3 | bash
helm version
```

### Now we have to install jupyterhub , but we use older version which is 0.11.x 

[https://z2jh.jupyter.org/en/0.11.x/jupyterhub/installation.html](https://z2jh.jupyter.org/en/0.11.x/jupyterhub/installation.html)

- Make Helm aware of the JupyterHub Helm chart repository so you can install the JupyterHub chart from it without having to use a long URL name.
```bash
helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
```
```bash
helm repo update
```

- Check heml reposetory list :
```bash
helm repo list
helm search repo jupyterhub
```
- Now pull the values file which is jupyterhub.yaml file : 
```bash
helm show values jupyterhub/jupyterhub > /tmp/jupyterhub.yaml
```
- Now before edit that yaml file generate secret token using following command and copy that token
```bash
openssl rand -hex 32
```
- Now open the yaml file and find proxy -> secret token then in this paste that token you copied
```bash
nano /tmp/jupyterhub.yaml
```
**`Note:search for storageClassName and StorageClass and change it to your desired storage class. If default storage class no need to mention if not please specify your storage class name.`**


### Now we use helm to deploy jupyterhub 
```bash
helm installl jupyterhub jupyterhub/jupyterhub –values /tmp/jupyterhub.yaml
```
- It will take some time to install..

### Now check helm list to ensure that jupyterhub is installed successfully 
```bash
helm list
helm status jupyterhub
```

### Now run and check all the pods are running fine: 
```bash
kubectl get all 
kubectl get pods 
kubectl get deployments
```

### Now, check the service/proxy-public that we set up with MetalLB. Find the IP it's using, and open that  to access JupyterHub from outside.
