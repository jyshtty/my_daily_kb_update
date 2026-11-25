# Workloads

## Controller Hierarchy
```
Deployment ──manages──→ ReplicaSet ──manages──→ Pods
StatefulSet ─────────────────────────────────→ Pods (ordered, stable IDs)
DaemonSet ───────────────────────────────────→ 1 Pod per node
CronJob ──→ Job ─────────────────────────────→ Pods (run to completion)
```

## Comparison

| Workload | Use for | Pod identity | Scaling | Storage |
|---|---|---|---|---|
| **ReplicaSet** | Keep N identical pods running (rarely used directly) | Random | Any order | Shared/none |
| **Deployment** | Stateless apps: APIs, web | Random names | Any order | Usually none |
| **StatefulSet** | Databases, Kafka, ZooKeeper | Stable: `db-0`, `db-1` | Ordered | One PVC per pod |
| **DaemonSet** | Node agents: logging, monitoring, CNI | One per node | Follows nodes | Host paths |
| **Job** | Batch task that must finish | Random | `parallelism` / `completions` | Optional |
| **CronJob** | Scheduled jobs | Random | Per schedule | Optional |

## Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: api }
spec:
  replicas: 3
  selector: { matchLabels: { app: api } }
  template:
    metadata: { labels: { app: api } }
    spec:
      containers:
        - name: api
          image: myorg/api:1.4.2
```
- Updating the pod template creates a new ReplicaSet; rolling update shifts pods over
- Keeps old ReplicaSets for rollback (`revisionHistoryLimit`)

## StatefulSet
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: db }
spec:
  serviceName: db            # headless service → db-0.db.ns.svc.cluster.local
  replicas: 3
  selector: { matchLabels: { app: db } }
  template: { ... }
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: [ReadWriteOnce]
        resources: { requests: { storage: 20Gi } }
```
Guarantees:
- Stable network identity (`db-0`, `db-1`…) via a **headless Service**
- Stable storage: `data-db-0` PVC follows `db-0` when rescheduled
- Ordered create/scale/terminate (`podManagementPolicy: OrderedReady`, or `Parallel`)
- PVCs are **not** deleted on scale-down by default

## DaemonSet
- Runs one pod on every (or selected) node; new nodes get one automatically
- Needs tolerations to run on tainted nodes (e.g. control plane)
- Examples: Fluent Bit, node-exporter, Cilium, kube-proxy

## Job
```yaml
apiVersion: batch/v1
kind: Job
metadata: { name: report }
spec:
  completions: 5
  parallelism: 2
  backoffLimit: 4
  activeDeadlineSeconds: 600
  ttlSecondsAfterFinished: 3600
  template:
    spec:
      restartPolicy: OnFailure
      containers: [{ name: report, image: myorg/report:1.0 }]
```

## CronJob
```yaml
apiVersion: batch/v1
kind: CronJob
metadata: { name: nightly }
spec:
  schedule: "0 2 * * *"
  timeZone: "Etc/UTC"
  concurrencyPolicy: Forbid      # Allow | Forbid | Replace
  startingDeadlineSeconds: 300
  jobTemplate: { spec: { template: { spec: { restartPolicy: OnFailure, containers: [...] } } } }
```
Jobs may occasionally run twice or be missed — make them **idempotent**.

## Pitfalls

| Pitfall | Fix |
|---|---|
| Databases on Deployments | Use StatefulSet (or a managed DB / operator) |
| Selector doesn't match template labels | They must match; selector is immutable |
| Finished Jobs piling up | `ttlSecondsAfterFinished`, history limits |
| CronJob overlap | `concurrencyPolicy: Forbid` |
| Assuming StatefulSet = replication | It only gives identity/storage; the app handles replication |

## Interview Quick Answer
> Deployments run stateless apps via ReplicaSets with rolling updates. StatefulSets give pods stable names, ordered rollout and a PVC per pod for databases and brokers. DaemonSets run one pod per node for agents. Jobs run to completion, and CronJobs run Jobs on a schedule.

## Rule of Thumb
Stateless → Deployment, stateful → StatefulSet (or an operator), per-node → DaemonSet, finite → Job/CronJob.
