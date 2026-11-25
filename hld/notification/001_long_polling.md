# Long Polling

## What is Long Polling?

A technique where the client makes a request and the server **holds it open**
until new data is available (or a timeout occurs), then responds.
The client immediately sends another request — creating a pseudo-real-time connection.

```
Client: GET /notifications (holds connection open)
Server: ... waiting for data ...
Server: data available → responds with data
Client: immediately sends another GET /notifications
```

## Flow

```
Client                    Server
  |── GET /poll ─────────→|
  |                        | (holds request, waiting)
  |                        | (new event arrives)
  |←── 200 { data } ──────|
  |── GET /poll ─────────→|  ← immediately reconnects
  |                        | (holds again...)
```

## Implementation (FastAPI)

```python
import asyncio
from fastapi import FastAPI

app = FastAPI()
event_queue: asyncio.Queue = asyncio.Queue()

@app.get("/poll")
async def long_poll(timeout: int = 30):
    try:
        # Hold connection until event or timeout
        event = await asyncio.wait_for(event_queue.get(), timeout=timeout)
        return {"data": event}
    except asyncio.TimeoutError:
        return {"data": None}  # client reconnects

@app.post("/notify")
async def push_event(data: dict):
    await event_queue.put(data)
    return {"status": "queued"}
```

> The example above uses **one shared queue**, so whichever client polls first takes the event. Real systems need a queue **per user**, shown in Example 3 below.

## Examples

### Example 1: Timeline Walkthrough (investigator waiting for a case)

An investigator has a dashboard open. They want to see "new case assigned" without refreshing.

```
Time     Client (browser)                     Server
10:00:00 GET /poll?timeout=30  ─────────────→ holds the request, nothing to send
10:00:12                                      Supervisor assigns Case #1001 to this user
10:00:12 ←── 200 {"id":101,"msg":"Case #1001 assigned"}
10:00:12 shows banner, immediately sends
         GET /poll?timeout=30  ─────────────→ holds again
10:00:42                                      30 s pass, nothing happened
10:00:42 ←── 200 {"data": null}               (timeout, not an error)
10:00:42 GET /poll?timeout=30  ─────────────→ holds again ...
```

Two ways a request ends: **data arrives** (respond early) or **timeout** (respond empty). Either way the client reconnects at once.

### Example 2: Client Loop (JavaScript)

```javascript
async function pollForever(userId) {
  let lastId = 0;          // last event this client has seen
  let backoff = 1000;      // retry delay after errors

  while (true) {
    try {
      const res = await fetch(`/poll?user=${userId}&after=${lastId}&timeout=30`);
      if (res.status === 200) {
        const { events } = await res.json();
        for (const e of events) {
          showNotification(e.msg);
          lastId = e.id;
        }
      }
      backoff = 1000;      // success (data or empty timeout) → reset
    } catch (err) {
      await new Promise(r => setTimeout(r, backoff));   // network error → wait
      backoff = Math.min(backoff * 2, 30000);           // exponential backoff, max 30 s
    }
  }
}
```
- An **empty timeout response** is normal, just loop again
- On a **network error**, back off, otherwise a server outage turns every client into a tight retry loop

### Example 3: Server With a Queue Per User (no missed events)

Problem with the simple version: events that arrive **between** two polls (while the client is reconnecting) are lost. Fix: store events with an increasing ID and let the client say "give me everything after ID X".

```python
import asyncio
from collections import defaultdict
from fastapi import FastAPI

app = FastAPI()

events: dict[str, list[dict]] = defaultdict(list)       # user_id → ordered events
waiters: dict[str, asyncio.Event] = defaultdict(asyncio.Event)
next_id = 0

@app.post("/notify/{user_id}")
async def notify(user_id: str, body: dict):
    global next_id
    next_id += 1
    events[user_id].append({"id": next_id, "msg": body["msg"]})
    waiters[user_id].set()                               # wake the waiting request
    return {"id": next_id}

@app.get("/poll")
async def poll(user: str, after: int = 0, timeout: int = 30):
    def pending():
        return [e for e in events[user] if e["id"] > after]

    if pending():                                        # missed while reconnecting
        return {"events": pending()}

    waiters[user].clear()
    try:
        await asyncio.wait_for(waiters[user].wait(), timeout)
    except asyncio.TimeoutError:
        return {"events": []}
    return {"events": pending()}
```

Walkthrough:
```
Client polls with after=0   → nothing pending → waits
POST /notify/u1 {"msg":"A"} → id 1 stored, waiter woken → client gets [A]
POST /notify/u1 {"msg":"B"} → id 2 stored while client is reconnecting
Client polls with after=1   → B is already pending → returned instantly (nothing lost)
```
In production, keep events in **Redis** (list or stream) or a DB with a TTL instead of process memory, so any server instance can answer the poll.

### Example 4: Payment Status Page

User pays via UPI and lands on "Waiting for payment confirmation…". The page needs to flip to "Success" as soon as the bank confirms, which may take 2–60 seconds.

```
Browser: GET /payments/789/status?wait=25 ────────→ server holds
         (bank callback arrives at 8 s)
         ←── 200 {"status":"SUCCESS"}           → page shows tick, loop stops
```
```python
@app.get("/payments/{pid}/status")
async def payment_status(pid: str, wait: int = 25):
    deadline = asyncio.get_event_loop().time() + wait
    while asyncio.get_event_loop().time() < deadline:
        status = await db.get_status(pid)
        if status != "PENDING":
            return {"status": status}          # final state → respond immediately
        await asyncio.sleep(0.5)
    return {"status": "PENDING"}               # client polls again
```
Here the loop stops once the status is final. The server checks the DB every 500 ms, simple, but production systems would wake on a pub/sub message instead.

### Example 5: Other Typical Uses

| Scenario | What the client waits for |
|---|---|
| Ride-hailing screen | "Driver has accepted" / "Driver arrived" |
| Chat widget fallback | New message when WebSocket is blocked by a proxy |
| Admin dashboard | New alert or job finished |
| CI/CD page | Build status change |
| Order tracking | "Your order has been shipped" |
| Case management tool | "A new AML case has been assigned to you" |

## Practical Settings

| Setting | Typical value | Why |
|---|---|---|
| Server hold timeout | 20–30 s | Stay below proxy/load-balancer idle timeouts (often 60 s), or they cut the request |
| Client request timeout | A few seconds above the server timeout | Avoid the client giving up first |
| Reconnect on success | Immediately | Minimises the gap where events could be missed |
| Reconnect on error | Exponential backoff + jitter | Prevents a thundering herd after an outage |
| Event IDs (`after=`) | Always | No lost or duplicate events |

## Pros and Cons

| Pros | Cons |
|---|---|
| Works everywhere (plain HTTP) | Server holds open connections (resource cost) |
| No special infrastructure | Not truly real-time — gap between response and reconnect |
| Firewall/proxy friendly | Each client = one hanging request on server |
| Simple to implement | Not efficient at scale (thousands of clients) |

## vs Other Real-Time Techniques

| Technique | Connection | Latency | Server Load | Use Case |
|---|---|---|---|---|
| **Long Polling** | HTTP, reconnects | Low-medium | Medium | Simple notifications |
| **SSE** | HTTP persistent | Low | Low | Server→client stream |
| **WebSocket** | Persistent TCP | Very low | Low | Bidirectional chat |
| **Short Polling** | HTTP, frequent | High | High | Avoid this |

## When to Use
- Notification systems where real-time is "nice to have" not critical
- Environments where WebSockets are blocked (strict proxies/firewalls)
- Simple use cases where SSE/WebSocket setup overhead is not justified
