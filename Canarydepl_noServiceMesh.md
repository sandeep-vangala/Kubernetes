To switch traffic in a canary deployment on Kubernetes **without a service mesh**, **without EKS** (assuming a vanilla or self-managed k8s cluster), and **using Helm**, you can use one or more of the following approaches:

***

## 1. Weighted Traffic Splitting Using Ingress Controllers

### a. **NGINX Ingress Controller with Canary Support**
- Modern Ingress controllers like **NGINX** support "canary" features via annotations, letting you send a fixed percentage of HTTP traffic to a canary service.
- **How it works:**
  - Deploy two versions of your app (`stable` and `canary`) using Helm charts.
  - Set up two Kubernetes Services: one for the stable deployment and one for the canary.
  - Create two Ingress rules for the same host/path:
    - The main rule targets the stable Service.
    - The canary rule uses annotations like:
      - `nginx.ingress.kubernetes.io/canary: "true"`
      - `nginx.ingress.kubernetes.io/canary-weight: "10"` (routes 10% of traffic to canary)
- Gradually increase the `canary-weight` annotation to shift more traffic to the new version.

### b. **Traffic Splitting Logic**
- This method allows you to shift traffic gradually (e.g., 10% → 25% → 50% → 100%) by updating the annotation, all via Helm values and upgrades.

***

## 2. Manual Pod Replica Adjustments (Without Advanced Ingress)

If you aren't using an ingress controller with canary support, you can **approximate canary deployments by adjusting replica counts**, but this method is very coarse:
- Deploy both versions with Helm, using the same Service selector (so requests hit any pod).
- Control traffic split by varying the number of replicas:
  - For a rough 80/20 split, run 8 replicas of stable and 2 of canary.
- This approach is **imprecise** (because pod load balancing by the Service is not truly weighted by version, especially at small scales), but can work for very basic canaries.[1]

***

## 3. Helm and Canary-Ready Controllers

### a. **Helm + Kruise Rollout Addon**
- OpenKruise's kruise-rollout and other addons allow you to add canary rollout capability to regular Kubernetes workloads (Deployment/StatefulSet), triggered and managed by Helm updates.
- The traffic switch is usually handled by Ingress rules or Gateway API, as above, not Helm itself.[2]

***

## 4. Automation & Rollback

- Automate updates by scripting Helm value changes (for canary-weight or replica count) in your CI pipeline.
- Monitor key metrics and, if issues are observed, roll back by restoring canary-weight to 0% or scaling back canary pods.

***

## **Summary Table**

| Method                                   | Traffic Switch Control      | Helm Usage? | Extra Controller Needed?       | Precision     |
|-------------------------------------------|----------------------------|-------------|-------------------------------|---------------|
| NGINX Ingress with canary annotations     | Weighted annotations       | Yes         | NGINX Ingress                  | High          |
| Manual replica adjustment                 | Vary pod counts            | Yes         | No (k8s core+helm)             | Low           |
| Kruise Rollout (with ingress/gateway)     | Managed by Kruise addon    | Yes         | Kruise Rollout, Ingress/Gateway| High          |

***

**Key Points:**
- **No service mesh needed**: Use Ingress controller canary features for true weighted traffic.
- **Helm friendly**: All changes (Ingress rules, annotations, replica counts) can be managed as Helm values in your chart.
- **Monitoring is critical**: Always monitor canary version during traffic shifts to determine when to proceed, halt, or rollback.[3][1][2]

***

**References:**  
- Leverage the documented approaches using ingress annotations or Helm with addons for traffic switching in canary deployments without the complexity of a service mesh or managed EKS features.[3][1][2]

[1] https://devtron.ai/blog/canary-deployment-strategy/
[2] https://www.alibabacloud.com/blog/the-canary-rollout-of-the-helm-chart-application-is-coming_599232
[3] https://www.netdata.cloud/academy/canary-deployment/
[4] https://devtron.ai/blog/canary-deployment/
[5] https://docs.flagger.app/usage/deployment-strategies
[6] https://developer.hashicorp.com/consul/tutorials/control-network-traffic/service-splitters-canary-deployment
[7] https://codefresh.io/learn/software-deployment/what-are-canary-deployments/
[8] https://developer.harness.io/docs/continuous-delivery/deploy-srv-diff-platforms/kubernetes/cd-k8s-ref/traffic-shifting-step/
[9] https://docs.gitlab.com/user/project/canary_deployments/
[10] https://argo-rollouts.readthedocs.io/en/stable/features/canary/
[11] https://istio.io/latest/docs/concepts/traffic-management/
[12] https://aws.amazon.com/blogs/containers/create-a-pipeline-with-canary-deployments-for-amazon-eks-with-aws-app-mesh/
[13] https://developer.harness.io/docs/continuous-delivery/deploy-srv-diff-platforms/helm/step-reference/helm-canary-deploy/
[14] https://aws.amazon.com/blogs/containers/create-a-pipeline-with-canary-deployments-for-amazon-ecs-using-aws-app-mesh/
[15] https://codefresh.io/learn/software-deployment/canary-deployment-in-kubernetes-the-basics-and-a-quick-tutorial/
[16] https://octopus.com/blog/kubernetes-canary-deployments
[17] https://zeet.co/blog/kubernetes-canary-deployment
[18] https://developer.harness.io/docs/continuous-delivery/deploy-srv-diff-platforms/kubernetes/kubernetes-executions/create-a-kubernetes-canary-deployment/
[19] https://spacelift.io/blog/kubernetes-deployment-strategies
