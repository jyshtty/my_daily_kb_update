# Ingress and Gateway API

## Why
A `LoadBalancer` Service per app is costly and has no HTTP routing. **Ingress** (and its successor **Gateway API**) gives one entry point with host/path routing and TLS.

```
Internet → Cloud LB → Ingress controller (NGINX/Envoy/…)
                         ├─ api.example.com      → Service api
                         ├─ example.com/         → Service web
                         └─ example.com/images   → Service images
```

## Ingress
Ingress is **only a spec**; you need an **Ingress controller** running (ingress-nginx, Traefik, HAProxy, AWS Load Balancer Controller, GKE Ingress).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
spec:
  ingressClassName: nginx
  tls:
    - hosts: [example.com]
      secretName: example-tls
  rules:
    - host: example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend: { service: { name: api, port: { number: 80 } } }
          - path: /
            pathType: Prefix
            backend: { service: { name: web, port: { number: 80 } } }
```

| Feature | Ingress support |
|---|---|
| Host / path routing | Yes |
| TLS termination | Yes (Secret with cert; cert-manager automates) |
| Canary, header routing, rewrites, rate limits | **Controller-specific annotations** (not portable) |
| TCP/UDP, gRPC | Limited / annotations |

Note: the community ingress-nginx project is being retired (announced end of maintenance March 2026); new setups are moving to Gateway API implementations.

## Gateway API
The newer, role-oriented, more expressive standard (GA for HTTP routing since v1.0, 2023).

```
GatewayClass   (infra provider: which controller)      ← platform/infra team
   └─ Gateway  (listeners: ports, hostnames, TLS)      ← cluster operator
        └─ HTTPRoute / GRPCRoute / TLSRoute / TCPRoute  ← app teams
             └─ Services
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: api }
spec:
  parentRefs: [{ name: public-gw }]
  hostnames: [api.example.com]
  rules:
    - matches: [{ path: { type: PathPrefix, value: / } }]
      backendRefs:
        - { name: api-v1, port: 80, weight: 90 }
        - { name: api-v2, port: 80, weight: 10 }   # native canary split
```

## Ingress vs Gateway API

| | Ingress | Gateway API |
|---|---|---|
| Status | Stable, frozen | Actively developed standard |
| Role separation | One object | GatewayClass / Gateway / Routes |
| Traffic splitting, header match | Annotations | Built in |
| Protocols | HTTP(S) | HTTP, gRPC, TLS, TCP, UDP |
| Cross-namespace routing | Awkward | Built in with ReferenceGrant |
| Implementations | Many | Envoy Gateway, Istio, Cilium, NGINX Gateway Fabric, Kong, GKE, AWS |

## Pitfalls

| Pitfall | Fix |
|---|---|
| Ingress created but nothing happens | Install a controller; set `ingressClassName` |
| Relying on annotations → vendor lock-in | Prefer Gateway API |
| Manual certificate renewals | cert-manager |
| `pathType` surprises | Use `Prefix` or `Exact` explicitly |
| Ingress controller as SPOF | Multiple replicas, PDB, anti-affinity |

## Interview Quick Answer
> Ingress is an L7 routing spec implemented by an ingress controller, giving host/path routing and TLS behind a single load balancer. Gateway API is its successor: it splits responsibilities into GatewayClass, Gateway and Routes, and natively supports traffic splitting, header matching and more protocols. For new clusters I'd choose Gateway API.

## Rule of Thumb
One shared entry point per cluster, TLS via cert-manager, and Gateway API for anything new.
