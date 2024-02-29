---
title: Kubernetes Cluster on a VM
description: Use kubeadm to spin a kubernetes cluster on a virtual machine.
date: 2024-02-29 15:00:00 +0500
category: kubernetes
tags:
  - kubernetes
  - kubeadm
---

## What we are going to setup

In this post, we are going to cover how to install a Kubernetes cluster on Ubuntu 20.04 LTS VM with kubeadm with one control plane node and one worker node.

## Prerequisites

1. 2 Ubuntu 20.04 VMs with 2GB RAM and 2 vCPUs each.
2. SSH access with sudo priveleges.

## Step 1 (Static IP address assignment and turn off swap on both VMs.)

#### Controlplane Setup

On a subnet 10.211.55.0/24, assign static IP address to the **controlplane node**. The process is as follows:

- SSH user@CONTROLPLANE_IP

- RUN `sudo vim /etc/netplan/00-installer-config.yaml` and update the config as:

```yaml
network:
  ethernets:
    enp0s5:
      dhcp4: no
      addresses: [10.211.55.21/24]
      gateway4: 10.211.55.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
  version: 2
```

- RUN `sudo netplan try` and post netplan tests, press enter and the new IP address will be assigned.
- Update hosts:

  1.  RUN `sudo hostnamectl set-hostname controlplane`
  2.  RUN `vim /etc/hosts` and add:

  ```
  127.0.1.1 controlplane

  # Kubernetes Hosts
  10.211.55.21 controlplane
  10.211.55.22 node01
  ```

- Turn off swap by running `sudo swapoff -a` and disable swap on reboot by unmounting it; RUN `sudo vim /etc/fstab` and comment out swap by placing # at the start of swap entry.

#### Worker Node Setup

On a subnet 10.211.55.0/24, assign static IP address to the **worker node**. The process is as follows:

- SSH user@WORKERNODE_IP

- RUN `sudo vim /etc/netplan/00-installer-config.yaml` and update the config as:

```yaml
network:
  ethernets:
    enp0s5:
      dhcp4: no
      addresses: [10.211.55.22/24]
      gateway4: 10.211.55.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
  version: 2
```

- RUN `sudo netplan try` and post netplan tests, press enter and the new IP address will be assigned.
- Update hosts:

  1.  RUN `sudo hostnamectl set-hostname node01`
  2.  RUN `vim /etc/hosts` and add:

  ```
  127.0.1.1 node01

  # Kubernetes Hosts
  10.211.55.21 controlplane
  10.211.55.22 node01
  ```

- Turn off swap by running `sudo swapoff -a` and disable swap on reboot by unmounting it; RUN `sudo vim /etc/fstab` and comment out swap by placing # at the start of swap entry.

## Step 2 (Bridge Traffic)

SSH into **both VMs** and run the following to forward IPv4 and letting iptables see bridged traffic:

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

## Step 3 (Install Container runtime, kubeadm, kubectl, and kubelet)

SSH into **both VMs** and run the following to install container runtime, kubeadm, kubectl(\*) and kubelet:

```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl docker.io
sudo apt-mark hold kubelet kubeadm kubectl docker.io
```

(\*) kubectl installation to be skipped on a worker node

## Step 4a (Initialize controlplane node)

SSH into **controlplane node** and run:

- `sudo kubeadm config images pull`
- `sudo kubeadm init --pod-network-cidr=192.168.0.0/16`

This might take 3-7 minutes depending on the speed of your internet. Once completed, several instructions will be provided in the logs e.g. generate kube config, etc:

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Run above to create the kubeconfig file as mentioned in the logs.

At this point, kubeadm join command should be printed in the logs as well to join the worker node with the cluster. If somehow misplaced, run `sudo kubeadm token create --print-join-command` and copy the command.

## Step 4b (Join worker node)

SSH into **worker node** and run `sudo kubeadm join ....`, the command that was copied from previous step.

## Step 5 (Verify nodes)

On the **controlplane node**, run `kubectl get nodes -o wide` and verify both nodes are there.

## Step 6 (Install CNI)

At the point, **nodes will be in NotReady status**. That is because there is no CNI plugin setup to provide networking for containers and pods. SSH into controlplane & setup Calico CNI by running:

```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
```

Wait couple of minutes and run `kubectl get nodes` to verify that the Nodes status turns to Ready or run `watch kubectl get nodes` to live monitor the status

## Example Deployment

For a sanity check, make an nginx deployment to verify the cluster is up and running:

```
kubectl create deployment app --image=nginx:1.25-alpine --replicas=3
```

Check the pods are running; `kubectl get pods`
