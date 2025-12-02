# Deployment Flow

This document explains the end-to-end deployment flow for Project 3, from a
developer commit in Project 1 to running pods in the EKS cluster. It covers
both the automated dev path and the manual prod promotion process.

---

# 1. Branch Strategy Overview

Project 1 uses two primary branches with different deployment behaviors:

- `main`
  - Classic push-based CI/CD (Project 1 original behavior).
  - Builds and pushes image to ECR.
  - Directly applies Kubernetes manifests to the cluster using `kubectl`.
  - Does **not** interact with the Project 3 GitOps repo.

- `project3-gitops`
  - GitOps-based CI/CD (Project 3 integration).
  - Builds and pushes image to ECR.
  - Updates the Project 3 GitOps repo (dev overlay).
  - Argo CD syncs the change and deploys to EKS via pull-based CD.
  - The direct `kubectl` deploy job is effectively disabled for this branch.

---

# 2. Dev Deployment Flow (Automated – GitOps)

This is the primary flow used by Project 3.

1. **Developer Commit (Project 1, `project3-gitops` branch)**
   - Developer pushes changes (e.g., to `app/index.html`).
   - Push targets the `project3-gitops` branch.

2. **GitHub Actions: Build and Push to ECR**
   - The `CI/CD to EKS` workflow runs.
   - Job `build-and-push`:
     - Checks out the repository.
     - Configures AWS credentials.
     - Logs in to Amazon ECR.
     - Builds a Docker image from `app/`.
     - Tags the image with the Git SHA.
     - Pushes the SHA-tagged image to ECR.

3. **Update Project 3 GitOps Repository**
   - The same job clones `theseanchristopher/project3-app-config`.
   - It checks out the `main` branch of the GitOps repo.
   - Installs `yq`.
   - Updates `overlays/dev/kustomization.yaml`:
     - Finds the `images` entry with `name: project1-nginx`.
     - Sets `newTag:` to the current Git SHA.
   - Commits and pushes the change back to the GitOps repository.

4. **Argo CD Sync (Dev Application)**
   - Argo CD watches the GitOps repo and detects the new commit.
   - The `project3-nginx-dev` application:
     - Uses `overlays/dev` as its path.
     - Renders the manifests via Kustomize.
     - Applies the changes to the `project3-dev` namespace.
   - The Deployment in `project3-dev` is updated to use the new image tag.

5. **Result**
   - The dev environment is automatically updated with the new version.
   - The prod environment remains unchanged until manually promoted.

---

# 3. Prod Deployment Flow (Manual Promotion)

Prod uses a controlled, manual promotion process.

1. **Verify Dev**
   - Ensure `project3-dev` is healthy.
   - Confirm the dev image tag is correct via:
     - Argo CD UI, or
     - `overlays/dev/kustomization.yaml`.

2. **Update Prod Overlay**
   - Edit `overlays/prod/kustomization.yaml` in the Project 3 repo.
   - Set `newTag:` to the desired SHA (usually the dev tag).

3. **Commit and Push**
   - Commit the change to the GitOps repo.
   - Push to the `main` branch.

4. **Argo CD Sync (Prod Application)**
   - Argo CD detects the change for the `project3-nginx-prod` application.
   - It renders and applies the prod overlay.
   - The Deployment in `project3-prod` updates to the promoted image tag.

5. **Result**
   - Prod runs the approved image version.
   - Dev can continue to move ahead with newer tags without affecting prod.

---

# 4. Classic CI/CD Path (Reference – Main Branch)

For completeness, the workflow still documents a classic path on `main`:

- Job `deploy`:
  - Runs only if `github.ref == 'refs/heads/main'`.
  - Uses `kubectl` to apply manifests directly from the Project 1 repo.
  - Performs a smoke test against the Ingress endpoint.

This job is effectively **disabled** when using the `project3-gitops` branch,
because that branch is dedicated to GitOps-based deployment via Argo CD.

---

# 5. Summary

- **Dev (GitOps)**
  - Triggered by commits to `project3-gitops`.
  - CI builds image and updates the dev overlay in the GitOps repo.
  - Argo CD deploys changes to `project3-dev`.

- **Prod (GitOps, Manual Promotion)**
  - Triggered by intentional edits to the prod overlay.
  - Argo CD deploys changes to `project3-prod`.

- **Main Branch (Classic CI/CD)**
  - Maintained for reference and backward compatibility.
  - Not used by Project 3’s GitOps flow.
