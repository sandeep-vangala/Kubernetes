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


