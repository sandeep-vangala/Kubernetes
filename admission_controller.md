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

If you have a specific use case (e.g., integrating with ExternalDNS from your previous question or enforcing a particular policy), let me know, and I can provide a tailored example!
