Perfect 👍 — here’s a **printable runbook** for doing a **stable, safe EKS upgrade with Terraform in production**. It’s formatted as a step-by-step operational checklist you can actually use during a maintenance window.

---

# 📘 Runbook — Safe EKS Upgrade with Terraform (Production)

---

## 🎯 Goal

Upgrade an EKS cluster managed by Terraform to a **stable, supported Kubernetes version** (latest patch in standard support). Ensure **zero/low downtime**, avoid deprecated APIs, and stay compliant with AWS/EKS support lifecycle.

---

## 🛠️ Prerequisites

* **IAM Access**: Terraform runner role must have `eks:*`, `ec2:*`, `iam:*`, `ssm:*`.
* **Terraform Backend**: Remote state (S3 + DynamoDB lock).
* **Current Inventory**:

  * Current EKS version (`kubectl version` or `aws eks describe-cluster`).
  * Node group versions & AMIs.
  * Add-ons: VPC CNI, CoreDNS, kube-proxy.
  * Controllers: cluster-autoscaler, ingress, metrics-server.

---

## 🚦 Step 0 — Pre-flight Checks

1. **Pick stable target version**

   * Go to [EKS Kubernetes versions page](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html).
   * Choose a minor that’s in **standard support**, and pin to the **latest patch**. Example: `1.33.4`.
   * Update `variables.tf`:

     ```hcl
     variable "k8s_version" {
       type    = string
       default = "1.33.4"
     }
     ```

2. **Scan manifests for deprecations**

   ```bash
   kubectl krew install deprecations
   kubectl deprecations --k8s-version=1.33.4
   ```

   Or use `pluto detect-files` on Helm charts.

3. **Validate readiness**

   * Ensure **PodDisruptionBudgets** exist for critical workloads.
   * Verify readiness/liveness probes.
   * Scale up replicas for critical services if needed.

4. **Backups**

   * DB snapshots (RDS, EBS, EFS).
   * Application state backup.

---

## 🔄 Step 1 — Upgrade Control Plane

Edit Terraform:

```hcl
resource "aws_eks_cluster" "this" {
  name    = var.name
  version = var.k8s_version  # e.g. "1.33.4"
  # other config...
}
```

Apply control plane only:

```bash
terraform plan -target=aws_eks_cluster.this
terraform apply -target=aws_eks_cluster.this
```

**Verification**:

```bash
kubectl version --short
# Server Version: v1.33.4
kubectl get nodes
# Nodes still on old version → expected
```

---

## 🔄 Step 2 — Upgrade Node Groups

Update Terraform:

```hcl
resource "aws_eks_node_group" "workers" {
  cluster_name    = aws_eks_cluster.this.name
  node_group_name = "workers"
  version         = var.k8s_version
  update_config {
    max_unavailable = 1
  }
  # optional for full pin:
  # release_version = "1.33.x-yyyymmdd"
}
```

Apply:

```bash
terraform plan -target=aws_eks_node_group.workers
terraform apply -target=aws_eks_node_group.workers
```

**During rollout**:

* Watch pods migrate:

  ```bash
  kubectl get pods -o wide
  ```
* Gracefully drain old nodes:

  ```bash
  kubectl cordon <old-node>
  kubectl drain <old-node> --ignore-daemonsets --delete-emptydir-data
  ```

---

## 🔄 Step 3 — Upgrade EKS Add-ons

Pin versions in Terraform:

```hcl
resource "aws_eks_addon" "vpc_cni" {
  cluster_name       = aws_eks_cluster.this.name
  addon_name         = "vpc-cni"
  addon_version      = "v1.16.1-eksbuild.1"
  resolve_conflicts  = "OVERWRITE"
}

resource "aws_eks_addon" "coredns" {
  cluster_name       = aws_eks_cluster.this.name
  addon_name         = "coredns"
  addon_version      = "v1.11.1-eksbuild.4"
  resolve_conflicts  = "OVERWRITE"
}

resource "aws_eks_addon" "kube_proxy" {
  cluster_name       = aws_eks_cluster.this.name
  addon_name         = "kube-proxy"
  addon_version      = "v1.33.4-eksbuild.2"
  resolve_conflicts  = "OVERWRITE"
}
```

Apply:

```bash
terraform plan
terraform apply
```

**Verification**:

```bash
kubectl get pods -n kube-system
```

---

## 🔄 Step 4 — Upgrade Helm-managed Controllers

Example: Cluster Autoscaler:

```bash
helm repo update
helm upgrade cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=<your-cluster> \
  --version 9.35.0 # version tested for 1.33.x
```

Repeat for ingress controllers, metrics-server, CSI drivers.

---

## ✅ Step 5 — Post-upgrade Validation

* Run smoke tests / e2e tests.
* Monitor dashboards (latency, error rate, pod restarts).
* Confirm API versions (`kubectl api-resources`).
* Ensure all workloads are healthy:

  ```bash
  kubectl get pods --all-namespaces
  ```

---

## 🔄 Rollback Strategy

* **Control Plane**: Cannot roll back minor. Only option: create new cluster with previous version and migrate workloads (blue/green).
* **Node Groups**: Roll back `version` or `release_version` in Terraform → apply.
* **Add-ons**: Downgrade by re-pinning `addon_version`.
* **Workloads**: Helm rollback or redeploy old chart version.

---

## 🔐 Key Best Practices

* Pin versions (cluster minor+patch, nodegroup release\_version, addon\_version, Helm chart versions).
* Never use `latest`.
* Test in **staging** before prod.
* Always upgrade **addons + controllers** immediately after control plane + nodes.
* Keep upgrades within AWS standard support window (14 months).

---

# 🟢 Quick Memory Mnemonic

**Plan → Control Plane → Nodes → Add-ons → Controllers → Validate**

---

Would you like me to also make a **Markdown cheatsheet version (1-page summary)** of this runbook that you can keep handy for interviews and on-call upgrades?



---

# 🧱 Kubernetes – Production Upgrade Notes

### 📌 Purpose:

This guide outlines best practices and step-by-step procedures for safely upgrading Kubernetes clusters (specifically EKS) in a production environment.

---

## 🔎 Overview of Kubernetes Release Notes (v1.32 → v1.33)

Before performing any upgrade, carefully review the official Kubernetes release notes to understand:

* **New features and enhancements**
* **Deprecated APIs** or removed components
* **Security patches**
* **Behavior changes** affecting workloads or controllers
* **Component version compatibility** (Kubelet, kube-proxy, etc.)

Refer to:

* [Kubernetes Changelog 1.33](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.33.md)
* [Kubernetes Changelog 1.32](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.32.md)

---

## ✅ Prerequisites

1. **Cordon Nodes**

   * Prevent scheduling new pods on nodes.
   * Avoid application updates or deployments during upgrade.

2. **Review Release Notes**

   * Go through the 1.32 → 1.33 change log thoroughly.
   * Identify any API deprecations, version mismatches, or breaking changes.

3. **Environment Upgrade Strategy**

   * Kubernetes upgrades on **EKS are irreversible**.
   * Always upgrade **lower environments (dev, QA)** first.
   * Wait at least **1 week** post-validation before upgrading **production**.

4. **Version Compatibility**

   * **Control Plane** and **Node Groups** must be on the **same Kubernetes version**.
   * Don’t proceed until they match.

5. **Cluster Autoscaler Compatibility**

   * Ensure Cluster Autoscaler version supports the target Kubernetes version.

6. **IP Address Availability**

   * Ensure **at least 5 free IP addresses** are available in each subnet during the upgrade.

7. **Kubelet Version Match**

   * The `kubelet` on nodes must match the version of the control plane.

---

## 🔄 Actual Upgrade Process

### 1. 🚀 Upgrade Control Plane (Approx. 30 mins)

* EKS-managed Kubernetes offers:

  * **High Availability**
  * **Disaster Recovery**
  * **Built-in Security**
  * **API Server Auto-Scaling**
* Control plane upgrade is automated and non-disruptive (but should still be tested beforehand).

### 2. 🧱 Upgrade Node Groups (1 hour+)

* Upgrade **Managed Node Groups** using **rolling updates**.
* This uses a **node-by-node rollout approach** to minimize disruption.

### 3. 🔧 Upgrade Add-ons (Approx. 30 mins)

* Includes:

  * **CoreDNS**
  * **Kube-Proxy**
  * **VPC CNI plugin**
* Ensure each add-on version is compatible with the upgraded control plane.

---

## 🧪 Post-Upgrade – Testing

* Run **functional, regression, and integration tests**.
* Validate that:

  * Workloads are running smoothly.
  * No pod crashes, deployment failures, or network issues.
  * Monitoring and logging are intact.

---

Certainly, here's the continuation of your **Kubernetes Production Upgrade Notes** – specifically for performing **EKS upgrades using Terraform**.

This section assumes you are managing EKS cluster, node groups, and add-ons via **Terraform**, and outlines a safe and production-grade upgrade approach.

---

## ☁️ Upgrading EKS via Terraform (IaC)

Using Terraform ensures version-controlled, repeatable upgrades of EKS clusters, node groups, and add-ons. However, caution is required for production systems.

---

### ⚙️ High-Level Terraform Upgrade Flow

1. **Update Terraform module versions**
2. **Update desired Kubernetes version**
3. **Apply changes for Control Plane (EKS Cluster)**
4. **Apply changes for Node Groups**
5. **Upgrade Add-ons via Terraform**
6. **Post-upgrade validations**

---

### 🔧 1. Update Kubernetes Version in EKS Module

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = ">=19.0.0"  # Use latest stable version

  cluster_name    = "your-cluster-name"
  cluster_version = "1.33"

  # ... rest of your EKS config
}
```

> 🔍 Make sure to read the [terraform-aws-eks module changelog](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/CHANGELOG.md) for any breaking changes.

---

### ⚙️ 2. Upgrade Managed Node Groups

Update the Kubernetes version in each managed node group:

```hcl
  managed_node_groups = {
    default = {
      desired_size = 3
      max_size     = 5
      min_size     = 1

      instance_types = ["t3.medium"]

      ami_type       = "AL2_x86_64"
      capacity_type  = "ON_DEMAND"
      version        = "1.33"   # <- Target version
    }
  }
```

> ☝️ When applied, Terraform will perform a **rolling upgrade** of the nodes in the group.

---

### 🧩 3. Upgrade Core Add-ons (CoreDNS, kube-proxy, VPC CNI)

Add the add-ons block or update their versions:

```hcl
module "eks" {
  # ... existing config

  eks_addons = {
    coredns = {
      addon_version     = "v1.11.1-eksbuild.1"
      resolve_conflicts = "OVERWRITE"
    }
    kube-proxy = {
      addon_version     = "v1.33.0-eksbuild.1"
      resolve_conflicts = "OVERWRITE"
    }
    vpc-cni = {
      addon_version     = "v1.18.1-eksbuild.1"
      resolve_conflicts = "OVERWRITE"
    }
  }
}
```

> ✅ Use the exact version from AWS-supported [EKS Add-on versions](https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html).

---

### 🚀 4. Terraform Commands

```bash
# Initialize any updated modules or providers
terraform init

# Review the planned changes (safe check)
terraform plan

# Apply the changes in stages
terraform apply
```

> ✅ Apply in the following order:
>
> 1. Control Plane upgrade
> 2. Node Group upgrade
> 3. Add-on upgrade

---

### 🧪 5. Post-Upgrade Validation (after Terraform apply)

* Confirm new Kubernetes version:

  ```bash
  kubectl version --short
  ```
* Check node group versions:

  ```bash
  kubectl get nodes -o wide
  ```
* Ensure pods are running fine:

  ```bash
  kubectl get pods -A
  ```
* Check add-on versions via console or:

  ```bash
  aws eks describe-addon --cluster-name <name> --addon-name <addon>
  ```

---

## ✅ Best Practices for Terraform-Based EKS Upgrades

| Step               | Description                                                                |
| ------------------ | -------------------------------------------------------------------------- |
| 🔒 Backup          | Ensure state file is backed up (e.g., remote S3 + DynamoDB)                |
| 🔀 Staging         | Upgrade in lower environments before production                            |
| 🔄 Rolling Updates | Ensure node groups use managed rollout strategy                            |
| 💬 Communication   | Notify stakeholders before upgrade window                                  |
| 🛡️ Monitoring     | Enable detailed monitoring via CloudWatch/Grafana during and after upgrade |
| 🧪 Testing         | Run integration and smoke tests after each upgrade phase                   |

---


