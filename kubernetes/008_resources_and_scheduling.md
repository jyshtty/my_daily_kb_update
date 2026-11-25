# Resources and Scheduling

## Requests vs Limits
```yaml
resources:
  requests: { cpu: 250m, memory: 256Mi }   # guaranteed; used by the scheduler
  limits:   { cpu: "1",  memory: 512Mi }   # hard cap enforced on the node
```

| | Requests | Limits |
|---|---|---|
| Used by | **Scheduler** (does the node have room?) | **kubelet / kernel** (cgroups) |
| CPU over limit | — | **Throttled** (slowed, not killed) |
| Memory over limit | — | **OOMKilled** (exit code 137) |

Units: CPU `1` = 1 vCPU, `500m` = half; memory `Mi`, `Gi`.
Common practice: set memory request = limit; set CPU requests, be careful with CPU limits (throttling hurts latency).

## QoS Classes

| Class | Rule | Eviction order under node pressure |
|---|---|---|
| **Guaranteed** | Every container: requests == limits for CPU and memory | Last |
| **Burstable** | At least one request/limit set, not Guaranteed | Middle |
| **BestEffort** | No requests or limits | **First** |

## OOMKilled vs Eviction
- **OOMKilled**: container exceeded its own memory limit → kernel kills it → restarts
- **Eviction**: node is low on memory/disk → kubelet evicts pods (BestEffort first, then Burstable using more than requested)

## Namespace Guardrails
- **ResourceQuota**: total CPU/memory/object counts per namespace
- **LimitRange**: default and min/max requests/limits per container

## How the Scheduler Works
```
Pending pod → Filter (which nodes can run it?) → Score (which is best?) → Bind
             resources, taints, affinity,        spread, balance,
             volume zone, ports                  image locality
```

## Steering Pods

### nodeSelector (simple)
```yaml
nodeSelector: { disktype: ssd }
```

### Node Affinity (expressive)
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:        # hard
      nodeSelectorTerms:
        - matchExpressions: [{ key: topology.kubernetes.io/zone, operator: In, values: [us-east-1a, us-east-1b] }]
    preferredDuringSchedulingIgnoredDuringExecution:       # soft
      - weight: 50
        preference: { matchExpressions: [{ key: instance-type, operator: In, values: [m6i.large] }] }
```

### Pod Affinity / Anti-Affinity
```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector: { matchLabels: { app: api } }
        topologyKey: kubernetes.io/hostname    # no two api pods on one node
```
Affinity: co-locate (app near its cache). Anti-affinity: spread replicas for HA.

### Taints and Tolerations
Taints **repel** pods from nodes; tolerations **allow** specific pods onto them.
```bash
kubectl taint nodes gpu-1 gpu=true:NoSchedule
```
```yaml
tolerations:
  - { key: gpu, operator: Equal, value: "true", effect: NoSchedule }
```
Effects: `NoSchedule`, `PreferNoSchedule`, `NoExecute` (also evicts running pods).
A toleration only *permits*; combine with affinity to *require* the node.

### Topology Spread Constraints
```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector: { matchLabels: { app: api } }
```
Evenly spreads replicas across zones/nodes — preferred over anti-affinity at scale.

### Priority and Preemption
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata: { name: critical }
value: 100000
```
If a high-priority pod can't fit, the scheduler **evicts lower-priority pods** to make room.

## Pitfalls

| Pitfall | Fix |
|---|---|
| No requests → overpacked nodes, noisy neighbours | Always set requests (enforce with LimitRange) |
| Requests far above usage → wasted money | Right-size with VPA recommendations / metrics |
| Tight CPU limits → latency spikes | Monitor throttling; relax or remove CPU limits |
| Pods `Pending`: "Insufficient cpu" | Lower requests or add nodes (autoscaler) |
| All replicas on one node/zone | Anti-affinity / topology spread |

## Interview Quick Answer
> Requests are what the scheduler reserves; limits are hard caps — CPU over limit is throttled, memory over limit is OOMKilled. Requests and limits define QoS classes, which decide eviction order. I steer placement with node affinity, spread replicas with anti-affinity or topology spread constraints, dedicate nodes with taints and tolerations, and protect critical workloads with PriorityClasses.

## Rule of Thumb
Always set requests, set memory limits, spread replicas across zones, and use taints to reserve special nodes.
