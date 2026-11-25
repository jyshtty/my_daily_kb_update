# ConfigMaps and Secrets

## Why
Keep configuration **out of images** so the same image runs in dev, staging and prod.

| | ConfigMap | Secret |
|---|---|---|
| For | Non-sensitive config (URLs, flags, files) | Passwords, tokens, keys, TLS certs |
| Storage | Plain text in etcd | **base64-encoded** (not encrypted!) in etcd; encryption at rest optional |
| Size limit | 1 MiB | 1 MiB |
| Mounted as tmpfs | No | Yes (in memory on node) |

## ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: app-config }
data:
  LOG_LEVEL: info
  FEATURE_X: "true"
  app.properties: |
    cache.ttl=300
    timeout=5s
```

## Secret
```yaml
apiVersion: v1
kind: Secret
metadata: { name: db-creds }
type: Opaque
stringData:                # plain values; stored as base64 under data:
  username: app
  password: s3cr3t
```
```bash
kubectl create secret generic db-creds --from-literal=password=s3cr3t
kubectl create secret tls example-tls --cert=tls.crt --key=tls.key
```

Secret types: `Opaque`, `kubernetes.io/tls`, `kubernetes.io/dockerconfigjson` (image pull), `kubernetes.io/service-account-token`.

## Consuming Them

```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      envFrom:
        - configMapRef: { name: app-config }        # all keys as env vars
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef: { name: db-creds, key: password }
      volumeMounts:
        - { name: cfg, mountPath: /etc/app, readOnly: true }
  volumes:
    - name: cfg
      configMap: { name: app-config }
```

| Method | Updates when ConfigMap/Secret changes? |
|---|---|
| Env vars | **No** — only on pod restart |
| Volume mount | **Yes**, eventually (kubelet sync, ~1 min) — app must re-read the file |
| `subPath` mount | **No** |

Trigger rollouts on config change: hash of config in a pod annotation (Helm `checksum/config`), or tools like Reloader.

## Immutable
`immutable: true` prevents accidental changes and reduces API server watch load. Change = create a new object (e.g. `app-config-v2`).

## Securing Secrets

| Practice | Why |
|---|---|
| **Encryption at rest** (`EncryptionConfiguration`, KMS provider) | base64 is not encryption |
| **RBAC**: restrict `get/list/watch` on secrets | `list` reveals all secret values |
| External secret stores: **External Secrets Operator**, **Secrets Store CSI Driver** (Vault, AWS Secrets Manager, Azure Key Vault) | Central rotation, audit |
| GitOps: **Sealed Secrets** or **SOPS** | Never commit plain Secrets to Git |
| Prefer volume mounts over env vars | Env vars leak into logs, crash dumps, `/proc` |
| Workload identity (IRSA, GKE WI) | Avoid static cloud credentials entirely |

## Pitfalls

| Pitfall | Fix |
|---|---|
| Thinking Secrets are encrypted | Enable encryption at rest + KMS |
| Secrets in Git | Sealed Secrets / SOPS / external store |
| Config changed but app didn't pick it up | Restart via checksum annotation |
| Huge config blobs | 1 MiB limit; use object storage |
| Pod fails: referenced key missing | `optional: true` or create it first |

## Interview Quick Answer
> ConfigMaps hold non-sensitive configuration and Secrets hold credentials; both are injected as env vars or mounted files. Secrets are only base64-encoded, so I enable encryption at rest with KMS, lock them down with RBAC, and usually sync them from Vault or a cloud secret manager with External Secrets Operator. Env vars don't update live, so config changes trigger a rollout via a checksum annotation.

## Rule of Thumb
Config in ConfigMaps, secrets in an external store synced into Secrets, and never commit plaintext secrets.
