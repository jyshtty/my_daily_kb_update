# API Gateway

## What It Is
A single entry point for clients that routes requests to backend services and handles **cross-cutting concerns** in one place.

```
Mobile ─┐
Web ────┼──→ API Gateway ──→ User Service
3rd party┘    (auth, rate limit,   Order Service
               routing, TLS...)    Payment Service
```

## Responsibilities

| Concern | What the gateway does |
|---|---|
| **Routing** | `/users/*` → user-service, `/orders/*` → order-service |
| **Authentication** | Validate JWT / API key / OAuth token once at the edge |
| **Authorization** | Coarse checks (scopes, roles) |
| **Rate limiting & throttling** | Per client / per key limits (see `004_rate_limiting.md`) |
| **TLS termination** | HTTPS at the edge, plain or mTLS inside |
| **Request / response transformation** | Header rewrite, protocol translation (REST ↔ gRPC) |
| **Aggregation** | Combine several service calls into one response |
| **Caching** | Cache common GET responses |
| **Observability** | Central logging, metrics, tracing, correlation IDs |
| **Resilience** | Timeouts, retries, circuit breaking |
| **Versioning** | `/v1`, `/v2` routing |

## API Gateway vs Load Balancer vs Reverse Proxy

| | Load Balancer | Reverse Proxy | API Gateway |
|---|---|---|---|
| Main job | Spread traffic across instances of **one** service | Sit in front of servers, hide them | Manage and route **APIs** across many services |
| Layer | L4 / L7 | L7 | L7 |
| Knows API semantics? | No | Little | Yes (auth, quotas, versions) |
| Examples | AWS ELB/ALB, HAProxy | NGINX, Envoy | Kong, AWS API Gateway, Apigee, Envoy-based |

Typical stack: `Client → CDN → Load Balancer → API Gateway → services (each with its own LB)`. Tools overlap — NGINX and Envoy can play all three roles.

## Backend for Frontend (BFF)
One gateway per client type, so each gets a tailored API.
```
Mobile app ──→ Mobile BFF ──┐
Web app ─────→ Web BFF ─────┼──→ Microservices
Partner API ─→ Public GW ───┘
```
Avoids one bloated "one-size-fits-all" gateway.

## Request Flow Example
1. Client sends `GET /orders/42` with a JWT
2. Gateway terminates TLS, validates the JWT, checks rate limit
3. Looks up route, forwards to order-service with user ID header
4. Order service replies; gateway adds headers, logs, returns response

## Pitfalls

| Pitfall | Mitigation |
|---|---|
| **Single point of failure** | Run multiple gateway instances behind an LB, multi-AZ |
| **Extra latency hop** | Keep it thin, avoid heavy logic |
| **Bottleneck / scaling limit** | Stateless gateways, scale horizontally |
| **Business logic creeping in** | Gateway = cross-cutting only; logic belongs in services |
| **Tight coupling to every team's release** | Declarative config, per-team ownership, BFF split |
| **Auth only at the edge** | Keep service-level authorization too (zero trust) |

## Gateway vs Service Mesh
- **Gateway**: north–south traffic (client → system)
- **Service mesh** (Istio, Linkerd): east–west traffic (service → service), mTLS, retries, telemetry via sidecars

## Interview Quick Answer
> An API gateway is the single entry point that routes client requests to microservices and centralizes auth, rate limiting, TLS, logging and transformation. It differs from a load balancer, which just spreads traffic across instances. Make it stateless and horizontally scaled to avoid a SPOF, keep business logic out of it, and consider a BFF per client type.

## Rule of Thumb
Put cross-cutting edge concerns in the gateway, keep business logic in services, and run the gateway as a redundant, stateless tier.
