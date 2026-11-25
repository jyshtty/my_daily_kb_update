# Idempotency

## What It Is
An operation is **idempotent** if performing it many times has the **same effect as performing it once**.

```
set_balance(100)   → idempotent   (repeat = still 100)
add_to_balance(10) → NOT idempotent (repeat = +10 each time)
```

## Why It Matters
Networks and queues are unreliable. Clients **retry** after timeouts, brokers **redeliver** messages, and you can't tell whether the first attempt worked.

```
Client ──charge $50──→ Server (charges, response lost)
Client ──(timeout) retries──→ Server charges AGAIN  ✗ double charge
```

Delivery guarantees:

| Guarantee | Meaning | Reality |
|---|---|---|
| At-most-once | May lose messages | Simple, lossy |
| **At-least-once** | May duplicate | Most common |
| Exactly-once | Once and only once | Practically = **at-least-once + idempotent consumer** |

## HTTP Methods

| Method | Idempotent? | Notes |
|---|---|---|
| GET, HEAD, OPTIONS | Yes | Also safe (no side effects) |
| PUT | Yes | Replaces a resource |
| DELETE | Yes | Deleting twice leaves the same end state |
| **POST** | **No** | Creates / triggers — needs an idempotency key |
| PATCH | Depends | Idempotent only if it sets absolute values |

## Idempotency Key Pattern
Client generates a unique key per logical operation and sends it on every retry.

```
POST /payments
Idempotency-Key: 7f3c9e2a-...
```

Server flow:
```
1. Look up key in store
2. Found, completed   → return the stored response (don't re-execute)
3. Found, in progress → return 409 / wait
4. Not found          → insert key as "in progress", execute,
                         store response, mark complete
```

```sql
CREATE TABLE idempotency_keys (
  key         TEXT PRIMARY KEY,        -- unique constraint is the guard
  request_hash TEXT,                   -- detect same key, different payload
  status      TEXT,                    -- in_progress | completed
  response    JSONB,
  created_at  TIMESTAMP
);
```

Key points:
- The **unique constraint** (or Redis `SET NX`) makes "first writer wins" atomic
- Store a hash of the request; same key + different body → reject (422)
- Expire keys after a window (e.g. 24 h)
- Write the business change and the key record in the **same DB transaction**

## Idempotent Consumers (Messaging)
```python
def handle(msg):
    with db.transaction():
        if processed_ids.exists(msg.id):     # dedupe table
            return                           # already done, just ack
        apply_business_change(msg)
        processed_ids.add(msg.id)
    ack(msg)
```
Other techniques:
- **Natural idempotence**: `UPSERT`, `SET status = 'shipped'` instead of increments
- **Conditional / versioned writes**: `UPDATE ... WHERE version = n`
- **Deterministic IDs**: derive the key from business data (`order_id + step`)

## Where It Is Needed
- Payments and refunds
- Order creation, inventory updates
- Message / webhook consumers (see `004_dead_letter_queue.md` replay)
- Any API a client may retry, plus saga steps and compensations

## Pitfalls

| Pitfall | Fix |
|---|---|
| Key check and action not atomic | Same transaction, or unique constraint |
| Key generated per attempt, not per operation | Client creates it once, reuses on retry |
| Side effects outside the transaction (email, external API) | Outbox pattern, or pass the key to the downstream system |
| Unbounded key storage | TTL / cleanup |
| Returning a different response on replay | Store and return the original response |

## Interview Quick Answer
> Idempotency means repeating an operation doesn't change the result beyond the first time. Because retries and at-least-once delivery are unavoidable, I make POST endpoints idempotent with a client-supplied idempotency key stored with a unique constraint, and make consumers dedupe by message ID in the same transaction as the business change. That gives effectively-once processing.

## Rule of Thumb
Assume every request and message will be delivered more than once, and design so that it doesn't matter.
