---
title: "Kubernetes on Rocky Linux with Containerd"
date: 2022-02-28T17:44:22-07:00
draft: false
---

This guide shows steps to install Kubernetes on Rocky Linux using kubeadm. I chose to use Rocky Linux for it's stability and RedHat compatibility. For my lab, I am installing Kubernetes with 1 master and 2 worker nodes, but you can install as many master and worker nodes as you would like, including just 1 node.

I have chosen to use containerd over Docker and CRI-O due to it's simplicity and lightweight nature. I will not be needing to use any docker specific commands as all containers should be running in the Kubernetes system.

## System Requirements

- At least 2 cores per node
- 4GB of RAM per master node
- 2GB of RAM per worker node
- 50GB of storage per node

These are not hard system requirements, so feel free to try running this with lower system specifications. These are the lowest requirements I feel comfortable running.

## Rocky Linux Preparation

#### Installing Rocky Linux

Rocky Linux can be downloaded from the official Rocky Linux Site: https://rockylinux.org/

For this guide, I am using Rocky Linux 8.5 with the Minimal ISO.

1. Boot to the Rocky Linux installer
   - I am using a mixture of Unraid and ESXi as a KVM hosts. In this case, I create a new VM, and mount the ISO file as a virtual disk. 
2. For Installation Destination, select Custom, then Done
3. I changed the partitioning scheme to Standard Partition over LVM, as I am more familiar with expanding these type of volumes if need be later on.
4. Select 'Click Here to create them automatically.'
5. Delete the swap partition and home partition, and set the desired capacity of / to the largest available size.
   - I kept the File System as xfs, but you can switch to ext4 here as well.
   - Removing the swap partition is required for kubernetes
   - Removing the home partition allows everything to be under the root directory, making storage management easier.
6. For Software Selection, I chose 'Minimal Install' as this used the least amount of resources.
   - We will be installing all required packages manually later on
7. For Network & Host Name, set a host name (unique for each node), and turn on the Ethernet adapter.
   - I chose kubemaster1, kubenode1, and kubenode2
   - I also set my router's DHCP server to always assign the same IP address to each node.
8. Create a root user password, and standard user. Select install, and wait until Rocky Linux is fully installed. Select Reboot. Repeat this installation for each node.

## Install Prerequisites 

#### SSH

I find it easier to work with the VM over SSH instead of a virtual monitor. Rocky Linux minimal install comes with OpenSSH enabled by default, so you can SSH into the VM.

#### Installing Container Runtime

##### Update system, patch firewall, and set SELinux to permissive

```bash
sudo dnf update -y

sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=10251/tcp
sudo firewall-cmd --permanent --add-port=10252/tcp
sudo firewall-cmd --permanent --add-port=10255/tcp
sudo firewall-cmd --reload

sudo setenforce 0
sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/g' /etc/selinux/config
```

##### Enable overlay and netfilter (required for Kubernetes and containerd networking), and configure sysctl

```bash
sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

sudo sysctl --system
```

##### Add official docker repository, and install containerd

(We aren't installing docker here, but the containerd package is located in the docker repository)

```bash
sudo yum install -y yum-utils iproute-tc
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y containerd.io
```

##### Set up containerd configuration

```bash
sudo mkdir -p /etc/containerd 
sudo containerd config default > config.toml
sudo mv config.toml /etc/containerd/config.toml
```

##### Restart containerd and verify it is running

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd
```

Check to make sure that the containerd service is active

#### Installing Kubernetes Prerequisites

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

Now that we have containerd, our container backend installed and working, let's install kubernetes tools

##### Add kubernetes repository, install kubelet, kubeadm, kubectl, and enable kubelet

Make sure to accept any keys on the update step

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

sudo yum update
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```

Here is a good time to reboot, and run the same steps on each worker node.

```bash
sudo reboot
```

## Install Kubernetes 

#### Installing Kubernetes Master Node

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/ 

Run these steps only on the master node

##### Bootstrap the kubernetes master using kubeadm

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

We are using this pod network cidr in particular for installing flannel, our kubernetes networking CNI

This step may take a while, once it finishes, you should see this message:

+ Your Kubernetes control-plane has initialized successfully!

##### If this command fails, do not run kubeadm init again. Instead, you must first reset kubeadm by running this command, then fix any errors and try again

```bash
sudo kubeadm reset
```

##### Take note of your kubeadm join command. If you lose it, you can also regenerate this command by using

```bash
sudo kubeadm token create --print-join-command
```

##### Move kubectl config to the .kube directory

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### Installing Kubernetes Worker Nodes

##### On all nodes, run the kubeadm join command from the end of the master node setup

```bash
sudo kubeadm join 111.111.111.111:6443 --token my.token --discovery-token-ca-cert-hash sha256:mycerthash
```

##### On the master node, run this command and verify all nodes show up

Note that the nodes will show as 'NotReady' until the CNI has been installed

```bash
kubectl get nodes
```

#### Install networking CNI (Flannel)

https://github.com/flannel-io/flannel 

##### On the master node (or anywhere you have kubectl installed, and the config file placed in ~/.kube/config), run the following commands

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

kubectl get pods --all-namespaces
```

Wait until the coredns pod is running and ready

##### Once the codedns pod is running, verify that all nodes status is 'Ready'

```bash
kubectl get nodes
```

#### Remove Master Taint (if only using 1 node)

Removing the master node taint allows us to run pods on the master node. If you only have 1 node, you will want to run everything on it, so the taint must be removed.

##### Run this command on the master (or anywhere kubectl is configured to connect to the cluster)

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

Note that the command ends with master- with the - meaning remove

That means, you can re-taint the node by running:

```
kubectl taint node nodename node-role.kubernetes.io/master=true:NoSchedule
```

From here, your kubernetes cluster should be fully functioning.

## Install Additional Kubernetes Tools

##### Install k9s, a cli based kubernetes dashboard

https://github.com/derailed/k9s 

```bash
wget https://github.com/derailed/k9s/releases/download/v0.25.18/k9s_Linux_x86_64.tar.gz 
tar -xzf k9s_Linux_x86_64.tar.gz 
chmod +x k9s 4sudo mv k9s /usr/local/bin/k9s
```

##### Install kubectl-aliases, a programmatically generated list of kubectl aliases

https://github.com/ahmetb/kubectl-aliases 

```bash
wget https://raw.githubusercontent.com/ahmetb/kubectl-aliases/master/.kubectl_aliases 
vim ~/.bashrc
```

##### Add the following to the end of the file:

```bash
[ -f ~/.kubectl_aliases ] && source ~/.kubectl_aliases 
function kubectl() { echo "+ kubectl $@">&2; command kubectl $@; }
```

If you use zsh instead of bash, edit  ~/.zshrc instead of ~/.bashrc

##### Reboot

```bash
sudo reboot
```
