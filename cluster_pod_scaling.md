# Notes: Scaling and High Availability in Kubernetes

## 1. Introduction: Why Scaling & HA Matter
- **Scalability** and **high availability (HA)** are fundamental for reliable, resilient production systems.
- Main goals:
  - **Handle increased traffic**: Application or infrastructure should scale as demand rises.
  - **Tolerate failures**: Apps and platform survive crashes or disruptions without experiencing downtime.

## 2. Scaling at Multiple Levels
- **Pod/Application Scaling**: Adjust how many instances (replicas) of a particular app/component are running.
- **Node/Cluster Scaling**: Adjust the size/capacity of the infrastructure (nodes in the Kubernetes cluster).

***

## 3. Pod Scaling Mechanisms

### Horizontal Pod Autoscaler (HPA)
- **Purpose**: Automatically adjusts *number of pod replicas* for deployments or statefulsets.
- **How it works**:
  - Monitors certain **metrics** (CPU by default, but can use custom metrics like requests per second, memory, etc.).
  - When metric exceeds threshold (e.g., average CPU >60%), more pods are launched (up to a set max).
  - Pods are reduced if load drops → **saves resources/cost**.
- **Requirements/Best Practices**:
  - Application should be **stateless** or handle scaling well (e.g., **externalize session management**).
  - Example: Configure HPA to keep average CPU at 60%; if above, scale more pods (up to 10).
- **Advanced**:
  - Can work with custom metrics for things like **requests per second**.

### Vertical Pod Autoscaler (VPA)
- **Purpose**: Suggests or automatically adjusts **resource requests/limits** of pods based on actual usage.
- **How it works**:
  - VPA in "active mode" may cause pod restarts (as pods need to be recreated with new sizes).
  - Often used in "recommendation mode" in production (observe and then adjust limits manually).
  - Can schedule VPA actions during low-traffic times.
- **Interaction with HPA**:
  - Can be used together (newer Kubernetes versions improve compatibility).
  - In practice, most rely on HPA for *scaling out* and VPA as a guide for *right-sizing* pods.

***

## 4. Cluster-Level Scaling

### Cluster Autoscaler (CA)
- **Purpose**: Adds/removes worker nodes as needed, ensuring there’s capacity for pending pods.
- **How it works**:
  - Watches for pods that can’t be scheduled due to lack of resources.
  - Signals cloud provider to add more nodes (e.g., AWS EKS integrates CA with Auto Scaling Groups/Managed Node Groups).
  - Automatically scales down underutilized nodes (if pods can be rescheduled elsewhere).
- **Critical for**: Handling *large, sudden traffic spikes* (e.g., flash sales).
- **AWS Karpenter**: An advanced autoscaler alternative for faster and more flexible node provisioning (e.g., dynamic instance types).
- **Best Practices**:
  - **Consider lead time**: Node startup can take a few minutes; slight over-provisioning helps avoid delays or downtime.
  - **Autoscaling policies**: Should ensure there's some headroom to absorb traffic changes.

***

## 5. High Availability Patterns

### Multi-Zone Deployments (within a region)
- **How**: Run nodes across **multiple Availability Zones (AZs)**.
- Kubernetes can spread pods using **topology spread constraints**.
- **Cloud controllers/load balancers** (like AWS ELB) auto-distribute traffic to healthy pods in all AZs.
- **Main approach for HA**: Multi-AZ within a region is sufficient for most production clusters.

### Multi-Region Deployments
- **Why**: Handle complete regional outages (useful for fintech, strict DR requirements).
- **How**:
  - Use **separate clusters in multiple regions**.
  - Application-level or DNS-based (e.g., global load balancer, active-passive or active-active).
  - Cluster Federation (advanced, often complex), but simpler approaches prefer using CI/CD pipelines to deploy to both clusters.
  - Data replication must be considered separately (e.g., DB replication).
- **When needed**: Only if the business demands **disaster recovery (DR)** beyond what multi-AZ provides.

***

## 6. Self-Healing and Health-Checks

### Liveness/Readiness Probes
- Kubernetes will automatically:
  - Restart containers that fail **liveness probes** or crash.
  - Reschedule pods when a node fails.
- **Pod Disruption Budget (PDB)**:
  - Controls minimum number of pods available during maintenance or voluntary node shutdowns (e.g., node drain).
  - Example: With 3 replicas and PDB `minAvailable: 2`, Kubernetes won’t disrupt more than 1 pod at a time.

***

## 7. High Availability of Kubernetes Components

### Control Plane
- **If self-managing**: Always run multiple control plane nodes to avoid a single point of failure.

### Core Kubernetes Add-ons
- **CoreDNS**: Critical for service discovery. Should have at least 2 replicas. If it fails, the cluster can’t resolve services.
- **Metrics Server**: Important if used for HPA decisions.
- **General Rule**: Avoid deploying single-instances of any critical control-plane or add-on.

***

## 8. Singleton Workloads and Non-replicable Apps
- **Avoid single-pod deployments** in production.
- If some workloads truly can’t be replicated (e.g., legacy third-party apps):
  - Use a **statefulset with a backup**.
  - Or replace with an **external HA alternative** if available.

***

## Summary Table: Key Strategies

| Scaling/HA Technique   | Layer         | When/Why to Use      | Notes            |
|-----------------------|---------------|----------------------|------------------|
| HPA                   | Pod/App       | Variable workload    | Stateless apps   |
| VPA                   | Pod/App       | Right-sizing pods    | May restart pods |
| Cluster Autoscaler    | Node/Cluster  | On-demand capacity   | Needs headroom   |
| Multi-AZ              | Infra/App     | Most prod workloads  | Basic HA         |
| Multi-Region          | Infra/App     | DR, strict SLAs      | Advanced, complex|
| PDB                   | App           | Controlled disruption| Set min replicas |
| Multi control-plane   | Control plane | Avoid K8s downtime   | Critical add-ons |

***

## 9. Best Practices Checklist

- Use HPA and VPA appropriately—monitor and tune.
- Enable Cluster Autoscaler with enough node headroom.
- Run nodes across multiple AZs for default HA.
- Consider multi-region only if required for DR.
- Set up liveness/readiness probes and PDBs for all critical workloads.
- Run control plane and critical add-ons with high-availability configs.
- Avoid single-instance pods in production. For stateful/non-replicable apps, find HA alternatives.

***

**In summary:**  
To build a robust, production-ready Kubernetes cluster, combine application auto-scaling, node auto-scaling, and infrastructure HA—backed by failover, health checks, and redundancy at every critical tier.
