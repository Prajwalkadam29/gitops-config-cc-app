# ☸️ GitOps Configuration: Currency Converter App

This repository serves as the declarative Single Source of Truth for the Kubernetes deployment state of the Currency Converter microservice.

By adopting a strict GitOps methodology, we have entirely decoupled Continuous Integration (Jenkins) from Continuous Deployment (ArgoCD). Jenkins is completely stripped of Kubernetes API credentials; its sole responsibility is to build, sign, and push artifacts, and then update the configuration files in this repository. ArgoCD acts as the reconciliation engine, continuously monitoring this repository and pulling the defined state into the target Amazon EKS cluster.

---

## The GitOps Paradigm in this Pipeline

In traditional "Push-based" CI/CD, the CI server (Jenkins) runs `kubectl apply` or `helm upgrade` directly against the cluster. This introduces massive security risks (the CI server needs cluster admin credentials) and configuration drift (manual cluster edits aren't reflected in Git).

**In our "Pull-based" GitOps pipeline:**
1. **Security:** Jenkins never talks to Kubernetes. It only needs an IAM role to push to ECR and a standard GitHub PAT to commit to this repository.
2. **Auditability:** Every deployment, configuration change, and rollback is a Git commit. `git log` is the deployment history.
3. **Drift Prevention:** If someone manually edits a Deployment in the cluster via `kubectl`, ArgoCD immediately detects the drift and forcefully overwrites it back to the state defined in this repository (`selfHeal: true`).
4. **Rollbacks:** Rollbacks are no longer complex pipeline executions. They are simply a `git revert` command.

---

## 📂 Repository Structure & File Manifest

ArgoCD relies on a strict directory structure to map Git paths to Kubernetes namespaces.

```
.
├── apps/
│   └── currency-converter-app/
│       └── helm/
│           ├── Chart.yaml              # Helm chart metadata
│           ├── values.yaml             # Base defaults shared by ALL environments
│           ├── values-dev.yaml         # Dev overrides (Targeted by Jenkins CI)
│           ├── values-staging.yaml     # Staging overrides (Targeted by Jenkins CI)
│           ├── values-prod.yaml        # Prod overrides (Requires human PR approval)
│           └── templates/              # Kubernetes Manifest Templates
│               ├── deployment.yaml     # Stateless App Deployment
│               ├── service.yaml        # Internal ClusterIP routing
│               ├── hpa.yaml            # Horizontal Pod Autoscaling (70% CPU target)
│               ├── networkpolicy.yaml  # Ingress isolation restrictions
│               └── externalsecret.yaml # Dynamic AWS Secrets Manager injection
├── argocd-appset/
│   └── appset.yaml                     # ArgoCD Matrix Generator definition
└── CODEOWNERS                          # Defines mandatory PR reviewers for prod promotion

```

### Manifest Deep-Dive

* `appset.yaml`: The brain of the ArgoCD deployment. It uses a Matrix Generator combining the Git path (`apps/*`) with a list of environments (`dev`, `staging`, `prod`) to dynamically spawn isolated namespaces (e.g., `currency-converter-app-dev`) and deploy the Helm charts without duplicating YAML files.

* `Chart.yaml`: Defines the Helm chart metadata, including the chart name, description, and versioning.

* `values.yaml`: Contains the safe, base defaults shared across all environments, such as the base ECR repository URI, default resource limits, and health probe paths.

* `values-<env>.yaml`: Contains environment-specific overrides. This includes targeted replica counts, the exact AWS Secret path for that environment, and crucially, the dynamically updated `image.tag` written by the Jenkins CI pipeline.

* `deployment.yaml`: Configured for strict Pod Security Standards. It enforces a `readOnlyRootFilesystem` and drops all Linux capabilities. Because Spring Boot requires a writable `/tmp` to start Tomcat, a temporary `emptyDir` volume is mounted to satisfy the application without compromising the root filesystem security.

* `service.yaml`: Defines the Kubernetes Service, providing internal `ClusterIP` routing to expose the application pods to other resources within the cluster.

* `hpa.yaml`: Implements a HorizontalPodAutoscaler, automatically scaling the number of application replicas up or down based on a target CPU utilization of 70%.

* `externalsecret.yaml`: Implements the External Secrets Operator (ESO) CRD. It bridges the gap between AWS Secrets Manager and Kubernetes native Secrets securely at runtime.

* `networkpolicy.yaml`: Enforces a Zero-Trust network boundary inside the cluster, ensuring the application only accepts traffic from the NGINX Ingress Controller and Prometheus metrics scrapers.

* `CODEOWNERS`: Enforces branch protection rules by requiring designated platform/SRE team members to review and approve any Jenkins-generated Pull Requests before production deployment.

---

## The CI/CD Handoff Flow

Our pipeline utilizes a Jenkins Multibranch configuration to handle branches differently based on their purpose:

### 1. Feature Branch Flow (Continuous Integration)
When a developer pushes code to a `feature/*` branch, Jenkins provides rapid feedback without altering the cluster or GitOps repository:

* **Jenkins CI:** Compiles code -> Scans Secrets -> Runs SAST (SonarQube) -> Builds Docker Image -> Scans Image (Trivy).
* **Result:** The pipeline stops here. No artifacts are pushed to ECR, and the GitOps configuration remains untouched.


### 2. Main Branch Flow (Continuous Delivery)
When a developer merges application code to the `main` branch, the full release process is triggered:

**1. Jenkins CI:** Compiles code -> Scans Secrets -> Runs SAST (SonarQube) -> Builds Docker Image -> Scans Image (Trivy) -> Pushes to ECR -> **Cryptographically Signs the Image (Cosign)**.

**2. The Handoff:** Jenkins runs a Groovy script to clone this repository, uses `yq` to update the `image.tag` inside `apps/currency-converter-app/helm/values-<env>.yaml`, and pushes the commit.

* **Dev/Staging:** Jenkins commits directly to the main branch.

* **Production:** Jenkins creates a Pull Request. Branch protection rules and the `CODEOWNERS` file enforce that an SRE/Lead must review and merge the PR.

**3. ArgoCD CD:** Detects the commit to `main`, renders the Helm template, and synchronizes the cluster state.

**4. Admission Control:** Before Kubernetes spins up the Pod, the **Kyverno** admission controller intercepts the request, reaches out to ECR, and verifies the Cosign cryptographic signature. If the signature is missing or invalid, the deployment is blocked.

---

## Design Choices & Security Implementations

### 1. Matrix ApplicationSet (DRY Configuration)

Instead of maintaining three separate ArgoCD Application YAMLs, we utilize one ApplicationSet with a Matrix Generator. This completely adheres to the DRY (Don't Repeat Yourself) principle. Adding a new microservice is as simple as creating a new folder under apps/—ArgoCD will automatically discover it and deploy it to all three environments.

### 2. Zero-Trust Secrets Management (ESO)

No plain-text secrets or base64-encoded credentials exist in this repository.
The values-<env>.yaml files only contain the reference name of the secret in AWS (e.g., awsSecretName: "dev/currency-converter/api-keys"). The externalsecret.yaml manifest instructs the External Secrets Operator to fetch the payload securely via an IAM Role for Service Accounts (IRSA) and inject it into the pods via `envFrom`.


### 3. Hardened Pod Security Standards

To comply with Kyverno security policies, the application runs under strict constraints:

* `allowPrivilegeEscalation: false`
* `seccompProfile: { type: RuntimeDefault }`
* `readOnlyRootFilesystem: true`
* `capabilities: { drop: ["ALL"] }`

---

## ⚖️ Architectural Trade-Offs

While this setup is highly robust, the following trade-offs were deliberately accepted for this specific iteration:

**1. Namespaced Environments vs. Multi-Cluster Isolation:**
Currently, dev, staging, and prod are isolated by Kubernetes Namespaces and NetworkPolicies within a single EKS cluster. While highly cost-effective and perfectly valid for lower environments, a true mission-critical production environment would utilize physically separate EKS clusters to eliminate the "noisy neighbor" problem and blast radius of cluster-wide control plane failures.

**2. RunAsNonRoot Validation limitation:**
While the containers are configured to run securely, the strict runAsNonRoot: true boolean was intentionally omitted from the securityContext. This is because the underlying Docker image utilizes a text-based user (USER spring), which the native Kubelet admission webhook cannot statically validate against numeric ID requirements prior to container initialization. Security is instead enforced via the RuntimeDefault seccomp profile and readonly filesystem.

**3. Mutable ECR Image Tags:**
The ECR repository currently allows mutable image tags to support rapid CI overwrites during active development cycles. In a strict production environment, ECR should be configured for immutable tags to guarantee that the image tag verified in staging is cryptographically identical to the one promoted to production.

---

## Quick Troubleshooting

1. **ArgoCD reports "app path does not exist"**
   Ensure your ApplicationSet template specifically references the helm directory: `path: '{{path}}/helm'`.

2. **Pods are failing to start with Secret missing errors**
   Verify that the IAM Role associated with the `external-secrets-sa` ServiceAccount has `secretsmanager:GetSecretValue` permissions for the exact AWS Secret ARN defined in your `values-<env>.yaml`. 

    Check the operator status:
    ```bash
    kubectl describe externalsecret currency-converter-api-secret -n currency-converter-app-dev
    ```   

3. **Pods stuck in CreateContainerConfigError**
   This usually indicates a volume mount failure or a security context clash. Verify that the `emptyDir` mount for `/tmp` is correctly mapped in both `volumeMounts` and `volumes` in `deployment.yaml`.

---
