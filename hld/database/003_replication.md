# Database Replication

## What It Is
**Replication** keeps copies of the same data on multiple nodes.

| Goal | How replication helps |
|---|---|
| **Availability** | If a node dies, another copy serves traffic |
| **Read scalability** | Spread reads across replicas |
| **Latency** | Place copies near users (multi-region) |
| **Durability** | Data survives disk / node / AZ loss |

> Replication = **same data on many nodes**. Sharding = **different data on different nodes**. Large systems use both.

## Replication Topologies

### 1. Leader–Follower (primary–replica)
```
Writes → Leader ──replication log──→ Follower 1 (reads)
                                  └→ Follower 2 (reads)
```
- All writes go to one leader; followers apply the leader's log
- Simple, no write conflicts
- Used by: PostgreSQL, MySQL, MongoDB (replica set), Redis
- Weakness: leader is the write bottleneck and a failover point

### 2. Multi-Leader
```
Region A Leader ⇄ Region B Leader ⇄ Region C Leader
```
- Each region accepts writes locally, then syncs to the others
- Good for multi-region and offline-capable clients
- Weakness: **write conflicts** (same row edited in two regions) — need conflict resolution (last-write-wins, CRDTs, app logic)

### 3. Leaderless (Dynamo-style)
```
Client writes to N replicas, waits for W acks
Client reads from R replicas, picks newest
Quorum rule:  W + R > N
```
- No single leader; any node can accept reads/writes
- Used by: Cassandra, DynamoDB, Riak
- Tunable consistency; needs read repair / anti-entropy to heal stale replicas

## Synchronous vs Asynchronous

| | Synchronous | Asynchronous |
|---|---|---|
| Write acknowledged when | Follower also confirmed | Leader wrote locally |
| Durability | Strong — no data loss on leader crash | May lose recent writes on failover |
| Write latency | Higher (waits on follower) | Low |
| Availability | A slow follower can block writes | Followers don't block writes |
| Typical use | 1 sync follower + rest async (**semi-sync**) | Most read replicas |

## Replication Lag Problems
With async replication, followers can be behind. Users see odd behaviour:

| Anomaly | Example | Fix |
|---|---|---|
| **Read-your-writes violated** | Post a comment, refresh, comment missing | Read own recent writes from leader, or track write timestamp |
| **Monotonic reads violated** | Page shows new data, refresh shows older data | Pin a user to one replica |
| **Consistent prefix violated** | Reply appears before the question | Keep causally related writes in the same partition |

## Failover
1. Detect leader failure (heartbeat timeout)
2. Elect a new leader (most up-to-date follower)
3. Reconfigure clients / proxies to the new leader
4. Old leader rejoins as follower

Dangers:
- **Split brain** — two nodes both think they are leader → data divergence. Prevent with quorum-based election and **fencing**
- Async failover can **lose** writes the old leader had not shipped
- Timeout too short → false failovers; too long → long outage

## Replication Methods
| Method | Notes |
|---|---|
| Statement-based | Ships SQL; breaks on non-deterministic functions (`NOW()`, `RAND()`) |
| WAL shipping | Ships low-level disk log; tied to storage format/version |
| **Logical (row-based)** | Ships row changes; version-independent, enables CDC |

## Interview Quick Answer
> Replication copies data across nodes for availability, read scale and durability. Leader–follower is the default; multi-leader helps multi-region writes but needs conflict resolution; leaderless uses quorums (W + R > N). Async replication is fast but has lag and can lose writes on failover; sync is safer but slower. Handle lag with read-your-writes routing and avoid split brain with quorum election and fencing.

## Rule of Thumb
Start with leader–follower and async replicas. Add sync replication where you cannot lose writes. Reach for multi-leader or leaderless only when you truly need multi-region writes or very high availability.
