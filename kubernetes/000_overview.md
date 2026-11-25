# Kubernetes — Overview

## What It Is
**Kubernetes (K8s)** is a container orchestrator. You declare the **desired state** ("run 3 replicas of this image, exposed on port 80") and Kubernetes keeps the **actual state** matching it: scheduling, restarting, scaling, networking and rolling out changes.

```
You:  kubectl apply -f app.yaml   (desired state)
K8s:  observe → diff → act → repeat   (reconciliation loop)
```

## Index

| # | File | What it covers |
|---|---|---|
| 001 | [Architecture](001_architecture.md) | Control plane, node components, what happens on `kubectl apply` |
| 002 | [Pods](002_pods.md) | Pod lifecycle, multi-container, init and sidecar containers |
| 003 | [Workloads](003_workloads.md) | ReplicaSet, Deployment, StatefulSet, DaemonSet, Job, CronJob |
| 004 | [Services and Networking](004_services_and_networking.md) | Service types, kube-proxy, DNS, CNI, NetworkPolicy |
| 005 | [Ingress and Gateway API](005_ingress_and_gateway_api.md) | HTTP routing into the cluster |
| 006 | [ConfigMaps and Secrets](006_configmaps_and_secrets.md) | Configuration and sensitive data |
| 007 | [Storage](007_storage.md) | Volumes, PV, PVC, StorageClass |
| 008 | [Resources and Scheduling](008_resources_and_scheduling.md) | Requests/limits, QoS, affinity, taints, priority |
| 009 | [Autoscaling](009_autoscaling.md) | HPA, VPA, Cluster Autoscaler, Karpenter, KEDA |
| 010 | [Health Probes](010_health_probes.md) | Liveness, readiness, startup probes |
| 011 | [Deployment Strategies](011_deployment_strategies.md) | Rolling, recreate, blue-green, canary, PDB |
| 012 | [RBAC and Security](012_rbac_and_security.md) | ServiceAccounts, RBAC, Pod Security, image security |
| 013 | [Helm and Kustomize](013_helm_and_kustomize.md) | Packaging and templating manifests |
| 014 | [Observability](014_observability.md) | Logs, metrics, tracing, events |
| 015 | [Troubleshooting and kubectl](015_troubleshooting_and_kubectl.md) | Common failures and a kubectl cheat sheet |

## Core Objects at a Glance

| Object | Purpose |
|---|---|
| Pod | Smallest deployable unit; one or more containers |
| Deployment | Stateless app, rolling updates |
| StatefulSet | Stateful app with stable identity and storage |
| Service | Stable virtual IP / DNS name for a set of pods |
| Ingress / Gateway | HTTP(S) entry into the cluster |
| ConfigMap / Secret | Configuration and credentials |
| PVC | Request for persistent storage |
| Namespace | Logical isolation and scoping |

## Rule of Thumb
Describe what you want declaratively, keep manifests in Git, and let controllers do the work.
