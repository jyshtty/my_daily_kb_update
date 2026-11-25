# Kubernetes Architecture

## Big Picture
```
                    ┌──────────────── Control Plane ────────────────┐
kubectl / CI ──────→│ kube-apiserver ←→ etcd                         │
                    │      ↑      ↑                                  │
                    │ scheduler  controller-manager  cloud-controller│
                    └──────┬─────────────────────────────────────────┘
                           │ (watch / update via API server)
        ┌──────────────────┼──────────────────┐
   ┌────▼─────┐       ┌────▼─────┐       ┌────▼─────┐
   │ Node 1   │       │ Node 2   │       │ Node 3   │
   │ kubelet  │       │ kubelet  │       │ kubelet  │
   │ kube-proxy│      │ kube-proxy│      │ kube-proxy│
   │ runtime  │       │ runtime  │       │ runtime  │
   │ [pods]   │       │ [pods]   │       │ [pods]   │
   └──────────┘       └──────────┘       └──────────┘
```

## Control Plane Components

| Component | Role |
|---|---|
| **kube-apiserver** | Front door; REST API; authN, authZ, admission; the **only** component that talks to etcd |
| **etcd** | Consistent key-value store (Raft) holding all cluster state |
| **kube-scheduler** | Picks a node for each unscheduled pod (filter → score → bind) |
| **kube-controller-manager** | Runs controllers: Deployment, ReplicaSet, Node, Job, EndpointSlice, ServiceAccount… |
| **cloud-controller-manager** | Cloud integration: load balancers, node lifecycle, routes |

## Node Components

| Component | Role |
|---|---|
| **kubelet** | Agent on each node; ensures containers of assigned pods run; runs probes; reports status |
| **kube-proxy** | Programs iptables/IPVS rules so Service IPs reach pods (can be replaced by eBPF CNIs like Cilium) |
| **Container runtime** | Runs containers via CRI — containerd, CRI-O (dockershim removed in 1.24) |

Add-ons: CoreDNS, CNI plugin (Calico, Cilium, AWS VPC CNI), metrics-server, ingress controller.

## Reconciliation (Controller Pattern)
```
loop:
  desired = read spec from API server
  actual  = observe the world
  if actual != desired: act to close the gap
```
Everything is level-triggered and eventually consistent — controllers keep retrying until state matches.

## What Happens on `kubectl apply -f deploy.yaml`
1. kubectl sends the manifest to the **API server**
2. API server: authenticate → authorize (RBAC) → **admission** (mutating, then validating webhooks) → schema validation
3. Object persisted in **etcd**
4. **Deployment controller** sees new Deployment → creates a **ReplicaSet**
5. **ReplicaSet controller** creates **Pod** objects (no node yet)
6. **Scheduler** sees pending pods → chooses nodes → writes `nodeName`
7. **kubelet** on that node sees the pod → pulls image via runtime → starts containers → sets up networking via CNI and volumes via CSI
8. kubelet reports status; **EndpointSlice controller** adds ready pods to the Service

## High Availability
| Concern | Practice |
|---|---|
| API server | Multiple replicas behind a load balancer |
| etcd | 3 or 5 members (odd, quorum); regular backups |
| Scheduler / controller-manager | Multiple replicas, one active via **leader election** (Lease objects) |
| Managed offerings | EKS, GKE, AKS run the control plane for you |

## Pitfalls

| Pitfall | Fix |
|---|---|
| No etcd backups | Snapshot etcd regularly (`etcdctl snapshot save`) |
| Even number of etcd members | Use 3 or 5 |
| Heavy webhooks slowing every request | Timeouts, `failurePolicy`, scope with selectors |
| Treating pods as pets | Use controllers; pods are disposable |

## Interview Quick Answer
> The control plane has the API server (single entry point, backed by etcd), the scheduler (assigns pods to nodes) and the controller manager (reconciliation loops). Each node runs a kubelet that starts containers through the container runtime and kube-proxy that implements Services. On `kubectl apply` the API server validates and stores the object, controllers create ReplicaSets and Pods, the scheduler binds pods to nodes, and kubelets run them.

## Rule of Thumb
Everything goes through the API server, state lives in etcd, and controllers continuously reconcile desired vs actual.
