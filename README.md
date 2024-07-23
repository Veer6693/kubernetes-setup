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
**Helm:**
- use official website to install helm : [Setup Helm](https://z2jh.jupyter.org/en/stable/kubernetes/setup-helm.html)

### Network Requirements

| Node | IP Address  | Hostname     |
|------|-------------|--------------|
| Master | 10.10.20.36 | k8s-master   |
| Worker | 10.10.20.32 | k8s-worker1  |

## Cluster Setup

### Set Hostname and Update Hosts File

On each node, set the hostname and update the hosts file:

```bash
sudo hostnamectl set-hostname "k8s-master"  # for master
sudo hostnamectl set-hostname "k8s-worker1"  # for worker
sudo apt install -y nano
sudo nano /etc/hosts
```


