# Storage

## The Problem
Container filesystems are ephemeral — a restart wipes them. Kubernetes separates **how storage is requested** from **how it is provided**.

```
Pod ──uses──→ PVC (request: 20Gi, RWO) ──bound to──→ PV (actual disk) ←── StorageClass (provisioner)
   developer                                           admin / dynamic          e.g. AWS EBS CSI
```

## Volume Types

| Type | Lifetime | Use |
|---|---|---|
| `emptyDir` | Pod | Scratch space, cache, sharing between containers (`medium: Memory` for tmpfs) |
| `configMap` / `secret` / `projected` | Pod | Config files |
| `hostPath` | Node | Node agents only — security risk, avoid for apps |
| `persistentVolumeClaim` | Independent of pod | Databases, durable data |
| Ephemeral CSI volumes | Pod | Per-pod scratch from a CSI driver |

## PersistentVolume (PV) and PersistentVolumeClaim (PVC)

| Object | Who creates | What |
|---|---|---|
| **PV** | Admin or dynamic provisioner | A piece of real storage (EBS volume, NFS share…) |
| **PVC** | Developer | A request: size, access mode, class |
| **StorageClass** | Admin | Template for dynamic provisioning |

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: data }
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: gp3
  resources: { requests: { storage: 20Gi } }
---
# in the pod spec
volumes:
  - name: data
    persistentVolumeClaim: { claimName: data }
```

## StorageClass
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: gp3 }
provisioner: ebs.csi.aws.com
parameters: { type: gp3 }
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```
- **Dynamic provisioning**: creating a PVC automatically creates the PV/disk
- `WaitForFirstConsumer`: provision in the zone where the pod is scheduled (avoids zone mismatch)
- A default StorageClass is used when `storageClassName` is omitted

## Access Modes

| Mode | Meaning | Typical backend |
|---|---|---|
| **RWO** ReadWriteOnce | Read-write by one **node** | Block: EBS, Persistent Disk, Azure Disk |
| **ROX** ReadOnlyMany | Read-only by many nodes | NFS, some block |
| **RWX** ReadWriteMany | Read-write by many nodes | NFS, EFS, Azure Files, CephFS |
| **RWOP** ReadWriteOncePod | Read-write by one **pod** | CSI volumes |

## Reclaim Policy

| Policy | When PVC is deleted |
|---|---|
| **Delete** (dynamic default) | PV and underlying disk deleted |
| **Retain** | PV kept (`Released`); admin must clean up / reuse manually |

Use `Retain` for important production data.

## CSI (Container Storage Interface)
Standard plugin interface for storage vendors. Provides provisioning, attach/mount, **resize**, and **VolumeSnapshots**.
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata: { name: data-snap }
spec:
  volumeSnapshotClassName: ebs-snap
  source: { persistentVolumeClaimName: data }
```

## StatefulSets and Storage
`volumeClaimTemplates` create one PVC per replica (`data-db-0`, `data-db-1`); a rescheduled pod reattaches its own PVC.

## Pitfalls

| Pitfall | Fix |
|---|---|
| PVC stuck `Pending` | No default StorageClass, wrong class name, or quota |
| Pod stuck: volume in another AZ | `WaitForFirstConsumer`, zone-aware scheduling |
| Expecting RWX from EBS | Use EFS/NFS for shared write |
| Data lost on PVC delete | `reclaimPolicy: Retain`, snapshots, backups (Velero) |
| `Multi-Attach error` during rollout with RWO | Use `Recreate` strategy or StatefulSet |

## Interview Quick Answer
> Pods request storage through PVCs, which bind to PVs representing real disks. StorageClasses enable dynamic provisioning through CSI drivers, with access modes (RWO, RWX, ROX, RWOP) and reclaim policies (Delete or Retain). StatefulSets give each replica its own PVC. I use `WaitForFirstConsumer` to avoid zone mismatch, Retain plus snapshots for important data.

## Rule of Thumb
Developers ask with PVCs, admins define StorageClasses, and anything you can't lose gets Retain, snapshots and backups.
