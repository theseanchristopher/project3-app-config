# OIDC and IRSA (IAM Roles for Service Accounts)

This document explains how EKS OIDC and IAM Roles for Service Accounts (IRSA)
are used in the Project 3 environment, especially for the AWS Load Balancer
Controller.

---

# 1. Background

## 1.1 What is OIDC in EKS?

Amazon EKS can be configured with an OpenID Connect (OIDC) identity provider.
This allows IAM roles to be assumed by Kubernetes service accounts using
short-lived credentials, instead of long-lived access keys.

## 1.2 What is IRSA?

IAM Roles for Service Accounts (IRSA) is an AWS feature that allows you to:

- Associate an IAM role with a Kubernetes service account.
- Let pods using that service account assume the IAM role via OIDC.
- Avoid storing AWS credentials inside pods.

This is the recommended pattern for granting AWS permissions to workloads
running inside EKS.

---

# 2. Enabling OIDC for the EKS Cluster

The first step is to associate an OIDC provider with the EKS cluster.

Using `eksctl`:

```bash
eksctl utils associate-iam-oidc-provider   --cluster <EKS_CLUSTER_NAME>   --approve
```

This command:

- Creates an IAM OIDC identity provider for the cluster.
- Allows IAM roles to trust tokens issued by the cluster.

You can verify the OIDC provider in the AWS IAM console under
**Identity providers**.

---

# 3. Creating an IAM Role for a Service Account

To allow a controller (such as the AWS Load Balancer Controller) to manage AWS
resources, you create an IAM role and associate it with a Kubernetes service
account.

Example using `eksctl`:

```bash
eksctl create iamserviceaccount   --cluster <EKS_CLUSTER_NAME>   --namespace kube-system   --name aws-load-balancer-controller   --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy   --approve   --override-existing-serviceaccounts
```

This does the following:

- Creates an IAM role with the specified policy attached.
- Creates (or updates) a Kubernetes service account
  `kube-system/aws-load-balancer-controller`.
- Annotates the service account with the IAM role ARN and the OIDC provider
  information.

---

# 4. How Pods Assume the IAM Role

Once IRSA is configured:

1. A pod uses the annotated service account (for example,
   `aws-load-balancer-controller` in `kube-system`).
2. When the pod makes AWS API calls, the AWS SDK inside the pod retrieves
   credentials by:
   - Reading projected service account tokens.
   - Exchanging them with AWS STS via the OIDC provider.
3. AWS STS issues temporary credentials for the IAM role.
4. The pod uses these temporary credentials to access AWS resources, such as:
   - ELBv2 (ALBs)
   - Target groups
   - Listeners
   - Related networking resources

No static AWS access keys are stored inside the pod.

---

# 5. Why IRSA Matters in Project 3

In Project 3:

- The AWS Load Balancer Controller runs in EKS and needs permission to manage
  ALBs.
- IRSA provides these permissions **securely**, without embedding credentials
  in manifests or container images.
- The trust is based on:
  - The cluster’s OIDC provider.
  - The specific service account identity.

This is an important topic in interviews, as it demonstrates:

- Understanding of cloud-native security best practices.
- Familiarity with how EKS integrates with IAM.
- Awareness of least-privilege and credential management patterns.

---

# 6. Summary

- OIDC enables EKS to issue tokens that AWS IAM can trust.
- IRSA maps Kubernetes service accounts to IAM roles.
- The AWS Load Balancer Controller uses IRSA to manage AWS resources securely.
- This pattern is widely used for controllers and applications that need AWS
  API access (e.g., external-dns, cert-manager with Route 53, custom apps).
