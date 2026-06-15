# High Throughput System Design

## What is Throughput?
Requests (or messages) processed per second. High throughput = handle millions of ops/sec.

## Core Strategies

### 1. Horizontal Scaling
Add more machines instead of bigger ones.
```
Load Balancer → [Server 1] [Server 2] [Server 3] ...
```
- Stateless services scale easily — store session in Redis, not in-process
- Use consistent hashing to distribute load evenly

### 2. Async Processing (Decouple with Queues)
Never make users wait for slow operations.
```
Request → API → Queue (Kafka/RabbitMQ) → Worker → DB
           ↓
        200 OK (immediately)
```
- Heavy tasks (email, image resize, ML inference) go to queue
- Workers process at their own pace

### 3. Caching
Stop hitting the DB for the same data repeatedly.
```
Request → Cache hit? → return (microseconds)
               ↓ miss
           DB query → store in cache → return
```
- **L1**: in-process (HashMap) — fastest, not shared
- **L2**: Redis/Memcached — shared across instances, milliseconds
- Cache invalidation strategies: TTL, write-through, write-behind

### 4. Database Optimizations
- **Read replicas**: route reads to replicas, writes to primary
- **Sharding**: split data across multiple DBs by key (user_id % N)
- **Connection pooling**: reuse DB connections (PgBouncer)
- **Denormalization**: trade storage for query speed

### 5. Batching
Group small operations into larger ones.
```python
# Instead of 1000 individual inserts
db.insert_many(records)  # one round trip

# Batch Kafka messages instead of one-by-one
producer.send_batch(messages)
```

### 6. Non-blocking I/O
- Use async frameworks (FastAPI, Node.js, Netty)
- A single thread handles thousands of concurrent connections
- Never block a thread on network/DB I/O

## Throughput Killers to Avoid
| Anti-pattern | Problem | Fix |
|---|---|---|
| Synchronous external calls | Blocks thread | Async + queue |
| N+1 queries | DB overloaded | Batch/join |
| No connection pooling | New connection per request | PgBouncer/HikariCP |
| Single DB node for all reads | Bottleneck | Read replicas |
| Large JSON payloads | Network/parse overhead | Protobuf/compression |

## Rule of Thumb
Measure first. Profile before optimizing. The bottleneck is almost always I/O, not CPU.
