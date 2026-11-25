# CAP Theorem

## What is CAP?

In a distributed system you can only guarantee **2 of 3**:

- **C — Consistency**: every read returns the most recent write
- **A — Availability**: every request gets a response (not guaranteed to be latest)
- **P — Partition Tolerance**: system keeps working even if network splits nodes

**Network partitions always happen** in real distributed systems — so you must choose between C and A when a partition occurs.

## The Three Trade-offs

### CP (Consistency + Partition Tolerance)
- Sacrifices availability — system rejects requests rather than return stale data
- Examples: **HBase, Zookeeper, MongoDB (default config)**
- Use when: banking, inventory, anything where stale data causes real harm

### AP (Availability + Partition Tolerance)
- Sacrifices consistency — nodes may return stale data during partition
- Examples: **Cassandra, DynamoDB, CouchDB**
- Use when: social feeds, analytics, DNS — eventual consistency is acceptable

### CA (Consistency + Availability)
- Sacrifices partition tolerance — only works on a single node or trusted network
- Examples: **Traditional RDBMS (PostgreSQL, MySQL)** on a single node
- Not viable for truly distributed systems

## Visual

```
          Consistency
              /\
             /  \
            / CP \
           /      \
          /--------\
         / CA   AP  \
        /____________\
   Availability    Partition
                   Tolerance
```

## PACELC Extension

CAP only covers partition scenarios. **PACELC** adds normal operation trade-offs:
- If Partition → choose A or C
- Else (normal) → choose Latency or Consistency

| System | Partition | Normal |
|---|---|---|
| DynamoDB | AP | EL (low latency) |
| Spanner | CP | EC (strong consistency) |
| Cassandra | AP | EL |

## Key Rule
Pick CP when wrong data = real damage. Pick AP when availability matters more than perfect consistency.
