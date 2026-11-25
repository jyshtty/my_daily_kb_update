# Deployment Strategies

## Overview

| Strategy | Downtime | Extra capacity | Rollback speed | Native in K8s? |
|---|---|---|---|---|
| **Rolling update** | None | Small (`maxSurge`) | Minutes | Yes (default) |
| **Recreate** | Yes | None | Minutes | Yes |
| **Blue-green** | None | 2× | Instant (switch back) | Via Service selector / tools |
| **Canary** | None | Small | Fast | Via Ingress/Gateway weights, Argo Rollouts, Flagger, mesh |

## Rolling Update (default)
```yaml
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%          # extra pods allowed above replicas (→ up to 13)
      maxUnavailable: 0      # never go below 10 ready pods
  minReadySeconds: 10        # pod must be ready this long to count
```
```
v1 v1 v1 v1  →  v1 v1 v1 v1 v2  →  v1 v1 v1 v2 v2  → ... →  v2 v2 v2 v2
```
- Relies on **readiness probes** to know a new pod is healthy
- Both versions serve traffic during the rollout → must be **backward compatible** (APIs, DB schema)
- `maxUnavailable: 0, maxSurge: 1` = safest; higher values = faster

## Recreate
```yaml
strategy: { type: Recreate }
```
Kill all old pods, then start new ones. Use when two versions can't run together, or with RWO volumes.

## Blue-Green
```
Service selector: version=blue  →  [blue pods v1]   (live)
                                    [green pods v2]  (deployed, tested)
switch selector to version=green → instant cutover; switch back to roll back
```
Pros: instant switch and rollback, test prod-like env. Cons: double resources, DB migrations still shared.

## Canary
Send a small % of traffic to the new version, watch metrics, then increase.
```
5% → 25% → 50% → 100%      (abort and roll back if error rate / latency regress)
```
Options:
- **Gateway API** `HTTPRoute` weighted `backendRefs`
- **Argo Rollouts** / **Flagger**: automated steps + analysis against Prometheus metrics
- **Service mesh** (Istio, Linkerd): precise % and header-based routing
- Crude: replica ratio under one Service (9 v1 + 1 v2 ≈ 10%)

```yaml
# Argo Rollouts excerpt
strategy:
  canary:
    steps:
      - setWeight: 10
      - pause: { duration: 10m }
      - analysis: { templates: [{ templateName: error-rate }] }
      - setWeight: 50
      - pause: { duration: 10m }
```

## Rollback
```bash
kubectl rollout status deploy/api
kubectl rollout history deploy/api
kubectl rollout undo deploy/api                 # previous revision
kubectl rollout undo deploy/api --to-revision=3
kubectl rollout pause|resume deploy/api
```
`progressDeadlineSeconds` (default 600) marks a stuck rollout as failed — it does **not** auto-roll back; use CI/CD or Argo Rollouts for that.
In GitOps (Argo CD / Flux), roll back by reverting the Git commit.

## PodDisruptionBudget (PDB)
Limits **voluntary** disruptions (node drains, upgrades, autoscaler scale-down) — not crashes.
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: api }
spec:
  minAvailable: 2           # or maxUnavailable: 1
  selector: { matchLabels: { app: api } }
```
Note: rolling updates of Deployments are governed by `maxUnavailable`, not PDBs.

## Database Migrations (expand–contract)
1. **Expand**: add new column/table (old code ignores it)
2. Deploy code that writes both / reads new
3. Backfill data
4. **Contract**: remove old column after all pods use the new one

## Pitfalls

| Pitfall | Fix |
|---|---|
| No readiness probe → broken pods get traffic | Always add readiness |
| Breaking API/schema change during rolling update | Expand–contract, versioned APIs |
| PDB `minAvailable` = replicas | Blocks all drains; leave headroom |
| Canary without metrics | Automate analysis (error rate, p99 latency) |
| `latest` tag → no real rollout | Immutable tags/digests |

## Interview Quick Answer
> Kubernetes Deployments do rolling updates controlled by `maxSurge` and `maxUnavailable`, gated by readiness probes, with `kubectl rollout undo` for rollback. Recreate is for versions that can't coexist. Blue-green runs two full environments and switches the Service selector; canary shifts a small percentage of traffic and promotes based on metrics, usually with Argo Rollouts, Flagger or Gateway API weights. PDBs protect availability during voluntary disruptions like node drains.

## Rule of Thumb
Rolling updates for most services, automated canaries for risky changes, and always backward-compatible releases.
