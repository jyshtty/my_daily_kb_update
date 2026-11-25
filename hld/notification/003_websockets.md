# WebSockets

## What is a WebSocket?

A persistent, full-duplex TCP connection between client and server.
Both sides can send messages at any time without re-establishing a connection.

```
HTTP (polling):  Client → request → Server → response → connection closed
WebSocket:       Client ←──── persistent connection ────→ Server
                              both can send anytime
```

## Handshake

WebSocket starts as an HTTP request, then upgrades:
```
Client: GET /ws HTTP/1.1
        Upgrade: websocket
        Connection: Upgrade

Server: HTTP/1.1 101 Switching Protocols
        Upgrade: websocket
```
After the 101 response, the protocol switches from HTTP to WebSocket frames.

## Implementation (FastAPI)

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import dict

app = FastAPI()

# Connection registry: user_id → WebSocket
connections: dict[str, WebSocket] = {}

@app.websocket("/ws/{user_id}")
async def websocket_endpoint(websocket: WebSocket, user_id: str):
    await websocket.accept()
    connections[user_id] = websocket
    try:
        while True:
            data = await websocket.receive_text()   # blocks until message arrives
            await websocket.send_text(f"echo: {data}")
    except WebSocketDisconnect:
        del connections[user_id]

# Send to a specific user from anywhere in the app
async def notify_user(user_id: str, message: str):
    ws = connections.get(user_id)
    if ws:
        await ws.send_text(message)
```

## Scaling WebSockets Across Multiple Servers

Single server: in-memory dict works fine.
Multiple servers: a user's socket lives on one server — other servers can't reach it directly.

**Solution: Redis Pub/Sub**
```
Server 1 (user A connected)     Server 2 (user B connected)
         ↑                               ↑
         └──────── Redis Pub/Sub ────────┘
                         ↑
               Event publisher (any service)
```

```python
import aioredis

redis = aioredis.from_url("redis://localhost")

# Publisher (any server)
async def publish_event(user_id: str, message: str):
    await redis.publish(f"user:{user_id}", message)

# Subscriber (each server runs this on startup)
async def listen():
    pubsub = redis.pubsub()
    await pubsub.psubscribe("user:*")
    async for msg in pubsub.listen():
        if msg["type"] == "pmessage":
            user_id = msg["channel"].split(":")[1]
            ws = connections.get(user_id)
            if ws:
                await ws.send_text(msg["data"])
```

## WebSocket vs SSE vs Long Polling

| | WebSocket | SSE | Long Polling |
|---|---|---|---|
| Direction | Bidirectional | Server → Client only | Client → Server (pull) |
| Protocol | WS (TCP upgrade) | HTTP persistent | HTTP reconnect |
| Latency | Lowest | Low | Medium |
| Firewall friendly | Sometimes blocked | Yes | Yes |
| Browser support | All modern | All modern | All |
| Use case | Chat, gaming, live collab | Live feed, notifications | Simple notifications |

## Message Types
WebSocket frames can carry:
- **Text** — JSON payloads (most common)
- **Binary** — images, files, protobuf
- **Ping/Pong** — keepalive (handled by most frameworks automatically)

## Heartbeat / Keepalive
Idle connections get dropped by load balancers and proxies.
Send periodic pings to keep the connection alive:
```python
import asyncio

async def heartbeat(websocket: WebSocket):
    while True:
        await asyncio.sleep(30)
        await websocket.send_text("ping")
```

## Common Pitfalls

| Pitfall | Fix |
|---|---|
| No reconnect logic on client | Exponential backoff reconnect |
| Storing sockets in-memory on multi-server setup | Redis Pub/Sub |
| No auth on WS endpoint | Validate token on handshake (HTTP headers) |
| Sending to disconnected socket | Wrap sends in try/except, clean up registry |
| Load balancer drops idle connections | Set idle timeout > heartbeat interval |

## When to Use WebSockets
- Chat applications
- Live collaboration (Google Docs-style)
- Real-time dashboards (stock prices, live metrics)
- Multiplayer games
- Any feature requiring sub-second bidirectional communication
