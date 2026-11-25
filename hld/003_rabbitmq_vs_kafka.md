# RabbitMQ vs Kafka

## Core Philosophy

| | RabbitMQ | Kafka |
|---|---|---|
| Model | **Message broker** — routes and delivers messages | **Event log** — stores ordered stream of events |
| Consumer model | Messages pushed to consumers, deleted after ack | Consumers pull at their own pace, messages retained |
| Ordering | Per-queue ordering | Per-partition ordering |
| Replay | No — message gone after consumed | Yes — replay from any offset |

## How They Work

### RabbitMQ
```
Producer → Exchange → Queue → Consumer (ack) → message deleted
```
- Exchange routes messages to queues using routing keys
- Message disappears after consumer acknowledges
- Think: **task queue / job dispatcher**

### Kafka
```
Producer → Topic (Partition 1, 2, 3...) → Consumer Group (offset tracked)
                    ↑
            messages retained for N days
```
- Messages stored in ordered log, consumers track their own offset
- Multiple consumer groups can read the same messages independently
- Think: **event stream / audit log**

## Feature Comparison

| Feature | RabbitMQ | Kafka |
|---|---|---|
| Throughput | ~50k msg/sec | Millions msg/sec |
| Latency | Sub-millisecond | Low ms |
| Message replay | No | Yes |
| Message ordering | Per-queue | Per-partition |
| Consumer groups | Competing consumers | Independent groups |
| Message retention | Until consumed/TTL | Time-based (default 7 days) |
| Protocol | AMQP | Custom binary |
| Complexity | Lower | Higher |

## When to Use Which

### Use RabbitMQ when:
- Task queues — one worker should process each job exactly once
- Complex routing — fan-out, topic exchange, direct routing
- Low latency delivery matters
- You need per-message acknowledgement and retry logic
- Example: send email, process payment, resize image

### Use Kafka when:
- Event streaming — multiple systems need the same events
- Audit logs / event sourcing — need full history
- Very high throughput (millions/sec)
- Replay needed — reprocess historical events
- Example: user activity stream, CDC (change data capture), analytics pipeline

## Common Misconception
Kafka is NOT a drop-in replacement for RabbitMQ.
- Kafka is a log — great for streaming and replay
- RabbitMQ is a broker — great for task distribution and routing
- Many production systems use both: Kafka for event streams, RabbitMQ for job queues

## Rule of Thumb
If consumers need to process a task once → RabbitMQ.
If multiple systems need the same events, or you need history → Kafka.
