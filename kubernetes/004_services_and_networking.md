# Services and Networking

## Networking Model
- Every pod gets its **own IP**
- All pods can reach all pods **without NAT** (unless NetworkPolicy restricts it)
- Implemented by a **CNI plugin**: Calico, Cilium, Flannel, AWS VPC CNI, Azure CNI

Pod IPs change on every restart, so you need a **Service** for a stable address.

## Service
A stable virtual IP + DNS name that load-balances to pods matching a label selector.
```
client → Service "api" (10.96.0.15:80) ──→ pod 10.1.0.4:8080
                                       ├─→ pod 10.1.1.7:8080
                                       └─→ pod 10.1.2.9:8080
                 (endpoints = ready pods matching selector)
```
```yaml
apiVersion: v1
kind: Service
metadata: { name: api }
spec:
  type: ClusterIP
  selector: { app: api }
  ports: [{ port: 80, targetPort: 8080 }]
```

## Service Types

| Type | Reachable from | How |
|---|---|---|
| **ClusterIP** (default) | Inside cluster | Virtual IP |
| **NodePort** | Outside, via `<NodeIP>:30000–32767` | Opens a port on every node |
| **LoadBalancer** | Internet / VPC | Cloud provisions an external LB → NodePort/pods |
| **ExternalName** | Inside cluster | DNS CNAME to an external host; no proxying |
| **Headless** (`clusterIP: None`) | Inside cluster | DNS returns pod IPs directly; used by StatefulSets |

## kube-proxy Modes

| Mode | Notes |
|---|---|
| **iptables** (default) | Rules per service/endpoint; random backend; O(n) rule chains at large scale |
| **IPVS** | Kernel load balancer; hash tables, more LB algorithms; better at many services |
| **nftables** (newer) | Successor to iptables mode |
| **eBPF replacement** (Cilium) | No kube-proxy; fastest, richer observability |

Session affinity: `sessionAffinity: ClientIP`.
`externalTrafficPolicy: Local` preserves client IP and avoids an extra hop, but only nodes with pods receive traffic.

## DNS (CoreDNS)
```
<service>.<namespace>.svc.cluster.local      → ClusterIP
<pod-name>.<service>.<namespace>.svc.cluster.local → pod IP (headless/StatefulSet)
```
Within the same namespace, just `api` works. Note the `ndots:5` default can cause extra lookups for external names — use FQDNs with a trailing dot for hot paths.

## EndpointSlices
Controllers track **ready** pod IPs for each Service in EndpointSlices. Pods failing readiness are removed → no traffic.

## NetworkPolicy
Firewall rules for pods. Enforced only if the CNI supports it (Calico, Cilium — not plain Flannel).
Once a pod is selected by any policy, traffic not allowed is **denied**.

```yaml
# Default deny all ingress in namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny }
spec:
  podSelector: {}
  policyTypes: [Ingress]
---
# Allow frontend → api on 8080
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-frontend }
spec:
  podSelector: { matchLabels: { app: api } }
  ingress:
    - from: [{ podSelector: { matchLabels: { app: frontend } } }]
      ports: [{ port: 8080 }]
```
Remember egress to DNS (UDP/TCP 53) when adding default-deny egress.

## Pitfalls

| Pitfall | Fix |
|---|---|
| Selector typo → Service has no endpoints | `kubectl get endpointslices -l kubernetes.io/service-name=api` |
| `port` vs `targetPort` confusion | `port` = Service, `targetPort` = container |
| One LoadBalancer Service per app (cost) | Use Ingress / Gateway |
| NetworkPolicy "not working" | CNI doesn't enforce it |
| Long-lived connections (gRPC) not balanced | L7 LB / service mesh / client-side balancing |

## Interview Quick Answer
> Every pod gets its own IP via the CNI, but pod IPs are ephemeral, so a Service provides a stable virtual IP and DNS name that load-balances to ready pods. ClusterIP is internal, NodePort exposes a port on each node, LoadBalancer provisions a cloud LB, and headless Services return pod IPs for StatefulSets. kube-proxy implements this with iptables/IPVS, CoreDNS handles names, and NetworkPolicies restrict traffic.

## Rule of Thumb
ClusterIP for internal traffic, Ingress/Gateway for HTTP from outside, and default-deny NetworkPolicies per namespace.
