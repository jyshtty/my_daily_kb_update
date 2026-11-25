# Health Probes

## The Three Probes

| Probe | Question | On failure |
|---|---|---|
| **Liveness** | Is the container stuck/dead? | kubelet **restarts** the container |
| **Readiness** | Can it serve traffic right now? | Pod **removed from Service endpoints** (not restarted) |
| **Startup** | Has it finished starting? | Liveness/readiness are paused until it passes; kills container if it never passes |

```
container start ──→ [startup probe] ──pass──→ [liveness + readiness run continuously]
                         │ fails beyond threshold → restart
```

## Probe Mechanisms

| Type | Example |
|---|---|
| `httpGet` | `GET /healthz` returns 200–399 |
| `tcpSocket` | Port accepts a connection |
| `exec` | Command exits 0 |
| `grpc` | gRPC health checking protocol |

## Example
```yaml
containers:
  - name: api
    image: myorg/api:1.4.2
    ports: [{ containerPort: 8080 }]
    startupProbe:
      httpGet: { path: /healthz, port: 8080 }
      periodSeconds: 5
      failureThreshold: 30          # up to 150 s to start
    livenessProbe:
      httpGet: { path: /healthz, port: 8080 }
      periodSeconds: 10
      timeoutSeconds: 2
      failureThreshold: 3
    readinessProbe:
      httpGet: { path: /ready, port: 8080 }
      periodSeconds: 5
      failureThreshold: 2
```

## Key Fields

| Field | Default | Meaning |
|---|---|---|
| `initialDelaySeconds` | 0 | Wait before first probe (prefer a startup probe) |
| `periodSeconds` | 10 | How often to probe |
| `timeoutSeconds` | 1 | Probe timeout |
| `failureThreshold` | 3 | Failures before acting |
| `successThreshold` | 1 | Successes to be considered OK (readiness can be >1) |

## What Each Endpoint Should Check

| Endpoint | Check | Don't check |
|---|---|---|
| **Liveness** `/healthz` | Process is responsive (event loop not deadlocked) | Database, downstream services |
| **Readiness** `/ready` | Warmed up, has config, can reach **its own critical** dependencies, not overloaded | Things that would make **all** pods unready together |

## Graceful Shutdown and Readiness
On termination the pod is removed from endpoints, but propagation takes a moment:
```yaml
lifecycle:
  preStop:
    exec: { command: ["sleep", "5"] }   # let endpoints/LBs stop routing first
```
Then the app handles SIGTERM, finishes in-flight requests and exits.

## Pitfalls

| Pitfall | Consequence | Fix |
|---|---|---|
| Liveness checks the database | DB blip → **every pod restarts** → outage gets worse | Liveness = process health only |
| Same endpoint for liveness and readiness | Overloaded pod gets killed instead of shed | Separate endpoints |
| Liveness too aggressive on slow start | `CrashLoopBackOff` | Add startup probe |
| Timeout 1 s under load / GC pauses | False restarts | Increase timeout / threshold |
| No readiness probe | Traffic hits pods still starting → errors during rollout | Always add readiness |
| Heavy health checks | Wasted CPU | Keep them cheap |

## Interview Quick Answer
> Liveness tells kubelet when to restart a stuck container, readiness controls whether a pod receives Service traffic, and startup protects slow-starting apps by delaying the other two. Liveness should only check the process itself — never shared dependencies — or one database blip can restart the whole fleet. Readiness gates rollouts and, with a preStop delay, enables graceful shutdown.

## Rule of Thumb
Always use readiness, keep liveness simple and lenient, and add a startup probe for slow starters.
