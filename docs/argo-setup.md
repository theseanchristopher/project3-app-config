# Argo CD Setup

This document describes how Argo CD is installed and configured for Project 3,
and how it manages the GitOps-based deployment of the application into the EKS
cluster.

---

## 1. Argo CD Installation

### 1.1 Namespace

Argo CD runs in its own namespace:

```bash
kubectl create namespace argocd
```

If the namespace already exists, this command is a no-op.

### 1.2 Install Argo CD Components

Argo CD was installed using the official Argo CD manifests:

```bash
kubectl apply -n argocd   -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This creates:

- Argo CD API server
- Application controller
- Repo server
- Redis
- Argo CD UI (server)

You can verify that the pods are running:

```bash
kubectl get pods -n argocd
```

---

## 2. Accessing the Argo CD UI

### 2.1 Port-Forward (Local Access)

For local access via port-forward:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then open:

- https://localhost:8080

### 2.2 Initial Admin Password

The initial admin password is stored in a Kubernetes secret:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd   -o jsonpath="{.data.password}" | base64 -d && echo
```

Log in to the UI as:

- Username: `admin`
- Password: (value from the command above)

The admin password should be changed immediately after the first login.

---

## 3. Configuring Argo CD for Project 3

### 3.1 Git Repository

Argo CD is configured to watch the Project 3 GitOps repository:

- Repository: `theseanchristopher/project3-app-config`
- Default branch: `main`

The repository contains:

- `apps/` – Argo CD Application manifests
- `base/` – Shared Kubernetes manifests
- `overlays/dev` – Dev Kustomize overlay
- `overlays/prod` – Prod Kustomize overlay

### 3.2 Applications

Two Argo CD Application objects are defined in `apps/`:

- `apps/project3-dev-app.yaml`
- `apps/project3-prod-app.yaml`

These are applied to the cluster:

```bash
kubectl apply -f apps/project3-dev-app.yaml
kubectl apply -f apps/project3-prod-app.yaml
```

Argo CD then manages:

- `project3-dev` namespace (dev overlay)
- `project3-prod` namespace (prod overlay)

---

## 4. Sync Policy

Both applications use automated sync:

- **Dev**
  - Auto-sync enabled
  - Prune enabled
  - Self-heal enabled
- **Prod**
  - Auto-sync enabled
  - Prune enabled
  - Self-heal enabled
  - Tag changes are made manually in the prod overlay

This means:

- Dev will automatically deploy each new tag written by the CI pipeline.
- Prod will only change when the prod overlay’s `newTag` is manually updated.

---

## 5. CLI Access (Optional)

To use the Argo CD CLI:

1. Download the `argocd` CLI binary.
2. Log in to the Argo CD server:

```bash
argocd login <ARGOCD_SERVER>   --username admin   --password <PASSWORD>   --insecure
```

3. List applications:

```bash
argocd app list
```

4. Sync a specific application:

```bash
argocd app sync project3-nginx-dev
argocd app sync project3-nginx-prod
```

The CLI is optional, but useful for scripting and debugging.
