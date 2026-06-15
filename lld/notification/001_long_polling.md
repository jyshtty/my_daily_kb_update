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
