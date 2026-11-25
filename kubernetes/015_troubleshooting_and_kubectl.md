# Troubleshooting and kubectl

## Debugging Flow
```
kubectl get pods  →  status?
   ├─ Pending            → describe pod → scheduling / PVC / quota
   ├─ ImagePullBackOff   → image name, tag, registry auth
   ├─ CrashLoopBackOff   → logs --previous, exit code, probes, config
   ├─ Running, not Ready → readiness probe, dependencies
   └─ Running & Ready but no traffic → Service selector, endpoints, Ingress, NetworkPolicy
```

## Common Failures

| Symptom | Likely causes | Check |
|---|---|---|
| **Pending** | Insufficient CPU/memory, taints with no toleration, affinity unmatched, PVC unbound, quota exceeded | `kubectl describe pod` → Events (`FailedScheduling`) |
| **ImagePullBackOff / ErrImagePull** | Wrong image/tag, private registry without `imagePullSecrets`, rate limits, no network | `describe pod`, verify image exists |
| **CrashLoopBackOff** | App error on start, missing env/config, bad command, liveness probe killing it, dependency unreachable | `kubectl logs --previous`, exit code |
| **OOMKilled** (exit 137) | Memory limit too low or leak | `describe pod` → Last State: `OOMKilled`; raise limit / fix leak |
| **CreateContainerConfigError** | Missing ConfigMap/Secret or key | `describe pod` |
| **Running but not Ready** | Readiness probe failing | Events, probe endpoint |
| **Evicted** | Node memory/disk pressure | `describe node`, requests too low |
| **Terminating forever** | Finalizers, stuck volume detach, node gone | Check `metadata.finalizers` |
| **Node NotReady** | kubelet down, disk/memory pressure, network | `describe node`, kubelet logs |

### Exit Codes
| Code | Meaning |
|---|---|
| 0 | Completed successfully |
| 1 | App error |
| 126 / 127 | Command not executable / not found |
| 137 | SIGKILL (often OOMKilled) |
| 139 | Segfault |
| 143 | SIGTERM (graceful stop) |

## Service Not Reachable
```bash
kubectl get svc api -o wide
kubectl get endpointslices -l kubernetes.io/service-name=api    # empty → selector/readiness issue
kubectl run tmp --rm -it --image=busybox -- sh
  wget -qO- http://api.shop.svc.cluster.local
  nslookup api.shop
kubectl get networkpolicy -n shop
kubectl port-forward svc/api 8080:80                            # bypass Ingress
```

## Interactive Debugging
```bash
kubectl exec -it api-xyz -- sh
kubectl debug -it api-xyz --image=nicolaka/netshoot --target=api   # ephemeral container (distroless images)
kubectl debug node/worker-1 -it --image=busybox                     # shell on a node
```

## kubectl Cheat Sheet

### Context and namespaces
```bash
kubectl config get-contexts
kubectl config use-context prod
kubectl config set-context --current --namespace=shop
```

### Inspect
```bash
kubectl get pods -o wide -A
kubectl get pods -l app=api -w
kubectl get deploy,svc,ingress -n shop
kubectl describe pod api-xyz
kubectl get pod api-xyz -o yaml
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'
kubectl get events --sort-by=.lastTimestamp
kubectl top pods --sort-by=cpu
kubectl api-resources
kubectl explain deployment.spec.strategy
```

### Change
```bash
kubectl apply -f app.yaml            # declarative (preferred)
kubectl diff -f app.yaml             # preview changes
kubectl delete -f app.yaml
kubectl scale deploy/api --replicas=5
kubectl set image deploy/api api=myorg/api:1.4.3
kubectl rollout restart deploy/api
kubectl rollout undo deploy/api
kubectl edit deploy/api              # quick fixes only — drifts from Git
```

### Nodes
```bash
kubectl cordon worker-1              # stop new pods
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data
kubectl uncordon worker-1
kubectl taint nodes worker-1 key=value:NoSchedule
```

### Generate YAML
```bash
kubectl create deploy api --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl create secret generic db --from-literal=pw=x --dry-run=client -o yaml
```

### Access
```bash
kubectl auth can-i create pods -n shop
kubectl port-forward pod/api-xyz 8080:8080
kubectl cp api-xyz:/tmp/dump.txt ./dump.txt
```

Handy tools: **k9s** (TUI), **kubectx/kubens**, **stern** (multi-pod logs), **kubectl-neat**.

## Pitfalls

| Pitfall | Fix |
|---|---|
| Running commands against the wrong cluster | Show context in prompt; separate kubeconfigs |
| `kubectl edit` in prod | Change Git and re-apply |
| Deleting pods to "fix" without finding cause | Read logs/events first |
| Ignoring Events | `describe` almost always tells you why |

## Interview Quick Answer
> I start with `kubectl get pods` to see the status, then `kubectl describe` for events and `kubectl logs --previous` for crashes. Pending means a scheduling problem (resources, taints, affinity, PVC), ImagePullBackOff means an image or registry auth problem, CrashLoopBackOff means the app or a probe is failing, and exit 137 usually means OOMKilled. For traffic issues I check the Service's endpoints, DNS from a debug pod, NetworkPolicies and the Ingress, and I use `kubectl debug` for distroless images.

## Rule of Thumb
`get` → `describe` → `logs` → `exec`/`debug`: the Events section usually tells you the answer.
