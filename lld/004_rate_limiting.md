# Rate Limiting

## What is Rate Limiting?
Controlling how many requests a client can make in a given time window.
Protects services from abuse, DoS attacks, and resource exhaustion.

## Algorithms

### 1. Token Bucket (Most Common)
Bucket holds N tokens. Each request consumes 1 token. Tokens refill at a fixed rate.
Allows short bursts up to bucket capacity.

```
Bucket capacity: 10 tokens
Refill rate: 2 tokens/sec

t=0s: bucket=10, request → bucket=9  ✓
t=0s: 5 rapid requests → bucket=4   ✓ (burst allowed)
t=0s: 5 more requests → bucket=-1   ✗ (rejected)
t=1s: bucket=6 (refilled 2)
```

```python
import time

class TokenBucket:
    def __init__(self, capacity: int, refill_rate: float):
        self.capacity = capacity
        self.tokens = capacity
        self.refill_rate = refill_rate  # tokens per second
        self.last_refill = time.time()

    def allow(self) -> bool:
        now = time.time()
        elapsed = now - self.last_refill
        self.tokens = min(self.capacity, self.tokens + elapsed * self.refill_rate)
        self.last_refill = now
        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False
```

### 2. Sliding Window Counter
Tracks request count in a rolling time window — more accurate than fixed window.

```python
import redis
from datetime import datetime

def is_allowed(user_id: str, limit: int, window_sec: int) -> bool:
    r = redis.Redis()
    now = datetime.utcnow().timestamp()
    key = f"ratelimit:{user_id}"
    pipe = r.pipeline()
    pipe.zremrangebyscore(key, 0, now - window_sec)   # remove old entries
    pipe.zadd(key, {str(now): now})                    # add current request
    pipe.zcard(key)                                    # count in window
    pipe.expire(key, window_sec)
    results = pipe.execute()
    return results[2] <= limit
```

### 3. Fixed Window Counter
Simplest — count resets every N seconds. Can allow 2x burst at window boundary.

```python
def is_allowed(user_id: str, limit: int, window_sec: int) -> bool:
    r = redis.Redis()
    key = f"ratelimit:{user_id}:{int(time.time() // window_sec)}"
    count = r.incr(key)
    if count == 1:
        r.expire(key, window_sec)
    return count <= limit
```

### 4. Leaky Bucket
Requests processed at a fixed rate — excess queued or dropped. Smooths out bursts.
Good for: output rate limiting (e.g. API calls to a downstream service).

## Algorithm Comparison

| Algorithm | Burst Handling | Accuracy | Complexity |
|---|---|---|---|
| Token Bucket | Allows bursts | Good | Medium |
| Sliding Window | Smooth | Best | Medium |
| Fixed Window | 2x burst at boundary | Moderate | Low |
| Leaky Bucket | No burst (smoothed) | Good | Medium |

## Where to Apply Rate Limits

| Layer | Tool | Granularity |
|---|---|---|
| API Gateway | Kong, AWS API GW | Per API key / IP |
| Nginx | limit_req_zone | Per IP |
| Application | Redis + middleware | Per user / endpoint |
| Service mesh | Istio/Envoy | Per service |

## Headers to Return
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 43
X-RateLimit-Reset: 1719000000
Retry-After: 30   (on 429)
```

## HTTP Status
- `429 Too Many Requests` — standard rate limit response

## Key Rule
Rate limit at the edge (API Gateway) for global protection.
Rate limit in the app for per-user/per-feature granularity.
