# Helm and Kustomize

## The Problem
Same app, many environments (dev, staging, prod) with small differences: replicas, image tag, resources, hostnames. Copy-pasting YAML doesn't scale.

| | Helm | Kustomize |
|---|---|---|
| Approach | **Templating** (Go templates + values) | **Overlay / patching** of plain YAML |
| Packaging | Charts, versioned, shareable via repos/OCI | No packaging |
| Release tracking | Yes (revisions, rollback) | No (kubectl applies) |
| Built into kubectl | No | Yes (`kubectl apply -k`) |
| Learning curve | Templates can get complex | Simple, pure YAML |
| Best for | Distributing apps, 3rd-party software | Your own apps' env differences |

## Helm

### Chart Structure
```
mychart/
├── Chart.yaml          # name, version, appVersion, dependencies
├── values.yaml         # defaults
├── values-prod.yaml    # env overrides (convention)
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── _helpers.tpl
    └── NOTES.txt
```

### Template Example
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources: {{- toYaml .Values.resources | nindent 12 }}
```
```yaml
# values.yaml
replicaCount: 2
image: { repository: myorg/api, tag: "1.4.2" }
resources: { requests: { cpu: 100m, memory: 128Mi } }
```

### Commands
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install api ./mychart -n shop -f values-prod.yaml
helm upgrade --install api ./mychart -f values-prod.yaml --set image.tag=1.4.3
helm template ./mychart -f values-prod.yaml     # render locally
helm diff upgrade api ./mychart                 # (helm-diff plugin)
helm history api && helm rollback api 3
helm uninstall api
```
Releases are stored as Secrets in the namespace. Helm 3 has no Tiller.

## Kustomize

### Layout
```
k8s/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/kustomization.yaml
    └── prod/
        ├── kustomization.yaml
        └── replicas-patch.yaml
```

```yaml
# base/kustomization.yaml
resources: [deployment.yaml, service.yaml]
```
```yaml
# overlays/prod/kustomization.yaml
resources: [../../base]
namespace: shop-prod
images:
  - name: myorg/api
    newTag: "1.4.2"
patches:
  - path: replicas-patch.yaml
configMapGenerator:
  - name: app-config
    literals: [LOG_LEVEL=warn]      # name gets a hash suffix → rollout on change
```
```bash
kubectl kustomize overlays/prod        # render
kubectl apply -k overlays/prod
```

## Using Them Together
- Helm for third-party charts (ingress controller, Prometheus, cert-manager)
- Kustomize for your own services, or to patch rendered Helm output
- **GitOps**: Argo CD and Flux support both natively and continuously sync from Git

## Pitfalls

| Pitfall | Fix |
|---|---|
| Huge, unreadable Helm templates | Keep charts simple; library charts for shared bits |
| `--set` flags in CI scripts drift from Git | Values files committed to Git |
| Not pinning chart versions | `--version` / lock dependencies |
| Secrets in values files | External Secrets / SOPS / Sealed Secrets |
| Manual `helm upgrade` from laptops | GitOps (Argo CD / Flux) |

## Interview Quick Answer
> Helm packages Kubernetes apps as versioned charts with Go templates and values files, and tracks releases so you can upgrade and roll back. Kustomize keeps plain YAML and applies environment overlays and patches, built into kubectl. I typically use Helm for third-party software, Kustomize or simple charts for our own services, and Argo CD or Flux to apply everything from Git.

## Rule of Thumb
Helm to distribute, Kustomize to customize, Git as the source of truth.
