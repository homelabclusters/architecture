# Homelab Kubernetes Architecture and Setup (K3s)

This document details the full architecture and step-by-step instructions for setting up a homelab environment using K3s, with s3 as the master node and s1/s2 as worker nodes. Services include Prometheus, Grafana, Jenkins with GitHub integration and GitOps workflow, a static host website, a Go-based website, and a Plex media server.

---

## Overview

### Nodes
- **s3** – Master Node (Control Plane)
- **s1** – Worker Node
- **s2** – Worker Node

### Services
- **Kubernetes** – Lightweight with K3s
- **Prometheus + Grafana** – Monitoring
- **Jenkins** – CI/CD with GitHub webhooks
- **Self-hosted Docker Registry** – For local image storage
- **Static Website** – Personal homepage
- **Go Website** – Custom Go app
- **Plex Media Server** – Local media server

---

## 1. Base System Setup

### OS: Ubuntu 22.04

### Common Tasks on All Nodes (s1, s2, s3)
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl vim git openssh-server
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### Set Static IPs and Hostnames
Update `/etc/hosts` on each node:
```
192.168.1.100  s3
192.168.1.101  s1
192.168.1.102  s2
```

Set hostnames:
```bash
sudo hostnamectl set-hostname s1  # or s2/s3
```

Enable passwordless SSH:
```bash
ssh-keygen -t rsa
ssh-copy-id user@s1
ssh-copy-id user@s2
ssh-copy-id user@s3
```

---

## 2. K3s Installation

### On Master (s3)
```bash
curl -sfL https://get.k3s.io | sh -
```

Get the node token:
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

### On Workers (s1, s2)
```bash
curl -sfL https://get.k3s.io | K3S_URL="https://s3:6443" K3S_TOKEN="<node-token>" sh -
```

Verify nodes:
```bash
kubectl get nodes
```

---

## 3. Helm Installation (on s3)
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

---

## 4. Monitoring Stack: Prometheus + Grafana

Add Helm repo and install:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack
```

Access Grafana:
- Default URL: `http://<NodeIP>:3000`
- Default login: `admin / prom-operator`

---

## 5. Self-Hosted Jenkins with GitHub Integration

### Install Jenkins:
```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update

helm install jenkins jenkins/jenkins
```

Get admin password:
```bash
kubectl exec --namespace default -it svc/jenkins -- cat /var/jenkins_home/secrets/initialAdminPassword
```

Expose Jenkins via NodePort or Ingress.

### GitHub Integration
- Add GitHub Webhook to repo
- Install Jenkins plugins:
  - GitHub
  - Git
  - Webhook Trigger -> github
  - Pipeline

### Jenkinsfile.
```groovy
pipeline {
    agent any
    stages {
        stage('Clone') {
            steps { git 'https://github.com/brodiep21/<jenkinssetup>/repo.git' }
        }
        stage('Build & Push') {
            steps {
                sh 'docker build -t s3:5000/myapp:latest .'
                sh 'docker push s3:5000/myapp:latest'
            }
        }
        stage('Deploy') {
            steps {
                sh 'kubectl apply -f k8s/deployment.yaml'
            }
        }
    }
}
```

---

## 6. Local Docker Registry

Deploy a local registry:
```bash
kubectl create deployment registry --image=registry:2
kubectl expose deployment registry --type=NodePort --port=5000
```

Push images:
```bash
docker tag myapp:latest s3:5000/myapp:latest
docker push s3:5000/myapp:latest
```

---

## 7. Host Website (Static)

Create a simple Nginx deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: host-site
spec:
  replicas: 1
  selector:
    matchLabels:
      app: host-site
  template:
    metadata:
      labels:
        app: host-site
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        configMap:
          name: host-site-content
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: host-site-content
  data:
    index.html: |
      <html><body><h1>My Homelab Site</h1></body></html>
```

---

## 8. Go Website

Dockerize the Go app and create a deployment pointing to it:
```Dockerfile
FROM golang:alpine
WORKDIR /app
COPY . .
RUN go build -o main .
CMD ["./main"]
```

Deploy via YAML or Helm chart.

---

## 9. Plex Media Server

Use Helm:
```bash
helm repo add k8s-at-home https://k8s-at-home.com/charts/
helm repo update

helm install plex k8s-at-home/plex --set persistence.config.enabled=true --set persistence.media.enabled=true --set persistence.media.mountPath=/mnt/media
```

Make sure media is accessible via a shared volume or NFS.

---

## 10. GitOps (optional extension)

Using argocd for GITOPS.

Install ArgoCD:
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Expose via ingress and link to your GitHub repo for declarative deployments.

---

**This document is evolving. Adjustments and expansions will be made as services and features are added.**

