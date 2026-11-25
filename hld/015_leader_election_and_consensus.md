# Leader Election and Consensus

## Why It Matters
Distributed nodes must **agree** on things: who the leader is, what the next log entry is, what the cluster config is. Without agreement you get split brain and data corruption.

```
Node A ─┐
Node B ─┼── must agree on ONE leader / ONE order of writes
Node C ─┘
```

## Problem: Split Brain
Network partition → both sides think the other is dead and elect a leader.
```
[A  B]  ✗ partition ✗  [C]
 leader=A                leader=C   ← two leaders accept writes, data diverges
```
Solution: a leader needs a **majority (quorum)**. Only one side of a partition can have a majority.

## Quorum
For `N` nodes, majority = `⌊N/2⌋ + 1`.

| Nodes (N) | Quorum | Failures tolerated |
|---|---|---|
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |
| 4 | 3 | 1 (no better than 3 nodes) |

Use an **odd** number of nodes. Tolerating `f` failures needs `2f + 1` nodes.

## Consensus Algorithms

| Algorithm | Notes | Used by |
|---|---|---|
| **Raft** | Designed to be understandable; strong leader | etcd, Consul, CockroachDB, TiKV |
| **Paxos** | Original, notoriously hard; Multi-Paxos in practice | Google Chubby/Spanner |
| **ZAB** | Atomic broadcast for ordered updates | ZooKeeper |
| **Viewstamped Replication** | Similar family to Raft | Research / some DBs |

## Raft in Brief

### Roles
`Follower` → `Candidate` → `Leader`

### Leader Election
```
1. Follower hears no heartbeat before a randomized election timeout
2. Becomes Candidate, increments term, votes for itself, requests votes
3. Nodes vote once per term (and only for candidates with up-to-date logs)
4. Majority of votes → becomes Leader, sends heartbeats
5. Higher term seen → step down to follower
```
- **Term** = logical clock; stale leaders are rejected
- Randomized timeouts avoid repeated split votes

### Log Replication
```
Client → Leader appends entry → sends AppendEntries to followers
        → majority acknowledge → entry COMMITTED → applied → reply to client
```
Committed entries are never lost, because any future leader must have them.

## Fencing: Protecting Against Zombie Leaders
An old leader paused (GC, network) may wake up and still believe it leads.
- Each leadership has a **monotonic fencing token** (term / epoch)
- Storage rejects writes with an older token than it has seen
```
Leader A (token 33) pauses → B elected (token 34) writes
A wakes and writes with token 33 → storage rejects it
```

## Leases
A leader holds a **time-limited lease** and must renew it.
- If it can't renew, it must stop acting as leader
- Depends on bounded clock drift — combine with fencing for safety

## Practical Tools
Don't implement consensus yourself. Use:

| Tool | Provides |
|---|---|
| **etcd / Consul / ZooKeeper** | Distributed locks, leader election, config, service discovery |
| **Kubernetes Lease objects** | Leader election for controllers |
| **Database primary election** | Patroni (PostgreSQL), MongoDB replica set elections |

Typical use: only one node runs a scheduler / cron job / singleton consumer — it grabs a lock in etcd/ZooKeeper; others stand by.

## Trade-offs
- Consensus costs latency (a round trip to a majority) and caps write throughput through the leader
- Loses availability without a quorum (a CP choice — see `001_cap_theorem.md`)
- Cross-region consensus is slow; keep consensus groups small and local where possible

## Interview Quick Answer
> Consensus lets nodes agree on a leader and an ordered log despite failures. Raft elects a leader using terms, randomized timeouts and majority votes, then replicates log entries to a majority before committing. Quorums prevent split brain, and fencing tokens stop old leaders from writing. In practice I use etcd, ZooKeeper or Consul rather than building it.

## Rule of Thumb
Keep consensus clusters small (3 or 5 nodes), always require a majority, and protect resources with fencing tokens.
