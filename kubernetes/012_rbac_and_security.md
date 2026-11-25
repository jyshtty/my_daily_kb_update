# RBAC and Security

## Layers of Defence
```
Supply chain (images) → API access (authN/authZ/admission) → Workload (pod security) → Network → Data (secrets)
```

## API Request Path
```
request → Authentication (who?) → Authorization (RBAC: allowed?) → Admission (mutate/validate) → etcd
```
- **AuthN**: client certs, OIDC tokens (SSO), cloud IAM (EKS access entries, GKE IAM), ServiceAccount tokens
- **AuthZ**: RBAC (plus Node authorizer, webhooks)
- **Admission**: Pod Security Admission, ValidatingAdmissionPolicy (CEL), Kyverno, OPA Gatekeeper

## ServiceAccounts
Identity for **pods** (users are external, not K8s objects).
```yaml
apiVersion: v1
kind: ServiceAccount
metadata: { name: api, namespace: shop }
automountServiceAccountToken: false     # opt in only where the app needs the API
```
- Tokens are short-lived, projected and audience-bound (since 1.22+)
- Map to cloud IAM with **workload identity**: IRSA / EKS Pod Identity, GKE Workload Identity, Azure Workload Identity — no static cloud keys

## RBAC

| Object | Scope | Purpose |
|---|---|---|
| **Role** | Namespace | Permissions within a namespace |
| **ClusterRole** | Cluster | Cluster-wide resources or reusable permission sets |
| **RoleBinding** | Namespace | Grant a Role/ClusterRole to subjects in one namespace |
| **ClusterRoleBinding** | Cluster | Grant a ClusterRole everywhere |

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: pod-reader, namespace: shop }
rules:
  - apiGroups: [""]
    resources: [pods, pods/log]
    verbs: [get, list, watch]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: api-reads-pods, namespace: shop }
subjects:
  - { kind: ServiceAccount, name: api, namespace: shop }
roleRef: { kind: Role, name: pod-reader, apiGroup: rbac.authorization.k8s.io }
```
```bash
kubectl auth can-i list secrets -n shop --as=system:serviceaccount:shop:api
```
RBAC is **additive only** — no deny rules.

Dangerous permissions: `secrets get/list`, `pods/exec`, `create pods` (can mount any SA), `escalate`, `bind`, `impersonate`, wildcards `*`, `cluster-admin`.

## Pod Security Standards (PSA)
Built-in admission controller (PodSecurityPolicy was removed in 1.25).

| Level | Allows |
|---|---|
| **Privileged** | Everything |
| **Baseline** | Blocks known privilege escalations (privileged, hostNetwork, hostPath…) |
| **Restricted** | Hardened: non-root, drop all caps, seccomp, no privilege escalation |

```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
```

## securityContext
```yaml
securityContext:                 # pod level
  runAsNonRoot: true
  runAsUser: 10001
  fsGroup: 10001
  seccompProfile: { type: RuntimeDefault }
containers:
  - name: api
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities: { drop: [ALL] }
```

## Secrets Protection
- Encryption at rest with a **KMS** provider
- Restrict secret access via RBAC; audit logs on secret reads
- External stores (Vault, cloud secret managers) — see `006_configmaps_and_secrets.md`

## Image and Supply Chain Security

| Practice | Tooling |
|---|---|
| Minimal base images (distroless, alpine) | Smaller attack surface |
| Scan images for CVEs in CI and registry | Trivy, Grype, ECR/Artifact Registry scanning |
| Sign and verify images | Sigstore cosign + Kyverno/policy-controller |
| Pin by digest, allow only trusted registries | Admission policies |
| SBOMs | Syft, SLSA provenance |

## Other Controls
- **NetworkPolicies**: default deny (see `004_services_and_networking.md`)
- **Audit logging** on the API server
- Runtime threat detection: Falco, Tetragon
- Keep the cluster and nodes patched; restrict API endpoint access (private endpoint)
- Benchmarks: CIS Kubernetes Benchmark (kube-bench)

## Pitfalls

| Pitfall | Fix |
|---|---|
| Apps using `default` SA with broad rights | Dedicated SA per app, least privilege |
| `cluster-admin` handed out widely | Namespaced Roles, break-glass process |
| Containers running as root | `runAsNonRoot`, Restricted PSA |
| Static cloud keys in Secrets | Workload identity |
| Unscanned / unsigned images | CI scanning, signature verification |

## Interview Quick Answer
> I secure Kubernetes in layers: OIDC/IAM authentication and least-privilege RBAC with a dedicated ServiceAccount per app; Pod Security Admission at Restricted with non-root, read-only filesystems and dropped capabilities; default-deny NetworkPolicies; secrets encrypted with KMS and sourced from Vault or a cloud secret manager; workload identity instead of static keys; and scanned, signed images enforced by admission policies like Kyverno.

## Rule of Thumb
Least privilege everywhere: minimal RBAC, non-root pods, default-deny network, and only trusted, signed images.
