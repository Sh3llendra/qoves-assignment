# QOVES Take-Home Platform

A GitOps-managed Kubernetes platform running a trivial HTTP API backed by PostgreSQL, deployed via ArgoCD on minikube with Calico CNI, Sealed Secrets, default-deny NetworkPolicies, HPA, and Prometheus observability.

---

## Repository Layout

```
qoves-assignment/
├── app/                          # Provided application (source + container)
│   ├── main.py
│   ├── requirements.txt
│   └── Dockerfile
├── bootstrap/
│   └── root-application.yaml     # One-time manual apply — app-of-apps root
├── applications/                 # ArgoCD child Applications (one per component)
│   ├── secrets.yml               # Sealed Secrets controller (wave 0)
│   ├── database.yml              # PostgreSQL StatefulSet + PVC (wave 1)
│   ├── ingress.yml               # NetworkPolicies (wave 1)
│   ├── api.yml                   # API Deployment + Service + Ingress + HPA (wave 2)
│   ├── prometheus.yml            # kube-prometheus-stack Helm chart (wave 2)
│   └── monitoring.yml            # ServiceMonitor + PrometheusRule (wave 3)
├── manifests/
│   ├── data/                     # PostgreSQL StatefulSet, Service, SealedSecret
│   ├── network/                  # Default-deny + allow-list NetworkPolicies
│   ├── api/                      # API Deployment, Service, Ingress, HPA
│   └── monitoring/               # ServiceMonitor, PrometheusRule
├── WRITEUP.md                    # Required writeup (5 sections)
├── README.md
└── .gitignore
```

---

## Architecture

- **GitOps controller:** ArgoCD with app-of-apps pattern — `bootstrap/root-application.yaml` points at `applications/`, each child Application targets a `manifests/` subdirectory or a Helm chart
- **Sync-wave ordering:** sealed-secrets (0) → database + network (1) → api + prometheus (2) → monitoring (3)
- **Secrets:** Bitnami Sealed Secrets — encrypted in git, decrypted in-cluster
- **Networking:** Default-deny ingress + egress in the `qoves` namespace, explicit allow rules for DNS, ingress→API, API→Postgres, Prometheus scraping
- **Observability:** kube-prometheus-stack (Prometheus + Alertmanager + Grafana), ServiceMonitor for `/metrics`, PrometheusRule with `QovesAPIHealthzFailing` alert
- **Scaling:** HPA on the API Deployment (2–5 replicas, 70% CPU target)

---

## Prerequisites

| Tool | Install |
|------|---------|
| Docker Desktop | Running and healthy |
| minikube | `brew install minikube` |
| kubectl | `brew install kubectl` |
| Helm | `brew install helm` |
| kubeseal | `brew install kubeseal` |

---

## Quick Start

### 1. Cluster & CNI

```bash
minikube start --nodes=2 --cni=calico --driver=docker --cpus=2 --memory=3g --kubernetes-version=stable
minikube addons enable ingress
minikube addons enable metrics-server
```

### 2. Build the API Image

```bash
eval $(minikube docker-env)
cd app && docker build -t qoves-api:0.1.0 . && cd ..
```

### 3. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.11.7/manifests/install.yaml
kubectl wait --for=condition=available deploy/argocd-server -n argocd --timeout=300s
```

### 4. Deploy via GitOps (Two-Phase)

**Phase 1 — Apply the root application:**

```bash
kubectl apply -f bootstrap/root-application.yaml
```

ArgoCD discovers `applications/`, deploys sealed-secrets controller first (wave 0), then the rest.

**Phase 2 — Generate and push the sealed secret:**

Wait for the sealed-secrets controller, then seal the database credentials:

```bash
kubectl wait --for=condition=available deploy/sealed-secrets -n kube-system --timeout=300s

kubectl create secret generic qoves-db-credentials \
  --namespace=qoves \
  --from-literal=POSTGRES_USER=qoves \
  --from-literal=POSTGRES_PASSWORD=changeme \
  --from-literal=POSTGRES_DB=qovesdb \
  --from-literal=DATABASE_URL="postgresql://qoves:changeme@postgres.qoves.svc:5432/qovesdb" \
  --dry-run=client -o yaml > /tmp/db-secret-plain.yaml

kubeseal --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  --format=yaml < /tmp/db-secret-plain.yaml > manifests/data/qoves-db-sealed.yaml

rm /tmp/db-secret-plain.yaml

git add manifests/data/qoves-db-sealed.yaml
git commit -m "add real sealed secret"
git push origin main
```

### 5. Validate

```bash
kubectl get applications -n argocd
kubectl get pods,svc,ingress,netpol -n qoves
kubectl get hpa -n qoves

# Health check through the ingress
kubectl port-forward svc/ingress-nginx-controller -n ingress-nginx 8080:80 &
curl -H "Host: qoves.local" http://127.0.0.1:8080/healthz

# NetworkPolicy blocks unauthorized egress
kubectl run test --rm -it --image=busybox:1.36 -n qoves --restart=Never -- wget -qO- --timeout=3 http://example.com

# Grafana dashboard
kubectl port-forward svc/kps-grafana -n monitoring 3000:80 &
# Open http://localhost:3000 — admin / prom-operator
```

---

## Making a Change (GitOps Flow)

1. Edit a manifest in `manifests/` or an application in `applications/`
2. `git add . && git commit -m "description" && git push origin main`
3. ArgoCD detects the change and reconciles the cluster automatically
4. Verify with `kubectl get applications -n argocd`

No `kubectl apply` or `kubectl edit` on running workloads.

---

## Writeup

See [WRITEUP.md](WRITEUP.md) for the full writeup covering:

1. **Run it** — setup steps, repo layout, GitOps flow
2. **Decisions** — CNI, secrets, database, scaling signal (ADR-style)
3. **What minikube did for me** — what bare metal would require
4. **Production gaps** — HA, backups, TLS, real secret backend, multi-cluster
5. **One runbook** — database pod failure recovery
