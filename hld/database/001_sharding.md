# Database Sharding

## What is Sharding?
Splitting a large database horizontally across multiple machines (shards).
Each shard holds a subset of the data — no single machine holds everything.

```
Without sharding:    [All 1B users on one DB]  ← bottleneck

With sharding:       Shard 1: users 0-333M
                     Shard 2: users 333M-666M
                     Shard 3: users 666M-1B
```

## Sharding Strategies

### 1. Range-Based Sharding
Split by value range of a key.
```
user_id 1–1M       → Shard 1
user_id 1M–2M      → Shard 2
user_id 2M–3M      → Shard 3
```
- Simple to implement and reason about
- Risk of **hot spots** — new users always go to last shard

### 2. Hash-Based Sharding
```python
shard_id = hash(user_id) % num_shards
```
- Even distribution
- Adding shards remaps almost all keys (use consistent hashing to fix this)

### 3. Consistent Hash Sharding
See `005_consistent_hashing.md` — add/remove shards with minimal key remapping.
Used by: Cassandra, DynamoDB, Redis Cluster.

### 4. Directory-Based Sharding
A lookup table maps each key to its shard.
```
user_id → shard_id (stored in a routing DB)
```
- Flexible — easy to rebalance
- Routing DB becomes a bottleneck/single point of failure

### 5. Geographic Sharding
Route by user region.
```
EU users → EU shard (GDPR compliance)
US users → US shard
```

## Shard Key Selection — Most Important Decision

Bad shard key → hot spots, uneven distribution, cross-shard queries.

| Criteria | Good | Bad |
|---|---|---|
| Cardinality | High (many unique values) | Low (e.g. boolean) |
| Distribution | Even write/read spread | Monotonically increasing (timestamps) |
| Query pattern | Most queries use shard key | Queries frequently span shards |

**Avoid**: auto-increment IDs as shard keys — all writes go to the last shard.
**Prefer**: user_id, tenant_id, hashed values.

## Cross-Shard Queries (The Big Problem)
Queries that span multiple shards are expensive — must query all shards and merge.

```sql
-- Single shard (fast)
SELECT * FROM orders WHERE user_id = 123

-- Cross-shard (slow — fan out to all shards)
SELECT COUNT(*) FROM orders WHERE status = 'pending'
```

Solutions:
- Denormalize data so most queries stay within one shard
- Use a separate analytics DB (Redshift, BigQuery) for cross-shard aggregations
- Maintain a global secondary index

## Rebalancing Shards
When a shard gets too large, split it. Options:
1. **Consistent hashing** — minimal remapping
2. **Double writes** — write to old and new shard during migration
3. **Read from old, write to new** — migrate gradually

## Tools That Handle Sharding For You
| Tool | Sharding Approach |
|---|---|
| **Cassandra** | Consistent hashing, automatic |
| **MongoDB** | Range or hash, config servers route |
| **PostgreSQL (Citus)** | Hash sharding extension |
| **Vitess** | MySQL sharding at scale (used by YouTube) |
| **Redis Cluster** | Hash slots (16384 slots across nodes) |

## Sharding vs Partitioning
- **Partitioning**: splitting data within a single DB instance (logical)
- **Sharding**: splitting across multiple DB instances (physical, different machines)

## Key Rule
Shard only when you have exhausted vertical scaling, read replicas, and caching.
Sharding adds enormous operational complexity — delay it as long as possible.
