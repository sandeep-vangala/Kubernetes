---

## **Kubernetes Scaling & High Availability (HA) — Notes**

**Purpose:**
Scalability and HA ensure apps handle traffic spikes & failures **without downtime**.
Covers **pod-level scaling**, **node/infra scaling**, and **HA patterns**.

---

### **1. Horizontal Pod Autoscaler (HPA)**

**Definition:**

* Automatically adjusts **number of pod replicas** based on observed metrics.
* Default metric = CPU utilization (can also use memory, requests/sec via custom metrics).

**Why use:**

* Handles **variable workloads** (e.g., traffic spikes).
* Saves resources by scaling down in low demand.

**Example scenario:**

* Target CPU = 60% avg.
* If >60% → HPA increases replicas (up to `maxReplicas`).
* If < target → scales down.

**Considerations:**

* Works best with **stateless apps** (sessions externalized).
* Good for web workloads, APIs.
* Needs metrics pipeline for custom scaling.

**Example config (concept):**

```
maxReplicas: 10
targetCPUUtilizationPercentage: 60
```

---

### **2. Vertical Pod Autoscaler (VPA)**

**Definition:**

* Adjusts **resource requests/limits** (CPU, memory) for pods over time.

**Modes:**

* **Recommendation** mode → suggests sizes, no restarts.
* **Active** mode → applies changes, may restart pods.

**Best practice:**

* Use in **recommendation mode** for production (observe, then adjust manually).
* Or apply changes during **low-traffic windows**.

**Interaction with HPA:**

* Can use both, but carefully (avoid conflicts).
* HPA = scale **out** (replicas), VPA = scale **up/down** (resources per pod).

---

### **3. Cluster Autoscaler (CA)**

**Definition:**

* Scales **nodes** (infrastructure) up/down based on pod scheduling needs.

**How it works:**

* Watches for **unschedulable pods**.
* Requests more nodes from cloud provider.

**On AWS EKS:**

* Works with **Auto Scaling Groups / Managed Node Groups**.
* Min/max nodes set in ASG.
* CA adds/removes nodes accordingly.

**Alternatives:**

* **Karpenter**: Faster node provisioning, flexible instance selection.

**Notes:**

* Node provisioning takes time (minutes).
* Keep **buffer capacity** to avoid scaling delays.

---

### **4. Multi-AZ & Multi-Region for HA**

**Multi-AZ (common in prod):**

* Distribute nodes across multiple **Availability Zones**.
* Kubernetes & AWS ELB handle spreading automatically.

**Multi-Region (advanced, DR use cases):**

* Needed for **entire region outage** scenarios.
* Implemented via **multiple clusters** (one per region).
* Use **DNS-based failover** (active-passive or active-active).
* Complex: usually handled at app/DNS level.

**Example:**

* Fintech app → DR cluster in another region with DB replication.

---

### **5. Self-Healing & Health Checks**

**Built-in K8s features:**

* Restarts failed containers.
* Reschedules pods from failed nodes.

**Best practices:**

* Configure **liveness & readiness probes**.
* Use **Pod Disruption Budgets (PDBs)** to control voluntary disruptions.

**Example:**

* Deployment with 3 replicas, `minAvailable=2` in PDB → ensures at least 2 pods always up during maintenance.

---

### **6. HA of Kubernetes Components**

**If self-managing cluster:**

* Multiple **control plane nodes**.
* Multiple **CoreDNS pods** (DNS = critical service discovery).
* Ensure redundancy for metrics-server (needed for HPA).

---

### **7. Avoid Singletons**

* Avoid **single pod deployments** for production.
* If can’t be replicated (e.g., proprietary tools), use:

  * StatefulSet with backup.
  * External HA solution.

---

✅ **Key Takeaway:**
HPA handles **pod scaling**, VPA right-sizes resources, CA/Karpenter scales **nodes**, and HA patterns (multi-AZ, health checks, redundancy) keep your apps running smoothly — even under failures or traffic spikes.

---

If you want, I can also **add a visual cheat sheet diagram** linking **HPA → CA → Karpenter → Multi-AZ → DR** so the scaling and HA relationships are easier to remember. That would make this notes page feel like a complete “study-ready” reference.
