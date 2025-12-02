# Project 3 – CI → GitOps → Argo CD → EKS

This repository is part of a multi-repository architecture.  

Project 1 builds and pushes application images, while **this** repository defines the GitOps deployment configuration consumed by Argo CD to deploy into Amazon EKS.

This project implements a full GitOps workflow using GitHub Actions, Kustomize, and Argo CD. The workflow builds container images in Project 1, updates manifests in this GitOps repository, and Argo CD keeps the cluster synchronized with the desired state.

---

## Overview

**Build → GitOps Update → Argo CD Sync → EKS Deployment**

1. **Project 1 CI/CD**
   - Builds and pushes application images to Amazon ECR  
   - Updates the dev image tag in this GitOps repository

2. **GitOps Repository (Project 3)**
   - Stores Kubernetes manifests using `base` + `overlays`
   - Separates dev and prod environments using Kustomize

3. **Argo CD Applications**
   - Watches this repository
   - Automatically deploys dev
   - Deploys prod only on manual promotion

4. **Amazon EKS**
   - Runs the dev and prod workloads in isolated namespaces

---

## Repository Structure

```
project3-app-config/
├── apps/
│   ├── project3-dev-app.yaml
│   └── project3-prod-app.yaml
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   ├── ingress-dev-patch.yaml
    │   └── namespace-dev.yaml
    └── prod/
        ├── kustomization.yaml
        └── namespace-prod.yaml
```

---

## Kustomize Layout

The base defines shared resources; overlays provide environment‑specific customizations.

### Base
Contains resources common to both environments:

- Deployment  
- Service  
- Ingress  
- HPA  

### Overlays
Each environment has its own overlay:

- A unique namespace  
- Optional patches  
- Independent image tags per environment  
- Independent Argo CD Application  

---

## Argo CD Applications

Argo CD continuously monitors the following:

### Dev Application
- Syncs **automatically**
- Uses `overlays/dev`
- Applies into namespace `project3-dev`

### Prod Application
- Syncs automatically, **but only when the prod overlay changes**
- Uses `overlays/prod`
- Applies into namespace `project3-prod`
- Tag updates are **manual**
- Prod does **not** receive dev images automatically

---

# Manual Promotion: Dev to Prod

The dev environment deploys automatically whenever Project 1 updates the base image tag.

The prod environment updates only when manually promoted.

Use the steps below to promote a dev image to prod.

---

## 1. Determine the current dev image tag

You can check the dev tag from either location:

### Option A – Argo CD UI
- Open the `project3-nginx-dev` application  
- View the image under the application details or pod metadata  

### Option B – Git Repository

```bash
cat overlays/dev/kustomization.yaml
```

Look for the `newTag` field under `images`.

---

## 2. Update the prod overlay with the desired tag

Edit:

```
overlays/prod/kustomization.yaml
```

Replace the value in `newTag` with the dev tag:

```yaml
images:
  - name: project1-nginx
    newName: 349821334974.dkr.ecr.us-east-1.amazonaws.com/project1-nginx
    newTag: <DEV_TAG_HERE>
```

---

## 3. Commit and push the change

```bash
git add overlays/prod/kustomization.yaml
git commit -m "Promote image <DEV_TAG_HERE> to prod"
git push origin main
```

Argo CD will detect the change and automatically sync the prod environment.

---

## 4. Verify the deployment

```bash
kubectl get pods -n project3-prod -o wide
```

Ensure the running pods reflect the promoted tag.

---

# Documentation

Additional detailed documentation is available in the `docs/` directory:

- [Project Architecture](docs/architecture.md)
- [Argo CD Setup](docs/argo-setup.md)
- [Kustomize Structure](docs/kustomize-structure.md)
- [Deployment Flow](docs/deployment-flow.md)
- [AWS Load Balancer Controller](docs/load-balancer-controller.md)
- [OIDC & IRSA](docs/oidc-irsa.md)

The architecture diagram is located under `images/project3-architecture.png`.
