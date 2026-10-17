# QOVES Platform Writeup

## 1. Run It

### Prerequisites

- Docker (running)
- minikube v1.37.0+
- kubectl
- kubeseal CLI
- Helm (for ArgoCD install verification)

### Cluster Bootstrap

```bash
minikube start --nodes=2 --cni=calico --driver=docker --cpus=2 --memory=3g --kubernetes-version=stable
minikube addons enable ingress
minikube addons enable metrics-server
```

### Build the Container Image

```bash
eval $(minikube docker-env)
cd app && docker build -t qoves-api:0.1.0 . && cd ..
```

### Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.11.7/manifests/install.yaml
kubectl wait --for=condition=available deploy/argocd-server -n argocd --timeout=300s
```

### Generate the Sealed Secret

```bash
kubectl create namespace qoves

kubectl create secret generic qoves-db-credentials \
  --namespace=qoves \
  --from-literal=POSTGRES_USER=qoves \
  --from-literal=POSTGRES_PASSWORD=changeme \
  --from-literal=POSTGRES_DB=qovesdb \
  --from-literal=DATABASE_URL="postgresql://qoves:changeme@postgres.qoves.svc:5432/qovesdb" \
  --dry-run=client -o yaml | \
kubeseal --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  --format=yaml > manifests/data/qoves-db-sealed.yaml
```

Commit and push the sealed secret (never the plaintext).

### Deploy via GitOps

```bash
git add -A && git commit -m "populate manifests" && git push origin main
kubectl apply -f bootstrap/root-application.yaml
```

ArgoCD discovers `applications/`, creates child Applications, and reconciles all manifests into the cluster.

### Repo Layout

```
qoves-assignment/
  app/                          # Provided application
    main.py
    Dockerfile
    requirements.txt
  bootstrap/
    root-application.yaml       # Root ArgoCD Application (app-of-apps)
  applications/                 # Child ArgoCD Applications
    secrets.yml                 # Sealed Secrets controller (wave 0)
    database.yml                # PostgreSQL StatefulSet + PVC (wave 1)
    ingress.yml                 # NetworkPolicies (wave 1)
    api.yml                     # API Deployment + Service + Ingress + HPA (wave 2)
    prometheus.yml              # kube-prometheus-stack Helm chart (wave 2)
    monitoring.yml              # ServiceMonitor + PrometheusRule (wave 3)
  manifests/
    data/
      postgres-statefulset.yaml
      postgres-service.yaml
      qoves-db-sealed.yaml
    api/
      deployment.yaml
      service.yaml
      ingress.yaml
      hpa.yaml
    network/
      00-default-deny.yaml
      10-allow.yaml
    monitoring/
      servicemonitor.yaml
      prometheusrule.yaml
  WRITEUP.md
```

### Making a Change (GitOps Flow)

1. Edit manifest(s) locally (e.g. bump replica count in `manifests/api/deployment.yaml`)
2. `git commit` and `git push origin main`
3. ArgoCD detects the diff within its poll interval (default 3 min) or sync manually: `argocd app sync root`
4. The controller applies the change; `selfHeal: true` reverts any manual drift

### Validation

```bash
# Add host entry
echo "$(minikube ip) qoves.local" | sudo tee -a /etc/hosts

# Health check through ingress
curl -H "Host: qoves.local" http://$(minikube ip)/healthz

# All resources
kubectl get pods,svc,ingress,netpol -n qoves

# ArgoCD app tree
kubectl get applications -n argocd

# Prove NetworkPolicy blocks unauthorized egress
kubectl run test --rm -it --image=busybox -n qoves -- wget -qO- --timeout=3 http://example.com
# Expected: timeout (blocked by default-deny egress)
```

---

## 2. Decisions

### CNI: Calico

**Decision:** Calico as the CNI plugin via `minikube start --cni=calico`.

**Alternatives:** Cilium (eBPF-based, richer L7 policies), Flannel (simple but no NetworkPolicy enforcement), minikube default (kindnet — silently ignores NetworkPolicy).

**Why:** The assignment requires enforced NetworkPolicy. minikube's default CNI does not enforce policies — they are silently no-ops. Calico is the most widely deployed CNI for NetworkPolicy enforcement, has excellent documentation, low resource overhead on minikube, and is the CNI hinted in the assignment. Cilium would also work but adds complexity (eBPF kernel requirements) that is unnecessary for this scope.

### Secrets: Sealed Secrets

**Decision:** Bitnami Sealed Secrets for encrypting credentials into git.

**Alternatives:** SOPS with age/PGP (requires ArgoCD plugin config), External Secrets Operator with Vault (heavier dependency, needs a running Vault instance).

**Why:** Sealed Secrets is the simplest fully-local option. The controller runs in-cluster, `kubeseal` encrypts on the client side, and the SealedSecret resource is safe to commit. No external dependency. In production, I would pair this with or migrate to ESO + HashiCorp Vault for centralized rotation and audit logging, but for a local minikube demo Sealed Secrets delivers the guarantee — no plaintext in git — with minimal moving parts.

### Database: Raw StatefulSet

**Decision:** A single-replica PostgreSQL StatefulSet with a PVC, rather than CloudNativePG or Helm chart.

**Alternatives:** CloudNativePG operator (automated failover, backups, WAL archiving), Zalando Postgres Operator, Crunchy PGO, Bitnami Helm chart.

**Why:** The assignment asks for a trivial API with one database. A raw StatefulSet makes every piece visible and defensible — there is no operator magic to explain. The PVC uses `ReadWriteOnce`, which binds the pod to one node; data survives pod restarts but not node loss without a backup strategy. CloudNativePG would be the right choice in production (see Production Gaps).

### Scaling Signal: CPU-based HPA

**Decision:** HPA targeting 70% average CPU utilization, min 2 / max 5 replicas.

**Alternatives:** Custom metrics (request rate via Prometheus Adapter), KEDA with external metrics.

**Why:** CPU is the simplest signal and works out of the box with the metrics-server addon. However, for this specific API — a lightweight Flask app behind gunicorn where requests are I/O-bound (waiting on Postgres) — CPU is not the ideal signal. Request latency or request rate (`http_requests_total`) via Prometheus Adapter would better reflect actual load. CPU-based scaling risks under-reacting to I/O-bound saturation and over-reacting to brief CPU spikes. I chose CPU for simplicity and noted this tradeoff.

---

## 3. What minikube Did for Me

On real bare metal, I would need to build or manage each of the following that minikube provides automatically:

**Control-plane bootstrap:** minikube runs `kubeadm init` internally, generates certs, configures the API server, controller-manager, and scheduler. On bare metal: I would run `kubeadm init` myself, manage the CA, distribute kubeconfig files, and configure HA with multiple control-plane nodes behind a load balancer (e.g. HAProxy or kube-vip).

**CNI installation:** minikube installs Calico manifests automatically with `--cni=calico`. On bare metal: I would apply the Calico manifests (or use the Tigera operator), configure BGP peering if needed, and manage IP pool sizing for the pod CIDR.

**Ingress load-balancing:** minikube's ingress addon deploys an nginx ingress controller and binds it to the node IP. On bare metal: I would need an external load balancer (MetalLB for L2/BGP advertisement, or HAProxy/Nginx in front of worker nodes) to route external traffic to the ingress controller pods. There is no cloud LoadBalancer service type.

**Storage provisioner:** minikube provides a built-in `hostPath` provisioner that auto-creates PVs for any PVC. On bare metal: I would deploy a CSI driver (e.g. Longhorn, Rook-Ceph, or OpenEBS) or configure NFS/iSCSI-backed PVs manually. `hostPath` is not suitable for production — it ties data to a single node with no replication.

**etcd and its backup:** minikube runs a single etcd instance as a static pod. On bare metal: I would run a 3- or 5-node etcd cluster (odd quorum), configure TLS peer/client certs, set up automated snapshots (`etcdctl snapshot save`) on a cron, store them off-cluster (S3 or NFS), and regularly test restores. etcd is the single point of truth for the entire cluster — losing it without a backup means rebuilding from scratch.

---

## 4. Production Gaps

**High availability:** Single-replica Postgres is a single point of failure. Production needs a Postgres HA setup (CloudNativePG with streaming replication, or Patroni) with automated failover. The API deployment has 2 replicas but `PodAntiAffinity` should spread them across nodes/zones.

**Backups:** No backup strategy exists. Production needs: scheduled `pg_dump` or WAL archiving to object storage (S3/MinIO), with tested restore procedures. etcd snapshots should be taken regularly. PVC snapshots via VolumeSnapshot if the CSI driver supports it.

**Real secret backend:** Sealed Secrets encrypts at rest in git but doesn't support rotation, audit logging, or dynamic credentials. Production should use External Secrets Operator + HashiCorp Vault (or AWS Secrets Manager / GCP Secret Manager) for centralized management, automatic rotation, and audit trails.

**TLS:** The ingress serves plain HTTP. Production needs cert-manager with Let's Encrypt (or internal CA) for automatic TLS certificate provisioning and renewal. All internal service-to-service traffic should use mTLS via a service mesh (Istio/Linkerd) or Calico's WireGuard encryption.

**Upgrade strategy:** No rollout strategy beyond Kubernetes default (RollingUpdate). Production needs: `PodDisruptionBudgets` to prevent total outage during node drains, a defined upgrade path for Kubernetes versions (drain, cordon, upgrade kubelet/kubeadm), and ArgoCD progressive delivery (Argo Rollouts with canary/blue-green).

**Multi-cluster:** Everything runs in a single minikube cluster. Production needs: separate clusters for staging/production, potentially across regions. GitOps would target different overlays per cluster (Kustomize overlays or Helm value files per environment).

**Observability depth:** Only one alert rule exists. Production needs: full alerting (SLO-based — latency percentiles, error rates, saturation), structured logging (Loki or ELK), distributed tracing (OpenTelemetry + Jaeger/Tempo), and Grafana dashboards for the golden signals.

**Resource quotas and LimitRanges:** No namespace-level quotas. Production should set `ResourceQuota` and `LimitRange` on the `qoves` namespace to prevent runaway resource consumption and ensure every pod has requests/limits.

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
2. `git commit -m "increase postgres memory limit to 1Gi" && git push origin main`
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
kubectl get pod postgres-0 -n qoves         # Running
curl -H "Host: qoves.local" http://$(minikube ip)/healthz  # "ok"
```

**Step 6 — Post-incident:**

- Confirm the alert resolves automatically once `/healthz` returns 200
- Check ArgoCD: `kubectl get applications -n argocd` — all apps should be Healthy/Synced
- If any data was lost, restore from the most recent `pg_dump` backup (not applicable in this minikube setup — see Production Gaps)
