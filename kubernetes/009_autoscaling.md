# Autoscaling

## Three Dimensions
```
More pods      → HPA / KEDA       (horizontal)
Bigger pods    → VPA              (vertical)
More nodes     → Cluster Autoscaler / Karpenter
```
Typical flow: load ↑ → HPA adds pods → pods `Pending` (no room) → node autoscaler adds nodes.

## Horizontal Pod Autoscaler (HPA)
Scales replicas based on metrics. Needs **metrics-server** (CPU/memory) or a metrics adapter (Prometheus Adapter) for custom metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: api }
  minReplicas: 3
  maxReplicas: 30
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300      # avoid flapping
```

Formula:
```
desiredReplicas = ceil(currentReplicas × currentMetric / targetMetric)
e.g. 4 pods at 90% CPU, target 60% → ceil(4 × 90/60) = 6
```
- Utilization is relative to **requests** → requests must be set
- Metric types: Resource, Pods (custom per pod), Object, External

## Vertical Pod Autoscaler (VPA)
Adjusts container **requests/limits** based on observed usage.

| Mode | Behaviour |
|---|---|
| `Off` | Recommendations only (great for right-sizing) |
| `Initial` | Apply at pod creation only |
| `Recreate` / `Auto` | Evict pods to apply new values |
| `InPlaceOrRecreate` | Uses in-place pod resize where supported (newer K8s) |

Don't use VPA and HPA on the **same CPU/memory metric** — they fight. Combine VPA with HPA on custom metrics, or VPA in `Off` mode.

## Cluster Autoscaler (CA)
- Watches for **unschedulable pods** → adds nodes to a node group (ASG / MIG / VMSS)
- Removes nodes that are underutilized and whose pods can move elsewhere
- Works with predefined node groups (instance types fixed per group)
- Respects PDBs and `cluster-autoscaler.kubernetes.io/safe-to-evict` annotations

## Karpenter
Node provisioner (AWS-first, also Azure) that launches **right-sized nodes directly**, without node groups.

| | Cluster Autoscaler | Karpenter |
|---|---|---|
| Model | Scales existing node groups | Picks instance types per pending pods |
| Speed | Slower (ASG) | Faster, direct API |
| Bin packing / consolidation | Basic scale-down | Active consolidation, replaces with cheaper nodes |
| Spot handling | Via node groups | Native, diversified instance types |

## KEDA (Event-Driven Autoscaling)
Scales on **event sources** — Kafka lag, SQS depth, RabbitMQ queue length, Prometheus queries, cron — and can scale **to zero**.
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: { name: worker }
spec:
  scaleTargetRef: { name: worker }
  minReplicaCount: 0
  maxReplicaCount: 50
  triggers:
    - type: aws-sqs-queue
      metadata: { queueURL: https://sqs..., queueLength: "20", awsRegion: us-east-1 }
```
KEDA creates and manages an HPA under the hood.

## Choosing

| Workload | Scaler |
|---|---|
| Web/API, CPU-bound | HPA on CPU (or RPS via custom metric) |
| Queue workers | KEDA on queue depth / lag |
| Unknown sizing | VPA in `Off` mode for recommendations |
| Nodes | Karpenter (AWS) or Cluster Autoscaler |
| GPU / LLM inference | Custom metrics: queue length, concurrency, KV-cache use |

## Pitfalls

| Pitfall | Fix |
|---|---|
| HPA shows `<unknown>` | Install metrics-server; set requests |
| Flapping | `behavior` stabilization windows |
| Scale-up too slow for spikes | Over-provision buffer (pause pods), faster metrics, pre-scale |
| Node scale-down blocked | PDBs too strict, local storage, `safe-to-evict: false` |
| HPA + VPA on same metric | Separate the metrics |
| Cold start (image pull, JVM) | Smaller images, readiness probes, minReplicas > 0 |

## Interview Quick Answer
> HPA scales pod replicas using CPU, memory or custom metrics relative to requests; VPA right-sizes requests; Cluster Autoscaler or Karpenter adds nodes when pods can't be scheduled and removes idle ones. For queue-driven workloads I use KEDA, which scales on queue depth or Kafka lag and can scale to zero. I add stabilization windows to avoid flapping and keep HPA and VPA off the same metric.

## Rule of Thumb
HPA for pods, Karpenter or Cluster Autoscaler for nodes, KEDA for queues, and VPA mainly for sizing recommendations.
