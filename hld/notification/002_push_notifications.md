# Push Notifications

## What are Push Notifications?

A push notification is a message that an application or backend server sends to a user's device **without the user having to open the application or manually request the information**.

Examples:
- Your banking app sends: "₹5,000 debited from your account."
- Gmail sends: "You received a new email."
- Amazon sends: "Your order has been shipped."
- Siron ACM could notify an investigator: "A new AML case has been assigned to you."

In system-design terms: the server **initiates** sending data to the client — the client does not need to ask.
This is the opposite of polling, where the client asks repeatedly ("anything new? anything new?").

```
Polling (client pulls)                Push (server pushes)

Client: "Any new case?"  → Server     Server: "New case assigned!" → Client
Client: "Any new case?"  → Server     (client did nothing, app may be closed)
Client: "Any new case?"  → Server
Server: "Yes, one."      → Client
```

## How It Actually Works (Plain Language)

The part people find confusing: **your backend does not talk to the phone directly.**
A phone with the app closed has no open connection to your server. Instead, the phone's operating system keeps **one shared connection** to Google (Android) or Apple (iOS), and your server hands the message to them.

Think of it like the post office:
- **Your backend** = the sender who writes the letter
- **FCM / APNs** = the post office (Google's Firebase Cloud Messaging, Apple Push Notification service)
- **Device token** = the home address of one specific app install on one specific phone
- **The phone's OS** = the postman who delivers it to the right app (and shows the banner)

### Step by step: "A new AML case has been assigned to you"

```
 ① Setup (once, when the investigator installs/logs in to the app)
    App ──asks OS for a device token──→ OS ──registers with──→ FCM/APNs
    App ←──────── device token (e.g. "dX3f...9a") ────────────┘
    App ──"Investigator #42's token is dX3f...9a"──→ Your Backend  (stored in DB)

 ② Something happens (any time later, app can be closed)
    Supervisor assigns Case #1001 to Investigator #42
        │
        ↓
    Backend looks up #42's device token(s) in the DB
        │
        ↓
    Backend ──HTTPS: {token, title, body}──→ FCM/APNs

 ③ Delivery
    FCM/APNs ──over the phone's always-on OS connection──→ Phone OS
    Phone OS shows banner: "New AML case assigned to you"

 ④ User taps the banner
    App opens and fetches the full case details from the backend API
```

Key points:
1. **Step ① happens once** — the token is saved against the user in your database.
2. **Step ② is triggered by an event in your system**, not by the user.
3. **The notification carries only a short message.** The app fetches the real data (the case) when opened — never put sensitive details in the payload.
4. **Tokens change** (reinstall, new phone, OS refresh), so apps re-send the token on each launch, and the backend removes tokens FCM reports as invalid.
5. **One user can have many devices** — store a list of tokens per user.

### Same idea inside a browser or open app
If the app/page is **open**, there's no need for the post office: the server can push directly over an already-open connection (SSE or WebSocket, covered below). Mobile push (FCM/APNs) exists for when the app is **closed**.

| Situation | What delivers the message |
|---|---|
| App/page is open | SSE or WebSocket (direct connection to your server) |
| App is closed or in the background | FCM (Android) / APNs (iOS) / Web Push (browsers) |
| User is offline entirely | Stored in a notification inbox, delivered on next connect |

## Key Terms

| Term | Meaning |
|---|---|
| **Device token / registration token** | Unique address of an app install, issued by FCM/APNs |
| **FCM** | Firebase Cloud Messaging — Google's push service (Android, iOS, web) |
| **APNs** | Apple Push Notification service — required for iOS delivery |
| **Payload** | The small JSON message (title, body, optional data) |
| **Notification message** | OS displays it automatically |
| **Data message** | Delivered silently to the app, which decides what to do |
| **Topic** | Named group (e.g. `aml-team`) so you can send to many devices at once |

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
