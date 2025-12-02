# AWS Load Balancer Controller

This document describes how the AWS Load Balancer Controller is integrated into
the EKS cluster used by Project 3. The controller is responsible for creating
and managing AWS Application Load Balancers (ALBs) based on Kubernetes Ingress
resources.

---

# 1. Purpose of the AWS Load Balancer Controller

The AWS Load Balancer Controller:

- Watches Kubernetes Ingress resources and Service annotations.
- Creates and configures Application Load Balancers (ALBs) in AWS.
- Attaches ALBs to the appropriate target groups (backed by Kubernetes Pods).
- Keeps AWS resources in sync with the desired state defined in Kubernetes
  manifests.

In this project:

- The application is exposed via an Ingress resource.
- The Ingress uses AWS Load Balancer Controller annotations to configure the ALB.
- The ALB provides external HTTPS access to the application.

---

# 2. Prerequisites

To use the AWS Load Balancer Controller with IRSA:

- EKS cluster with OIDC provider enabled.
- IAM policy that grants the controller permissions to manage AWS load balancers.
- IAM role for service account (IRSA) bound to the controller’s Kubernetes
  service account.

OIDC and IRSA details are documented in `docs/oidc-irsa.md`.

---

# 3. Installing the AWS Load Balancer Controller

## 3.1 Create the IAM Policy

An IAM policy is created to grant the controller permission to manage ALBs, target
groups, listeners, and related resources.

Example:

```bash
aws iam create-policy   --policy-name AWSLoadBalancerControllerIAMPolicy   --policy-document file://iam-policy.json
```

(where `iam-policy.json` contains the recommended AWS Load Balancer Controller
policy from the AWS documentation)

## 3.2 Create IAM Role for Service Account (IRSA)

Using `eksctl`, an IAM role is associated with a Kubernetes service account:

```bash
eksctl create iamserviceaccount   --cluster <EKS_CLUSTER_NAME>   --namespace kube-system   --name aws-load-balancer-controller   --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy   --approve   --override-existing-serviceaccounts
```

This creates:

- An IAM role with the required permissions.
- A Kubernetes service account annotated to assume that IAM role via OIDC.

## 3.3 Install the Controller via Helm

The controller is typically installed using Helm:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller   -n kube-system   --set clusterName=<EKS_CLUSTER_NAME>   --set serviceAccount.create=false   --set serviceAccount.name=aws-load-balancer-controller   --set region=<AWS_REGION>   --set vpcId=<VPC_ID>
```

This uses the existing IRSA-enabled service account created earlier.

---

# 4. Ingress Configuration

The application uses an Ingress resource in the base manifests, which is
consumed by both dev and prod overlays.

Key annotations (example):

```yaml
metadata:
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
```

The dev overlay may patch the host:

```yaml
spec:
  rules:
    - host: dev.project3.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80
```

A prod overlay would point to a production host, such as
`project3.example.com`.

---

# 5. Verifying the Controller

To verify that the AWS Load Balancer Controller is running:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

To see the created Ingress resources:

```bash
kubectl get ingress -A
```

You should see:

- Ingress objects in `project3-dev` and `project3-prod`.
- Corresponding ALBs in the AWS console.

---

# 6. Summary

- The AWS Load Balancer Controller integrates Kubernetes Ingress with AWS ALBs.
- It uses IRSA to obtain permissions securely via a Kubernetes service account.
- Ingress resources in the Project 3 GitOps repo drive how ALBs are created
  and configured in AWS.
