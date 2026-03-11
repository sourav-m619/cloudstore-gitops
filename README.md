# cloudstore-gitops
This repository is the **absolute, declarative Single Source of Truth (SSOT)** for the CloudStore GKE cluster. It contains all Kubernetes manifests, Helm values, Service Mesh configurations, and Observability definitions required to run the platform.

⚠️ **RESTRICTED OPERATION: Manual mutations via `kubectl apply`, `edit`, or `delete` are strictly forbidden in the production environment and will be aggressively overwritten by the GitOps controller.**



---

## ⚙️ The GitOps Reconciliation Loop

CloudStore utilizes a pull-based GitOps model powered by **ArgoCD**.

1.  **Automated CI Commits:** The CI pipelines in `cloudstore-app` push immutable, SHA-tagged images to Artifact Registry. A bot holding a scoped `GITOPS_TOKEN` immediately patches the `apps/backend/deployment.yaml` or `apps/frontend/deployment.yaml` in this repository and commits the change.
2.  **Drift Detection:** ArgoCD polls this repository (via webhook or 3-minute intervals). It calculates the AST difference between the Git state and the live GKE cluster state.
3.  **Auto-Sync & Prune:** ArgoCD applies the changes, executing a zero-downtime rolling update. Any orphaned resources in the cluster not defined in Git are automatically pruned.

---

## 📂 Deep Directory Structure

We use a layered approach, separating the business logic (`apps/`) from the foundational cluster capabilities (`platform/`).

```text
cloudstore-gitops/
├── apps/
│   ├── backend/
│   │   ├── deployment.yaml      # Node.js API (Mutated by CI)
│   │   ├── service.yaml         # ClusterIP
│   │   ├── hpa.yaml             # HorizontalPodAutoscaler (CPU/Memory targets)
│   │   └── external-secret.yaml # ESO CRD mapped to GCP Secret Manager
│   └── frontend/
│       ├── deployment.yaml      # React/Nginx UI (Mutated by CI)
│       └── service.yaml         # ClusterIP
├── platform/
│   ├── istio/
│   │   ├── gateway.yaml         # Istio IngressGateway (Port 80/443 mapping)
│   │   ├── virtual-service.yaml # L7 Routing (/api -> backend, / -> frontend)
│   │   └── peer-authentication.yaml # STRICT mTLS enforcement
│   └── observability/
│       ├── otel-collector.yaml  # OpenTelemetry Collector (DaemonSet, :4317 gRPC)
│       ├── prometheus.yaml      # Metrics retention and scraping configs
│       ├── grafana.yaml         # Dashboard provisioning
│       ├── jaeger.yaml          # Trace retention and UI
│       └── kiali.yaml           # Mesh topology RBAC and deployment
└── argocd/
    ├── app-of-apps.yaml         # Root ArgoCD App orchestrating all child apps
    ├── cloudstore-apps.yaml     # ArgoCD definition tracking the apps/ directory
    └── cloudstore-platform.yaml # ArgoCD definition tracking the platform/ directory
🔐 Zero-Touch Secrets Management (ESO)
To maintain a zero-trust posture, no secrets (not even base64 encoded Secret manifests) exist in this repository. We utilize the External Secrets Operator (ESO) configured with GCP Workload Identity.

How it works:
A ClusterSecretStore is configured to authenticate to GCP Secret Manager using the backend-sa Kubernetes Service Account (which maps to the GCP IAM service account via WIF).

The ExternalSecret manifest instructs ESO to fetch db-password.

ESO dynamically generates a native Kubernetes Secret inside the cluster.

Example external-secret.yaml:

YAML
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: backend-db-credentials
  namespace: cloudstore
spec:
  refreshInterval: "1h"
  secretStoreRef:
    name: gcp-store
    kind: ClusterSecretStore
  target:
    name: db-credentials-secret # The actual K8s Secret created
    creationPolicy: Owner
  data:
  - secretKey: DB_PASSWORD
    remoteRef:
      key: db-password
🕸️ Network Topology & Service Mesh (Istio)
CloudStore leverages Istio for advanced traffic management and zero-trust internal networking.

Ingress: External traffic hits the GCP Load Balancer configured via the Istio IngressGateway component.

L7 Routing: VirtualService definitions route traffic cleanly. The frontend is ignorant of the backend's internal cluster DNS; it calls /api, which Istio routes to the backend service.

mTLS (Mutual TLS): PeerAuthentication is set to STRICT cluster-wide. All pod-to-pod communication is encrypted, and plaintext connections are rejected at the sidecar level.

📊 Telemetry & Observability (OTEL)
Our observability stack entirely removes the need for sidecar monitoring agents in favor of a centralized pipeline.

OTEL Collector: Deployed as a DaemonSet across the GKE nodes. It exposes gRPC port 4317.

Application Telemetry: The Node.js and React applications are instrumented via the OTEL SDK and transmit spans/metrics directly to the local node's Collector IP.

Data Fan-Out: The Collector processes and exports data:

Metrics -> Prometheus (Scraped on port :8889)

Traces -> Jaeger

Logs -> Google Cloud Logging (via GCP exporter)

Visualization: Grafana consumes Prometheus data, while Kiali hooks into Prometheus and Istio to generate live, interactive mesh topology maps.

🚨 Operations & Disaster Recovery Runbook
Local Validation
Before opening a Pull Request, validate your manifests locally to prevent ArgoCD sync failures:

Bash
# Validate generic Kubernetes YAML syntax
kubeval apps/backend/*.yaml

# Lint Istio configurations
istioctl analyze platform/istio/
Emergency Rollbacks (The GitOps Way)
If a bad deployment slips through CI and causes a production outage, do not use kubectl rollout undo.

Find the offending commit in the Git history.

Revert the commit locally or via the GitHub UI:

Bash
git revert <bad-commit-sha>
git push origin main
ArgoCD will detect the reverted state within 3 minutes (or instantly if webhook triggered) and sync the cluster back to the previous stable state.

Bootstrapping a New Cluster
In a total cluster loss scenario, disaster recovery takes less than 10 minutes:

Run the Terraform pipeline in cloudstore-app to spin up a fresh GKE cluster.

Install ArgoCD on the new cluster.

Apply the Root Application:

Bash
kubectl apply -f argocd/app-of-apps.yaml
ArgoCD will recursively apply the platform and apps definitions, completely restoring the cluster state from this repository.
