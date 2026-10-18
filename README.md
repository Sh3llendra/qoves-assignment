# QOVES Take-Home Platform

GitOps-managed Kubernetes platform on minikube. ArgoCD app-of-apps, Calico CNI, Sealed Secrets, default-deny NetworkPolicies, HPA, Prometheus + Grafana.

## Architecture

```
git push → GitHub → ArgoCD (app-of-apps) → minikube (Calico CNI)

Sync-wave order:
  0  sealed-secrets controller
  1  PostgreSQL StatefulSet + PVC, NetworkPolicies
  2  API Deployment + Ingress + HPA, kube-prometheus-stack
  3  ServiceMonitor + PrometheusRule
```

## Repo Layout

```
app/                    Flask API (main.py, Dockerfile, requirements.txt)
bootstrap/              Root ArgoCD Application
applications/           Child ArgoCD Applications (*.yml, sync-wave ordered)
manifests/data/         PostgreSQL StatefulSet, Service, SealedSecret
manifests/api/          API Deployment, Service, Ingress, HPA
manifests/network/      Default-deny + allow-list NetworkPolicies
manifests/monitoring/   ServiceMonitor, PrometheusRule
```

## Setup

Full from-scratch instructions in [WRITEUP.md](WRITEUP.md) — Section 1.

## Writeup

[WRITEUP.md](WRITEUP.md) covers the five required sections:

1. **Run it** — setup, repo layout, GitOps flow, validation
2. **Decisions** — CNI, secrets, database, scaling signal
3. **What minikube did for me** — control plane, CNI, ingress, storage, etcd
4. **Production gaps** — HA, backups, TLS, secret backend, multi-cluster
5. **One runbook** — database pod failure recovery
