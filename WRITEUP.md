# QOVES Platform — Writeup

## 1. Run It

Complete from-scratch setup. Every command is here — nothing is assumed to be pre-existing.

### Prerequisites

| Tool | Install | Purpose |
|------|---------|---------|
| Docker Desktop | Running and healthy | Container runtime for minikube |
| minikube | `brew install minikube` | Local Kubernetes cluster |
| kubectl | `brew install kubectl` | Cluster CLI |
| Helm | `brew install helm` | Required by ArgoCD for Helm-based apps |
| kubeseal | `brew install kubeseal` | Encrypts secrets client-side |

### Step 1 — Start the Cluster with Calico CNI

```bash
minikube start \
  --nodes=2 \
  --cni=calico \
  --driver=docker \
  --cpus=2 \
  --memory=3g \
  --kubernetes-version=stable
```

Enable required addons:

```bash
minikube addons enable ingress
minikube addons enable metrics-server
```

Verify Calico is running on all nodes:

```bash
kubectl get pods -n kube-system -l k8s-app=calico-node
# Every node should have a calico-node pod in Running state
```

> **Note:** If the ingress addon creates a stale validating webhook that blocks Ingress creation later, delete it:
> `kubectl delete validatingwebhookconfiguration ingress-nginx-admission`

### Step 2 — Build and Load the API Image

```bash
eval $(minikube docker-env)
cd app && docker build -t qoves-api:0.1.0 . && cd ..
```

The Deployment uses `imagePullPolicy: Never` so Kubernetes uses the locally built image.

### Step 3 — Install ArgoCD

This is the only imperative install. Everything after this is GitOps-managed.

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.11.7/manifests/install.yaml
kubectl wait --for=condition=available deploy/argocd-server -n argocd --timeout=300s
```

Retrieve the admin password (for UI access):

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Optional — access the ArgoCD UI:

```bash
kubectl port-forward svc/argocd-server -n argocd 8443:443 &
# Open https://localhost:8443 — admin / <password from above>
```

### Step 4 — Deploy via GitOps (Two-Phase Approach)

The sealed-secrets controller must be running in-cluster before `kubeseal` can encrypt locally. This creates a chicken-and-egg: the controller is deployed by ArgoCD, but the sealed secret must exist for Postgres to start. Solution: deploy in two phases.

**Phase 1 — Apply the root application:**

The repo already has a placeholder `qoves-db-sealed.yaml`. Push all manifests and apply:

```bash
git add -A && git commit -m "populate manifests" && git push origin main
kubectl apply -f bootstrap/root-application.yaml
```

ArgoCD discovers `applications/`, creates child Applications in sync-wave order:
1. **Wave 0:** `sealed-secrets` — installs the controller in `kube-system`
2. **Wave 1:** `qoves-data` + `qoves-network` — Postgres StatefulSet, NetworkPolicies
3. **Wave 2:** `qoves-api` + `prometheus-stack` — API Deployment, kube-prometheus-stack
4. **Wave 3:** `qoves-monitoring` — ServiceMonitor + PrometheusRule

The Postgres and API pods will be in `CreateContainerConfigError` — this is expected because the secret doesn't exist yet.

Wait for the sealed-secrets controller:

```bash
kubectl wait --for=condition=available deploy/sealed-secrets -n kube-system --timeout=300s
```

**Phase 2 — Generate the real sealed secret and push:**

```bash
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
git commit -m "add real sealed secret for database credentials"
git push origin main
```

ArgoCD syncs the sealed secret → the controller decrypts it into a native Secret → Postgres and the API pods start.

### Step 5 — Validate Everything

Wait for all pods to be Running:

```bash
kubectl get pods -n qoves -w
# postgres-0, qoves-api-xxx, qoves-api-yyy — all 1/1 Running
```

Check all ArgoCD applications:

```bash
kubectl get applications -n argocd
# All 7 apps should show Synced / Healthy
```

**Test the API through the ingress:**

```bash
kubectl port-forward svc/ingress-nginx-controller -n ingress-nginx 8080:80 &

curl -H "Host: qoves.local" http://127.0.0.1:8080/
# Expected: "hello from the QOVES take-home API"

curl -H "Host: qoves.local" http://127.0.0.1:8080/healthz
# Expected: "ok"

curl -H "Host: qoves.local" http://127.0.0.1:8080/metrics
# Expected: Prometheus exposition format
```

> **Why port-forward?** On macOS with Docker driver, the minikube IP (192.168.49.x) is inside a Docker network and not directly reachable from the host. Port-forwarding the ingress controller is the reliable approach. Alternatively, `minikube tunnel` (requires sudo) maps the ingress to 127.0.0.1:80.

**Test NetworkPolicy enforcement:**

```bash
# Prove default-deny egress blocks unauthorized traffic
kubectl run test-egress --rm -it --image=busybox:1.36 -n qoves --restart=Never -- \
  wget -qO- --timeout=3 http://example.com
# Expected: timeout (blocked)

# Prove DNS still resolves (allowed by allow-dns-egress policy)
kubectl run test-dns --rm -it --image=busybox:1.36 -n qoves --restart=Never -- \
  nslookup postgres.qoves.svc.cluster.local
# Expected: resolves successfully
```

**Test data persistence:**

```bash
kubectl delete pod postgres-0 -n qoves
kubectl wait --for=condition=ready pod/postgres-0 -n qoves --timeout=120s
curl -H "Host: qoves.local" http://127.0.0.1:8080/healthz
# Expected: "ok" — PVC preserved the data across restart
```

**Test HPA:**

```bash
kubectl get hpa -n qoves
# Should show CPU target and current utilization
```

**Test Prometheus scraping:**

```bash
kubectl port-forward svc/kps-kube-prometheus-stack-prometheus -n monitoring 9090:9090 &
curl -s 'http://localhost:9090/api/v1/query?query=http_requests_total' | python3 -m json.tool | head -20
# Expected: metric results with path="/", path="/healthz"

curl -s 'http://localhost:9090/api/v1/rules' | python3 -m json.tool | grep QovesAPIHealthzFailing
# Expected: the alert rule is loaded
```

**Test Grafana:**

```bash
kubectl port-forward svc/kps-grafana -n monitoring 3000:80 &
# Open http://localhost:3000
# Login: admin / prom-operator
# Go to Explore → Prometheus → query: http_requests_total
```

**Verify no plaintext secrets in the repo:**

```bash
grep -r "changeme" manifests/ || echo "No plaintext secrets found - PASS"
```

### Repo Layout

```
qoves-assignment/
├── app/                          # Provided application
│   ├── main.py
│   ├── Dockerfile
│   └── requirements.txt
├── bootstrap/
│   └── root-application.yaml     # Root ArgoCD Application (app-of-apps)
├── applications/                 # Child ArgoCD Applications
│   ├── secrets.yml               #   Sealed Secrets controller (wave 0)
│   ├── database.yml              #   PostgreSQL StatefulSet + PVC (wave 1)
│   ├── ingress.yml               #   NetworkPolicies (wave 1)
│   ├── api.yml                   #   API Deployment + Service + Ingress + HPA (wave 2)
│   ├── prometheus.yml            #   kube-prometheus-stack Helm chart (wave 2)
│   └── monitoring.yml            #   ServiceMonitor + PrometheusRule (wave 3)
├── manifests/
│   ├── data/
│   │   ├── postgres-statefulset.yaml
│   │   ├── postgres-service.yaml
│   │   └── qoves-db-sealed.yaml
│   ├── api/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── ingress.yaml
│   │   └── hpa.yaml
│   ├── network/
│   │   ├── 00-default-deny.yaml
│   │   └── 10-allow.yaml
│   └── monitoring/
│       ├── servicemonitor.yaml
│       └── prometheusrule.yaml
├── WRITEUP.md
├── README.md
└── .gitignore
```

### Making a Change (GitOps Flow)

1. Edit manifest(s) locally (e.g. change `maxReplicas` in `manifests/api/hpa.yaml`)
2. `git add . && git commit -m "description" && git push origin main`
3. ArgoCD detects the diff within ~3 minutes (its default poll interval) and auto-syncs
4. `selfHeal: true` means any manual `kubectl edit` drift is reverted automatically

No `kubectl apply` or `kubectl edit` on running workloads — git is the only path to change the cluster.

---

## 2. Decisions

**CNI — Calico.** Alternatives: Cilium (eBPF, richer L7 policies), Flannel (no NetworkPolicy), minikube default (kindnet — silently ignores policies). Calico enforces NetworkPolicy out of the box, is widely deployed, and has lower overhead than Cilium on a local cluster. The assignment hints at it directly.

**Secrets — Sealed Secrets.** Alternatives: SOPS+age (needs ArgoCD plugin), ESO+Vault (needs a running Vault). Sealed Secrets is the simplest fully-local option: encrypt client-side, commit ciphertext, controller decrypts in-cluster. In production I would use ESO + Vault for rotation and audit trails.

**Database — Raw StatefulSet.** Alternatives: CloudNativePG (automated failover, WAL archiving), Zalando operator, Bitnami Helm chart. A raw StatefulSet makes every piece visible — no operator to explain. The PVC uses ReadWriteOnce, binding the pod to one node. Data survives pod restarts but not node loss. CloudNativePG is the right production choice.

**Scaling signal — CPU.** Alternatives: request rate via Prometheus Adapter, KEDA. CPU works out of the box with metrics-server. But this API is I/O-bound (waiting on Postgres), so CPU under-represents real load. Request rate or latency via Prometheus Adapter would be a better signal. I chose CPU for simplicity and noted the tradeoff.

---

## 3. What minikube Did for Me

On real bare metal, I would need to build or manage each of the following that minikube provides automatically:

**Control-plane bootstrap.** minikube runs `kubeadm init`, generates certs, configures API server, controller-manager, scheduler. On bare metal: run `kubeadm init` manually, manage CA distribution, configure HA control planes behind LB, HAProxy or kube-vip.

**CNI install.** minikube applies Calico manifests automatically. On bare metal: apply Tigera operator manifests, configure BGP peering, manage IP pool sizing for the pod CIDR.

**Ingress load-balancing.** minikube's addon deploys nginx ingress controller bound to the node IP. On bare metal: deploy MetalLB for L2/BGP advertisement or HAProxy in front of workers. No cloud LoadBalancer exists.

**Storage provisioner.** minikube provides a hostPath provisioner that auto-creates PVs. On bare metal: deploy a CSI driver (Longhorn, Rook-Ceph, OpenEBS) or configure NFS/iSCSI PVs. hostPath is not production-suitable — data is node-local with no replication.

**etcd.** minikube runs a single etcd static pod. On bare metal: run a 3- or 5-node etcd cluster with TLS, automated snapshots via `etcdctl snapshot save` on a cron stored off-cluster, and tested restores. Losing etcd without a backup means rebuilding from scratch. truth for the entire cluster — losing it without a backup means rebuilding from scratch.

---

## 4. Production Gaps

**HA.** Single-replica Postgres is a SPOF. Needs CloudNativePG with streaming replication or Patroni. API replicas need PodAntiAffinity across nodes/zones.

**Backups.** No backup strategy. Needs scheduled `pg_dump` or WAL archiving to S3/MinIO, regular etcd snapshots, and tested restore procedures.

**Secret backend.** Sealed Secrets lacks rotation, audit logging, dynamic credentials. Needs ESO + HashiCorp Vault or cloud-native secret managers.

**TLS.** Ingress serves plain HTTP. Needs cert-manager with Let's Encrypt, mTLS via service mesh (Istio/Linkerd) or Calico WireGuard for internal traffic.

**Upgrades.** No PodDisruptionBudgets, no progressive delivery. Needs PDBs, a defined Kubernetes upgrade path, and Argo Rollouts for canary/blue-green deploys.

**Multi-cluster.** Single cluster. Needs staging/production separation with Kustomize overlays or Helm value files per environment.

**Observability.** One alert rule. Needs SLO-based alerting, structured logging (Loki), distributed tracing (OpenTelemetry + Tempo), and Grafana dashboards for golden signals.

**Resource quotas.** No namespace-level quotas. Needs ResourceQuota and LimitRange to cap consumption.

---

## 5. One Runbook: Database Pod Dies

**Failure mode:** The `postgres-0` pod crashes or is evicted (OOMKilled, node failure, manual deletion).

### Detection

- `QovesAPIHealthzFailing` alert fires (Prometheus detects `/healthz` returning 503 for > 2 minutes)
- ArgoCD shows the `qoves-data` application as Degraded
- `kubectl get pods -n qoves` shows `postgres-0` in CrashLoopBackOff or missing

### Recovery Steps

**Step 1 — Assess the damage:**

```bash
kubectl describe pod postgres-0 -n qoves
kubectl logs postgres-0 -n qoves --previous
kubectl get events -n qoves --sort-by='.lastTimestamp' | head -20
```

**Step 2 — Check PVC status:**

```bash
kubectl get pvc -n qoves
# postgres-data-postgres-0 should show Bound — data is intact
```

Because the StatefulSet uses a PVC with `ReadWriteOnce`, the data persists on disk even when the pod dies. The StatefulSet controller will automatically restart the pod and re-attach the same PVC.

**Step 3 — If the pod does not auto-recover (CrashLoopBackOff):**

Check if the issue is resource-related:

```bash
kubectl top pod postgres-0 -n qoves
kubectl describe node $(kubectl get pod postgres-0 -n qoves -o jsonpath='{.spec.nodeName}')
```

If OOMKilled — fix via git (GitOps flow):

1. Edit `manifests/data/postgres-statefulset.yaml` — increase memory limit
2. `git add . && git commit -m "increase postgres memory limit to 1Gi" && git push origin main`
3. ArgoCD syncs the change; the StatefulSet rolls the pod with new limits
4. Verify: `kubectl get pod postgres-0 -n qoves -w` — wait for Running

**Step 4 — If the node itself died:**

The PVC is `ReadWriteOnce` — it is bound to the old node. If the node comes back, the pod restarts. If the node is permanently gone:

```bash
kubectl delete pod postgres-0 -n qoves --force --grace-period=0
# The scheduler places it on a surviving node
# The PVC may need manual intervention if the underlying volume is node-local (hostPath)
```

With minikube's hostPath provisioner, data on a dead node is lost. This is a known limitation — see Production Gaps for the backup strategy that would mitigate this.

**Step 5 — Verify recovery:**

```bash
kubectl get pod postgres-0 -n qoves                                          # Running
kubectl port-forward svc/ingress-nginx-controller -n ingress-nginx 8080:80 &
curl -H "Host: qoves.local" http://127.0.0.1:8080/healthz                   # "ok"
```

**Step 6 — Post-incident:**

- Confirm the `QovesAPIHealthzFailing` alert resolves automatically once `/healthz` returns 200
- Check ArgoCD: `kubectl get applications -n argocd` — all apps should be Healthy/Synced
- If any data was lost, restore from the most recent `pg_dump` backup (not applicable in this minikube setup — see Production Gaps)
