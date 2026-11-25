# Push Notifications

## What are Push Notifications?

Server **initiates** sending data to the client — client does not need to ask.
Opposite of polling where the client asks repeatedly.

## Types of Push

### 1. Server-Sent Events (SSE)
One-way stream from server to client over HTTP.
```
Client: GET /stream (keep-alive connection)
Server: keeps sending events as they happen
        data: {"type": "notification", "msg": "New message"}
        data: {"type": "notification", "msg": "Order shipped"}
```

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import asyncio

app = FastAPI()

async def event_stream(user_id: str):
    while True:
        event = await get_next_event(user_id)
        yield f"data: {event}\n\n"

@app.get("/stream/{user_id}")
async def stream(user_id: str):
    return StreamingResponse(event_stream(user_id), media_type="text/event-stream")
```

### 2. WebSocket (Bidirectional)
Full-duplex connection — both server and client can send at any time.
```python
from fastapi import WebSocket

@app.websocket("/ws/{user_id}")
async def websocket_endpoint(websocket: WebSocket, user_id: str):
    await websocket.accept()
    register(user_id, websocket)
    try:
        while True:
            data = await websocket.receive_text()  # receive from client
            await websocket.send_text(f"echo: {data}")  # send to client
    except:
        unregister(user_id)
```

### 3. Mobile Push (FCM / APNs)
Platform-managed push for mobile apps — server sends to Firebase/Apple, they deliver.
```python
import firebase_admin
from firebase_admin import messaging

message = messaging.Message(
    notification=messaging.Notification(title="New message", body="You have a reply"),
    token=device_token,  # stored when user installs app
)
messaging.send(message)
```

## Comparison

| | SSE | WebSocket | Mobile Push (FCM) |
|---|---|---|---|
| Direction | Server → Client | Bidirectional | Server → Device |
| Protocol | HTTP | WS (TCP) | HTTPS to FCM/APNs |
| Works when app closed? | No | No | Yes |
| Firewall friendly | Yes (HTTP) | Sometimes blocked | Yes |
| Use case | Live feed, notifications | Chat, gaming | Mobile alerts |

## Architecture for Scale

```
Event (order placed, message sent)
    → Publish to Kafka/Redis Pub-Sub
        → Notification Service (subscribed)
            → WebSocket manager (in-memory user→socket map)
                → send to connected user
            → FCM/APNs (if user is offline/mobile)
```

## Key Decisions

1. **Connection registry** — where do you store user_id → socket mapping?
   - Single server: in-memory HashMap
   - Multi-server: Redis Pub-Sub (each server subscribes, broadcasts to local sockets)

2. **Offline users** — what happens if user is not connected?
   - Store in DB / notification inbox
   - Deliver on next connection or via mobile push

3. **At-least-once vs at-most-once**
   - Use message IDs + client-side dedup for reliability

## Rule of Thumb
SSE for simple server-to-client feeds. WebSocket for interactive real-time features.
Mobile push for alerts when the app is closed.
