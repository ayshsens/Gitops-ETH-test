# GitOps Ethereum Node Deployment

GitOps setup for deploying an Ethereum node (Geth) on Kubernetes with monitoring, logging

## Prerequisites

Required tools and versions:
- Minikube v1.32.0+
- kubectl v1.28.0+
- Helm v3.12.0+
- OpenSSL for certificate generation
- Git

Minimum hardware requirements:
- CPU: 4 cores
- RAM: 8GB
- Storage: 300GB

## Repository Structure

```

```

## Quick Start


```bash
# Clone repository
git clone https://github.com/ayushvgcsns/Gitops-ETH.git
cd Gitops-ETH

# Start Minikube
minikube start --cpus 8 --memory 7168 --disk-size 300g --nodes 2
minikube addons enable ingress

# Install core components
kubectl create namespace ethereum
kubectl create namespace monitoring
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -f manifests/argocd/application.yaml

# Install Prometheus & Grafana
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  -f charts/monitoring/values-custom.yaml \
  --namespace monitoring

# Generate certificates
cd manifests/certificates
./generate-certs.sh
cd ../..
```

## Detailed Setup Guide

### 1. Local Environment Setup

a. Start Minikube cluster:
```bash
minikube start --nodes 2 --cpus 4 --memory 8192 --disk-size 50g
```

b. Verify cluster status:
```bash
kubectl get nodes
minikube status
```

### 2. Certificate Generation

Generate self-signed certificates for secure communication:

```bash
cd manifests/certificates
./generate-certs.sh
```

This script will:
- Generate a root CA certificate
- Create server certificates for all services
- Create Kubernetes TLS secrets

### 3. Core Components Installation

a. Create necessary namespaces:
```bash
kubectl create namespace ethereum
kubectl create namespace monitoring
kubectl create namespace argocd
```

b. Install Ingress-Nginx controller:
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  -f manifests/ingress-nginx/values-custom.yaml \
  --namespace ingress-nginx --create-namespace
```

### 4. ArgoCD Setup

a. Install ArgoCD:
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

b. Deploy application configuration:
```bash
kubectl apply -f manifests/argocd/application.yaml
```

c. Get ArgoCD admin password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 5. Geth Deployment

The Geth node will be automatically deployed by ArgoCD. Monitor the deployment:

```bash
kubectl -n ethereum get pods
kubectl -n ethereum get services
kubectl -n ethereum get ingress
```

### 6. Monitoring Stack

Install Prometheus and Grafana:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  -f charts/monitoring/values-custom.yaml \
  --namespace monitoring
```

## Accessing Services

Add the following entries to your `/etc/hosts` file:
```
127.0.0.1 geth-rpc.local
127.0.0.1 argocd.local
127.0.0.1 grafana.local
```

Enable port-forwarding:
```bash
# ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Grafana
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80

# Geth RPC
kubectl port-forward svc/geth -n ethereum 8545:8545
```

Access services:
- ArgoCD UI: https://argocd.local:8080
- Grafana: http://grafana.local:3000
- Geth RPC: https://geth-rpc.local:8545

Default credentials:
- ArgoCD: admin / (get password from secret)
- Grafana: admin / prom-operator

## Troubleshooting

1. Check pod status:
```bash
kubectl get pods -A
kubectl describe pod <pod-name> -n <namespace>
```

2. View logs:
```bash
kubectl logs -f <pod-name> -n <namespace>
```

3. Common issues:

a. Geth not syncing:
```bash
kubectl logs -f $(kubectl get pod -l app=geth -n ethereum -o jsonpath='{.items[0].metadata.name}') -n ethereum
```

b. ArgoCD sync issues:
```bash
kubectl -n argocd get applications
kubectl -n argocd describe application geth
```

c. Certificate issues:
```bash
kubectl -n ethereum get secrets
kubectl -n ethereum describe secret geth-tls
```

## Cleanup

Remove all resources:

```bash
# Delete ArgoCD applications
kubectl delete -f manifests/argocd/application.yaml

# Uninstall Helm releases
helm uninstall -n ethereum geth
helm uninstall -n monitoring monitoring
helm uninstall -n ingress-nginx ingress-nginx

# Delete namespaces
kubectl delete namespace ethereum monitoring ingress-nginx argocd

# Stop and delete Minikube cluster
minikube stop
minikube delete
```

