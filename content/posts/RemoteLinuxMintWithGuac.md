---
title: "Remote Linux Mint environment setup for Guacamole"
date: 2022-03-09T14:45:21-07:00
draft: false
---

This document provides instructions (mostly for myself) for installing Linux Mint in a virtual machine, and providing access to it over VNC and SSH using apache guacamole.
This also goes through installing utilities I use often in these remote instances.
This guide assumes you already have an instance of guacamole running that works fine.

## Table of Contents
- [Install Mint](#install-mint)
- [Configure SSH and VNC](#configure-ssh-and-vnc)
  - [Enable SSH](#enable-ssh)
  - [Enable VNC](#enable-vnc)
- [Install Docker & Docker-Compose](#install-docker--docker-compose)
  - [Install Docker](#install-docker)
  - [Fix docker Permissions](#fix-docker-permissions)
  - [Install Docker-Compose](#install-docker-compose)
- [Install zsh, ohmyzsh, and kube utils](#install-zsh-ohmyzsh-and-kube-utils)
  - [Install zsh, ohmyzsh, and kubectl aliases](#install-zsh-ohmyzsh-and-kubectl-aliases)
  - [Configure kubernetes networking](#configure-kubernetes-networking)
  - [Install kubectl, kubeadm, and kubelet](#install-kubectl-kubeadm-and-kubelet)
  
## Install Mint
- Download Linux Mint from https://linuxmint.com/download.php
  - Use the one based on Ubuntu over the new Debian one, as we are using Linux Mint as a "Ubuntu machine with snap removed".
  - I used the Xfce edition, but Mate and Cinnamon also work well over VNC
- Install Mint to the VM
  - For disk use, mint uses a Non-LVM style partitioning scheme by default, and does not include a swap or dedicated home partition
    - This is my preferred style, so I will keep it as is
  - If you are planning on doing mission critical work, you may want to consider using btrfs for use with timeshift to automatically back up (snapshot) your system
- Remove virtual install media and reboot

## Configure SSH and VNC
### Enable SSH
Linux Mint does not include openssh-server by default, so we will need to enable this
- From your virtual monitor, run the following commands:
```bash
sudo apt update -y
sudo apt install openssh-server
sudo systemctl enable --now openssh-server
sudo reboot
```
### Enable VNC
Now that we have ssh access, I recommend sshing into your VM
- Run these commands to enable VNC access (on port 5900)
```bash
sudo apt update
sudo apt install x11vnc vim
sudo mkdir /etc/x11vnc
sudo x11vnc --storepasswd /etc/x11vnc/vncpwd
```
Now that we have the folder, edit the following file to create a systemd service
```bash
sudo vim /lib/systemd/system/x11vnc.service
```
Add the following to the file:
```bash
[Unit]
Description=Start x11vnc at startup.
After=multi-user.target

[Service]
Type=simple
ExecStart=/usr/bin/x11vnc -auth guess -forever -noxdamage -repeat -rfbauth /etc/x11vnc/vncpwd -rfbport 5900 -shared

[Install]
WantedBy=multi-user.target
```
Now, enable service and reboot
```bash
sudo systemctl daemon-reload
sudo systemctl enable x11vnc.service
sudo systemctl start x11vnc.service
sudo reboot
```

## Install Docker & Docker-Compose
I use these 2 utilities often in the remote VMs
### Install Docker
Run the following:
```bash
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  focal stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```
Note that this is hard-coded to focal, as the Linux Mint version used is based on Ubuntu Focal release
### Fix docker Permissions
```bash
sudo groupadd docker
sudo usermod -aG docker $USER
```
### Install Docker-Compose
Run these commands, then reboot:
```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo reboot
```

## Install zsh, ohmyzsh, and kube utils
### Install zsh, ohmyzsh, and kubectl aliases
```bash
sudo apt install zsh fonts-powerline -y
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

wget https://raw.githubusercontent.com/ahmetb/kubectl-aliases/master/.kubectl_aliases

wget https://github.com/derailed/k9s/releases/download/v0.25.18/k9s_Linux_x86_64.tar.gz
tar -xzf k9s_Linux_x86_64.tar.gz
chmod +x k9s
sudo mv k9s /usr/local/bin/k9s

vim ~/.zshrc
```
Change the following in your .zshrc file:
```bash
ZSH_THEME="agnoster"

plugins=(
        git
        kubectl
        vscode
        shrink-path
)

[ -f ~/.kubectl_aliases ] && source ~/.kubectl_aliases
function kubectl() { echo "+ kubectl $@">&2; command kubectl $@; }
```
Reboot after
```bash
sudo reboot
```
### Configure kubernetes networking
```bash
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system

sudo vim /etc/docker/daemon.json
```
Add the following lines:
```json
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```
Restart docker
```bash
sudo systemctl restart docker
```

### Install kubectl, kubeadm, and kubelet
```bash
sudo apt update
sudo apt install apt-transport-https ca-certificates curl

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo reboot
```