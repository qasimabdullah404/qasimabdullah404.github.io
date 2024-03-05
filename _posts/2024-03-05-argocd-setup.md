---
title: GitOps with ArgoCD
description: Use ArgoCD to maintain Kubernetes application state
date: 2024-03-5 15:00:00 +0500
category: kubernetes
tags:
  - kubernetes
  - argocd
  - gitops
---

## What we are going to setup

In this post, we are going to cover how to setup ArgoCD to maintain the Kuberenetes applicatio state

## Prerequisites

1. Kubernetes Cluster available (I will be using minikube for this one)
2. SSH access with sudo priveleges.

## Setup ArgoCD

To setup ArgoCD, run:

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Verify that the ArgoCD pods are running - `kubectl -n argocd get pods`

## Access ArgoCD UI

Accessing the ArgoCD is a two step part; first get the admin user password by running `kubectl -n argocd get secrets argocd-initial-admin-secret -o yaml | grep password | awk '{print $2}' | base64 -d` and then run `k -n argocd port-forward svc/argocd-server 8085:443`. Login at 127.0.0.1:8085 and use `admin` as username and password from the first step.

## Create ArgoCD application manifest file

Create ArgoCD application manifest(argo-app.yaml) by using the following as an example (remember to change the repository URL to yours):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/qasimabdullah404/argocd-test.git
    targetRevision: HEAD
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: argoapp
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
    automated:
      selfHeal: true
      prune: true
```

This tells ArgoCD to look for manifests at https://github.com/qasimabdullah404/argocd-test/tree/main/manifests and sync the cluster stateto mentioned manifests. The sync policy states:

- Create the namespace if it doesn't exist within the cluster
- `selfHeal` will revert any changes made from the terminal e.g. `kubectl` commands to update image etc
- `prune` will delete the resources if it detects the manifest doesn't mention those anymore

## Create a GitHub repository to store manifests

To setup resources within the cluster i.e. deployments, etc; create a GitHub repository e.g. https://github.com/qasimabdullah404/argocd-test/tree/main/manifests and store your YAML manifest(s).

## Apply ArgoCD application manifest

Run `kubectl -n argocd apply -f ./argo-app.yaml` and head over to the ArgoCD UI to see the application created. This will sync the cluster and create any missing resources that are mentioned at the GitHub repository.

## TEST - make a minimal change to observe ArgoCD ain action

At the manifests path over at the GitHub repository, edit the manifest and update the image to a newer tag and commit. ArgoCD will incorporate the change within 3 minutes.
