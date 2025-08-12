
---

### **What is an External DNS Controller?**

An **External DNS controller** is a Kubernetes component that automates the management of DNS records for services exposed externally, typically via Ingress or Service resources. It watches Kubernetes resources (like Ingress objects) for specific annotations, then creates, updates, or deletes DNS records in a DNS provider (e.g., AWS Route53, Google Cloud DNS) to map domain names to the external addresses of load balancers or other endpoints.

The controller simplifies DNS management by ensuring that DNS records stay in sync with your Kubernetes resources, eliminating manual DNS configuration.

---

### **How It Watches Annotations**

The ExternalDNS controller monitors Kubernetes resources (primarily Ingress and Service objects) for changes, specifically looking for annotations that define how DNS records should be created. For example, in an Ingress resource, annotations like `external-dns.alpha.kubernetes.io/hostname` specify the desired DNS name (e.g., `api.mycompany.com`).

When the controller detects a change (e.g., a new Ingress, updated annotations, or a deleted resource), it interacts with the DNS provider’s API to:
- **Create** a DNS record mapping the specified hostname to the load balancer’s address.
- **Update** the record if the load balancer’s address changes.
- **Delete** the record if the Ingress or Service is removed.

---

### **Key Components in the Setup**

1. **Kubernetes Cluster**: Running workloads with Ingress or Service resources.
2. **Ingress Controller**: A component like AWS ALB Ingress Controller or NGINX Ingress Controller that provisions a load balancer to expose services.
3. **ExternalDNS Controller**: A Kubernetes deployment that watches resources and manages DNS records.
4. **DNS Provider**: A service like AWS Route53 where DNS records are created/updated.
5. **Annotations**: Metadata on Ingress/Service objects that ExternalDNS uses to determine DNS settings.

---

### **Step-by-Step Setup for ExternalDNS with AWS Route53**

Here’s a practical guide to setting up ExternalDNS in a Kubernetes cluster with AWS Route53, focusing on how it watches annotations and manages DNS records.

#### **Prerequisites**
- A Kubernetes cluster (e.g., EKS, k3s, or any cluster with AWS integration).
- An Ingress controller (e.g., AWS ALB Ingress Controller) installed and configured.
- An AWS account with Route53 hosted zone for your domain (e.g., `mycompany.com`).
- IAM permissions for ExternalDNS to manage Route53 records.
- `kubectl` configured to interact with your cluster.

#### **Step 1: Set Up IAM Permissions for ExternalDNS**
ExternalDNS needs permissions to interact with Route53. Create an IAM policy and attach it to a role associated with your Kubernetes nodes or a service account.

1. **Create an IAM Policy**:
   ```json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Effect": "Allow",
               "Action": [
                   "route53:ChangeResourceRecordSets",
                   "route53:ListResourceRecordSets",
                   "route53:ListHostedZones"
               ],
               "Resource": "*"
           }
       ]
   }
   ```
   Save this as `external-dns-policy.json` and create it:
   ```bash
   aws iam create-policy --policy-name ExternalDNSPolicy --policy-document file://external-dns-policy.json
   ```

2. **Attach the Policy to a Role**:
   - If using EKS, associate the policy with the node group’s IAM role or create a Kubernetes service account with an IAM role (IRSA).
   - For IRSA, create a service account and annotate it:
     ```bash
     eksctl create iamserviceaccount \
       --name external-dns \
       --namespace kube-system \
       --cluster <your-cluster-name> \
       --attach-policy-arn arn:aws:iam::<account-id>:policy/ExternalDNSPolicy \
       --approve
     ```

#### **Step 2: Install ExternalDNS**
Deploy the ExternalDNS controller in your cluster using a Helm chart or a manifest.

1. **Using Helm** (recommended for simplicity):
   Add the Bitnami Helm repository:
   ```bash
   helm repo add bitnami https://charts.bitnami.com/bitnami
   helm repo update
   ```

2. **Install ExternalDNS**:
   Create a `values.yaml` file to configure ExternalDNS for Route53:
   ```yaml
   provider: aws
   aws:
     region: us-east-1
     zoneType: public
   domainFilters:
     - mycompany.com
   serviceAccount:
     create: false
     name: external-dns
   txtOwnerId: my-cluster-id
   ```

   Install the chart:
   ```bash
   helm install external-dns bitnami/external-dns \
     --namespace kube-system \
     -f values.yaml
   ```

   - `provider: aws`: Specifies Route53 as the DNS provider.
   - `domainFilters`: Limits ExternalDNS to manage records in `mycompany.com`.
   - `serviceAccount`: Uses the IAM-enabled service account created earlier.
   - `txtOwnerId`: A unique identifier to avoid conflicts if multiple clusters manage the same DNS zone.

3. **Verify Installation**:
   ```bash
   kubectl get pods -n kube-system -l app.kubernetes.io/name=external-dns
   ```

#### **Step 3: Configure an Ingress Resource with Annotations**
Create an Ingress resource that ExternalDNS will monitor. Here’s an example for an AWS ALB Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: alb
    external-dns.alpha.kubernetes.io/hostname: api.mycompany.com
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
  - host: api.mycompany.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-service
            port:
              number: 80
```

Key annotations:
- `external-dns.alpha.kubernetes.io/hostname`: Specifies the DNS name (`api.mycompany.com`) that ExternalDNS will create in Route53.
- `kubernetes.io/ingress.class: alb`: Indicates the AWS ALB Ingress Controller.

Apply the Ingress:
```bash
kubectl apply -f ingress.yaml
```

#### **Step 4: How ExternalDNS Works with Annotations**
1. **Monitoring**:
   - ExternalDNS continuously watches Ingress and Service resources in the cluster using the Kubernetes API.
   - It looks for annotations like `external-dns.alpha.kubernetes.io/hostname`.

2. **Processing**:
   - When it detects the `api.mycompany.com` hostname in the Ingress, ExternalDNS queries the Ingress controller to get the load balancer’s DNS name (e.g., `alb-123456.us-east-1.elb.amazonaws.com`).
   - It then creates a DNS record in Route53, typically a CNAME or Alias record, mapping `api.mycompany.com` to the ALB’s DNS name.

3. **Record Creation**:
   - ExternalDNS creates an A record (Alias) or CNAME in the `mycompany.com` hosted zone in Route53.
   - Example Route53 record:
     ```
     Name: api.mycompany.com
     Type: A (Alias)
     Value: alb-123456.us-east-1.elb.amazonaws.com
     ```

4. **Updates and Deletion**:
   - If the Ingress is updated (e.g., hostname changes), ExternalDNS updates the DNS record.
   - If the Ingress is deleted, ExternalDNS removes the corresponding DNS record.

#### **Step 5: Verify DNS Records**
1. Check the Route53 hosted zone in the AWS Console or via CLI:
   ```bash
   aws route53 list-resource-record-sets --hosted-zone-id <your-hosted-zone-id>
   ```
   Look for the `api.mycompany.com` record pointing to the ALB DNS name.

2. Test DNS resolution:
   ```bash
   dig api.mycompany.com
   ```
   Ensure it resolves
<img width="677" height="698" alt="image" src="https://github.com/user-attachments/assets/471d1a3b-4591-424f-8a5c-f594992df86a" />
