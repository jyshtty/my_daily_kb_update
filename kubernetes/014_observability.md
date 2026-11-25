# Observability

## Three Pillars (+ Events)

| Signal | Answers | Typical stack |
|---|---|---|
| **Metrics** | Is something wrong? How much? | Prometheus + Grafana, Datadog, CloudWatch |
| **Logs** | What exactly happened? | Fluent Bit / Vector → Loki, Elasticsearch/OpenSearch, CloudWatch |
| **Traces** | Where in the request path is the time spent? | OpenTelemetry → Tempo, Jaeger, X-Ray |
| **K8s events** | What did the cluster do? (scheduling, kills, pulls) | `kubectl get events`, event exporters |

## Logging
```
container stdout/stderr → node /var/log/containers/*.log → DaemonSet agent (Fluent Bit) → backend
```
- Apps should log to **stdout/stderr** in **structured JSON**
- `kubectl logs` only shows the current/previous container on that node — logs vanish with the pod, so ship them centrally
- Add metadata: namespace, pod, labels, trace ID

```bash
kubectl logs deploy/api -c api --since=10m -f
kubectl logs pod/api-xyz --previous          # logs from the crashed container
```

## Metrics

### metrics-server
Lightweight, in-memory CPU/memory metrics for `kubectl top` and HPA. **Not** for dashboards or history.
```bash
kubectl top nodes
kubectl top pods -n shop --sort-by=memory
```

### Prometheus Stack (kube-prometheus-stack)
```
Prometheus ──scrapes──→ /metrics endpoints (apps, kubelet/cAdvisor, kube-state-metrics, node-exporter)
     │
     ├─→ Grafana (dashboards)
     └─→ Alertmanager (Slack, PagerDuty)
```

| Source | Provides |
|---|---|
| **cAdvisor** (in kubelet) | Container CPU, memory, network |
| **kube-state-metrics** | Object state: replicas, pod phase, restarts, PVC status |
| **node-exporter** | Node CPU, disk, filesystem |
| App `/metrics` | Business and RED metrics |

Discovery via `ServiceMonitor` / `PodMonitor` CRDs (Prometheus Operator).
At scale: Thanos, Cortex, Mimir or managed Prometheus for long-term storage.

### What to Monitor
| Method | Metrics |
|---|---|
| **RED** (services) | Rate, Errors, Duration |
| **USE** (resources) | Utilization, Saturation, Errors |
| **Golden signals** | Latency, traffic, errors, saturation |

Useful alerts:
```promql
# Pods restarting
increase(kube_pod_container_status_restarts_total[15m]) > 3
# CPU throttling
rate(container_cpu_cfs_throttled_periods_total[5m]) / rate(container_cpu_cfs_periods_total[5m]) > 0.25
# Deployment not at desired replicas
kube_deployment_status_replicas_available < kube_deployment_spec_replicas
```

## Tracing
- Instrument with **OpenTelemetry** SDKs (or auto-instrumentation via the OTel Operator)
- Run an **OTel Collector** (DaemonSet or Deployment) to receive, sample and export
- Propagate `traceparent` headers across services; put trace IDs in logs to correlate
- Service meshes can emit spans for every hop

## Events
```bash
kubectl get events -n shop --sort-by=.lastTimestamp
kubectl describe pod api-xyz          # Events section at the bottom
```
Events expire (default ~1 h) — export them (e.g. kubernetes-event-exporter) if you need history.

## SLOs
Define SLIs (e.g. % of requests < 300 ms and non-5xx) and alert on **error-budget burn rate**, not on every CPU spike.

## Pitfalls

| Pitfall | Fix |
|---|---|
| Relying on `kubectl logs` | Central log shipping |
| High-cardinality labels (user ID) in metrics | Keep labels bounded; use logs/traces for detail |
| Alert fatigue | Alert on symptoms/SLOs, not causes |
| No correlation between signals | Shared trace IDs, consistent labels |
| Unbounded log volume cost | Sampling, retention tiers, drop debug logs in prod |

## Interview Quick Answer
> I collect metrics with Prometheus (cAdvisor, kube-state-metrics, node-exporter and app metrics) and visualize them in Grafana with Alertmanager for SLO-based alerts; metrics-server only feeds `kubectl top` and HPA. Logs go to stdout as JSON and are shipped by a Fluent Bit DaemonSet to Loki or OpenSearch. Traces use OpenTelemetry through a collector, with trace IDs in logs to correlate. Kubernetes events help explain scheduling and restart issues.

## Rule of Thumb
Structured logs to stdout, Prometheus for metrics, OpenTelemetry for traces, and alerts on user-facing symptoms.
