# Dead Letter Queue (DLQ)

## What It Is
A **Dead Letter Queue** is a separate queue where messages go when they **cannot be processed successfully**.
Instead of being lost, or retried forever and blocking the main queue, the failed message is parked for inspection, alerting and later replay.

```
Producer → Main Queue → Consumer ──(success)──→ ack, done
                           │
                           └──(fails N times / rejected / expired)──→ DLQ → alert + inspect + replay
```

Think: **quarantine for poison messages**.

## Why You Need One

| Without DLQ | With DLQ |
|---|---|
| Poison message retried forever → consumer stuck in a loop | Message moved aside after N attempts |
| Head-of-line blocking — good messages wait behind a bad one | Main queue keeps flowing |
| Message dropped silently → data loss | Message preserved with failure context |
| No visibility into failures | DLQ depth is a clear metric to alert on |

## When a Message Gets Dead-Lettered

| Reason | Example |
|---|---|
| **Max retries exceeded** | Downstream API keeps returning 500 |
| **Rejected / nacked without requeue** | Consumer decides message is invalid |
| **Deserialization / schema error** | Malformed JSON, unknown schema version |
| **Validation failure** | Missing required field, bad business data |
| **TTL expired** | Message sat in queue longer than its time-to-live |
| **Queue length limit exceeded** | Overflow policy set to dead-letter |

## Transient vs Permanent Failures
The most important design decision: **don't send everything to the DLQ immediately.**

| Type | Examples | Handling |
|---|---|---|
| **Transient** | Timeout, 503, DB lock, rate limit | Retry with **exponential backoff + jitter**, then DLQ |
| **Permanent** | Bad JSON, schema mismatch, 400, unknown ID | Send to DLQ **immediately** — retrying won't help |

```
fail → transient? ──yes──→ retry (1s, 2s, 4s, 8s...) ──still failing after N──→ DLQ
          │
          no (permanent) ─────────────────────────────────────────────────────→ DLQ
```

## How Major Brokers Do It

### AWS SQS
- Configure a **redrive policy** on the source queue: `deadLetterTargetArn` + `maxReceiveCount`
- After a message is received `maxReceiveCount` times without being deleted, SQS moves it to the DLQ
- **DLQ redrive** lets you move messages back to the source queue from the console/API
- DLQ type must match source: FIFO → FIFO, Standard → Standard
- Set DLQ retention **longer** than the source queue (retention is based on original enqueue time)

```json
{
  "deadLetterTargetArn": "arn:aws:sqs:us-east-1:123456789012:orders-dlq",
  "maxReceiveCount": 5
}
```

### RabbitMQ
- Built-in via **Dead Letter Exchange (DLX)** — set `x-dead-letter-exchange` (and optionally `x-dead-letter-routing-key`) on the queue or through a policy
- Triggers: `basic.reject`/`basic.nack` with `requeue=false`, message TTL expiry, queue length limit
- Dead-lettered message gets an `x-death` header (reason, original queue, count, time)
- Retry with delay: main queue → DLX → "wait" queue with TTL → dead-letters back to main queue

```python
channel.queue_declare(
    queue="orders",
    arguments={
        "x-dead-letter-exchange": "orders.dlx",
        "x-dead-letter-routing-key": "orders.dead",
    },
)
```

### Kafka
- **No built-in DLQ** in the broker — it's a log, there is no per-message "reject"
- Implemented at the application layer: on failure, produce the record to a `orders.DLT` / `orders-dlq` topic, then commit the offset
- **Kafka Connect**: `errors.tolerance=all` + `errors.deadletterqueue.topic.name=...`
- **Spring Kafka**: `DefaultErrorHandler` + `DeadLetterPublishingRecoverer`
- Common pattern: retry topics with increasing delay (`orders-retry-1m`, `orders-retry-10m`) → `orders-dlt`

### Others
| System | DLQ support |
|---|---|
| Azure Service Bus | Built-in `$DeadLetterQueue` sub-queue per queue/subscription (`MaxDeliveryCount`) |
| Google Pub/Sub | Dead-letter topic on a subscription (`max_delivery_attempts`, 5–100) |
| AWS SNS / Lambda / EventBridge | DLQ (SQS) for failed async deliveries / invocations |

## Consumer-Side Pattern (Generic)

```python
MAX_ATTEMPTS = 5

def handle(message):
    attempts = int(message.headers.get("x-attempts", 0))
    try:
        process(message.body)
        ack(message)
    except PermanentError as e:
        send_to_dlq(message, reason=str(e), attempts=attempts)
        ack(message)                      # remove from main queue
    except TransientError as e:
        if attempts + 1 >= MAX_ATTEMPTS:
            send_to_dlq(message, reason=str(e), attempts=attempts + 1)
            ack(message)
        else:
            retry_later(message, attempts=attempts + 1,
                        delay=backoff_with_jitter(attempts))
            ack(message)
```

## What to Store With a Dead-Lettered Message
Enough context to debug and replay without guessing:
- Original payload (unchanged)
- Source queue / topic, partition, offset
- Failure reason + error type / stack trace snippet
- Attempt count, first and last failure timestamps
- Correlation / trace ID, consumer service + version

## Operating a DLQ

### Monitor
- **Alert when DLQ depth > 0** (or above a small threshold) — a DLQ nobody watches is just a slower way to lose data
- Track DLQ inflow rate; a sudden spike usually means a bad deploy or downstream outage
- Watch age of oldest message

### Triage
1. Inspect messages and group by failure reason
2. Fix the root cause — code bug, schema change, downstream dependency
3. **Replay** (redrive) back to the main queue — or discard if truly invalid

### Replay safely
- Consumers must be **idempotent** — replayed messages may already be partially processed
- Replay at a throttled rate so you don't overload the fixed service
- Ordering may break: replayed messages arrive after newer ones. If order matters, handle it with versioning / sequence numbers

## Pitfalls

| Pitfall | Fix |
|---|---|
| Nobody monitors the DLQ | Alert on depth, assign an owner |
| DLQ retention shorter than time to investigate | Set long retention (e.g. 14 days on SQS) |
| Retrying permanent errors N times | Classify errors, dead-letter permanent ones immediately |
| `maxReceiveCount = 1` | One transient blip sends messages to DLQ — use 3–5+ |
| Non-idempotent consumers + replay | Idempotency keys / dedupe table |
| Losing failure context | Attach reason, attempts, trace ID as headers / attributes |
| Replay loop — message fails again and returns to DLQ forever | Fix root cause first; cap replays |
| Strict ordering + DLQ | Dead-lettering one message breaks order for that key — consider pausing the partition/key instead |

## Interview Quick Answer
> A DLQ is a holding queue for messages that fail processing after a configured number of retries, or that are rejected, expired, or malformed. It prevents poison messages from blocking the main queue and avoids silent data loss. You retry transient errors with backoff, dead-letter permanent ones immediately, attach failure metadata, alert on DLQ depth, fix the root cause, then replay with idempotent consumers.

## Rule of Thumb
Retry what might succeed later. Dead-letter what never will. Always watch the DLQ, and only replay once the cause is fixed.
