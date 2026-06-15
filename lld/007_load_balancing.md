# Load Balancing

## What is Load Balancing?
Distributing incoming traffic across multiple servers to avoid overloading any single one.

```
Clients → Load Balancer → [Server 1]
                        → [Server 2]
                        → [Server 3]
```

## Load Balancing Algorithms

### 1. Round Robin
Requests go to each server in turn, sequentially.
```
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 3
Request 4 → Server 1 (back to start)
```
- Simple, even distribution
- Bad if servers have different capacities or request weights

### 2. Weighted Round Robin
Servers get traffic proportional to their weight.
```python
servers = [("s1", 3), ("s2", 1)]  # s1 gets 3x traffic
# s1, s1, s1, s2, s1, s1, s1, s2...
```
- Use when servers have different specs (CPU/RAM)

### 3. Least Connections
Send to server with fewest active connections.
```python
def pick_server(servers):
    return min(servers, key=lambda s: s.active_connections)
```
- Better than round robin for long-lived connections (WebSockets, DB)
- Requires LB to track connection count per server

### 4. Least Response Time
Send to server with lowest avg response time + fewest connections.
- Best for latency-sensitive APIs
- Requires active health monitoring

### 5. IP Hash (Sticky Sessions)
Same client IP always goes to same server.
```python
server = servers[hash(client_ip) % len(servers)]
```
- Useful when session state is stored in-process (not recommended)
- Problem: uneven if many users share an IP (NAT)

### 6. Consistent Hashing
See `005_consistent_hashing.md` — distributes by key, survives node changes gracefully.

## Layer 4 vs Layer 7 Load Balancing

| | L4 (Transport) | L7 (Application) |
|---|---|---|
| Operates on | TCP/UDP | HTTP headers, URL, cookies |
| Speed | Faster | Slower (parses payload) |
| Routing logic | IP + port only | URL path, host, headers |
| SSL termination | No | Yes |
| Examples | AWS NLB, HAProxy TCP | AWS ALB, Nginx, Traefik |

```
L7 routing example:
  /api/*     → API servers
  /static/*  → CDN / static servers
  /ws/*      → WebSocket servers
```

## Health Checks
LB must stop sending traffic to unhealthy servers.

```
Active:  LB pings /health every 10s — remove if 3 consecutive failures
Passive: LB detects failed responses — remove after N errors
```

```python
# FastAPI health endpoint
@app.get("/health")
async def health():
    return {"status": "ok"}
```

## Session Persistence (Sticky Sessions)

Problem: user logs in on Server 1, next request hits Server 2 — session lost.

Solutions (in order of preference):
1. **Stateless sessions** — store in Redis/JWT, any server can handle any request ✓ best
2. **Cookie-based stickiness** — LB sets cookie to pin user to server
3. **IP hash** — same IP → same server (fragile)

## Common Load Balancer Tools

| Tool | Type | Use Case |
|---|---|---|
| **Nginx** | L7, software | Web traffic, reverse proxy |
| **HAProxy** | L4/L7, software | High performance, TCP+HTTP |
| **AWS ALB** | L7, managed | HTTP/HTTPS on AWS |
| **AWS NLB** | L4, managed | TCP/UDP, ultra-low latency |
| **Traefik** | L7, software | Kubernetes, auto-discovery |
| **Envoy** | L7, software | Service mesh (Istio) |

## Global Load Balancing (GeoDNS)
Route users to the nearest data center using DNS.
```
User in EU → DNS → EU data center
User in US → DNS → US data center
```
Tools: AWS Route 53 latency routing, Cloudflare Load Balancing.

## Key Rules
- Prefer stateless services — makes any LB algorithm work
- Use least connections for WebSocket/long-lived connections
- Always implement health checks — never send traffic to dead servers
- L7 LB for HTTP apps; L4 LB for raw TCP (databases, game servers)
