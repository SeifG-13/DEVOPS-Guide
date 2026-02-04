# Kubernetes Volumes

## Volume Types Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes Volumes                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Ephemeral Volumes                                             │
│   ├── emptyDir       (pod lifetime)                             │
│   └── configMap/secret (configuration)                          │
│                                                                  │
│   Persistent Volumes                                            │
│   ├── hostPath       (node filesystem)                          │
│   ├── nfs            (network filesystem)                       │
│   └── PersistentVolumeClaim (abstracted storage)               │
│                                                                  │
│   Cloud Provider Volumes                                        │
│   ├── awsElasticBlockStore                                     │
│   ├── gcePersistentDisk                                        │
│   └── azureDisk                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## emptyDir

Temporary storage that exists for pod's lifetime.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: cache-volume
      mountPath: /cache
  - name: sidecar
    image: busybox
    volumeMounts:
    - name: cache-volume
      mountPath: /data
  volumes:
  - name: cache-volume
    emptyDir: {}
```

### emptyDir with Memory

```yaml
volumes:
- name: tmpfs-volume
  emptyDir:
    medium: Memory      # Use RAM instead of disk
    sizeLimit: 100Mi    # Limit size
```

## hostPath

Mounts file or directory from host node.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: host-data
      mountPath: /data
  volumes:
  - name: host-data
    hostPath:
      path: /var/data
      type: DirectoryOrCreate
```

### hostPath Types

| Type | Description |
|------|-------------|
| `""` (empty) | No checks |
| `DirectoryOrCreate` | Create directory if not exists |
| `Directory` | Directory must exist |
| `FileOrCreate` | Create file if not exists |
| `File` | File must exist |
| `Socket` | Unix socket must exist |

> **Warning**: hostPath is node-specific and not portable!

## Persistent Volumes (PV) and Claims (PVC)

```
┌─────────────────────────────────────────────────────────────────┐
│                 PV and PVC Relationship                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Administrator                    Developer                    │
│        │                               │                        │
│        ▼                               ▼                        │
│   ┌───────────────┐           ┌───────────────┐                │
│   │ Persistent    │           │ Persistent    │                │
│   │ Volume (PV)   │◄─────────►│ Volume Claim  │                │
│   │               │   Bind    │ (PVC)         │                │
│   │ 10Gi storage  │           │ Request 5Gi   │                │
│   └───────────────┘           └───────────────┘                │
│        │                               │                        │
│        │                               ▼                        │
│        │                        ┌───────────────┐              │
│        │                        │     Pod       │              │
│        └───────────────────────►│  volumeMounts │              │
│                                 └───────────────┘              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-storage
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
```

### PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-storage
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual
```

### Using PVC in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: pvc-storage
```

## Access Modes

| Mode | Abbreviation | Description |
|------|--------------|-------------|
| ReadWriteOnce | RWO | Single node read-write |
| ReadOnlyMany | ROX | Multiple nodes read-only |
| ReadWriteMany | RWX | Multiple nodes read-write |
| ReadWriteOncePod | RWOP | Single pod read-write |

## Reclaim Policies

| Policy | Description |
|--------|-------------|
| Retain | Keep PV after PVC deletion (manual cleanup) |
| Delete | Delete PV when PVC is deleted |
| Recycle | Clear data and make PV available (deprecated) |

## Storage Classes

Dynamic provisioning of PersistentVolumes.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iopsPerGB: "10"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### Using StorageClass

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-storage  # Uses StorageClass
  resources:
    requests:
      storage: 10Gi
```

### Default StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
```

## NFS Volume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: nfs-server.example.com
    path: /exports/data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
```

## Cloud Provider Volumes

### AWS EBS

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: aws-ebs-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  awsElasticBlockStore:
    volumeID: vol-0123456789abcdef0
    fsType: ext4
```

### GCE Persistent Disk

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gce-pd-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  gcePersistentDisk:
    pdName: my-disk
    fsType: ext4
```

### Azure Disk

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: azure-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  azureDisk:
    diskName: myDisk
    diskURI: /subscriptions/.../myDisk
```

## Volume Snapshots

```yaml
# VolumeSnapshotClass
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapclass
driver: ebs.csi.aws.com
deletionPolicy: Delete
---
# VolumeSnapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: my-snapshot
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: my-pvc
---
# Restore from snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc
spec:
  dataSource:
    name: my-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## Projected Volumes

Combine multiple volume sources into one mount.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: projected-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: all-in-one
      mountPath: /projected-volume
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: mysecret
          items:
          - key: username
            path: my-group/my-username
      - configMap:
          name: myconfigmap
          items:
          - key: config
            path: my-group/my-config
      - downwardAPI:
          items:
          - path: labels
            fieldRef:
              fieldPath: metadata.labels
```

## CSI Volumes

Container Storage Interface for third-party storage.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: csi-storage-class
  resources:
    requests:
      storage: 10Gi
```

## Volume Management Commands

```bash
# List PersistentVolumes
kubectl get pv

# List PersistentVolumeClaims
kubectl get pvc

# Describe PV
kubectl describe pv pv-storage

# Describe PVC
kubectl describe pvc pvc-storage

# List StorageClasses
kubectl get storageclass
kubectl get sc

# Delete PVC
kubectl delete pvc pvc-storage
```

## Complete Example

```yaml
# StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
---
# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 20Gi
---
# Deployment using PVC
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
```

## Quick Reference

### Volume Types

| Type | Lifetime | Use Case |
|------|----------|----------|
| emptyDir | Pod | Scratch space, shared data |
| hostPath | Node | Node-specific data |
| PVC | Cluster | Persistent application data |
| configMap | Pod | Configuration files |
| secret | Pod | Sensitive data |

### PV Status

| Status | Description |
|--------|-------------|
| Available | Free, not bound to PVC |
| Bound | Bound to PVC |
| Released | PVC deleted, not yet reclaimed |
| Failed | Failed automatic reclamation |

### Key Commands

| Command | Description |
|---------|-------------|
| `kubectl get pv` | List PersistentVolumes |
| `kubectl get pvc` | List PersistentVolumeClaims |
| `kubectl get sc` | List StorageClasses |
| `kubectl describe pv` | Show PV details |

---

**Previous:** [11-secrets.md](11-secrets.md) | **Next:** [13-statefulsets.md](13-statefulsets.md)
