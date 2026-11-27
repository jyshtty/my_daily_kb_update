# Webhooks

## What is a Webhook?

A webhook is an **HTTP callback**: when something happens in system A, system A sends an HTTP `POST` to a URL that system B registered in advance.
Nobody polls. The provider calls **you**.

> API = you call them ("what's the status?").  Webhook = they call you ("the status changed!").

Examples:
- Stripe/Razorpay calls your server: "payment `pay_123` succeeded"
- GitHub calls your CI: "a commit was pushed to `main`"
- Twilio calls your app: "SMS delivered"
- A case system calls a downstream service: "AML case #1001 was assigned"

```
Polling (you pull)                        Webhook (they push)

You: "Payment done yet?" → Provider       Provider: POST https://you.com/hooks/payments
You: "Payment done yet?" → Provider                 {"event":"payment.succeeded","id":"pay_123"}
You: "Payment done yet?" → Provider       You: 200 OK
Provider: "Yes."          → You
```

## How It Works

```
① Register (once)
   You ──"call https://you.com/hooks/payments on payment events"──→ Provider
   Provider ──returns a shared SECRET for signing──→ You

② Event happens
   Customer pays → Provider creates event → looks up your registered URL

③ Delivery
   Provider ──HTTPS POST + signature header──→ Your endpoint
   Your endpoint: verify signature → save event → return 200 quickly

④ Processing (async)
   Worker reads saved event → updates order → sends email

⑤ If you fail (timeout / 5xx)
   Provider retries with backoff (e.g. minutes → hours, up to ~days)
```

## Webhook vs Other Notification Methods

| | Webhook | Polling / Long polling | WebSocket / SSE | Mobile push (FCM) |
|---|---|---|---|---|
| Who talks to whom | **Server → server** | Client → server | Server → browser/app | Server → device |
| Needs open connection | No (one-off request per event) | Repeated requests | Yes, persistent | Via OS |
| Audience | Other **systems/backends** | UI or clients | End-user UI | End users |
| Latency | Low | Medium | Very low | Low |
| Your infra needs | A public HTTPS endpoint | None | Connection handling | FCM/APNs setup |

Typical combination: a payment provider sends a **webhook** to your backend, and your backend then pushes to the user's screen via **WebSocket** or **FCM**.

## Example 1: Receiving a Webhook (FastAPI)

```python
import hmac, hashlib, json, os
from fastapi import FastAPI, Request, HTTPException

app = FastAPI()
SECRET = os.environ["WEBHOOK_SECRET"].encode()

def valid_signature(body: bytes, header: str) -> bool:
    expected = hmac.new(SECRET, body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, header)          # constant-time compare

@app.post("/hooks/payments")
async def payment_webhook(request: Request):
    body = await request.body()                           # RAW bytes, not re-serialised JSON
    if not valid_signature(body, request.headers.get("X-Signature", "")):
        raise HTTPException(status_code=401, detail="bad signature")

    event = json.loads(body)

    # Idempotency: the provider may deliver the same event more than once
    if await db.event_already_seen(event["id"]):
        return {"status": "duplicate"}

    await db.save_event(event)                            # persist first
    await queue.enqueue("process_payment_event", event["id"])   # work happens later
    return {"status": "received"}                         # respond fast (< few seconds)
```

Rules shown here:
1. **Verify the signature** over the raw body
2. **Dedupe** by event ID
3. **Save, enqueue, return 200 quickly**, don't do slow work in the request
4. Return a **non-2xx only when you want a retry**

## Example 2: Sending a Webhook (provider side)

```python
import hmac, hashlib, json, time, httpx

def sign(secret: bytes, body: bytes) -> str:
    return hmac.new(secret, body, hashlib.sha256).hexdigest()

def deliver(subscription, event):
    body = json.dumps(event).encode()
    headers = {
        "Content-Type": "application/json",
        "X-Signature": sign(subscription.secret, body),
        "X-Event-Id": event["id"],
        "X-Timestamp": str(int(time.time())),
    }
    try:
        r = httpx.post(subscription.url, content=body, headers=headers, timeout=5)
        ok = 200 <= r.status_code < 300
    except httpx.HTTPError:
        ok = False

    if not ok:
        schedule_retry(subscription, event)   # exponential backoff, see below
```

## Example 3: Payload and Headers

```http
POST /hooks/payments HTTP/1.1
Host: you.com
Content-Type: application/json
X-Event-Id: evt_8f3a
X-Timestamp: 1767225600
X-Signature: 5d41402abc4b2a76b9719d911017c592...

{
  "id": "evt_8f3a",
  "type": "payment.succeeded",
  "created": 1767225600,
  "data": { "payment_id": "pay_123", "amount": 500000, "currency": "INR" }
}
```

Two payload styles:

| Style | Content | Trade-off |
|---|---|---|
| **Fat payload** | Full object in the event | Fewer API calls, but may be stale or leak data |
| **Thin payload** | Only IDs, receiver fetches details via API | Always fresh and safer, one extra call |

## Reliability: Retries and Backoff

Webhooks are **at-least-once**. Networks fail, receivers go down.

```
Attempt 1  fail → wait 1 min
Attempt 2  fail → wait 5 min
Attempt 3  fail → wait 30 min
Attempt 4  fail → wait 2 h
...
After N attempts / ~3 days → give up → Dead Letter Queue + alert
```

- Use **exponential backoff + jitter**
- Treat `2xx` as success; `4xx` (except 408/429) usually means do not retry; `5xx`/timeout means retry
- After the final failure, move the event to a **dead letter queue** (see [004_dead_letter_queue.md](004_dead_letter_queue.md)) and let users **replay** from a dashboard
- **Ordering is not guaranteed**: events can arrive late or out of order. Include a timestamp or sequence number, and make handlers tolerant (e.g. ignore an older `status` than the one stored)

## Receiver Checklist

| Concern | What to do |
|---|---|
| **Authenticity** | Verify HMAC signature (or mTLS / provider-signed JWT) |
| **Replay attacks** | Check the timestamp header, reject if older than ~5 minutes |
| **Duplicates** | Store processed event IDs; make handlers idempotent (see [010_idempotency.md](../010_idempotency.md)) |
| **Speed** | Return 2xx fast; process asynchronously via a queue |
| **Out-of-order events** | Compare versions/timestamps before applying |
| **Transport** | HTTPS only; restrict to provider IP ranges if published |
| **Missed events** | Run a periodic **reconciliation job** that calls the provider's API to catch anything lost |

## Sender Checklist

| Concern | What to do |
|---|---|
| Sign every request | Per-subscriber secret, support secret rotation |
| Timeouts | Short (5–10 s) so slow receivers can't pile up workers |
| Isolation | Per-endpoint queues, so one failing receiver can't block others |
| Rate limiting | Cap delivery rate per endpoint |
| Auto-disable | Pause endpoints that fail for days, and notify the owner |
| SSRF protection | Block private/internal IPs and redirects when users supply URLs |
| Observability | Delivery log (status code, latency, attempts), manual replay |

## Scaling the Sender

```
Event source → Kafka / SQS → Webhook dispatcher workers → HTTP POST to each subscriber
                                   │
                                   ├─ retry queue (delayed)
                                   └─ DLQ after max attempts
```
- Decouple event creation from delivery with a queue
- Scale workers horizontally; partition by subscriber for isolation and per-endpoint ordering
- A slow subscriber must never slow the others (bulkhead, see [011](../011_circuit_breaker_and_resilience.md))

## Local Development
The provider can't reach `localhost`. Use a tunnel (ngrok, Cloudflare Tunnel) or the provider's CLI (`stripe listen`, `gh webhook forward`) to forward events to your machine, and keep sample payloads for tests.

## Webhook vs Polling, When to Choose

| Choose webhooks when | Choose polling when |
|---|---|
| Events are infrequent and unpredictable | You can't expose a public endpoint |
| You need near-real-time reaction | Provider offers no webhooks |
| You want to avoid wasted requests | You need to guarantee you see every state (poll as a safety net) |

Best practice: **webhooks as the fast path plus periodic polling/reconciliation as the safety net.**

## Pitfalls

| Pitfall | Fix |
|---|---|
| Doing heavy work before responding → provider times out and retries → duplicates | Ack first, process async |
| Verifying the signature on parsed JSON | Verify over the **raw** body |
| Assuming exactly-once, in-order delivery | Idempotent, order-tolerant handlers |
| No secret / unauthenticated endpoint | Anyone can forge events, always verify |
| Endpoint down for hours, events lost | Provider retries + your reconciliation job |
| Returning 200 even when processing failed | Persist before acking, return 5xx if you could not save |

## Interview Quick Answer
> A webhook is a server-to-server HTTP callback: a consumer registers a URL, and the provider POSTs an event to it when something happens, so nobody has to poll. Delivery is at-least-once, so the receiver verifies an HMAC signature on the raw body, checks the timestamp, dedupes by event ID, saves the event, returns 2xx quickly and processes asynchronously. The sender signs payloads, uses queues with exponential-backoff retries, isolates slow subscribers, and dead-letters after max attempts with replay. I'd add a reconciliation job as a safety net.

## Rule of Thumb
Acknowledge fast, verify everything, process idempotently, and never trust that an event arrived once or in order.
