# QOVES Take-Home Platform

A GitOps-managed Kubernetes platform running a trivial HTTP API backed by PostgreSQL, deployed via ArgoCD on minikube with Calico CNI, Sealed Secrets, default-deny NetworkPolicies, HPA, and Prometheus observability.

---

## Repository Layout

```
qoves-takehome/
├── app/                          # Provided application (source + container)
│   ├── main.py
│   ├── requirements.txt
│   └── Dockerfile
├── bootstrap/
│   └── root-app.yaml             # One-time manual apply — app-of-apps root
├── apps/                         # ArgoCD child Applications (one per component)
│   ├── sealed-secrets.yaml
│   ├── qoves-data.yaml
│   ├── qoves-network.yaml
│   ├── qoves-api.yaml
│   ├── prometheus-stack.yaml
│   └── qoves-monitoring.yaml
├── manifests/
│   ├── data/                     # PostgreSQL StatefulSet, Service, SealedSecret
│   ├── network/                  # Default-deny + allow-list NetworkPolicies
│   ├── api/                      # API Deployment, Service, Ingress, HPA
│   └── monitoring/               # ServiceMonitor, PrometheusRule
├── WRITEUP.md
└── .gitignore
```

---

## Prerequisites

| Tool | Install |
|------|---------|
| Docker Desktop | Running and healthy |
| minikube | `brew install minikube` |
| kubectl | `brew install kubectl` |
| Helm | `brew install helm` |
| ArgoCD CLI | `brew install argocd` |
| kubeseal | `brew install kubeseal` |

---

## Getting Started (from scratch)

### 1. Restructure the repo

The current repo uses `applications/` with `.yml` extensions. Align it with the target layout:

```bash
git mv applications apps
git mv apps/api.yml apps/qoves-api.yaml
git mv apps/database.yml apps/qoves-data.yaml
git mv apps/ingress.yml apps/qoves-network.yaml
git mv apps/secrets.yml apps/sealed-secrets.yaml
git mv apps/monitoring.yml apps/qoves-monitoring.yaml
git mv apps/prometheus.yml apps/prometheus-stack.yaml
git mv bootstrap/root-application.yaml bootstrap/root-app.yaml
mkdir -p manifests/{data,network,api,monitoring}
git add -A && git commit -m "restructure repo layout" && git push origin main
```

### 2. Cluster & CNI (Part A)

```bash
minikube delete
minikube start \
  --nodes=2 \
  --cni=calico \
  --driver=docker \
  --cpus=2 \
  --memory=4096 \
  --kubernetes-version=v1.30.0
```

**Why Calico:** minikube's default CNI does not enforce NetworkPolicy — policies silently do nothing. Calico enforces them out of the box and is lighter than Cilium for local development.

Enable required addons:

```bash
minikube addons enable ingress
minikube addons enable metrics-server
```

Verify:

```bash
kubectl get pods -n kube-system | grep calico
kubectl get pods -n ingress-nginx
```

### 3. GitOps Delivery (Part B)

Install ArgoCD by hand (the only imperative step):

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.13.3/manifests/install.yaml
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s
```

Retrieve the admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Apply the root app-of-apps — the **only** `kubectl apply` for workloads:

```bash
kubectl apply -f bootstrap/root-app.yaml
```

From this point forward, all changes flow through git. ArgoCD reconciles the cluster to match `apps/` and `manifests/`.

Optional — access the ArgoCD UI:

```bash
kubectl port-forward svc/argocd-server -n argocd 8443:443 &
# https://localhost:8443 — admin / <password>
```

### 4. Build & Load the API Image (Part C)

```bash
docker build -t qoves-api:0.1.0 app/
minikube image load qoves-api:0.1.0
```

### 5. Create Kubernetes Manifests (Parts C–H)

Create the following files under `manifests/`. Each file's content is described below — create them, then commit and push in one batch.

**`manifests/data/postgres-statefulset.yaml`** — Single-replica PostgreSQL StatefulSet with a 1Gi PVC, credentials sourced from `qoves-db-credentials` Secret, `pg_isready` probes, and resource requests/limits (100m/256Mi request, 500m/512Mi limit).

**`manifests/data/postgres-service.yaml`** — Headless Service (`clusterIP: None`) for the StatefulSet, exposing port 5432.

**`manifests/data/qoves-db-sealed.yaml`** — Generated via kubeseal (see Step 6).

**`manifests/api/deployment.yaml`** — 2-replica Deployment running `qoves-api:0.1.0` with `imagePullPolicy: Never` (loaded via minikube), `DATABASE_URL` from the sealed secret, HTTP probes on `/` port 8080, resource requests/limits (50m/64Mi request, 200m/128Mi limit).

**`manifests/api/service.yaml`** — ClusterIP Service mapping port 80 → 8080.

**`manifests/api/ingress.yaml`** — Ingress with `ingressClassName: nginx`, host `qoves.local`, routing `/` to the API service.

**`manifests/api/hpa.yaml`** — HorizontalPodAutoscaler targeting the API Deployment, 2–5 replicas, scaling at 70% average CPU utilization.

**`manifests/network/00-default-deny.yaml`** — Two policies: default-deny-ingress and default-deny-egress for the `qoves` namespace (empty `podSelector`, both `policyTypes`).

**`manifests/network/10-allow.yaml`** — Five allow policies:
- **DNS egress** — all pods → UDP/TCP 53 (easy to forget)
- **Ingress to API** — from `ingress-nginx` namespace → `app: qoves-api` on port 8080
- **API to DB** — `app: qoves-api` → `app: postgres` on port 5432
- **DB ingress from API** — `app: postgres` accepts from `app: qoves-api` on port 5432
- **Prometheus scrape** — from `monitoring` namespace → `app: qoves-api` on port 8080

**`manifests/monitoring/servicemonitor.yaml`** — ServiceMonitor selecting `app: qoves-api` in the `qoves` namespace, scraping `/metrics` on port `http` every 15s.

**`manifests/monitoring/prometheusrule.yaml`** — Alert `HealthzEndpointDown`: fires when >50% of `/healthz` requests return 503 for 2+ minutes. Actionable because it means the API has lost its only dependency (Postgres).

### 6. Seal the Database Secret (Part E)

Create a plaintext secret in `/tmp` (never in the repo), seal it, delete the plaintext:

```bash
cat > /tmp/qoves-db-secret-plain.yaml <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: qoves-db-credentials
  namespace: qoves
type: Opaque
stringData:
  POSTGRES_USER: qoves
  POSTGRES_PASSWORD: changeme-strong-password
  DATABASE_URL: "postgresql://qoves:changeme-strong-password@postgres.qoves.svc.cluster.local:5432/qoves"
EOF

kubeseal \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  --format=yaml \
  < /tmp/qoves-db-secret-plain.yaml \
  > manifests/data/qoves-db-sealed.yaml

rm /tmp/qoves-db-secret-plain.yaml
```

### 7. Write the Prometheus Stack ArgoCD Application

Update `apps/prometheus-stack.yaml` to install `kube-prometheus-stack` via Helm (chart version `61.2.0`) into the `monitoring` namespace. Key helm values: disable Grafana, enable Alertmanager, and set `serviceMonitorSelectorNilUsesHelmValues: false` so Prometheus picks up ServiceMonitors from all namespaces.

### 8. Commit, Push & Let ArgoCD Reconcile

```bash
git add manifests/ apps/
git commit -m "add all kubernetes manifests: data, api, network, monitoring"
git push origin main
```

ArgoCD auto-syncs. To force:

```bash
argocd app sync sealed-secrets
argocd app sync qoves-data
argocd app sync qoves-network
argocd app sync qoves-api
argocd app sync prometheus-stack
argocd app sync qoves-monitoring
```

### 9. Configure Local DNS

```bash
echo "$(minikube ip) qoves.local" | sudo tee -a /etc/hosts
```

---

## Validation

### Verify the stack is running

```bash
kubectl get pods,svc,ingress,netpol -A
kubectl get applications -n argocd
kubectl get hpa -n qoves
```

### Test endpoints through the ingress

```bash
curl http://qoves.local/
curl http://qoves.local/healthz
curl http://qoves.local/metrics
```

### Test NetworkPolicy enforcement

```bash
kubectl run nettest --rm -it --image=busybox:1.36 -n qoves --restart=Never -- \
  wget --timeout=3 -qO- http://google.com
# Expected: timeout (egress blocked)
```

### Test data persistence

```bash
kubectl exec -n qoves postgres-0 -- psql -U qoves -d qoves \
  -c "CREATE TABLE test (id serial PRIMARY KEY, val text);"
kubectl exec -n qoves postgres-0 -- psql -U qoves -d qoves \
  -c "INSERT INTO test (val) VALUES ('survives restart');"
kubectl delete pod postgres-0 -n qoves
kubectl wait --for=condition=ready pod/postgres-0 -n qoves --timeout=120s
kubectl exec -n qoves postgres-0 -- psql -U qoves -d qoves \
  -c "SELECT * FROM test;"
```

### Verify Prometheus scraping

```bash
kubectl port-forward svc/kube-prometheus-stack-prometheus -n monitoring 9090:9090 &
curl -s 'http://localhost:9090/api/v1/query?query=http_requests_total' | python3 -m json.tool
curl -s 'http://localhost:9090/api/v1/rules' | python3 -m json.tool | grep HealthzEndpointDown
```

### Verify no secrets in repo

```bash
grep -r "changeme" manifests/ || echo "No plaintext secrets found"
```

---

## Making a Change (GitOps Flow)

1. Edit a manifest in `manifests/` or an application in `apps/`
2. `git add . && git commit -m "description" && git push origin main`
3. ArgoCD detects the change and reconciles the cluster automatically
4. Verify with `argocd app list` or `kubectl get applications -n argocd`

No `kubectl apply` or `kubectl edit` on running workloads.

---

## Time Estimate

| Phase | Estimate |
|-------|----------|
| Repo restructure | 15 min |
| Cluster & CNI | 15–20 min |
| ArgoCD setup | 20–30 min |
| Manifests (all parts) | 45–60 min |
| NetworkPolicies | 30–40 min |
| Sealed Secrets | 30–40 min |
| Storage verification | 10 min |
| HPA | 15–20 min |
| Prometheus + alert | 30–40 min |
| Validation & proof | 15 min |
| WRITEUP.md | 60–90 min |
| **Total** | **5–7 hours** |
