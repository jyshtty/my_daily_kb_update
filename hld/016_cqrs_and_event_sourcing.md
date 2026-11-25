# CQRS and Event Sourcing

## CQRS (Command Query Responsibility Segregation)

### Idea
Use **different models** for writing and reading.

```
Commands (writes)                  Queries (reads)
 PlaceOrder, CancelOrder            GetOrderSummary, ListOrders
        │                                   ↑
        ↓                                   │
  Write model (normalized,         Read model (denormalized,
  enforces business rules)         shaped for the screen/API)
        │                                   ↑
        └──── events / change feed ────────┘   (async sync)
```

### Why
| Problem in one shared model | CQRS answer |
|---|---|
| Reads and writes have very different load | Scale each side independently |
| Complex joins slow down reads | Pre-built, denormalized read views |
| Domain rules clutter query code | Write model focuses on invariants |
| One DB tech can't suit both | e.g. PostgreSQL for writes, Elasticsearch/Redis for reads |

### Levels
1. **Same DB, separate code paths** — lightest form, often enough
2. **Separate read store**, updated asynchronously from the write side
3. **CQRS + Event Sourcing**

### Costs
- **Eventual consistency**: the read model lags the write model — a user may not see their change immediately (mitigate: return the new state from the command, optimistic UI, read-your-writes routing)
- More components, duplicated data, more operational work
- Needs idempotent projections and a way to rebuild read models

## Event Sourcing

### Idea
Store **the sequence of events** that happened, not just the current state. State = replaying events.

```
Traditional:  accounts(id=1, balance=70)          ← history lost

Event sourced: AccountOpened(100)
               MoneyWithdrawn(50)
               MoneyDeposited(20)    → replay → balance = 70
```

### Event Store
- **Append-only** log, partitioned per aggregate (e.g. per account ID)
- Events are immutable facts, named in the past tense: `OrderPlaced`, `PaymentFailed`
- Optimistic concurrency per stream: append only if expected version matches

```
Order-42:  v1 OrderPlaced → v2 ItemAdded → v3 PaymentReceived → v4 Shipped
```

### Snapshots
Replaying thousands of events is slow, so periodically store a snapshot of state and replay only events after it.

```
snapshot@v1000 + events v1001…v1050 → current state
```

### Projections
Consumers read the event stream and build **read models** (this is where CQRS plugs in).
Need a new view? Replay history into a new projection.

## Benefits
| Benefit | Detail |
|---|---|
| **Full audit trail** | Every change is recorded (finance, compliance) |
| **Time travel** | Rebuild state at any point; debug "how did we get here?" |
| **Rebuildable read models** | New reports or search indexes from existing history |
| **Natural fit for events** | Integrates with Kafka and event-driven systems |

## Challenges
| Challenge | Mitigation |
|---|---|
| **Event schema evolution** — old events live forever | Versioned events, upcasters, additive changes |
| Querying current state is not direct | Projections / CQRS |
| Eventual consistency | Design UI and APIs for it |
| Learning curve, more complexity | Use only where the benefits justify it |
| **GDPR / deletion** vs immutable log | Crypto-shredding (delete the encryption key) or store PII outside events |
| Side effects on replay (emails, payments) | Separate projections from side-effecting handlers |

## When to Use
| Good fit | Bad fit |
|---|---|
| Finance, ledgers, audit-heavy domains | Simple CRUD apps |
| Complex domain with rich history | Small teams with simple needs |
| Need for temporal queries and replay | Data with heavy deletion/ rewriting requirements |
| Read and write loads differ greatly | Strong, immediate consistency everywhere |

CQRS and Event Sourcing are **independent**: you can use CQRS without event sourcing, and event sourcing without CQRS (though it nearly always needs projections).

## Interview Quick Answer
> CQRS separates the write model from one or more read models so each can be optimized and scaled independently, at the cost of eventual consistency. Event sourcing stores an append-only sequence of immutable events and derives state by replaying them, giving a full audit trail and rebuildable projections. I'd use them for audit-heavy or complex domains and avoid them for simple CRUD, because of the added complexity and schema-evolution burden.

## Rule of Thumb
Start with plain CRUD. Add CQRS when read and write needs diverge, and add event sourcing only when history itself is a business requirement.
