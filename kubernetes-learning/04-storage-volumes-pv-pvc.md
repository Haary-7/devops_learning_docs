# Storage: Volumes, PV, PVC, and StorageClass

## Storage in Kubernetes

Containers are ephemeral - data written inside a container is lost when the container restarts. Kubernetes provides multiple ways to persist data.

## Volume Types Overview

| Volume Type | Lifecycle | Use Case |
|-------------|-----------|----------|
| `emptyDir` | Pod lifetime | Temp space, caching |
| `hostPath` | Node lifetime | DaemonSets, accessing node files |
| `configMap` | Independent | Configuration files |
| `secret` | Independent | Sensitive data |
| `persistentVolumeClaim` | Independent | Persistent application data |
| `nfs` | Independent | Shared network storage |
| `cephfs` | Independent | Ceph distributed storage |
| `iscsi` | Independent | Block storage |
| `awsElasticBlockStore` | Independent | AWS EBS |
| `gcePersistentDisk` | Independent | GCP Persistent Disk |
| `azureDisk` | Independent | Azure managed disk |
| `csi` | Independent | CSI driver storage |

## emptyDir

Created when Pod assigned to node, deleted when Pod removed.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: cache-volume
      mountPath: /cache
  
  - name: sidecar
    image: busybox
    command: ['sh', '-c', 'while true; do echo $(date) >> /cache/log.txt; sleep 5; done']
    volumeMounts:
    - name: cache-volume
      mountPath: /cache

  volumes:
  - name: cache-volume
    emptyDir:
      sizeLimit: 500Mi       # Optional size limit
      medium: Memory         # Optional: use tmpfs (RAM)
```

**Use Cases:**
- Scratch space for processing
- Checkpointing long computations
- Sharing files between containers in same pod
- Cache that can be regenerated

## hostPath

Mounts a file or directory from the host node.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
  - name: test
    image: busybox
    command: ['sh', '-c', 'cat /host-data/info.txt']
    volumeMounts:
    - name: host-volume
      mountPath: /host-data

  volumes:
  - name: host-volume
    hostPath:
      path: /data              # Path on the node
      type: Directory          # Type validation
      # Types: Directory, DirectoryOrCreate, File, FileOrCreate, Socket, CharDevice, BlockDevice
```

**Warning:** hostPath ties the pod to a specific node and has security implications.

## Persistent Volume (PV)

Cluster-wide storage resource provisioned by an administrator.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
  labels:
    type: local
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce           # RWO: Single node read-write
  # - ReadOnlyMany          # ROX: Multiple nodes read-only
  # - ReadWriteMany         # RWX: Multiple nodes read-write
  # - ReadWriteOncePod      # RWOP: Single pod read-write (K8s 1.22+)
  persistentVolumeReclaimPolicy: Retain  # Retain, Recycle, Delete
  storageClassName: manual
  mountOptions:
  - hard
  - nfsvers=4.1
  nfs:
    path: /exports/data
    server: nfs-server.example.com
  # OR local storage
  # local:
  #   path: /mnt/disks/ssd1
  #   fsType: ext4
```

### PV Lifecycle

```
Available → Bound → Released → (Recycled/Deleted)
                ↕
              PVC
```

| Phase | Description |
|-------|-------------|
| **Available** | Free resource, not yet claimed |
| **Bound** | Bound to a PVC |
| **Released** | PVC deleted, data remains |
| **Failed** | Failed automatic reclamation |

### Access Modes

| Mode | Abbreviation | Description | Supported By |
|------|-------------|-------------|--------------|
| ReadWriteOnce | RWO | Read-write by single node | Most storage |
| ReadOnlyMany | ROX | Read-only by many nodes | NFS, GlusterFS |
| ReadWriteMany | RWX | Read-write by many nodes | NFS, CephFS, EFS |
| ReadWriteOncePod | RWOP | Read-write by single pod | CSI volumes |

### Reclaim Policies

| Policy | Behavior |
|--------|----------|
| **Retain** | Data preserved, PV requires manual cleanup |
| **Recycle** | Data deleted (`rm -rf /volume/*`), PV becomes Available |
| **Delete** | Storage asset deleted from infrastructure |

## Persistent Volume Claim (PVC)

Request for storage by a user.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-data
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: gp2
  resources:
    requests:
      storage: 10Gi
  # Optional: Select specific PV
  # selector:
  #   matchLabels:
  #     type: local
```

### Using PVC in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-storage
spec:
  containers:
  - name: app
    image: mysql:8.0
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password"
    volumeMounts:
    - name: mysql-storage
      mountPath: /var/lib/mysql

  volumes:
  - name: mysql-storage
    persistentVolumeClaim:
      claimName: pvc-data
```

## StorageClass

Enables dynamic provisioning of PVs.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com    # CSI driver
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer  # or Immediate
allowVolumeExpansion: true
parameters:
  type: gp3
  fsType: ext4
  encrypted: "true"
  kmsKeyId: arn:aws:kms:...
  iopsPerGB: "50"
```

### Volume Binding Modes

| Mode | Description |
|------|-------------|
| **Immediate** | PV created when PVC is created |
| **WaitForFirstConsumer** | PV created when Pod using PVC is scheduled (considers node constraints) |

### Dynamic Provisioning Flow

```
User creates PVC
      │
      ▼
StorageClass selected (by name or default)
      │
      ▼
Provisioner creates storage (EBS, GPD, NFS, etc.)
      │
      ▼
PV automatically created and bound to PVC
      │
      ▼
Pod uses the PVC
```

## Complete MySQL Example with Dynamic Storage

```yaml
# 1. StorageClass (usually pre-existing)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: mysql-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
# 2. PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: database
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: mysql-storage
  resources:
    requests:
      storage: 50Gi
---
# 3. StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: database
spec:
  serviceName: mysql-headless
  replicas: 1
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
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "1000m"
            memory: "2Gi"
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping", "-h", "localhost"]
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 10
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: mysql-storage
      resources:
        requests:
          storage: 50Gi
```

## Volume Expansion

```bash
# Resize PVC (if StorageClass allows)
kubectl edit pvc mysql-pvc
# Change: storage: 50Gi → storage: 100Gi

# Check status
kubectl get pvc mysql-pvc
# Status shows FileSystemResizePending until node expands filesystem
```

## Storage Comparison Table

| Storage | RWO | ROX | RWX | Dynamic | Cloud |
|---------|-----|-----|-----|---------|-------|
| AWS EBS | ✓ | | | ✓ | AWS |
| GCE PD | ✓ | ✓ | | ✓ | GCP |
| Azure Disk | ✓ | | | ✓ | Azure |
| AWS EFS | ✓ | ✓ | ✓ | ✓ | AWS |
| Azure Files | ✓ | ✓ | ✓ | ✓ | Azure |
| NFS | ✓ | ✓ | ✓ | Manual | Any |
| CephFS | ✓ | ✓ | ✓ | ✓ | Any |
| Ceph RBD | ✓ | ✓ | | ✓ | Any |
| Local | ✓ | | | Manual | Any |
| Portworx | ✓ | ✓ | ✓ | ✓ | Any |
| Longhorn | ✓ | | ✓ | ✓ | Any |

## CSI (Container Storage Interface)

Standard plugin interface for storage providers.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com   # CSI driver
parameters:
  type: gp3
  encrypted: "true"
```

Popular CSI drivers:
- AWS EBS CSI
- GCE PD CSI
- Azure Disk CSI
- Ceph CSI
- Portworx
- Longhorn
- OpenEBS

## Volume Snapshots

```yaml
# 1. VolumeSnapshotClass
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapclass
driver: ebs.csi.aws.com
deletionPolicy: Retain
---
# 2. VolumeSnapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-snapshot
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: mysql-pvc
---
# 3. Restore from snapshot (PVC with dataSource)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-restored-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 50Gi
  dataSource:
    name: mysql-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

## Interview Questions and Answers

### Q: Difference between PV and PVC?

- **PV:** Cluster resource, provisioned by admin, independent of pods
- **PVC:** User request for storage, bound to a PV, used by pods

Think of PV as a physical hard drive and PVC as a partition request.

### Q: What happens to PV when PVC is deleted?

Depends on `persistentVolumeReclaimPolicy`:
- **Retain:** PV becomes Released, data preserved, manual cleanup needed
- **Delete:** PV and underlying storage deleted
- **Recycle:** Data wiped, PV becomes Available again

### Q: How does dynamic provisioning work?

1. User creates PVC with a `storageClassName`
2. Storage provisioner (CSI driver) watches for new PVCs
3. Provisioner creates storage on the backend (EBS, NFS, etc.)
4. PV is automatically created and bound to PVC
5. Pod can now use the PVC

### Q: What is WaitForFirstConsumer binding mode?

PV is not provisioned until a Pod using the PVC is scheduled. This ensures the storage is created in the same zone/region as the Pod, avoiding cross-zone data transfer costs.

### Q: Can multiple pods access the same PVC?

Depends on the access mode:
- **ReadWriteOnce:** Single node (multiple pods on same node)
- **ReadWriteMany:** Multiple nodes
- **ReadWriteOncePod:** Exactly one pod

### Q: How to resize a PVC?

1. StorageClass must have `allowVolumeExpansion: true`
2. Edit PVC and increase storage request
3. The underlying volume is expanded automatically
4. The filesystem on the node is expanded (may require Pod restart)
