# Kubernetes Cluster Deployment Project

This project contains Ansible playbooks and Kubernetes manifests for setting up a complete Kubernetes cluster with GitLab, monitoring, and other essential components. The setup includes a control plane node and worker nodes, along with various applications deployed on the cluster.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Setup Instructions](#setup-instructions)
  - [1. Initial Setup](#1-initial-setup)
  - [2. Kubernetes Installation](#2-kubernetes-installation)
  - [3. Control Plane Setup](#3-control-plane-setup)
  - [4. Worker Node Setup](#4-worker-node-setup)
  - [5. Network Plugin Installation](#5-network-plugin-installation)
  - [6. Ingress Controller Setup](#6-ingress-controller-setup)
  - [7. GitLab Deployment](#7-gitlab-deployment)
  - [8. Monitoring Stack Deployment](#8-monitoring-stack-deployment)
- [Additional Components](#additional-components)
- [Troubleshooting](#troubleshooting)

## Prerequisites

- Ubuntu servers for control plane and worker nodes
- SSH access to all servers with the `ubuntu` user
- Ansible installed on your local machine
- `kubernetes-dev.pem` SSH key for authentication

## Project Structure

```
kube-cluster/
├── gitlab/                      # GitLab deployment manifests
│   ├── postgresql/              # PostgreSQL for GitLab
│   ├── redis/                   # Redis for GitLab
│   └── ...                      # GitLab Kubernetes manifests
├── Jenkins X/                   # Jenkins X configuration
├── control-plane.yml            # Ansible playbook for control plane setup
├── disabling-swap.yml           # Ansible playbook to disable swap
├── enable-csi-driver.yml        # AWS EBS CSI driver setup
├── gitlab.yml                   # GitLab deployment playbook
├── hosts                        # Ansible inventory file
├── ingress-controller.yml       # NGINX Ingress Controller setup
├── initial.yml                  # Initial server setup playbook
├── install-calico.yml           # Calico CNI installation
├── kube-dependencies.yml        # Kubernetes dependencies installation
├── monitoring.yml               # Prometheus & Grafana setup
└── workers.yml                  # Worker nodes setup playbook
```

## Setup Instructions

### 1. Initial Setup

This step prepares all servers with the necessary user configuration:

```bash
ansible-playbook -i hosts initial.yml
```

This playbook:
- Creates the `ubuntu` user
- Configures passwordless sudo access
- Sets up SSH access

### 2. Kubernetes Dependencies Installation

Install Kubernetes components and dependencies on all nodes:

```bash
ansible-playbook -i hosts kube-dependencies.yml
```

This playbook:
- Installs Docker, curl, and other required packages
- Adds Kubernetes APT repository
- Installs kubelet, kubeadm, and kubectl
- Configures kubelet to use systemd cgroup driver

### 3. Disable Swap

Kubernetes requires swap to be disabled:

```bash
ansible-playbook -i hosts disabling-swap.yml
```

This playbook:
- Disables swap immediately
- Updates /etc/fstab to permanently disable swap
- Loads required kernel modules
- Configures sysctl parameters for Kubernetes

### 4. Control Plane Setup

Initialize the Kubernetes control plane:

```bash
ansible-playbook -i hosts control-plane.yml
```

This playbook:
- Initializes the Kubernetes control plane with kubeadm
- Sets up kubeconfig for the ubuntu user
- Prepares for Calico CNI installation

### 5. Worker Node Setup

Join worker nodes to the cluster:

```bash
ansible-playbook -i hosts workers.yml
```

This playbook:
- Generates a join command on the control plane
- Joins worker nodes to the cluster
- Configures kubelet to use the node's private IP

### 6. Network Plugin Installation

Install Calico CNI for pod networking:

```bash
ansible-playbook -i hosts install-calico.yml
```

This playbook:
- Downloads the Calico manifest
- Applies it to the cluster
- Waits for Calico pods to be ready

### 7. Ingress Controller Setup

Install NGINX Ingress Controller:

```bash
ansible-playbook -i hosts ingress-controller.yml
```

This playbook:
- Installs NGINX Ingress Controller
- Verifies the installation

### 8. GitLab Deployment

Deploy GitLab with PostgreSQL and Redis:

```bash
ansible-playbook -i hosts gitlab.yml
```

After running the playbook, apply the Kubernetes manifests on the control plane:

```bash
# On the control plane node
kubectl apply -f /home/ubuntu/gitlab/gitlab-ns.yml
kubectl apply -f /home/ubuntu/gitlab/gitlab-storageClass.yml
kubectl apply -f /home/ubuntu/gitlab/gitlab-pvc.yml
kubectl apply -f /home/ubuntu/gitlab/postgresql/
kubectl apply -f /home/ubuntu/gitlab/redis/
kubectl apply -f /home/ubuntu/gitlab/gitlab-deployment.yml
kubectl apply -f /home/ubuntu/gitlab/gitlab-svc.yml
```

### 9. Monitoring Stack Deployment

Deploy Prometheus and Grafana for monitoring:

```bash
ansible-playbook -i hosts monitoring.yml
```

This playbook:
- Creates a monitoring namespace
- Installs Helm
- Deploys kube-prometheus-stack (Prometheus, Grafana, Alertmanager)
- Configures NodePort services for access
- Sets up Ingress for web access

Access Grafana at NodePort 30000 and Prometheus at NodePort 30001.

## Additional Components

### AWS EBS CSI Driver

If running on AWS, you can enable the EBS CSI driver:

```bash
ansible-playbook -i hosts enable-csi-driver.yml
```

### Jenkins X

The project includes Jenkins X configuration in the `Jenkins X` directory.

## Troubleshooting

### Common Issues

1. **Pod networking issues**:
   - Verify Calico is running: `kubectl get pods -n kube-system | grep calico`
   - Check pod logs: `kubectl logs -n kube-system <calico-pod-name>`

2. **Worker nodes not joining**:
   - Check kubelet status: `systemctl status kubelet`
   - Review logs: `journalctl -xeu kubelet`

3. **GitLab deployment issues**:
   - Verify persistent volumes: `kubectl get pv,pvc -n gitlab`
   - Check pod status: `kubectl get pods -n gitlab`
   - View logs: `kubectl logs -n gitlab <pod-name>`

4. **Ingress not working**:
   - Verify ingress controller: `kubectl get pods -n ingress-nginx`
   - Check ingress resources: `kubectl get ingress --all-namespaces`

### Resetting the Cluster

If you need to reset the cluster and start over:

```bash
# On all nodes
sudo kubeadm reset
sudo rm -rf /etc/kubernetes/
sudo rm -rf $HOME/.kube/
```