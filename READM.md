README.md — GitOps Infrastructure for Argo CD & k3d

(You can copy-paste this directly into a README file in your repo)

# GitOps Infrastructure with Argo CD & k3d
Author: Geredi NIYIBIGIRA
Purpose: Fully automated GitOps pipeline using Argo CD, Kustomize, and k3d Kubernetes
Status: WORKING — Successfully Deployed
# Overview

This repository implements a complete GitOps workflow using:

k3d for a lightweight local Kubernetes cluster

Argo CD for GitOps Continuous Deployment

Kustomize for application management

GitHub as the single source of truth

The workflow supports:

✔ Automatic deployments when code changes
✔ Drift detection
✔ Auto-healing (cluster restored to Git state)
✔ Full application rollout (Deployment → ReplicaSet → Pod → Service)
✔ Environment structure (dev/prod-ready)
✔ Reproducible setup even after system shutdown

This README includes every step taken to build the system.

# Architecture Diagram (Conceptual)
 GitHub (gitops-infrastructure-geredi)
        |
        v
  Argo CD (running in k3d cluster)
        |
        v
Kubernetes resources deployed:
  - Deployment
  - Service
  - ReplicaSet
  - Pod

# 1️Install & Configure k3d Kubernetes
Install k3d

If not installed:

curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

Create the cluster
k3d cluster create argo-demo --servers 1


Verify node is ready:

kubectl get nodes


Expected output:

k3d-argo-demo-server-0   Ready   control-plane

# 2️Install Argo CD

Create Argo CD namespace:

kubectl create namespace argocd


Install using official manifests:

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml


Check pods:

kubectl get pods -n argocd

# Expose Argo CD Locally (Port-Forward)
kubectl port-forward svc/argocd-server -n argocd 8082:443


Open in browser:

https://localhost:8082

# Login to Argo CD

Get admin password:

# 1. Point this terminal to the correct config file
export KUBECONFIG=/home/niyge/.config/k3d/kubeconfig-argo-demo.yaml

# 2. Get the password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo




Login:

username: admin  
password: <decoded-password>

# Create GitOps Repository

GitHub repo:
 https://github.com/GerediNIYIBIGIRA/gitops-infrastructure-geredi

Clone it:

git clone https://github.com/GerediNIYIBIGIRA/gitops-infrastructure-geredi.git
cd gitops-infrastructure-geredi

# GitOps Folder Structure
gitops-infrastructure-geredi/
│
├── apps/
│   └── nginx-sample/
│        ├── deployment.yaml
│        ├── service.yaml
│        └── kustomization.yaml
│
├── clusters/
│   └── dev/
│        └── nginx-sample-app.yaml
│
└── README.md

# Application Manifests
apps/nginx-sample/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-sample
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-sample
  template:
    metadata:
      labels:
        app: nginx-sample
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80

apps/nginx-sample/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-sample
spec:
  type: ClusterIP
  selector:
    app: nginx-sample
  ports:
  - port: 80
    targetPort: 80

apps/nginx-sample/kustomization.yaml
resources:
  - deployment.yaml
  - service.yaml

# Argo CD Application Definition
clusters/dev/nginx-sample-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-sample
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/GerediNIYIBIGIRA/gitops-infrastructure-geredi.git
    targetRevision: HEAD
    path: apps/nginx-sample

  destination:
    server: https://kubernetes.default.svc
    namespace: default

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

# Deploy the GitOps Application
kubectl apply -f clusters/dev/nginx-sample-app.yaml


Result in Argo CD UI:

✔ Healthy

✔ Synced

✔ Deployment created

✔ ReplicaSet created

✔ Pod running

# Automatic Deployment (Git Push → ArgoCD Sync)

To change NGINX version:

Edit:

apps/nginx-sample/deployment.yaml


Change:

image: nginx:latest


to:

image: nginx:1.25


Commit & push:

git add .
git commit -m "Change nginx version to 1.25"
git push


Argo CD will:

Detect new commit

Apply new YAML

Create new RS

Roll pods

Become Healthy + Synced

# Drift Detection & Auto-Healing

Try breaking it:

kubectl delete pod -l app=nginx-sample
kubectl scale deployment nginx-sample --replicas=3


Argo CD will:

Detect drift

Auto heal

Return cluster to Git state

# How to Resume After Shutdown

When reopening your laptop:

Start Docker Desktop


k3d depends on Docker containers.

Start port-forward again
kubectl port-forward svc/argocd-server -n argocd 8082:443

## Run this command to fix the connection:
zl

Verify k3d cluster is running
kubectl get nodes

Argo CD UI should work normally
https://localhost:8082


Everything will continue from where you left off because:

Git stores your manifests

Argo CD stores config in the cluster

k3d preserves all resources

# What’s Next?

You are ready for:

Multi-environment GitOps (dev → staging → prod)

Argo Rollouts (canary / blue-green)

Secrets with SOPS/SealedSecrets

Ingress + DNS

Real application deployments

# Summary

You successfully built a production-ready GitOps pipeline with:

k3d Kubernetes cluster

Argo CD

Git Sync

Auto-sync

Drift detection

Auto-healing

Version-controlled deployments

This setup can be extended to real applications, real environments, and real production pipelines.

# Credits

Built by Geredi NIYIBIGIRA

