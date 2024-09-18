# Pet Voting App with KIND, Prometheus, and Grafana

This project demonstrates deploying a multi-node Kubernetes cluster using KIND (Kubernetes in Docker) with integrated Prometheus and Grafana for monitoring.

## Table of Contents
1. [Install KIND](#1-install-kind)
2. [Create KIND Cluster](#2-create-kind-cluster)
3. [Install Kubectl](#3-install-kubectl)
4. [Clone and Deploy the Voting App](#4-clone-and-deploy-the-voting-app)
5. [Set Up Argo CD](#5-set-up-argo-cd)
6. [Install Kubernetes Dashboard](#6-install-kubernetes-dashboard)
7. [Install HELM and Prometheus Stack](#7-install-helm-and-prometheus-stack)
8. [Monitoring with Prometheus and Grafana](#8-monitoring-with-prometheus-and-grafana)

## 1. Install KIND

```bash
[ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-arm64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

## 2. Create KIND Cluster

Define your cluster configuration in `config.yaml`:

```yaml
# A cluster with 3 control-plane nodes and 3 workers
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: control-plane
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
```

Create the cluster:

```bash
kind create cluster --config=config.yaml
```

Check cluster info and nodes:

```bash
kubectl cluster-info --context kind-kind
kubectl get nodes
kind get clusters
```

## 3. Install Kubectl

```bash
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.30.2/2024-07-12/bin/darwin/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version --short --client
```

## 4. Clone and Deploy the Voting App

Clone the repository:

```bash
git clone https://github.com/patel-aum/Pet-Voting-App.git
cd Pet-Voting/
```

Apply Kubernetes specifications:

```bash
kubectl apply -f k8s-specifications/
kubectl get all
```

Port-forward the services:

```bash
kubectl port-forward service/vote 5000:5000 --address=0.0.0.0 > /dev/null 2>&1 &
kubectl port-forward service/result 5001:5001 --address=0.0.0.0 > /dev/null 2>&1 &
```

## 5. Set Up Argo CD

Create a namespace for Argo CD:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Check Argo CD services:

```bash
kubectl get svc -n argocd
```

Expose Argo CD server using NodePort:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
```

Forward ports to access Argo CD:

```bash
kubectl port-forward -n argocd service/argocd-server 8443:443 > /dev/null 2>&1 &
```

Retrieve the Argo CD admin password:

```bash
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

## 6. Install Kubernetes Dashboard

Deploy Kubernetes dashboard:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
kubectl apply -f sa.yaml
```

Create a token for dashboard access:

```bash
kubectl -n kubernetes-dashboard create token admin-user
```

## 7. Install HELM and Prometheus Stack

Install HELM:

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Add HELM repositories:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://charts.helm.sh/stable
helm repo update
```

Create a namespace for monitoring:

```bash
kubectl create namespace monitoring
```

Install Prometheus and Grafana:

```bash
helm install kind-prometheus prometheus-community/kube-prometheus-stack --namespace monitoring \
  --set prometheus.service.nodePort=30000 \
  --set prometheus.service.type=NodePort \
  --set grafana.service.nodePort=31000 \
  --set grafana.service.type=NodePort \
  --set alertmanager.service.nodePort=32000 \
  --set alertmanager.service.type=NodePort \
  --set prometheus-node-exporter.service.nodePort=32001 \
  --set prometheus-node-exporter.service.type=NodePort
```

Check services in the monitoring namespace:

```bash
kubectl get svc -n monitoring
```

## 8. Monitoring with Prometheus and Grafana

Port-forward Prometheus:

```bash
kubectl port-forward svc/kind-prometheus-kube-prome-prometheus -n monitoring 9090:9090 --address=0.0.0.0 > /dev/null 2>&1 &
```

Port-forward Grafana:

```bash
kubectl port-forward svc/kind-prometheus-grafana -n monitoring 31000:80 --address=0.0.0.0 > /dev/null 2>&1 &
```

This setup will provide you with a local Kubernetes cluster using KIND, along with monitoring via Prometheus and Grafana, and automation via Argo CD.
