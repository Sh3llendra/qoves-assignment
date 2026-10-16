# QOVES Senior DevOps Take-Home — Implementation Guide

> **Estimated total time: 6–8 hours** (spread across steps below)

---

## Table of Contents

1. [Repo Restructure (Current Code Changes)](#1-repo-restructure)
2. [Part A — Cluster & CNI](#2-part-a--cluster--cni)
3. [Part B — GitOps Delivery (ArgoCD)](#3-part-b--gitops-delivery)
4. [Part C — App & Database Manifests](#4-part-c--app--database-manifests)
5. [Part D — Networking & Least Privilege](#5-part-d--networking--least-privilege)
6. [Part E — Secrets (Sealed Secrets)](#6-part-e--secrets)
7. [Part F — Storage & Data](#7-part-f--storage--data)
8. [Part G — Resources & Scaling (HPA)](#8-part-g--resources--scaling)
9. [Part H — Observability (Prometheus + Alert)](#9-part-h--observability)
10. [Validation & Proof](#10-validation--proof)
11. [WRITEUP.md](#11-writeupmd)

---

## Prerequisites

Install these tools on your Mac before starting:

```bash
# Homebrew installs
brew install minikube kubectl helm argocd kubeseal

# Docker Desktop must be installed and running
# (minikube will use the docker driver)
```

---

## 1. Repo Restructure

**Time estimate: 15 minutes**

Your current repo needs these changes to match the target layout. Since code is already pushed, do these as a single commit.

### 1a. Rename `applications/` → `apps/` and normalize file names

```bash
cd qoves-assignment

# Rename the directory
git mv applications apps

# Rename files to match target structure (.yml → .yaml, cleaner names)
git mv apps/api.yml apps/qoves-api.yaml
git mv apps/database.yml apps/qoves-data.yaml
git mv apps/ingress.yml apps/qoves-network.yaml
git mv apps/secrets.yml apps/sealed-secrets.yaml
git mv apps/monitoring.yml apps/qoves-monitoring.yaml
git mv apps/prometheus.yml apps/prometheus-stack.yaml
```

### 1b. Update `bootstrap/root-application.yaml` → `bootstrap/root-app.yaml`

```bash
git mv bootstrap/root-application.yaml bootstrap/root-app.yaml
```

Then update the `path` inside it from `apps` (it already says `apps`, but confirm after rename):

```yaml
# bootstrap/root-app.yaml — no content change needed if path was already 'apps'
# Just verify the source.path is: apps
```

### 1c. Update repoURL in all ArgoCD Application files

After renaming, the `repoURL` in every file under `apps/` already points to `https://github.com/Sh3llendra/qoves-assignment` — that's correct, keep it.

### 1d. Create the manifests directory structure

```bash
mkdir -p manifests/data manifests/network manifests/api manifests/monitoring
```

### 1e. Commit the restructure

```bash
git add -A
git commit -m "restructure: rename applications/ to apps/, normalize filenames, create manifests dirs"
git push origin main
```

---

## 2. Part A — Cluster & CNI

**Time estimate: 15–20 minutes** (Calico download can be slow)

### 2a. Delete any existing minikube cluster

```bash
minikube delete
```

### 2b. Start minikube with Calico CNI and 2 nodes

```bash
minikube start \
  --nodes=2 \
  --cni=calico \
  --driver=docker \
  --cpus=2 \
  --memory=4096 \
  --kubernetes-version=v1.30.0
```

**Why Calico?**
- minikube's default CNI (kindnet) does **not** enforce NetworkPolicy — policies silently do nothing.
- Calico enforces NetworkPolicy out of the box and is battle-tested in production.
- Cilium also works but is heavier for a local setup.

### 2c. Verify Calico is running

```bash
kubectl get pods -n kube-system | grep calico
```

You should see `calico-node-*` pods Running on both nodes.

### 2d. Enable required addons

```bash
minikube addons enable ingress
minikube addons enable metrics-server
```

Verify:

```bash
kubectl get pods -n ingress-nginx
kubectl get pods -n kube-system | grep metrics-server
```

---

## 3. Part B — GitOps Delivery

**Time estimate: 20–30 minutes**

### 3a. Install ArgoCD (by hand — this is allowed)

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.13.3/manifests/install.yaml
```

Wait for ArgoCD to be ready:

```bash
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s
```

### 3b. Get ArgoCD admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 3c. (Optional) Access ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8443:443 &
# Open https://localhost:8443 — login: admin / <password from above>
```

### 3d. Apply the root app-of-apps (the ONE manual apply)

```bash
kubectl apply -f bootstrap/root-app.yaml
```

This is the **only** `kubectl apply` you do for workloads. Everything else reconciles through ArgoCD from git.

### 3e. Verify ArgoCD picks up child apps

```bash
argocd app list
# or
kubectl get applications -n argocd
```

At this point the child apps will show as `OutOfSync` or `Missing` because the manifests directories are empty. That's expected — we'll fill them next.

---

## 4. Part C — App & Database Manifests

**Time estimate: 45–60 minutes**

### 4a. Build and tag the API image

```bash
cd app
docker build -t qoves-api:0.1.0 .
```

Load it into minikube (so the cluster can pull it without a registry):

```bash
minikube image load qoves-api:0.1.0
```

Go back to repo root:

```bash
cd ..
```

### 4b. Create `manifests/data/postgres-statefulset.yaml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: qoves
  labels:
    app: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16.3-bookworm
          ports:
            - containerPort: 5432
              name: pg
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: qoves-db-credentials
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: qoves-db-credentials
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_DB
              value: qoves
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          volumeMounts:
            - name: pgdata
              mountPath: /var/lib/postgresql/data
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "qoves"]
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            exec:
              command: ["pg_isready", "-U", "qoves"]
            initialDelaySeconds: 15
            periodSeconds: 20
  volumeClaimTemplates:
    - metadata:
        name: pgdata
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

### 4c. Create `manifests/data/postgres-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: qoves
  labels:
    app: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
      name: pg
```

### 4d. Create `manifests/api/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: qoves-api
  namespace: qoves
  labels:
    app: qoves-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: qoves-api
  template:
    metadata:
      labels:
        app: qoves-api
    spec:
      containers:
        - name: api
          image: qoves-api:0.1.0
          imagePullPolicy: Never
          ports:
            - containerPort: 8080
              name: http
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: qoves-db-credentials
                  key: DATABASE_URL
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
          readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 15
```

> **Note:** `imagePullPolicy: Never` because we loaded the image into minikube directly. In production you'd use a registry and `IfNotPresent`.

### 4e. Create `manifests/api/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: qoves-api
  namespace: qoves
  labels:
    app: qoves-api
spec:
  selector:
    app: qoves-api
  ports:
    - port: 80
      targetPort: 8080
      name: http
```

### 4f. Create `manifests/api/ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: qoves-api
  namespace: qoves
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: qoves.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: qoves-api
                port:
                  number: 80
```

### 4g. Commit and push — ArgoCD reconciles automatically

```bash
git add manifests/
git commit -m "add api deployment, postgres statefulset, service, and ingress manifests"
git push origin main
```

ArgoCD will detect the change and sync. You can force it:

```bash
argocd app sync qoves-data
argocd app sync qoves-api
```

### 4h. Add qoves.local to /etc/hosts

```bash
echo "$(minikube ip) qoves.local" | sudo tee -a /etc/hosts
```

---

## 5. Part D — Networking & Least Privilege

**Time estimate: 30–40 minutes**

### 5a. Create `manifests/network/00-default-deny.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: qoves
spec:
  podSelector: {}
  policyTypes:
    - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: qoves
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

### 5b. Create `manifests/network/10-allow.yaml`

```yaml
# Allow DNS egress for all pods (easy to forget!)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: qoves
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to: []
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
---
# Allow ingress-nginx → qoves-api
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-api
  namespace: qoves
spec:
  podSelector:
    matchLabels:
      app: qoves-api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
---
# Allow qoves-api → postgres
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: qoves
spec:
  podSelector:
    matchLabels:
      app: qoves-api
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
---
# Allow postgres to accept from api
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-db-ingress-from-api
  namespace: qoves
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: qoves-api
      ports:
        - protocol: TCP
          port: 5432
---
# Allow Prometheus to scrape the API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: qoves
spec:
  podSelector:
    matchLabels:
      app: qoves-api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 8080
```

### 5c. Commit and push

```bash
git add manifests/network/
git commit -m "add default-deny network policies and allow rules for app namespace"
git push origin main
```

### 5d. Test that NetworkPolicy actually blocks traffic

After ArgoCD syncs:

```bash
# Spin up a test pod in the qoves namespace
kubectl run nettest --rm -it --image=busybox:1.36 -n qoves --restart=Never -- sh

# Inside the pod, try to reach something you didn't allow (should fail/timeout):
wget --timeout=3 -qO- http://google.com
# Expected: timeout — egress is blocked (except DNS and postgres)

# Try to reach the API (should also fail — no policy allows pod-to-pod in general):
wget --timeout=3 -qO- http://qoves-api
# Expected: timeout

# Exit the test pod
exit
```

Capture this output as proof for submission.

### 5e. Verify the app still works through ingress

```bash
curl http://qoves.local/
# Expected: "hello from the QOVES take-home API"

curl http://qoves.local/healthz
# Expected: "ok" (200)
```

---

## 6. Part E — Secrets

**Time estimate: 30–40 minutes**

We use **Sealed Secrets** — the simplest fully-local option. The sealed-secrets controller is already defined in `apps/sealed-secrets.yaml` and will be installed by ArgoCD.

### 6a. Wait for Sealed Secrets controller to be running

```bash
kubectl get pods -n kube-system | grep sealed-secrets
```

If it's not up yet, sync it:

```bash
argocd app sync sealed-secrets
```

### 6b. Create the plaintext secret locally (NEVER commit this)

```bash
# Create a temp file — .gitignore already excludes secret*.yaml
cat > /tmp/qoves-db-secret-plain.yaml <<EOF
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
```

### 6c. Seal it with kubeseal

```bash
kubeseal \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  --format=yaml \
  < /tmp/qoves-db-secret-plain.yaml \
  > manifests/data/qoves-db-sealed.yaml
```

### 6d. Delete the plaintext immediately

```bash
rm /tmp/qoves-db-secret-plain.yaml
```

### 6e. Verify the sealed secret file looks right

```bash
cat manifests/data/qoves-db-sealed.yaml
# Should have kind: SealedSecret with encrypted values — no plaintext
```

### 6f. Commit and push the sealed secret

```bash
git add manifests/data/qoves-db-sealed.yaml
git commit -m "add sealed secret for database credentials (encrypted, safe for git)"
git push origin main
```

ArgoCD syncs → Sealed Secrets controller decrypts → creates the real `qoves-db-credentials` Secret in the `qoves` namespace.

### 6g. Verify the secret was created

```bash
kubectl get secret qoves-db-credentials -n qoves
# Should exist and show Opaque type
```

---

## 7. Part F — Storage & Data

**Time estimate: 10 minutes** (mostly verification, reasoning goes in WRITEUP.md)

### 7a. Verify the PVC is bound

```bash
kubectl get pvc -n qoves
# pgdata-postgres-0 should be Bound
```

### 7b. Test data persistence across pod restart

```bash
# Write some data
kubectl exec -n qoves postgres-0 -- psql -U qoves -d qoves -c "CREATE TABLE test (id serial PRIMARY KEY, val text);"
kubectl exec -n qoves postgres-0 -- psql -U qoves -d qoves -c "INSERT INTO test (val) VALUES ('survives restart');"

# Kill the pod
kubectl delete pod postgres-0 -n qoves

# Wait for it to come back
kubectl wait --for=condition=ready pod/postgres-0 -n qoves --timeout=120s

# Check data survived
kubectl exec -n qoves postgres-0 -- psql -U qoves -d qoves -c "SELECT * FROM test;"
# Should show: "survives restart"
```

Capture this output as proof.

---

## 8. Part G — Resources & Scaling

**Time estimate: 15–20 minutes**

Resource requests/limits are already set in the deployment and statefulset manifests above. Now add the HPA.

### 8a. Create `manifests/api/hpa.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: qoves-api
  namespace: qoves
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: qoves-api
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### 8b. Commit and push

```bash
git add manifests/api/hpa.yaml
git commit -m "add HPA for qoves-api deployment (CPU-based, 2-5 replicas)"
git push origin main
```

### 8c. Verify HPA

```bash
kubectl get hpa -n qoves
# Should show qoves-api with TARGETS showing current/70%
```

---

## 9. Part H — Observability

**Time estimate: 30–40 minutes**

### 9a. Write `apps/prometheus-stack.yaml`

This ArgoCD Application installs kube-prometheus-stack via Helm:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prometheus-stack
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 61.2.0
    helm:
      releaseName: kube-prometheus-stack
      values: |
        prometheus:
          prometheusSpec:
            serviceMonitorSelectorNilUsesHelmValues: false
            podMonitorSelectorNilUsesHelmValues: false
            ruleSelectorNilUsesHelmValues: false
        grafana:
          enabled: false
        alertmanager:
          enabled: true
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

### 9b. Create `manifests/monitoring/servicemonitor.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: qoves-api
  namespace: monitoring
  labels:
    app: qoves-api
spec:
  namespaceSelector:
    matchNames:
      - qoves
  selector:
    matchLabels:
      app: qoves-api
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

### 9c. Create `manifests/monitoring/prometheusrule.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: qoves-api-alerts
  namespace: monitoring
  labels:
    app: qoves-api
spec:
  groups:
    - name: qoves-api
      rules:
        - alert: HealthzEndpointDown
          expr: |
            sum(rate(http_requests_total{path="/healthz", status="503"}[5m]))
            /
            sum(rate(http_requests_total{path="/healthz"}[5m]))
            > 0.5
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "More than 50% of /healthz requests return 503 for 2+ minutes"
            description: "The API cannot reach the database — investigate postgres connectivity."
            runbook: "Check postgres-0 pod status, PVC binding, and NetworkPolicy."
```

**Why this alert?** A high 503 rate on `/healthz` means the API has lost database connectivity — the one dependency this service has. It's actionable (check postgres), not noisy (2-minute `for` window avoids blips), and maps to a real user-facing outage.

### 9d. Commit and push

```bash
git add apps/prometheus-stack.yaml manifests/monitoring/
git commit -m "add prometheus stack, servicemonitor, and healthz alert rule"
git push origin main
```

### 9e. Verify Prometheus is scraping

```bash
# Port-forward to Prometheus
kubectl port-forward svc/kube-prometheus-stack-prometheus -n monitoring 9090:9090 &

# Query the metric
curl -s 'http://localhost:9090/api/v1/query?query=http_requests_total' | python3 -m json.tool
# Should return results with path="/", path="/healthz", etc.
```

### 9f. Verify the alert rule loaded

```bash
curl -s 'http://localhost:9090/api/v1/rules' | python3 -m json.tool | grep HealthzEndpointDown
```

---

## 10. Validation & Proof

**Time estimate: 15 minutes**

Run these commands and capture output (screenshot or copy-paste) for submission:

```bash
# 1. All resources
kubectl get pods,svc,ingress,netpol -A

# 2. ArgoCD app tree
argocd app list
argocd app get root

# 3. Curl through ingress
curl http://qoves.local/
curl http://qoves.local/healthz
curl http://qoves.local/metrics

# 4. NetworkPolicy blocks traffic (run from a test pod)
kubectl run nettest --rm -it --image=busybox:1.36 -n qoves --restart=Never -- \
  wget --timeout=3 -qO- http://google.com
# Should timeout

# 5. DB data survives restart (already done in Part F)

# 6. HPA status
kubectl get hpa -n qoves

# 7. Sealed secret — no plaintext in repo
grep -r "changeme" manifests/ || echo "No plaintext secrets found ✓"
```

---

## 11. WRITEUP.md

**Time estimate: 60–90 minutes**

Create `WRITEUP.md` in the repo root covering these five sections. Below is the outline — fill in with your own reasoning:

### Section 1: Run It
- Prerequisites: minikube, kubectl, helm, kubeseal, argocd CLI
- `minikube start --nodes=2 --cni=calico ...`
- `kubectl apply -f bootstrap/root-app.yaml` — the one manual apply
- Repo layout explanation
- How to make a change: edit a manifest → commit → push → ArgoCD auto-syncs

### Section 2: Decisions (ADR-style)
- **CNI: Calico** — minikube default doesn't enforce NetworkPolicy; Calico does, lighter than Cilium for local dev.
- **Secrets: Sealed Secrets** — fully local, no external dependency like Vault; controller decrypts in-cluster; encrypted ciphertext is safe to commit.
- **Postgres: Raw StatefulSet** — simpler than CloudNativePG for a single-instance dev setup; PVC provides persistence; operator would add HA/backups in production.
- **Scaling signal: CPU** — adequate for this I/O-light API; in production, request latency (p99) or request rate via custom metrics would be more appropriate since this API is I/O-bound (waiting on Postgres).

### Section 3: What Minikube Did for Me
- **Control-plane bootstrap**: kubeadm, etcd, API server, scheduler, controller-manager — all provisioned automatically. On bare metal: you'd run kubeadm init, manage certs, configure HA with multiple control-plane nodes behind a load balancer.
- **CNI install**: minikube auto-applied Calico manifests. On bare metal: download and apply Calico operator + custom resource, configure IP pools, BGP peering if needed.
- **Ingress load-balancing**: minikube addon gives a single-node nginx-ingress. On bare metal: deploy nginx-ingress with a DaemonSet on edge nodes, front it with MetalLB or an external LB (HAProxy/keepalived VIP).
- **Storage provisioner**: minikube's hostPath provisioner auto-creates PVs. On bare metal: deploy a CSI driver (Longhorn, Rook-Ceph, or local-path-provisioner), manage disk allocation, plan for replication.
- **etcd + backup**: minikube runs a single etcd pod. On bare metal: run a 3- or 5-node etcd cluster, set up periodic snapshots (`etcdctl snapshot save`), test restore regularly.

### Section 4: Production Gaps
- **HA**: single Postgres instance — need streaming replication or CloudNativePG with 2+ replicas
- **Backups**: no automated DB backup — need pg_dump CronJob or WAL archiving to object storage
- **Real secret backend**: Sealed Secrets is fine for dev; production needs Vault or a cloud KMS (AWS Secrets Manager, GCP Secret Manager) via External Secrets Operator
- **TLS**: no TLS termination — need cert-manager + Let's Encrypt or a real certificate
- **Upgrades**: no strategy for Kubernetes version upgrades, no PodDisruptionBudgets
- **Multi-cluster**: single cluster is a SPOF — need federation or multi-cluster GitOps (ArgoCD ApplicationSets)
- **Image registry**: using `minikube image load` — need a private registry (Harbor) with vulnerability scanning
- **Audit logging**: no API server audit policy configured

### Section 5: One Runbook — "DB Pod Dies"

**Failure mode: postgres-0 pod is killed or OOMKilled**

1. **Detect**: Alert `HealthzEndpointDown` fires (>50% of /healthz returning 503 for 2m). PagerDuty/Slack notification arrives.
2. **Assess**: 
   ```bash
   kubectl get pods -n qoves -l app=postgres
   kubectl describe pod postgres-0 -n qoves
   kubectl logs postgres-0 -n qoves --previous
   ```
3. **If CrashLoopBackOff** (bad config):
   - Check if the PVC is still bound: `kubectl get pvc -n qoves`
   - Check events: `kubectl get events -n qoves --sort-by=.metadata.creationTimestamp`
   - Fix via git: update the manifest, commit, push → ArgoCD syncs
4. **If OOMKilled**:
   - Increase memory limit in `manifests/data/postgres-statefulset.yaml`
   - Commit → push → ArgoCD syncs → pod restarts with higher limit
5. **If node failure**:
   - Pod reschedules to remaining node (2-node cluster)
   - PVC with `ReadWriteOnce` means it can only attach to one node — if the old node is truly dead, the PV may need manual intervention (force-delete the old pod, or delete the VolumeAttachment)
6. **Verify recovery**:
   ```bash
   kubectl exec -n qoves postgres-0 -- pg_isready -U qoves
   curl http://qoves.local/healthz
   ```
7. **Post-incident**: check data integrity, review resource limits, consider adding PodDisruptionBudget.

---

## Time Breakdown Summary

| Part | Task | Estimate |
|------|------|----------|
| 1 | Repo restructure | 15 min |
| A | Cluster & CNI | 15–20 min |
| B | GitOps (ArgoCD) | 20–30 min |
| C | App & database manifests | 45–60 min |
| D | NetworkPolicies | 30–40 min |
| E | Sealed Secrets | 30–40 min |
| F | Storage verification | 10 min |
| G | HPA | 15–20 min |
| H | Prometheus + alert | 30–40 min |
| — | Validation & proof | 15 min |
| — | WRITEUP.md | 60–90 min |
| **Total** | | **~5–7 hours** |

---

## Quick Reference: Order of Operations

```
1. Restructure repo          (git mv, mkdir)
2. minikube start --cni=calico
3. Enable addons              (ingress, metrics-server)
4. Install ArgoCD             (kubectl apply)
5. kubectl apply -f bootstrap/root-app.yaml   ← ONLY manual apply
6. Build + load image         (docker build, minikube image load)
7. Create + commit manifests  (data, api, network, monitoring)
8. Seal the secret            (kubeseal)
9. Push to git                → ArgoCD reconciles everything
10. Validate + capture proof
11. Write WRITEUP.md
```
