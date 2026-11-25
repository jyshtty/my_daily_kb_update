# Pods

## What It Is
A **Pod** is the smallest deployable unit: one or more containers that share
- a **network namespace** (same IP, talk over `localhost`)
- **volumes**
- lifecycle and scheduling (always on the same node)

```
┌──────────── Pod (IP 10.1.2.3) ────────────┐
│  app container :8080   sidecar :9090      │
│        └──── shared volume /data ───┘      │
└────────────────────────────────────────────┘
```

Pods are **ephemeral** — never patch them by hand; use a controller (Deployment, StatefulSet…).

## Minimal Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  labels: { app: web }
spec:
  containers:
    - name: web
      image: nginx:1.27
      ports: [{ containerPort: 80 }]
```

## Pod Phases

| Phase | Meaning |
|---|---|
| **Pending** | Accepted, not yet running (scheduling, image pull) |
| **Running** | Bound to a node, at least one container running |
| **Succeeded** | All containers exited 0, won't restart |
| **Failed** | All containers terminated, at least one failed |
| **Unknown** | Node unreachable |

Container states: `Waiting` (e.g. `ImagePullBackOff`, `CrashLoopBackOff`), `Running`, `Terminated`.
`CrashLoopBackOff` is a *reason*, not a phase.

## Restart Policy

| Policy | Behaviour | Used by |
|---|---|---|
| `Always` (default) | Restart on any exit | Deployments |
| `OnFailure` | Restart only on non-zero exit | Jobs |
| `Never` | Never restart | One-shot jobs |

Restarts use exponential backoff (10 s, 20 s, 40 s … capped at 5 min).

## Init Containers
Run **sequentially, to completion**, before app containers start.
```yaml
spec:
  initContainers:
    - name: wait-for-db
      image: busybox
      command: ["sh", "-c", "until nc -z db 5432; do sleep 2; done"]
  containers:
    - name: app
      image: myapp:1.0
```
Use for: migrations, waiting for dependencies, fetching config, setting permissions.

## Multi-Container Patterns

| Pattern | Example |
|---|---|
| **Sidecar** | Log shipper, service-mesh proxy (Envoy), config reloader |
| **Ambassador** | Local proxy to an external service |
| **Adapter** | Convert app metrics to Prometheus format |

## Native Sidecars (K8s 1.29+ beta, GA 1.33)
An init container with `restartPolicy: Always` becomes a **sidecar**: starts before app containers, keeps running, and stops **after** them. Fixes old problems (Jobs never finishing because the sidecar kept running, sidecar starting too late).
```yaml
initContainers:
  - name: log-agent
    image: fluent-bit
    restartPolicy: Always
```

## Termination Flow
1. Pod marked Terminating; removed from Service endpoints
2. `preStop` hook runs (if any)
3. `SIGTERM` sent to containers
4. Wait `terminationGracePeriodSeconds` (default 30)
5. `SIGKILL` if still running

Handle SIGTERM to drain in-flight requests gracefully.

## Pitfalls

| Pitfall | Fix |
|---|---|
| Creating bare pods | Use Deployments/Jobs so they get recreated |
| App ignores SIGTERM | Handle it; add a short `preStop` sleep for endpoint propagation |
| Using `latest` image tag | Pin versions or digests |
| Too many containers in one pod | Only co-locate tightly coupled processes |

## Interview Quick Answer
> A pod is one or more containers sharing a network namespace and volumes, scheduled together on one node. It moves through Pending, Running, Succeeded/Failed. Init containers run to completion before the app, and native sidecars (init containers with `restartPolicy: Always`) run alongside it. Pods are disposable, so they're always managed by a controller.

## Rule of Thumb
One main process per pod, sidecars only for helpers, and always run pods through a controller.
