# Circuit Breaker and Resilience Patterns

## The Problem: Cascading Failure
One slow dependency can take down everything that calls it.

```
Client → Service A → Service B (slow, 30 s)
                        │
        A's threads pile up waiting → A exhausts its pool → A fails
        → everything calling A fails too
```
Resilience patterns **contain** failures so they don't spread.

## Circuit Breaker
Works like an electrical breaker: stop calling a failing dependency, give it time to recover.

```
        failures > threshold
 CLOSED ────────────────────→ OPEN
 (calls pass)                 (calls fail fast)
    ↑                             │ after cool-down timer
    │  trial calls succeed        ↓
    └──────────────────────── HALF-OPEN
         trial fails → back to OPEN   (limited trial calls)
```

| State | Behaviour |
|---|---|
| **Closed** | Normal; failures are counted in a sliding window |
| **Open** | Calls rejected immediately (no waiting) → return fallback |
| **Half-open** | Let a few probe calls through; success closes it, failure re-opens |

Typical config: open when failure rate > 50% over the last 20 calls (or slow-call rate high), wait 30 s, allow 3 probes.
Libraries: Resilience4j, Polly, Hystrix (legacy), Istio/Envoy outlier detection.

## Timeouts
- **Every** remote call needs a timeout — the default of "wait forever" is a bug
- Set below the caller's own deadline; propagate a **deadline** through the call chain
- Separate connect timeout and read timeout

## Retries
| Rule | Why |
|---|---|
| Retry only **transient** errors (timeouts, 503, 429) | Retrying 400/404 is pointless |
| Retry only **idempotent** operations (see `010_idempotency.md`) | Avoid double side effects |
| **Exponential backoff + jitter** | Spread retries out |
| Limit attempts and use a **retry budget** | Avoid amplification |

```
delay = random(0, min(cap, base * 2^attempt))      # "full jitter"
```

**Retry storm**: if 3 layers each retry 3×, one request becomes 27 against the failing service. Retry at one layer, and cap total retries.

## Bulkhead
Isolate resources so one failing part can't drain everything — like watertight ship compartments.
```
Thread pool / connection pool:
  Payments calls  → pool of 20
  Search calls    → pool of 20
  Recommendations → pool of 10     (if this hangs, the others still work)
```

## Fallbacks and Graceful Degradation
When a dependency fails, return something useful:
- Cached / stale data
- Default value (e.g. generic recommendations)
- Feature turned off (hide the widget)
- Queue the work for later

## Load Shedding and Back-pressure
- **Load shedding**: reject excess requests early (HTTP 503/429) to keep serving the rest
- **Back-pressure**: tell upstream to slow down (bounded queues, reactive streams, consumer lag signals)
- **Rate limiting**: see `004_rate_limiting.md`

## Health Checks
| Check | Purpose |
|---|---|
| **Liveness** | Is the process alive? Restart if not |
| **Readiness** | Can it take traffic now? Remove from LB if not |
| **Dependency/deep** | Used carefully — avoid turning one DB blip into a full outage |

## How They Fit Together
```
Request → timeout → retry (backoff+jitter) → circuit breaker → bulkhead → dependency
                                                   │
                                              fallback on failure
```

## Interview Quick Answer
> To stop cascading failures I set timeouts on every call, retry only idempotent transient errors with exponential backoff and jitter, wrap dependencies in a circuit breaker that fails fast when error rates rise, isolate resource pools with bulkheads, and provide fallbacks so the system degrades gracefully. I also shed load and apply back-pressure under overload.

## Rule of Thumb
Fail fast, retry carefully, isolate dependencies, and always have a fallback.
