---
title: "Kubernetes Resources 101"
date: 2022-03-07T15:33:15-07:00
draft: true
---

This article hopes to explain and further understanding of what kubernetes resources are, and how we utilize them. This guide touches in depth on what kubernetes resources are called, when they are used, and what they do.

Having this depth of knowledge can be invaluable working to support a kubernetes system. Knowing resource names and their functions can help you to understand what any given system is doing, and when something goes wrong, what to look at.

This guide is not complete, and will be updated over time. 

## Table of Contents
- [Assumed Knowledge](#assumed-knowledge)
  - [Images](#images)
  - [Containers](#containers)
  - [Linux distributions](#linux-distributions)
  - [Virtualization](#virtualization)
- [Layers of Abstraction](#layers-of-abstraction)
  - [Kubernetes Resources](#kubernetes-resources)
  - [Hypervisor / KVM](#hypervisor--kvm)
  - [Node](#node)
  - [Deployment](#deployment)
  - [Pod](#pod)
  - [Container](#container)
- [Deployments as instructions](#deployments-as-instructions)
  - [Deployments describe pods](#deployments-describe-pods)
  - [Deployments describe storage](#deployments-describe-storage)
  - [Deployments describe networking](#deployments-describe-networking)
  - [Deployments describe variables](#deployments-describe-variables)
- [Configmaps](#configmaps)
  - [A brief mention of helm](#a-brief-mention-of-helm)
  - [Environmental Variables](#environmental-variables)
  - [Secrets](#secrets)
  - [Updating a pod with new information](#updating-a-pod-with-new-information)
- [DaemonSets and ReplicaSets](#daemonsets-and-replicasets)
  - [DaemonSets](#daemonsets)
  - [Node Toleration and Tainting](#node-toleration-and-tainting)
  - [ReplicaSets](#replicasets)
- [Storage Basics](#storage-basics)
  - [Persistent Volumes and Persistent Volume Claims](#persistent-volumes-and-persistent-volume-claims)
  - [Storage Classes](#storage-classes)
- [Networking](#networking)
  - [Service](#service)
  - [Ingress](#ingress)
  - [CoreDNS and internal networking](#coredns-and-internal-networking)
- [Resource limiting](#resource-limiting)
  - [Limits vs Requests](#limits-vs-requests)
  - [Scheduling and Evictions](#scheduling-and-evictions)
- [RBAC](#rbac)
  - [Role-based Authentication](#role-based-authentication)
  - [Service Accounts](#service-accounts)
  - [Cluster Roles](#cluster-roles)
  - [Cluster Role Binding](#cluster-role-binding)
- [Testing tools](#testing-tools)
  - [Container runtime tools](#container-runtime-tools)
  - [Desktop tools running kubernetes and containers](#desktop-tools-running-kubernetes-and-containers)

## Assumed Knowledge
This guide assumes that you have a basic understanding on containerization, particularly regarding containers and what runs them.
However, I have included some basics of this knoweldge below. This will allow you to understand what you do and don't know, and what you may want to look into further to understand the rest of the guilde. I would especially make sure you have a strong understanding on how images are created. You should be able to create a Dockerfile without much trouble.
### Images
#### Images vs Containers
Images are not the same thing as containers, however it is common to mistake them for the same thing. Images are completely static and hosted in a repository, where a container is an active version of the images. You can have multiple containers using the same image. You can think of images as instructions + initial data for the container.
#### Image Creation
- Images are created using 2 methods
  - Using an initial set of instructions called a Dockerfile
    - These images can be build using tools such as docker build, kaniko, buildah, and podman
  - Saving an existing container with any changes
    -  This is not a common method of creating applications, but is common in creating base images from scratch
    -  Examples:
       -  You can create a container based on a linux distro, and install all the tools an application may use
       -  You can create a completely blank container and install a linux distro to use as a base
#### Base Images, Parent Images, and Child Images
- As touched on in [image creation](#image-creation), images can inherit data from other images
- A typical method of doing this is by using the FROM statement in your Dockerfile
- Example:
  - I have a set of containers that make up an application, and want a similar set of commands available while allowing nodes to avoid pulling the same information twice
    - First, I create an image with my distrobution of choice, then install all of my required tools. In this example, I want all of my images to be based on alpine linux 3.15, and have curl and java installed.
      - [Link to example base image](#base-image)
    - Next, I save this base image and push it to my image repo as myrepo.com/base-images/alpine-and-java:3.15
    - After this, I create multiple images based on the previous image (our "Base" image)
      - [Link to example application 1](#application-1)
      - [Link to example application 2](#application-2)
    - The advantage to this is that my fully updated alpine 3.15 with curl and openjdk8 installed is stored as a single layer. This single layer is only pulled once for all 3 images. See [Image Storage](#image-storage) for further details.
##### Base Image:
```yaml
FROM alpine:3.15 # First, let this image be based on alpine linux 3.15
RUN apk update -y && apk install -y \ # Second, update alpine with latest available packages, then install curl and openjdk8 (java 8)
    curl \
    openjdk8
```
##### Application 1:
```yaml
FROM myrepo.com/base-images/alpine-and-java:3.15 # Start with my base image
ADD ./app1 /app1 # Copy all files from the app1 folder to /app1 in the image
CMD curl mysite.com/scripts/startapp1.sh | /bin/ash # Run an executable from my site that starts my java application
```
##### Application 2:
```yaml
FROM myrepo.com/base-images/alpine-and-java:3.15 # Start with my base image
ADD ./app2 /app2 # Copy all files from the app2 folder to /app2 in the image
CMD ["java", "/app2/bin/app2.jar"] # Run a java file contained in /app2/bin
```

#### Image Storage
- Images are stored in image repositories (commonly shortened to repos)
- The client pulls an image from the repo
  - This can be done directly with the [container runtime](#container-runtimes), or using a [front end tool](#testing-tools) such as docker, crictl, and nerdtools
  - The client makes a REST call to the server (GET)  
  - This downloads each layer of an image not already present on the machine
  - This also downloads the image manifest, which lets the container system know which layers are associated with the image, and the order that the layers go in.
- Images are stored in layers
  - Each layer is an action in the dockerfile
  - Images are stored so that each layer is only stored once
    - This is useful for bandwidth and storage resources, as commonly used [base images](#base-images-parent-images-and-child-images) or layers may already be present in the repo or the client.
    - This is useful in the case of [base images](#base-images-parent-images-and-child-images):
      - Base images can be created, and pulled to a [node](#node) once
      - Each application built on the base image will only pull changed information (via later layers). This makes updating (where the base image is the same) and initial installs (where the base image is only pulled once for all applications) potentially much faster.
- Common image repositories include the following, which contain a wide varaity of features such as other artifact storage, authenitcation, users/groups, and automation:
  - Docker Registry
  - DockerHub
  - Quay (Provided by RedHat)
  - GCR (Google Container Registry)
  - ECR (Elastic Container Registry, part of Amazon AWS)
  - ACR (Azure Container Registry, part of Microsoft Azure)
  - Harbor
  - JFrog Artifactory
  - Sonatype Nexus
  - Gitlab

### Containers
#### Container Creation
- Containers are created from images
  - All containers are created from an image. Even a blank container is created from a blank image.
- (Almost) Everything a container does is ephemeral
  - This means that any changes you do in this container will dissapear when it is recreated from a new image
  - This is one of the main differences that seperate a VM and a container
  - An exception to this rule is if a container mounts a volume
    - Anything changed within the volume (usually set to a specific path) will remain changed
    - Any new container that has the same volume will be able to access these specific changes
    - Examples:
      - Postgres is a database application. You can use a volume to save the database to, so even if the container is recreated or upgraded, the database itself will not be changed.
      - You can create a specific folder in a volume that contains any user configured data such as users, preferences, uploaded data, code, etc. When you upgrade the application (using a new image), the data will not be removed.
#### Containers vs VMs
- Virtual Machines (VMs) act like standard computers, but (as the name implies) are virtual. 
- This allows you flexability in running multiple virtual machines for one physical server, and allows those machines to be further specialized.
- For example, you may have 1 VM running Ubuntu 20.04, another VM running CentOS 7, and another VM running Windows Server 2016.
  - Each of these machines are running completely seperately from each other.
  - Like a standard computer, changes are saved to a virtual hard drive, and you can run many things at the same time on the server.
- Containers act similar to a virtual machine. However, instead of virtualizing everything (innefficient), we can borrow some aspects from the host OS.
- Containers borrow the kernel of the host OS, and contain a bare-bones version of the OS. 
  - A limitation of this is that you can't run linux containers without the linux kernel running, or windows containers without the windows kernel running
  - Most server applications run on linux, so containers are more often built for linux.
  - You can create a virtual machine running linux on your host machine (Windows, MacOS), and run all of your containers in this virtual machine however
    - This is what 'Docker Desktop' and 'Rancher Desktop' do when running on Windows and MacOS.
- When running in the right conditions, containers are much faster than virtual machines. Since they are so light, we can run a lot of them in place of 1 VM.
  - This allows us to run 1 container per application, or multiple containers per application instead of 1 VM running multiple applications
    - This method allows for portability
- Containers are portable, where virtual machines are not to the same degree. 
  - Virtual Machines are large in file size (virtual hard drives), and are not expected to move from machine to machine freely
  - Containers (and images) are very small, and have a very light footprint. They are designed to not change once they are running, so you can bring up many containers across many different computer systems. Each of these containers can be running a micro-app (an application split into many different containers). 
  - Example:
    - I have 1 virtual machine that runs my web server, database, and API calls. I call this MyApp.
      - MyApp is getting popular on the internet, and my virtual machine is not able to keep up with all the requests.
      - To combat this, I created a second and third virtual machine that also runs MyApp. This was time consuming to set up, and now I need to worry about updating MyApp across 3 devices instead of 1.
    - I have 3 containers, web-server, database, and api-server. All together, I call this MyApp.
      - MyApp is getting popular on the internet, and my 3 containers are getting overloaded. This is especially true of the web-server container.
      - To combat this, I simply create more of the overloaded containers, and have my network load balance between those containers. 
      - I am now running 3 web-server containers, 2 api-server containers, and 1 database. 
      - To update my application, all I need to do is update my images. The containers are replaced with new containers with the new image.
#### Container Runtimes
- Container Runtimes are pieces of software that take your images, and run them as containers on your system.
- Common Container Runtime systems include:
  - Docker
  - Containerd
  - CRI-O
- Common ways to interact directly with these systems include
  - docker-cli
  - crictl
  - nerdctl
### Linux distributions
- Modern linux systems contain a lot more than the linux kernel on it's own
- Distributions (distros) are a common set of included systems that work with the linux kernel.
#### What is a distro
#### Common distros used in enterprise
### Virtualization
#### What is a VM
#### What is shared between a VM and the host OS

## Layers of Abstraction
### Kubernetes Resources
#### Anything can be a resource
### Hypervisor / KVM
#### Why utilize VMs instead of raw host interaction
#### Hypervisor type 1 vs type 2
#### Common hypervisor choices
### Node
#### Master vs Worker/Slave
### Deployment
#### What is a deployment
#### How is a deployment similar to an image
#### What can a deployment do
### Pod
#### What is a pod
#### How does a pod relate to a container
#### Why do some pods contain multiple containers
### Container
#### What is a container (to kubernetes)
#### What is an init-container

## Deployments as instructions
### Deployments describe pods
#### Deployments describe containers
### Deployments describe storage
### Deployments describe networking
### Deployments describe variables

## Configmaps
### A brief mention of helm
### Environmental Variables
### Secrets
### Updating a pod with new information

## DaemonSets and ReplicaSets
### DaemonSets
### Node Toleration and Tainting
### ReplicaSets

## Storage Basics
### Persistent Volumes and Persistent Volume Claims
### Storage Classes

## Networking
### Service
#### ClusterIP
#### NodePort
#### Load Balancer
### Ingress
#### Externally managed ingress
### CoreDNS and internal networking

## Resource limiting
### Limits vs Requests
### Scheduling and Evictions

## RBAC
### Role-based Authentication
### Service Accounts
### Cluster Roles
### Cluster Role Binding

## Testing tools
### Container runtime tools
- Crictl
- Nerdtools
### Desktop tools running kubernetes and containers
- Rancher Desktop
- Docker Desktop
- Minikube
- Kind
- RKE2
- K3s