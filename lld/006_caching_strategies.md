# Caching Strategies

## Why Cache?
DB queries take 1-100ms. Cache hits take microseconds.
Caching reduces latency, DB load, and cost at scale.

## Cache Placement

### 1. Client-Side Cache
Browser cache, mobile app cache. No server involvement.
```
Browser: Cache-Control: max-age=3600
```

### 2. CDN Cache
Edge servers cache static assets close to users.
```
User (NYC) → CDN edge (NYC) → cache hit → no origin request
```

### 3. Application Cache (most common)
In-process (dict/HashMap) or external (Redis/Memcached).
```python
import redis
r = redis.Redis()

def get_user(user_id: str):
    cached = r.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)          # cache hit
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    r.setex(f"user:{user_id}", 3600, json.dumps(user))  # cache + TTL
    return user
```

### 4. Database Cache (Query Cache)
DB-level cache (e.g. MySQL query cache). Generally avoided — use app-level instead.

## Cache Write Strategies

### Write-Through
Write to cache AND DB synchronously. Cache always consistent.
```
Write → Cache → DB (both updated together)
Read  → Cache (always fresh)
```
- Pro: no stale data
- Con: write latency (waits for both)
- Use: read-heavy, consistency critical

### Write-Behind (Write-Back)
Write to cache immediately, DB updated asynchronously.
```
Write → Cache (immediate) → queue → DB (async)
```
- Pro: low write latency
- Con: risk of data loss if cache dies before DB write
- Use: high write throughput, can tolerate brief inconsistency

### Write-Around
Write directly to DB, skip cache. Cache populated on next read.
```
Write → DB only
Read  → cache miss → DB → populate cache
```
- Pro: cache not polluted with rarely-read data
- Con: first read after write is always a cache miss
- Use: write-once read-rarely data (logs, archives)

## Cache Eviction Policies

| Policy | How | Best For |
|---|---|---|
| **LRU** (Least Recently Used) | Evict least recently accessed | General purpose |
| **LFU** (Least Frequently Used) | Evict least accessed overall | Skewed access patterns |
| **TTL** (Time To Live) | Expire after fixed time | Time-sensitive data |
| **FIFO** | Evict oldest entry | Simple queues |

## Cache Invalidation Strategies

### TTL-based
```python
r.setex("user:123", ttl=300, value=data)  # auto-expires in 5 min
```
Simple, but stale window = TTL duration.

### Event-based (Cache Aside)
Invalidate on write:
```python
def update_user(user_id, data):
    db.update(user_id, data)
    r.delete(f"user:{user_id}")  # invalidate — next read repopulates
```

### Versioned Keys
```python
# Bump version on schema change instead of invalidating
key = f"user:{user_id}:v3"
```

## Cache Stampede (Thundering Herd)
When a popular key expires, thousands of requests hit DB simultaneously.

**Fix: Probabilistic Early Expiry**
```python
import random, math

def get_with_early_expiry(key, ttl, beta=1.0):
    value, expiry = cache.get_with_expiry(key)
    remaining = expiry - time.time()
    if -beta * math.log(random.random()) > remaining:
        # Recompute slightly before expiry to avoid stampede
        value = recompute()
        cache.set(key, value, ttl)
    return value
```

Or use a distributed lock:
```python
if not r.set("lock:user:123", "1", nx=True, ex=5):
    time.sleep(0.1)
    return r.get("user:123")  # wait and retry
```

## Redis vs Memcached

| | Redis | Memcached |
|---|---|---|
| Data structures | Strings, hashes, lists, sets, sorted sets | Strings only |
| Persistence | Optional (RDB/AOF) | None |
| Replication | Yes | No |
| Pub/Sub | Yes | No |
| Multi-threading | Single (v6+ multi I/O) | Multi-threaded |
| Use when | Need rich structures, persistence, pub/sub | Pure cache, max throughput |

## Key Rule
Cache what is read often and changes rarely.
Always set a TTL — never cache indefinitely without an expiry.
