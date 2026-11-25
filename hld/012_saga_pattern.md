# Saga Pattern (Distributed Transactions)

## The Problem
In microservices each service owns its database, so a business operation spans several databases. A single ACID transaction isn't possible.

```
Place Order = Order Service + Payment Service + Inventory Service + Shipping Service
              (4 databases — no shared transaction)
```

## Why Not Two-Phase Commit (2PC)?

| | 2PC | Saga |
|---|---|---|
| Consistency | Strong, atomic | Eventual |
| Locks held | Across all participants until commit | Local transactions only |
| Availability | Coordinator failure blocks everyone | No global lock |
| Latency / scale | Poor | Good |
| Support | Needs XA-capable resources | Works with any store / message bus |

2PC is blocking and fragile at scale, so microservices mostly use **sagas**.

## What a Saga Is
A sequence of **local transactions**. Each step publishes an event or command that triggers the next. If a step fails, run **compensating transactions** to undo the earlier steps.

```
T1 Create order   → T2 Reserve inventory → T3 Charge payment → T4 Ship
                                                  ✗ fails
              C2 Release inventory ← C1 Cancel order   (compensate in reverse)
```

Compensation is a **semantic undo** (refund, cancel, release), not a database rollback — the original step really committed.

## Two Styles

### Choreography (event-driven, no central brain)
```
Order ──OrderCreated──→ Inventory ──StockReserved──→ Payment ──PaymentDone──→ Shipping
          ← OrderCancelled ←── PaymentFailed (each service listens and reacts)
```
- Pros: simple for few steps, loosely coupled, no single coordinator
- Cons: flow is implicit and scattered; hard to trace; risk of cyclic dependencies

### Orchestration (central coordinator)
```
              Saga Orchestrator
        ┌────────┼─────────┬────────┐
     Order   Inventory  Payment  Shipping
   (tells each service what to do next and handles failures)
```
- Pros: flow in one place, easy to monitor and change, clear failure handling
- Cons: orchestrator is extra component; risk of too much logic in it
- Tools: Temporal, AWS Step Functions, Camunda, Netflix Conductor

| Choose | When |
|---|---|
| Choreography | 2–4 steps, simple flow, autonomous teams |
| Orchestration | Many steps, branching, need visibility and timeouts |

## Design Rules
| Rule | Reason |
|---|---|
| Every step needs a **compensation** (or is the last "pivot" step) | To be able to roll back semantically |
| Steps and compensations must be **idempotent** (see `010_idempotency.md`) | Messages get redelivered |
| Compensations must be **retryable** until they succeed | They can't be allowed to fail permanently |
| Order steps: risky/hard-to-undo last | Fewer compensations needed |
| Track saga state (started, step N, compensating, done) | Recovery after crash |

## Atomically Publishing Events: the Outbox Pattern
Problem: update DB **and** publish an event — one can succeed while the other fails.

```
BEGIN;
  UPDATE orders SET status='CREATED';
  INSERT INTO outbox(event) VALUES ('OrderCreated');   -- same transaction
COMMIT;
Relay/CDC reads outbox → publishes to Kafka/RabbitMQ → marks sent
```
Guarantees at-least-once publishing; consumers must be idempotent.

## Isolation Problem
Sagas lack isolation — other requests can see intermediate state (order exists but payment not yet done).
Countermeasures:
- **Semantic locks** — mark records as `PENDING`
- Commutative updates, re-read values before acting
- Show "processing" states in the UI

## Pitfalls
- Forgetting a compensation path, or compensations that can fail
- Lost or duplicated messages without idempotency / outbox
- Long sagas with no timeouts → stuck orders
- Treating a saga like an ACID transaction

## Interview Quick Answer
> Because each microservice owns its data, I avoid 2PC and use a saga: a chain of local transactions, each with a compensating action if a later step fails. I choose choreography for short, simple flows and orchestration (e.g. Temporal) for complex ones. Steps are idempotent, events are published reliably with the outbox pattern, and the system is eventually consistent.

## Rule of Thumb
Prefer one service owning the whole transaction. When you must span services, use a saga with idempotent steps, compensations and an outbox.
