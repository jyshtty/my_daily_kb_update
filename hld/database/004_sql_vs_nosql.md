# SQL vs NoSQL

## Core Difference

| | SQL (Relational) | NoSQL |
|---|---|---|
| Data model | Tables, rows, fixed schema | Key-value, document, wide-column, graph |
| Schema | Strict, enforced up front | Flexible / schema-on-read |
| Relationships | Joins, foreign keys | Usually denormalized / embedded |
| Transactions | Full ACID, multi-row | Often single-record; some support multi-doc |
| Scaling | Vertical first, sharding is manual | Designed for horizontal scale |
| Consistency | Strong by default | Often tunable / eventual |
| Query | Powerful ad-hoc SQL | Access-pattern driven |
| Examples | PostgreSQL, MySQL, Oracle | Redis, MongoDB, Cassandra, DynamoDB, Neo4j |

## NoSQL Types

| Type | Model | Good for | Examples |
|---|---|---|---|
| **Key-value** | `key → blob` | Cache, sessions, counters, leaderboards | Redis, DynamoDB, Memcached |
| **Document** | `id → JSON doc` | Catalogs, user profiles, CMS | MongoDB, Couchbase |
| **Wide-column** | Row key → column families | Huge write volume, time series, IoT | Cassandra, HBase, Bigtable |
| **Graph** | Nodes + edges | Social graph, fraud rings, recommendations | Neo4j, Neptune |

## When to Choose SQL
- Data is **relational** and you need joins
- You need **ACID** transactions (payments, inventory, bookings)
- Schema is stable and integrity matters (constraints, foreign keys)
- Ad-hoc queries and reporting are common
- Scale fits one primary + read replicas (this covers most systems)

## When to Choose NoSQL
- **Massive scale** or write throughput beyond a single node
- Access patterns are known and simple (lookup by key)
- Schema changes often or data is semi-structured
- Need low latency at scale (caching, sessions)
- Geo-distributed, high-availability requirements

## Modelling Differences
```
SQL (normalized)                 NoSQL document (denormalized)
users ─< orders ─< order_items   { "user": "u1",
                                   "orders": [
 JOIN at read time                   { "id": 7, "items": [ ... ] } ] }
                                  one read, no join, data duplicated
```
- SQL: model the **data**, then write any query
- NoSQL: model the **queries**, then shape the data to fit them

## Trade-offs to Mention
| Topic | SQL | NoSQL |
|---|---|---|
| Joins | Native | Avoided or done in app |
| Duplication | Minimal | Accepted for speed |
| Updating duplicated data | Single place | Must update many places |
| Horizontal scaling | Hard (cross-shard joins/transactions) | Built in |
| Maturity / tooling | Very mature | Varies by engine |

## Common Misconceptions
- "NoSQL is faster" — only for the access pattern it was designed for
- "SQL doesn't scale" — it does a long way (replicas, partitioning, Citus, Vitess, Aurora)
- "NoSQL has no schema" — the schema just lives in application code
- Modern SQL (PostgreSQL JSONB) can store documents too

## Polyglot Persistence
Real systems mix stores:
```
PostgreSQL  → orders, payments (ACID)
Redis       → cache, sessions, rate limits
Elasticsearch → search
Cassandra   → activity / event feed
S3          → files and blobs
```

## Rule of Thumb
Default to a relational database. Pick NoSQL when a specific need — scale, access pattern, flexible schema, latency — clearly demands it, and use each store for what it is best at.
