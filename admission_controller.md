---

### **What is an Admission Controller in Kubernetes?**

An **admission controller** is a Kubernetes component that intercepts and processes requests to the Kubernetes API server before they are persisted or executed. They act as gatekeepers, enforcing policies, validating configurations, or mutating resources (e.g., adding defaults) during the creation, update, or deletion of Kubernetes objects (like pods, deployments, or services).

There are two types of admission controllers:
1. **Validating Admission Controllers**: Check if a request complies with defined policies and approve or reject it.
2. **Mutating Admission Controllers**: Modify the request (e.g., inject sidecar containers or set default values) before it’s processed.

Admission controllers run in two phases:
- **Mutating phase**: Modifies the object if needed.
- **Validating phase**: Ensures the object meets requirements.

---

### **Use Cases of Admission Controllers**

Admission controllers are critical for enforcing security, compliance, and operational best practices in a Kubernetes cluster. Here are common use cases:

1. **Security Policy Enforcement**:
   - **Pod Security Standards**: Enforce Pod Security Standards (PSS) or Pod Security Admission (PSA) to restrict pods from running as root, using privileged containers, or accessing host resources.
   - **Image Validation**: Ensure only approved container images (e.g., from a trusted registry) are used. For example, block images from unverified sources.
   - Example: Deny pods using `latest` tags or unapproved registries.

2. **Resource Management**:
   - **Resource Limits**: Automatically set CPU/memory limits and requests if not specified (mutating).
   - **Quota Enforcement**: Validate that resource creation doesn’t exceed namespace quotas.
   - Example: Add default resource limits to pods via a mutating webhook.

3. **Configuration Validation**:
   - Ensure Kubernetes objects adhere to organizational standards (e.g., require specific labels, annotations, or namespaces).
   - Example: Reject deployments missing a required `app` label.

4. **Network Policy Enforcement**:
   - Validate or inject network policies to control pod communication.
   - Example: Require all pods to have a network policy defined.

5. **Sidecar Injection**:
   - Automatically inject sidecar containers (e.g., for service meshes like Istio or Linkerd) into pods.
   - Example: Inject an Envoy proxy for traffic management.

6. **Custom Policy Enforcement**:
   - Use tools like **Open Policy Agent (OPA) Gatekeeper** or **Kyverno** to enforce custom policies via dynamic admission controllers.
   - Example: Enforce that all ingress resources use HTTPS or restrict certain namespaces to specific teams.

7. **Preventing Misconfigurations**:
   - Block invalid or risky configurations, such as pods with `hostNetwork: true` or excessive permissions.
   - Example: Deny pods that attempt to mount sensitive host paths.

8. **Compliance and Auditing**:
   - Ensure workloads comply with regulatory requirements (e.g., GDPR, HIPAA) by enforcing specific configurations.
   - Example: Require encryption annotations for persistent volumes.

---

### **Do We Use Admission Controllers in EKS by Default?**

Yes, Amazon EKS clusters have several **built-in admission controllers** enabled by default, as they are part of the Kubernetes API server’s default configuration. However, the exact set of enabled controllers depends on the Kubernetes version and EKS configuration.

#### **Default Admission Controllers in EKS**
EKS uses the same default admission controllers as upstream Kubernetes, with some AWS-specific tweaks. As of Kubernetes versions commonly used in EKS (e.g., 1.27–1.30 in 2025), the following admission controllers are typically enabled by default:

1. **NamespaceLifecycle**: Prevents deletion of default namespaces and ensures resources are deleted when their namespace is deleted.
2. **LimitRanger**: Enforces resource limits and requests if defined, preventing pods from consuming excessive resources.
3. **ServiceAccount**: Automatically assigns a default service account to pods if none is specified.
4. **DefaultStorageClass**: Assigns a default storage class to PersistentVolumeClaims (PVCs) if none is specified.
5. **ResourceQuota**: Enforces namespace resource quotas if defined.
6. **DefaultTolerationSeconds**: Sets default toleration times for pod taints.
7. **MutatingAdmissionWebhook**: Allows custom mutating webhooks (if configured).
8. **ValidatingAdmissionWebhook**: Allows custom validating webhooks (if configured).
9. **PodSecurity**: Enforces Pod Security Standards (replacing the deprecated PodSecurityPolicy in newer versions).
10. **NodeRestriction**: Limits what kubelets can modify, enhancing cluster security.
11. **EventRateLimit**: Limits the rate of events to prevent API server overload (optional, often enabled in EKS).

#### **Not Enabled by Default**
Some powerful admission controllers are **not** enabled by default in EKS (or Kubernetes) to avoid breaking workloads or requiring additional configuration:
- **PodSecurityPolicy** (deprecated in Kubernetes 1.21, removed in 1.25): Not enabled by default due to complexity.
- **ImagePolicyWebhook**: Requires external configuration for image validation.
- **AlwaysPullImages**: Forces image pulls for security but not enabled by default.
- Custom controllers like **OPA Gatekeeper** or **Kyverno** need to be installed separately.

#### **EKS-Specific Notes**
- **AWS-Specific Controllers**: EKS doesn’t add proprietary admission controllers but integrates with AWS services (e.g., IAM roles for service accounts). You may need to configure additional controllers for AWS-specific use cases, like validating ALB Ingress configurations.
- **Pod Security Admission (PSA)**: Since Kubernetes 1.25, EKS uses PSA by default to enforce Pod Security Standards. You can configure it to apply `privileged`, `baseline`, or `restricted` policies.
- **Customization**: You can enable/disable admission controllers in EKS by modifying the API server configuration, but this requires advanced access (e.g., via EKS managed node groups or custom AMIs) and is not typically done in managed EKS clusters.

#### **Verifying Enabled Admission Controllers in EKS**
To check which admission controllers are enabled in your EKS cluster:
1. Access the API server configuration (not directly editable in managed EKS clusters).
2. Alternatively, check the Kubernetes version and refer to its default admission controller list. For EKS, you can run:
   ```bash
   kubectl get --raw=/api/v1 | jq '.resources[] | select(.name=="admissionregistration.k8s.io/v1")'
   ```
   This confirms the presence of admission webhook APIs, but specific controller details may require AWS support or documentation for your EKS version.

---

### **Practical Example: Setting Up a Custom Admission Controller in EKS**

Let’s say you want to enforce a policy in your EKS cluster to block pods from running as root, using **Kyverno** as a dynamic admission controller.

#### **Step 1: Install Kyverno**
```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```

#### **Step 2: Create a Policy**
Create a Kyverno policy to enforce non-root containers:
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-non-root
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-run-as-non-root
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Pods must not run as root."
      pattern:
        spec:
          containers:
          - securityContext:
              runAsNonRoot: true
```

Apply the policy:
```bash
kubectl apply -f non-root-policy.yaml
```

#### **Step 3: Test the Policy**
Create a pod that violates the policy:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      runAsNonRoot: false
```

Apply it:
```bash
kubectl apply -f test-pod.yaml
```

The pod will be rejected with an error message indicating the policy violation.

---

### **Do You Need Custom Admission Controllers in EKS?**
By default, EKS’s built-in admission controllers cover basic security and resource management. However, for advanced use cases (e.g., custom security policies, service mesh integration, or compliance requirements), you’ll need to deploy tools like:
- **OPA Gatekeeper**: For policy-as-code with Rego.
- **Kyverno**: For YAML-based policies and simpler setup.
- **AWS-specific webhooks**: For validating AWS resources like ALB Ingress.

---

### **Key Considerations for EKS**
- **Managed Control Plane**: EKS manages the API server, so you cannot directly modify the `--enable-admission-plugins` flag. Use dynamic webhooks (via `ValidatingWebhookConfiguration` or `MutatingWebhookConfiguration`) for custom policies.
- **Performance**: Admission controllers add latency to API requests. Test custom controllers in a non-production environment.
- **IAM Integration**: Ensure proper IAM roles for controllers interacting with AWS services (e.g., Route53 for ExternalDNS, as in your previous query).
- **EKS Upgrades**: Kubernetes version upgrades in EKS may change default admission controllers. Check AWS documentation for your EKS version.

---

### **Summary**
- **Use Cases**: Admission controllers enforce security, compliance, resource limits, and custom policies, and can mutate resources (e.g., sidecar injection).
- **EKS Defaults**: EKS enables a standard set of Kubernetes admission controllers (e.g., NamespaceLifecycle, LimitRanger, PodSecurity), but advanced controllers like OPA or Kyverno require manual setup.
- **Custom Setup**: Use tools like Kyverno or OPA Gatekeeper for custom policies in EKS, leveraging webhooks for flexibility.



In **EKS (Elastic Kubernetes Service)**, an **Admission Controller** is a piece of code that intercepts API server requests *after authentication/authorization but before persistence into etcd*. It can **mutate** (modify) or **validate** resources being created/updated.

Since EKS is managed, AWS allows you to enable and use admission controllers via **MutatingAdmissionWebhook** and **ValidatingAdmissionWebhook**.

---

### 🔹 Use Cases of Admission Controllers in EKS

#### 1. **Security & Compliance**

* **Restricting privileged pods**
  Prevent developers from deploying pods with `privileged: true` or `hostNetwork: true`.
* **Enforcing resource requests/limits**
  Admission controller can reject pods without `cpu/memory` limits.
* **Restricting images**
  Ensure workloads only pull from approved registries (ECR, not Docker Hub).

---

#### 2. **Governance & Policy Enforcement**

* **Namespace-level policies**
  For example, deny creating resources directly in the `default` namespace.
* **Label/annotation enforcement**
  Ensure pods, services, or deployments have required labels (like `owner`, `cost-center`, `environment`).
* **Network policies enforcement**
  Prevent creating a pod in certain namespaces without a corresponding NetworkPolicy.

---

#### 3. **Operational Use Cases**

* **Automatic sidecar injection**
  Mutating webhook can inject sidecars (like Envoy for service mesh or Fluentd/Datadog agents).
  Example: Istio or App Mesh in EKS uses admission webhooks for automatic sidecar injection.
* **Defaulting values**
  Add defaults like tolerations, affinity, secrets, or volume mounts if a user forgets.
* **Quota enforcement**
  Prevent resource creation if it exceeds cluster/team quota.

---

#### 4. **Observability & Monitoring**

* **Inject monitoring/logging agents**
  Admission controller mutates pod specs to add Datadog, Prometheus exporters, or custom sidecars.
* **Traceability**
  Enforce annotations for trace IDs or logging correlation IDs.

---

### 🔹 EKS-Specific Examples

1. **App Mesh Sidecar Injection** → AWS App Mesh uses an admission controller to automatically inject Envoy sidecars into pods.
2. **OPA Gatekeeper in EKS** → Uses admission webhooks to enforce policies like "no container runs as root" or "all services must be internal-only".
3. **Kyverno on EKS** → Mutating/validating admission controller that applies security and compliance rules dynamically.
4. **Datadog Admission Controller** → Auto-injects the monitoring agent into EKS pods.

---

👉 In short:

* **Mutating Admission Controllers** = Modify requests (inject sidecars, add labels).
* **Validating Admission Controllers** = Enforce rules (deny privileged pods, block wrong namespaces).

---

For the interview

Perfect 👍 let’s build a **real-world EKS admission controller use case** you can directly use in interviews.

---

## 🎯 Scenario

Your company runs workloads on **Amazon EKS**. The **security team** raised a concern:

> Developers sometimes deploy pods running as `root`, which violates compliance.

You are asked to **enforce a policy** that prevents any pod in the cluster from running as root.

---

## 🛠️ Solution with Admission Controller (OPA Gatekeeper in EKS)

We deploy **OPA Gatekeeper**, which uses **ValidatingAdmissionWebhook** under the hood. It will **reject any pod** that does not comply.

---

### Step 1: Install Gatekeeper on EKS

```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml
```

---

### Step 2: Define a ConstraintTemplate (rule definition)

This template creates a reusable policy that checks for root user IDs.

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8sdisallowrootuser
spec:
  crd:
    spec:
      names:
        kind: K8sDisallowRootUser
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sdisallowrootuser

        violation[{"msg": msg}] {
          input.review.kind.kind == "Pod"
          container := input.review.object.spec.containers[_]
          not container.securityContext.runAsNonRoot
          msg := sprintf("Container %v is running as root. Set runAsNonRoot: true", [container.name])
        }
```

---

### Step 3: Create a Constraint (apply policy cluster-wide)

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDisallowRootUser
metadata:
  name: disallow-root
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

---

### Step 4: Test the Policy

**✅ Allowed Pod (compliant)**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
spec:
  containers:
    - name: app
      image: nginx
      securityContext:
        runAsNonRoot: true
```

This pod will be admitted successfully.

---

**❌ Blocked Pod (non-compliant)**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
spec:
  containers:
    - name: app
      image: nginx
```

This pod will be **rejected** with error:

```
admission webhook "validation.gatekeeper.sh" denied the request: 
Container app is running as root. Set runAsNonRoot: true
```

---

## ✅ Interview Answer (Storytelling)

\*"In my last project on EKS, we needed to enforce compliance so that no workloads ran as root. I implemented an admission controller using **OPA Gatekeeper** with a ValidatingAdmissionWebhook. We wrote a policy that automatically blocked pods missing `securityContext.runAsNonRoot: true`.

This not only enforced security but also reduced manual reviews during audits. For example, when a developer tried to deploy an Nginx pod without securityContext, Gatekeeper rejected it at admission. Later, we extended this approach to enforce registry restrictions and mandatory labels. This proactive policy enforcement significantly improved compliance and reduced misconfigurations."\*

---

Great 👍 let’s now build a **Mutating Admission Controller use case in EKS**.

---

## 🎯 Scenario

Your team wants **every pod in EKS** to automatically send logs to a centralized logging system (e.g., Fluentd/Datadog/Sidecar container).
Instead of asking developers to **manually add sidecars**, you implement a **Mutating Admission Controller** that **auto-injects a logging sidecar** into every pod.

---

## 🛠️ Solution with MutatingAdmissionWebhook

### Step 1: Create a Webhook Server

You run a small webhook service (Go/Python) that listens to admission requests and mutates the pod spec.
Example (simplified logic):

```python
# Python pseudo-code for webhook mutation
def mutate_pod(request):
    pod = request["object"]
    
    # Inject logging sidecar if not present
    sidecar = {
        "name": "fluentd-sidecar",
        "image": "fluent/fluentd:latest",
        "resources": {
            "limits": {"cpu": "100m", "memory": "200Mi"}
        }
    }
    
    pod["spec"]["containers"].append(sidecar)
    return pod
```

---

### Step 2: Register the Webhook in Kubernetes (EKS)

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: pod-logging-injector
webhooks:
  - name: log-injector.mydomain.com
    admissionReviewVersions: ["v1"]
    sideEffects: None
    rules:
      - operations: ["CREATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    clientConfig:
      service:
        name: logging-webhook-svc
        namespace: kube-system
        path: "/mutate"
      caBundle: <base64-cert>
```

Here:

* **Operations = CREATE** → Only mutates pods at creation.
* **clientConfig** → Points to the webhook service you deployed in EKS.

---

### Step 3: Test the Behavior

**Original Pod (developer manifest, no sidecar)**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-app
spec:
  containers:
    - name: app
      image: nginx
```

**What Actually Gets Created in EKS (after mutation)**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-app
spec:
  containers:
    - name: app
      image: nginx
    - name: fluentd-sidecar         # <-- auto-injected
      image: fluent/fluentd:latest
      resources:
        limits:
          cpu: 100m
          memory: 200Mi
```

---

## ✅ Interview Answer (Storytelling)

\*"In another project, we wanted centralized logging from all workloads in EKS, but developers often forgot to include sidecar containers for log forwarding. To solve this, I implemented a **Mutating Admission Controller** using a webhook.

Whenever a pod was created, the webhook intercepted the request and injected a **Fluentd logging sidecar** automatically. This removed the burden from developers, ensured consistent logging across all namespaces, and made audit compliance easier.

Later, we extended this approach for service mesh sidecars as well, very similar to how **Istio and App Mesh** inject Envoy proxies. This gave me deep hands-on experience with both **mutating and validating admission controllers** in Kubernetes."\*

---

👉 Now you have **two strong interview-ready stories**:

1. **Validating Admission Controller** → Enforce "no root user".
2. **Mutating Admission Controller** → Auto-inject logging/service-mesh sidecars.

---

Perfect 🔥 — let’s build you a **README.md** for this EKS Admission Controller project.
This will be **GitHub-ready** — you can just drop it in your repo and it looks professional for portfolio + interviews.

---

# 📌 Secure & Standardize Workloads in Amazon EKS with Admission Controllers

## 🚀 Overview

This project demonstrates how to use **Admission Controllers** in Amazon EKS to:

1. **Validate Pods** → Reject pods running as `root` (security compliance).
2. **Mutate Pods** → Automatically inject a logging sidecar (standardization).

By the end, you’ll have:

* A working EKS cluster.
* A **Validating Admission Controller** (OPA Gatekeeper).
* A **Mutating Admission Controller** (custom webhook).
* Tested enforcement and auto-injection on sample pods.

---

## 🛠️ Architecture

```plaintext
 Developer -> kubectl apply Pod
     |
     v
  +--------------------+
  |  EKS API Server    |
  +--------------------+
     |         |
     |         |
     v         v
 Validating   Mutating
 Webhook      Webhook
 (Gatekeeper) (Sidecar Injector)
     |
     v
   etcd -> Pod Created (only if rules are satisfied)
```

---

## ⚙️ Step 1: Setup EKS Cluster

### 1.1 Create an EKS Cluster (using eksctl)

```bash
eksctl create cluster \
  --name admission-demo \
  --region ap-south-1 \
  --nodegroup-name workers \
  --node-type t3.medium \
  --nodes 2
```

Verify cluster:

```bash
kubectl get nodes
```

---

## 🔒 Step 2: Validating Admission Controller (OPA Gatekeeper)

### 2.1 Install Gatekeeper

```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml
```

### 2.2 Create ConstraintTemplate

```yaml
# k8sdisallowrootuser-template.yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8sdisallowrootuser
spec:
  crd:
    spec:
      names:
        kind: K8sDisallowRootUser
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sdisallowrootuser

        violation[{"msg": msg}] {
          input.review.kind.kind == "Pod"
          container := input.review.object.spec.containers[_]
          not container.securityContext.runAsNonRoot
          msg := sprintf("Container %v is running as root. Set runAsNonRoot: true", [container.name])
        }
```

```bash
kubectl apply -f k8sdisallowrootuser-template.yaml
```

### 2.3 Apply Constraint

```yaml
# disallow-root-constraint.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDisallowRootUser
metadata:
  name: disallow-root
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

```bash
kubectl apply -f disallow-root-constraint.yaml
```

### 2.4 Test Policy

❌ Pod without `runAsNonRoot`:

```bash
kubectl run bad-pod --image=nginx
```

Expected → **Rejected**

✅ Pod with `runAsNonRoot`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
spec:
  containers:
    - name: nginx
      image: nginx
      securityContext:
        runAsNonRoot: true
```

```bash
kubectl apply -f good-pod.yaml
kubectl get pods
```

Expected → **Created**

---

## 🔄 Step 3: Mutating Admission Controller (Sidecar Injector)

### 3.1 Clone Webhook Example

```bash
git clone https://github.com/morvencao/kube-mutating-webhook-tutorial.git
cd kube-mutating-webhook-tutorial/deployment
```

### 3.2 Deploy Webhook

```bash
kubectl apply -f .
```

This deploys:

* A `mutating-webhook` service.
* A `MutatingWebhookConfiguration`.

### 3.3 Modify Sidecar Injection Logic

In the webhook code, update to inject a Fluentd container:

```yaml
containers:
  - name: fluentd-sidecar
    image: fluent/fluentd:latest
    resources:
      limits:
        cpu: "100m"
        memory: "200Mi"
```

Re-deploy webhook with updated image.

---

## 🧪 Step 4: Testing Mutation

Deploy a pod with label `mutate=true`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  labels:
    mutate: "true"
spec:
  containers:
    - name: app
      image: nginx
```

```bash
kubectl apply -f app-pod.yaml
kubectl get pod app-pod -o yaml
```

✅ You should see an **auto-injected Fluentd sidecar** in the pod spec.

---

## 🔀 Step 5: Combined Testing

| Scenario                                       | Expected Result                              |
| ---------------------------------------------- | -------------------------------------------- |
| Pod runs as root                               | ❌ Blocked by Validating Admission Controller |
| Pod runs as non-root, no sidecar               | ✅ Created, sidecar auto-injected             |
| Pod runs as non-root, sidecar already included | ✅ Created (no duplication)                   |

---

## 📸 Screenshots to Capture (for your repo/docs)

1. `kubectl get nodes` (EKS cluster running).
2. `kubectl run bad-pod` → Rejected message.
3. `kubectl describe pod app-pod` → Shows injected sidecar.
4. Gatekeeper logs (`kubectl logs -n gatekeeper-system <pod>`) showing validation events.

---

## 📚 Learnings from Project

* **Validating Admission Controller** → Used OPA Gatekeeper to enforce security rules.
* **Mutating Admission Controller** → Used a custom webhook to standardize deployments.
* **Practical Outcome** → Improved compliance, automation, and developer experience in EKS.

---

✅ This project can be extended to enforce:

* Mandatory labels (owner, cost-center).
* Registry restrictions (ECR-only).
* Auto-injection of monitoring/tracing agents.

---

Perfect 👌 — a clean **diagram** will make your README look polished and professional.
We can embed it using **Mermaid** (renders automatically on GitHub).

Here’s an updated section you can paste into your README:

---

## 🖼️ Admission Controller Workflow in EKS

```mermaid
flowchart TD
    A[Developer applies Pod manifest] --> B[EKS API Server]

    B --> C{Validating Admission Controller}
    C -- Violates Policy --> D[❌ Rejected (e.g., Runs as root)]
    C -- Passes Policy --> E[Proceed]

    E --> F{Mutating Admission Controller}
    F -- Adds Sidecar --> G[Mutated Pod Spec]
    F -- No Mutation --> H[Original Pod Spec]

    G --> I[etcd -> Pod Created ✅]
    H --> I[etcd -> Pod Created ✅]
```

---

### How it looks on GitHub:

* A developer applies a pod.
* First, the **Validating Admission Controller** (OPA Gatekeeper) checks compliance.

  * If non-compliant → request **rejected**.
  * If compliant → request moves forward.
* Next, the **Mutating Admission Controller** auto-injects a sidecar (e.g., Fluentd).
* Finally, the modified pod spec is stored in etcd → Pod created in cluster.

---
Awesome 🚀 — let’s make a **ready-to-push GitHub repo structure** for your EKS Admission Controller project.
This way you just need to `git init`, copy files, and push — and it looks like a professional portfolio project.

---

# 📂 Repo Structure

```
eks-admission-controllers/
├── README.md
├── diagrams/
│   └── admission-flow.mmd        # Mermaid diagram (workflow)
├── gatekeeper/                   # Validating Admission Controller
│   ├── k8sdisallowrootuser-template.yaml
│   ├── disallow-root-constraint.yaml
│   └── test-pods/
│       ├── good-pod.yaml
│       └── bad-pod.yaml
├── mutating-webhook/             # Mutating Admission Controller
│   ├── deployment.yaml           # Webhook service/deployment
│   ├── webhook-config.yaml       # MutatingWebhookConfiguration
│   ├── sidecar-patch.yaml        # Injected sidecar definition
│   └── test-pods/
│       └── app-pod.yaml
└── docs/
    ├── setup-eks.md              # Step 1: EKS setup
    ├── validating-controller.md  # Step 2: Gatekeeper usage
    ├── mutating-controller.md    # Step 3: Webhook usage
    └── combined-testing.md       # Step 4: Validation + Mutation
```

---

# 📝 File Contents

### `README.md`

(Use the README I gave you earlier — with the Mermaid diagram section included).

---

### `diagrams/admission-flow.mmd`

```mermaid
flowchart TD
    A[Developer applies Pod manifest] --> B[EKS API Server]

    B --> C{Validating Admission Controller}
    C -- Violates Policy --> D[❌ Rejected (e.g., Runs as root)]
    C -- Passes Policy --> E[Proceed]

    E --> F{Mutating Admission Controller}
    F -- Adds Sidecar --> G[Mutated Pod Spec]
    F -- No Mutation --> H[Original Pod Spec]

    G --> I[etcd -> Pod Created ✅]
    H --> I[etcd -> Pod Created ✅]
```

---

### `gatekeeper/k8sdisallowrootuser-template.yaml`

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8sdisallowrootuser
spec:
  crd:
    spec:
      names:
        kind: K8sDisallowRootUser
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sdisallowrootuser

        violation[{"msg": msg}] {
          input.review.kind.kind == "Pod"
          container := input.review.object.spec.containers[_]
          not container.securityContext.runAsNonRoot
          msg := sprintf("Container %v is running as root. Set runAsNonRoot: true", [container.name])
        }
```

---

### `gatekeeper/disallow-root-constraint.yaml`

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDisallowRootUser
metadata:
  name: disallow-root
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

---

### `gatekeeper/test-pods/good-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
spec:
  containers:
    - name: nginx
      image: nginx
      securityContext:
        runAsNonRoot: true
```

### `gatekeeper/test-pods/bad-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
spec:
  containers:
    - name: nginx
      image: nginx
```

---

### `mutating-webhook/deployment.yaml`

(Simplified — you’ll adapt from tutorial repo)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sidecar-webhook
  labels:
    app: sidecar-webhook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sidecar-webhook
  template:
    metadata:
      labels:
        app: sidecar-webhook
    spec:
      containers:
        - name: webhook
          image: <your-dockerhub-username>/sidecar-webhook:latest
          ports:
            - containerPort: 443
```

---

### `mutating-webhook/webhook-config.yaml`

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: sidecar-injector
webhooks:
  - name: injector.sandeep.dev
    admissionReviewVersions: ["v1"]
    sideEffects: None
    rules:
      - operations: ["CREATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    clientConfig:
      service:
        name: sidecar-webhook
        namespace: default
        path: "/mutate"
      caBundle: <base64-ca-cert>
```

---

### `mutating-webhook/sidecar-patch.yaml`

```yaml
containers:
  - name: fluentd-sidecar
    image: fluent/fluentd:latest
    resources:
      limits:
        cpu: "100m"
        memory: "200Mi"
```

---

### `mutating-webhook/test-pods/app-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  labels:
    mutate: "true"
spec:
  containers:
    - name: app
      image: nginx
```

---

### `docs/setup-eks.md`

* Step-by-step EKS setup using `eksctl`.
* Verify `kubectl get nodes`.

---

### `docs/validating-controller.md`

* How to install Gatekeeper.
* Apply policy.
* Test with good/bad pods.

---

### `docs/mutating-controller.md`

* Deploy webhook.
* Modify `sidecar-patch.yaml`.
* Apply pod and check injected sidecar.

---

### `docs/combined-testing.md`

* Try bad pod → rejected.
* Try good pod → accepted with sidecar injected.
* Show screenshots of results.

---

# 🚀 Next Steps for You

1. Create a repo:

   ```bash
   mkdir eks-admission-controllers && cd eks-admission-controllers
   git init
   ```
2. Copy above structure/files.
3. Add Mermaid diagram + screenshots to `README.md`.
4. Push to GitHub:

   ```bash
   git add .
   git commit -m "Initial commit - EKS Admission Controller project"
   git branch -M main
   git remote add origin git@github.com:<your-username>/eks-admission-controllers.git
   git push -u origin main
   ```

---

Perfect 🔥 — let’s make your **own mutating admission webhook** from scratch so you don’t need to rely on external repos.
We’ll build a **Go-based webhook server** (standard in K8s ecosystem), wrap it with a **Dockerfile**, and deploy it in EKS.

This way you’ll have **your own sidecar injector image** on Docker Hub/ECR → super impressive for interviews & portfolio 🚀.

---

# 🛠️ Mutating Admission Webhook – Go Implementation

## 📂 Project Structure

```
mutating-webhook/
├── main.go               # Webhook server code
├── Dockerfile            # Containerize the webhook
├── manifests/
│   ├── deployment.yaml   # Webhook Deployment + Service
│   ├── webhook.yaml      # MutatingWebhookConfiguration
│   └── rbac.yaml         # RBAC for webhook
```

---

## 📜 main.go (Go code)

This webhook intercepts pod creation and injects a **Fluentd sidecar** automatically.

```go
package main

import (
	"encoding/json"
	"fmt"
	"io/ioutil"
	"net/http"

	admissionv1 "k8s.io/api/admission/v1"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

func main() {
	http.HandleFunc("/mutate", handleMutate)
	fmt.Println("Starting webhook server on :8443")
	if err := http.ListenAndServeTLS(":8443", "/tls/tls.crt", "/tls/tls.key", nil); err != nil {
		panic(err)
	}
}

func handleMutate(w http.ResponseWriter, r *http.Request) {
	var admissionReview admissionv1.AdmissionReview
	body, _ := ioutil.ReadAll(r.Body)
	if err := json.Unmarshal(body, &admissionReview); err != nil {
		http.Error(w, "could not parse admission review", http.StatusBadRequest)
		return
	}

	reviewResponse := mutatePod(&admissionReview)

	response := admissionv1.AdmissionReview{
		TypeMeta: admissionReview.TypeMeta,
		Response: reviewResponse,
	}

	respBytes, _ := json.Marshal(response)
	w.Header().Set("Content-Type", "application/json")
	w.Write(respBytes)
}

func mutatePod(review *admissionv1.AdmissionReview) *admissionv1.AdmissionResponse {
	raw := review.Request.Object.Raw
	pod := corev1.Pod{}
	if err := json.Unmarshal(raw, &pod); err != nil {
		return &admissionv1.AdmissionResponse{
			Allowed: true,
		}
	}

	// Define sidecar
	sidecar := corev1.Container{
		Name:  "fluentd-sidecar",
		Image: "fluent/fluentd:latest",
		Resources: corev1.ResourceRequirements{
			Limits: corev1.ResourceList{
				"cpu":    resourceMustParse("100m"),
				"memory": resourceMustParse("200Mi"),
			},
		},
	}

	// Patch operation
	patch := []map[string]interface{}{
		{
			"op":    "add",
			"path":  "/spec/containers/-",
			"value": sidecar,
		},
	}

	patchBytes, _ := json.Marshal(patch)

	return &admissionv1.AdmissionResponse{
		UID:     review.Request.UID,
		Allowed: true,
		Patch:   patchBytes,
		PatchType: func() *admissionv1.PatchType {
			pt := admissionv1.PatchTypeJSONPatch
			return &pt
		}(),
	}
}

func resourceMustParse(val string) (res corev1.ResourceQuantity) {
	res = corev1.ResourceList{}.CPU()
	return
}
```

---

## 🐳 Dockerfile

```dockerfile
FROM golang:1.22 as builder
WORKDIR /app
COPY . .
RUN go mod init webhook && go mod tidy
RUN go build -o webhook main.go

FROM alpine:3.18
WORKDIR /root/
COPY --from=builder /app/webhook .
COPY tls/ /tls/
CMD ["./webhook"]
```

---

## 🔑 Certificates

Kubernetes requires TLS for admission webhooks.
Generate certs and create a secret:

```bash
openssl req -x509 -newkey rsa:4096 -keyout tls.key -out tls.crt -days 365 -nodes -subj "/CN=sidecar-webhook.default.svc"
kubectl create secret tls webhook-certs --cert=tls.crt --key=tls.key -n default
```

---

## 📜 manifests/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sidecar-webhook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sidecar-webhook
  template:
    metadata:
      labels:
        app: sidecar-webhook
    spec:
      containers:
        - name: webhook
          image: <your-dockerhub-username>/sidecar-webhook:latest
          ports:
            - containerPort: 8443
          volumeMounts:
            - name: tls-certs
              mountPath: /tls
              readOnly: true
      volumes:
        - name: tls-certs
          secret:
            secretName: webhook-certs
---
apiVersion: v1
kind: Service
metadata:
  name: sidecar-webhook
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    app: sidecar-webhook
```

---

## 📜 manifests/webhook.yaml

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: sidecar-injector
webhooks:
  - name: sidecar.sandeep.dev
    clientConfig:
      service:
        name: sidecar-webhook
        namespace: default
        path: "/mutate"
      caBundle: <base64-encoded-tls-crt>
    rules:
      - operations: ["CREATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
```

---

## 🧪 Test Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app
      image: nginx
```

Deploy:

```bash
kubectl apply -f app-pod.yaml
kubectl get pod app-pod -o yaml
```

✅ You should see an **auto-injected Fluentd sidecar** in the pod spec.

---

## 🚀 Workflow

1. Build & push image:

   ```bash
   docker build -t <dockerhub-username>/sidecar-webhook:latest .
   docker push <dockerhub-username>/sidecar-webhook:latest
   ```
2. Apply manifests:

   ```bash
   kubectl apply -f manifests/
   ```
3. Deploy test pod and check logs/spec.

---

⚡ With this, you’ll have:

* **Validating Admission Controller** (OPA Gatekeeper).
* **Mutating Admission Controller** (your own webhook in Go, containerized).

This is **portfolio gold** ✅ — shows deep Kubernetes knowledge + coding skills.

---






