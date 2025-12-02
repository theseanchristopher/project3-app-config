# Kustomize Structure

This document describes the Kustomize layout used in the Project 3 GitOps
repository (`project3-app-config`).

The repository follows a standard base + overlays pattern:

- `base/` – shared manifests
- `overlays/dev` – dev-specific overlay
- `overlays/prod` – prod-specific overlay

---

## 1. Repository Layout

```text
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

## 2. Base

The `base/` directory holds the common Kubernetes resources used by both
environments.

### 2.1 Base Kustomization

`base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - ingress.yaml
  - hpa.yaml
```

### 2.2 Base Resources

- `deployment.yaml`
  - Defines the main `nginx-deployment` for the application.
- `service.yaml`
  - Exposes the deployment inside the cluster.
- `ingress.yaml`
  - Configures the external access route (e.g., ALB ingress).
- `hpa.yaml`
  - Defines the Horizontal Pod Autoscaler for the deployment.

The base does not specify a namespace. Namespaces are layered on at the
overlay level.

---

## 3. Dev Overlay

The dev overlay customizes the base for the `project3-dev` namespace.

### 3.1 Dev Kustomization

`overlays/dev/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
  - namespace-dev.yaml

namespace: project3-dev

patchesStrategicMerge:
  - ingress-dev-patch.yaml

images:
  - name: project1-nginx
    newName: 349821334974.dkr.ecr.us-east-1.amazonaws.com/project1-nginx
    newTag: <LATEST_DEV_TAG>
```

Key points:

- `namespace: project3-dev` – all namespaced resources are deployed into
  `project3-dev`.
- `namespace-dev.yaml` – ensures the namespace exists.
- `ingress-dev-patch.yaml` – overrides hostnames or annotations for dev.
- `images:` – defines the container image; `newTag` is updated automatically
  by the Project 1 CI pipeline.

### 3.2 Namespace (Dev)

`overlays/dev/namespace-dev.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: project3-dev
  labels:
    app: project1-nginx
    environment: dev
```

---

## 4. Prod Overlay

The prod overlay customizes the base for the `project3-prod` namespace.

### 4.1 Prod Kustomization

`overlays/prod/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
  - namespace-prod.yaml

namespace: project3-prod

images:
  - name: project1-nginx
    newName: 349821334974.dkr.ecr.us-east-1.amazonaws.com/project1-nginx
    newTag: <PROD_TAG>
```

Key points:

- `namespace: project3-prod` – all namespaced resources deploy into
  `project3-prod`.
- `namespace-prod.yaml` – ensures the namespace exists.
- `images:` – `newTag` is updated *manually* as part of the promotion
  process.

### 4.2 Namespace (Prod)

`overlays/prod/namespace-prod.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: project3-prod
  labels:
    app: project1-nginx
    environment: prod
```

---

## 5. How Kustomize Fits into GitOps

- Argo CD points to `overlays/dev` and `overlays/prod` as the `path` for the
  respective applications.
- Kustomize composes:
  - `base/` resources
  - Overlay-specific namespace and patches
  - The correct image tag for each environment
- The final rendered manifests are what Argo CD applies to the cluster.

This structure keeps shared configuration in one place while allowing each
environment to manage its own namespace, ingress settings, and image tags.
