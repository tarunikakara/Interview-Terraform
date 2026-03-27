# Kubernetes Storage & Persistence

## Table of Contents
1. [Overview](#overview)
2. [Volumes](#volumes)
3. [Persistent Volumes (PV)](#persistent-volumes-pv)
4. [Persistent Volume Claims (PVC)](#persistent-volume-claims-pvc)
5. [Storage Classes](#storage-classes)
6. [Dynamic Provisioning](#dynamic-provisioning)
7. [CSI Drivers](#csi-drivers)
8. [Examples & Patterns](#examples--patterns)

---

## Overview

**Storage & Persistence** in Kubernetes addresses the challenge of preserving data beyond Pod lifetime. Since Pods are ephemeral (created and destroyed), applications need persistent storage to maintain data.

### Storage Hierarchy

```
┌─────────────────────────────────────────────────────┐
│              Ephemeral Storage (Pod-local)          │
│  Deleted when Pod is destroyed, lost automatically  │
└─────────────────────────────────────────────────────┘
                         ▲
                         │ Need Persistence?
                         ▼
┌─────────────────────────────────────────────────────┐
│                    Volumes                          │
│  Mounted directories in Pods (various types)        │
└─────────────────────────────────────────────────────┘
                         ▲
                         │ Need cluster lifecycle?
                         ▼
┌─────────────────────────────────────────────────────┐
│         Persistent Volumes (PV & PVC)              │
│  Persistent storage independent of Pod lifecycle    │
└─────────────────────────────────────────────────────┘
```

### Key Challenges Solved

| Problem | Solution |
|---------|----------|
| Data lost when Pod crashes | Volumes mount persistent storage |
| Manual storage allocation | Storage Classes enable automation |
| Storage management complexity | Dynamic Provisioning handles it |
| Storage driver constraints | CSI Drivers provide flexibility |

---

## Volumes

**Volumes** are directories made available to Pods that can contain data. They are defined in Pod spec and have explicit lifetime tied to Pod.

### Volume Lifecycle

```
Pod Created
    │
    ├─ Volume created
    │
    ├─ Volume mounted to containers
    │
Pod running (data persists across container restarts)
    │
    ├─ Container crashes → recreated → data still there
    │
Pod deleted
    └─ Volume deleted (for most volume types)
```

### Volume Types

#### 1. **emptyDir** - Temporary Pod-level Storage

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-emptydir
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: cache
      mountPath: /tmp/cache
  volumes:
  - name: cache
    emptyDir: {}
```

**Characteristics:**
- ✅ Created when Pod is created
- ✅ Shared among containers in same Pod
- ✅ Survives container restart
- ❌ Deleted when Pod is deleted
- ✅ Ideal for temporary cache, logs

**Example Use Case:** Sidecar container logs
```bash
# Main app writes to /tmp/logs
# Logging sidecar reads from /tmp/logs
# Both share the same volume
```

#### 2. **hostPath** - Access Node Filesystem

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-hostpath
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
      path: /var/data             # Node path
      type: DirectoryOrCreate     # Create if doesn't exist
```

**Characteristics:**
- ✅ Accesses node's filesystem
- ✅ Data persists across pod restarts
- ❌ Pod migrating to different node loses access
- ⚠️ Security risk (direct node access)
- ✅ Good for dev/testing only

**hostPath Types:**
```
Directory              # Path must exist
DirectoryOrCreate      # Create if doesn't exist
File                   # File must exist
FileOrCreate           # Create if doesn't exist
Socket                 # Unix socket must exist
CharDevice            # Character device must exist
BlockDevice           # Block device must exist
```

#### 3. **configMap** - Configuration Data

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.properties: |
    debug=true
    port=8080
  settings.json: |
    {
      "theme": "dark"
    }

---
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-configmap
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: app-config
      items:
      - key: app.properties
        path: app.properties
      - key: settings.json
        path: settings.json
```

**Result:**
```
/etc/config/
├── app.properties
└── settings.json
```

#### 4. **secret** - Sensitive Data

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: admin
  password: supersecret

---
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-secret
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: credentials
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: credentials
    secret:
      secretName: db-credentials
```

#### 5. **downwardAPI** - Pod Metadata

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-metadata
  labels:
    app: myapp
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: podinfo
      mountPath: /etc/podinfo
  volumes:
  - name: podinfo
    downwardAPI:
      items:
      - path: "pod-name"
        fieldRef:
          fieldPath: metadata.name
      - path: "pod-namespace"
        fieldRef:
          fieldPath: metadata.namespace
      - path: "pod-ip"
        fieldRef:
          fieldPath: status.podIP
      - path: "pod-labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "pod-cpu-limit"
        resourceFieldRef:
          containerName: app
          resource: limits.cpu
```

**Available Pod Metadata:**
```
metadata.name
metadata.namespace
metadata.uid
metadata.labels
metadata.annotations
status.podIP
status.hostIP
status.phase
```

#### 6. **projected** - Combine Multiple Sources

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-projected
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: all-data
      mountPath: /etc/all
  volumes:
  - name: all-data
    projected:
      sources:
      - configMap:
          name: app-config
      - secret:
          name: db-credentials
      - downwardAPI:
          items:
          - path: "pod-name"
            fieldRef:
              fieldPath: metadata.name
```

### Volume Summary

| Type | Lifetime | Data Persistence | Use Case |
|------|----------|------------------|----------|
| **emptyDir** | Pod | Container restart | Cache, temp files |
| **hostPath** | Pod/Node | Across Pod restarts | Dev/testing only |
| **configMap** | ConfigMap | As long as CM exists | Configuration |
| **secret** | Secret | As long as Secret exists | Credentials |
| **downwardAPI** | Pod | Pod lifecycle | Pod metadata |
| **projected** | Multiple sources | Source lifetime | Combined data |

---

## Persistent Volumes (PV)

**Persistent Volumes (PV)** are cluster-level storage resources created by administrators. They exist independent of any Pod.

### PV Characteristics
- **Cluster Resource**: Created at cluster level (not namespace-scoped)
- **Administrator-Provisioned**: Usually created by storage admin
- **Independent**: Exists outside Pod lifecycle
- **Lifecycle**: Manual or via Dynamic Provisioning
- **Status**: Available, Bound, Released, Failed

### Your PV Example

From your `pv-my.yml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
  labels:
    type: d-hostpath
spec:
  capacity:
    storage: 2Gi              # Total capacity
  volumeMode: Filesystem      # Filesystem or Block
  accessModes:
    - ReadWriteOnce           # rwx, rwo, rox
  persistentVolumeReclaimPolicy: Retain  # Delete, Retain, Recycle
  storageClassName: standard  # Storage class name
  hostPath:
    path: /usr/share/nginx/html/
```

### Access Modes

```yaml
accessModes:
  - ReadWriteOnce      # Single node read-write (rwo)
  - ReadOnlyMany       # Multiple nodes read-only (rox)
  - ReadWriteMany      # Multiple nodes read-write (rwx)
  - ReadWriteOncePod   # Single pod read-write (new)
```

**Supported by Backend:**

| Backend | RWO | ROX | RWX | RWOP |
|---------|-----|-----|-----|------|
| hostPath | ✅ | ✅ | ❌ | ✅ |
| Local | ✅ | ✅ | ❌ | ✅ |
| NFS | ✅ | ✅ | ✅ | ❌ |
| AWS EBS | ✅ | ❌ | ❌ | ❌ |
| GCP PD | ✅ | ❌ | ❌ | ❌ |
| Azure Disk | ✅ | ❌ | ❌ | ❌ |
| GlusterFS | ✅ | ✅ | ✅ | ❌ |

### Reclaim Policies

```yaml
persistentVolumeReclaimPolicy: Retain    # Manual cleanup needed
persistentVolumeReclaimPolicy: Delete    # Delete storage when PVC deleted
persistentVolumeReclaimPolicy: Recycle   # Clear data and make available
```

**Policy Behavior:**
```
┌─────────────────────┐
│    PVC Created      │
│   Requests 1Gi      │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────────────────────┐
│  Retain: PV stays, needs cleanup    │
│  Delete: PV deleted automatically   │
│  Recycle: PV wiped, made Available  │
└─────────────────────────────────────┘
```

### PV Lifecycle States

```
1. Available      → No PVC bound
                   └─ Ready to be claimed

2. Bound          → PVC successfully bound
                   └─ Actively used by Pod

3. Released       → PVC deleted, PV released
                   └─ Waiting for reclaim policy

4. Failed         → Error occurred
                   └─ Requires manual intervention
```

### Different PV Storage Types

#### hostPath PV (Development)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: host-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/pv
```

#### NFS PV (Shared Storage)
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
    server: 192.168.1.100
    path: "/export/data"
```

#### AWS EBS PV (Production)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: aws-ebs-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce
  awsElasticBlockStore:
    volumeID: vol-1234567890abcdef0
    fsType: ext4
```

#### GCP PD PV
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gcp-pd-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce
  gcePersistentDisk:
    pdName: gce-disk-1
    fsType: ext4
```

---

## Persistent Volume Claims (PVC)

**Persistent Volume Claims (PVC)** are requests for storage by users. They bind to PVs and provide persistent storage to Pods.

### PVC Lifecycle

```
┌──────────────────────────────────────┐
│   Pod needs persistent storage       │
└────────────┬─────────────────────────┘
             │
             ▼
    ┌─────────────────┐
    │   PVC Created   │  Requests storage
    └────────┬────────┘
             │
             ▼
    ┌──────────────────────────┐
    │  Find matching PV        │  Check capacity, access mode, class
    └────────┬─────────────────┘
             │
             ▼
    ┌──────────────────────────┐
    │  Bind PV to PVC          │
    └────────┬─────────────────┘
             │
             ▼
    ┌──────────────────────────┐
    │  Mount in Pod            │
    └────────┬─────────────────┘
             │
             ▼
    ┌──────────────────────────┐
    │  Data available to app   │
    └──────────────────────────┘
```

### Your PVC Example

From your `pvc-my.yml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi       # Request 1Gi
  selector:
    matchLabels:
      type: d-hostpath   # Match PV with this label
```

**Binding Process:**
1. PVC requests 1Gi with ReadWriteOnce
2. Kubernetes searches for available PVs
3. PV `my-pv` (2Gi, ReadWriteOnce, type=d-hostpath) matches
4. PV bound to PVC
5. Pod can now mount PVC

### Using PVC in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-using-pvc
spec:
  containers:
  - name: app
    image: nginx:latest
    volumeMounts:
    - name: data-storage
      mountPath: /usr/share/nginx/html/
  volumes:
  - name: data-storage
    persistentVolumeClaim:
      claimName: my-pvc
```

### Using PVC in Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:latest
        volumeMounts:
        - name: persistent-data
          mountPath: /data
      volumes:
      - name: persistent-data
        persistentVolumeClaim:
          claimName: my-pvc
```

### PVC Status

```bash
# Check PVC binding status
kubectl get pvc

# Output states:
# STATUS                            MEANING
# Pending                           Waiting for PV to bind
# Bound                             Successfully bound to PV
# Lost                              PV lost, can't recover
```

### PVC vs StatefulSet volumeClaimTemplates

**Manual PVC (one PVC for all Pods):**
```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: shared-pvc   # All pods share one PVC
```

**StatefulSet with volumeClaimTemplates (one PVC per Pod):**
```yaml
apiVersion: apps/v1
kind: StatefulSet
spec:
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
# Creates: data-mysql-0, data-mysql-1, data-mysql-2
```

---

## Storage Classes

**Storage Classes** define different storage types and parameters available in the cluster. They enable dynamic provisioning of volumes.

### Storage Class Purpose

```
┌────────────────────────────────────────────────────┐
│  Storage Class Definition                          │
│  ├─ Provisioner (AWS EBS, GCP PD, AzureDisk, etc) │
│  ├─ Parameters (size, iops, type, etc)            │
│  ├─ Reclaim Policy (Delete, Retain, Recycle)      │
│  └─ Volume Binding Mode (Immediate, WaitForFirst) │
└────────────────────────────────────────────────────┘
           │
           ▼ Referenced by PVC
      ┌─────────────────┐
      │   PVC Creation  │
      └────────┬────────┘
               │
               ▼ Triggers
      ┌──────────────────────┐
      │ Dynamic Provisioning │
      │ Creates PV auto      │
      └──────────────────────┘
```

### Basic Storage Class Example

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
allowVolumeExpansion: true
volumeBindingMode: Immediate
reclaimPolicy: Delete
```

### AWS EBS Storage Classes

```yaml
---
# SSD Storage (High Performance)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
allowVolumeExpansion: true
reclaimPolicy: Delete

---
# General Purpose (Balanced)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp2
provisioner: ebs.csi.aws.com
parameters:
  type: gp2
  encrypted: "true"
reclaimPolicy: Delete

---
# IOPS Optimized (Database)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-io1
provisioner: ebs.csi.aws.com
parameters:
  type: io1
  iops: "10000"
  encrypted: "true"
reclaimPolicy: Delete

---
# Throughput Optimized (Big Data)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-st1
provisioner: ebs.csi.aws.com
parameters:
  type: st1
  throughput: "500"
reclaimPolicy: Delete
```

### GCP Persistent Disk Storage Classes

```yaml
---
# SSD Storage
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcp-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
allowVolumeExpansion: true
reclaimPolicy: Delete

---
# Standard HDD
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcp-standard
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-standard
  replication-type: regional-pd
reclaimPolicy: Delete
```

### NFS Storage Class

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.1.100
  share: /export/nfs
allowVolumeExpansion: true
reclaimPolicy: Delete
```

### Volume Binding Modes

```yaml
# Immediate Binding (Default)
volumeBindingMode: Immediate
# PV allocated immediately when PVC created
# May cause pod scheduling issues

---
# Wait for First Consumer
volumeBindingMode: WaitForFirstConsumer
# PV allocated only when Pod references the PVC
# Pod can schedule on node where storage exists
```

### Storage Class with Snapshots

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-ebs-sc
provisioner: ebs.csi.aws.com
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
```

### Default Storage Class

```bash
# Mark as default (cluster can have only one default)
annotations:
  storageclass.kubernetes.io/is-default-class: "true"
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: default-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
```

---

## Dynamic Provisioning

**Dynamic Provisioning** automatically creates PVs when PVCs are requested. No need for administrators to manually create PVs.

### Static vs Dynamic

```
┌─────────────────────────────────────────────┐
│         Static Provisioning                 │
│  Admin creates PV → User creates PVC        │
│  Manual, scales poorly, error-prone         │
└─────────────────────────────────────────────┘

vs

┌──────────────────────────────────────────────┐
│      Dynamic Provisioning                    │
│  User creates PVC → Controller creates PV    │
│  Automatic, scalable, consistent             │
└──────────────────────────────────────────────┘
```

### Dynamic Provisioning Flow

```
1. PVC created with storageClassName
           │
           ▼
2. Controller detects PVC
           │
           ▼
3. Call provisioner (e.g., AWS EBS CSI)
           │
           ▼
4. Provisioner creates physical volume
           │
           ▼
5. Controller creates PV object
           │
           ▼
6. Bind PVC to PV
           │
           ▼
7. Pod mounts and uses volume
```

### Dynamic PVC Example

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-gp3      # Reference storage class
  resources:
    requests:
      storage: 20Gi              # Will auto-create 20Gi PV
```

**What Happens:**
1. PVC requests 20Gi using `ebs-gp3` storage class
2. AWS EBS CSI provisioner creates 20Gi EBS volume
3. PV object created automatically pointing to EBS volume
4. PVC bound to PV automatically
5. Pod can mount and use

### Dynamic with Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dynamic-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: app
        image: nginx:latest
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: dynamic-pvc

---
# Separate PVC resource
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-gp3
  resources:
    requests:
      storage: 20Gi
```

### Benefits vs Drawbacks

**Benefits:**
- ✅ Administrators don't pre-create PVs
- ✅ Automatic upon PVC request
- ✅ Scales easily with applications
- ✅ Consistent performance

**Drawbacks:**
- ❌ Requires CSI driver installation
- ❌ Cost if volumes not cleaned up
- ❌ Platform-specific configuration needed

---

## CSI Drivers

**CSI (Container Storage Interface)** is a standard that allows Kubernetes to communicate with external storage systems. CSI Drivers implement this interface.

### CSI Architecture

```
┌──────────────────────────────────┐
│      Kubernetes Control Plane     │
└────────────────┬─────────────────┘
                 │
    ┌────────────┴─────────────┐
    │                          │
    ▼                          ▼
┌──────────────┐      ┌──────────────┐
│ CSI Node     │      │ CSI Plugin   │
│ (Kubelet)    │      │ (Storage ops)│
└──────────────┘      └──────────────┘
    │                          │
    └────────────┬─────────────┘
                 │
                 ▼
        ┌─────────────────┐
        │  Storage System │
        │  (AWS, GCP, etc)│
        └─────────────────┘
```

### CSI Driver Components

#### 1. CSI Controller
- Creates/deletes volumes
- Attaches/detaches volumes to nodes
- Takes snapshots

#### 2. CSI Node Plugin
- Mounts volumes to pods
- Formats volumes
- Unmounts volumes

### Popular CSI Drivers

#### AWS EBS CSI Driver

```bash
# Install AWS EBS CSI driver
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  -n kube-system
```

**Usage:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-gp3
  resources:
    requests:
      storage: 50Gi
```

#### GCP Persistent Disk CSI Driver

```bash
# Usually pre-installed on GKE
kubectl create storageclass fast --provisioner=pd.csi.storage.gke.io \
  --parameters type=pd-ssd
```

#### Azure Disk CSI Driver

```bash
# Install Azure Disk CSI driver
helm repo add azuredisk-csi-driver https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/charts
helm install azuredisk-csi-driver azuredisk-csi-driver/azuredisk-csi-driver \
  -n kube-system
```

#### NFS CSI Driver

```bash
# Install NFS CSI driver (alpha/beta)
helm repo add nfs-csi-driver https://kubernetes-sigs.github.io/nfs-csi-driver
helm install nfs-csi-driver nfs-csi-driver/nfs-csi-driver \
  -n kube-system
```

### CSI Driver Installation Example

```yaml
# Storage Class pointing to CSI provisioner
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-storage
provisioner: ebs.csi.aws.com        # CSI driver name
parameters:
  type: gp3
  iops: "3000"
allowVolumeExpansion: true
reclaimPolicy: Delete

---
# CSI Node Plugin DaemonSet (simplified)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: csi-node
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: csi-node
  template:
    metadata:
      labels:
        app: csi-node
    spec:
      hostNetwork: true
      containers:
      - name: node-driver-registrar
        image: k8s.gcr.io/sig-storage/csi-node-driver-registrar:v2.0.0
      - name: ebs-plugin
        image: public.ecr.aws/ebs-csi-driver/ebs-csi-driver:v1.0.0
        volumeMounts:
        - name: kubelet-dir
          mountPath: /var/lib/kubelet
      volumes:
      - name: kubelet-dir
        hostPath:
          path: /var/lib/kubelet
```

### CSI vs In-Tree Drivers

| Aspect | CSI | In-Tree |
|--------|-----|---------|
| **Maintenance** | Separate repos | Kubernetes repo |
| **Updates** | Independent | Kubernetes release cycle |
| **New Features** | Faster | Slower |
| **Deprecation** | In progress | Old drivers being deprecated |
| **Installation** | Install via helm/yaml | Built-in |

### Custom CSI Driver

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: csi-provisioner
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: csi-provisioner
rules:
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "update"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: csi-provisioner
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: csi-provisioner
subjects:
- kind: ServiceAccount
  name: csi-provisioner
  namespace: kube-system
```

---

## Examples & Patterns

### Pattern 1: Database with Persistent Storage

```yaml
---
# Storage Class
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: database-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "5000"
  throughput: "250"
allowVolumeExpansion: true
reclaimPolicy: Delete

---
# StatefulSet with persistent storage
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  replicas: 1
  serviceName: postgres
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        ports:
        - containerPort: 5432
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: database-storage
      resources:
        requests:
          storage: 100Gi
```

### Pattern 2: Shared Cache with NFS

```yaml
---
# NFS Storage Class
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.1.100
  share: /export/cache

---
# PVC for shared cache
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cache-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 500Gi

---
# Multiple pods sharing same storage
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cache-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: cache
  template:
    metadata:
      labels:
        app: cache
    spec:
      containers:
      - name: app
        image: cache-app:latest
        volumeMounts:
        - name: cache
          mountPath: /cache
      volumes:
      - name: cache
        persistentVolumeClaim:
          claimName: cache-pvc
```

### Pattern 3: Backup with Snapshots

```yaml
---
# Storage Class with snapshot support
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: snapshot-enabled
provisioner: ebs.csi.aws.com
allowVolumeExpansion: true
reclaimPolicy: Delete

---
# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: snapshot-enabled
  resources:
    requests:
      storage: 50Gi

---
# Volume Snapshot Class
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapclass
driver: ebs.csi.aws.com
deletionPolicy: Delete

---
# Create snapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: data-snapshot
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: data-pvc
```

### Pattern 4: Logs with Local Storage

```yaml
---
# Local Storage Class
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer

---
# Local Persistent Volume
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/data
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node-1

---
# Pod using local storage
apiVersion: v1
kind: Pod
metadata:
  name: log-processor
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - node-1
  containers:
  - name: processor
    image: log-processor:latest
    volumeMounts:
    - name: logs
      mountPath: /logs
  volumes:
  - name: logs
    persistentVolumeClaim:
      claimName: local-pvc
```

---

## Best Practices

### Storage Planning
- ✅ Choose appropriate storage class for workload type
- ✅ Use `WaitForFirstConsumer` for local/node-specific storage
- ✅ Enable `allowVolumeExpansion: true` for growing workloads
- ✅ Set appropriate `reclaimPolicy` based on data criticality
- ❌ Don't use hostPath for production

### Performance
- ✅ Use SSD storage classes for databases
- ✅ Use appropriate IOPS for workload
- ✅ Monitor storage performance metrics
- ✅ Plan for storage growth
- ❌ Don't oversee workload requirements

### Security
- ✅ Enable encryption at rest
- ✅ Use secrets for sensitive data
- ✅ Backup critical persistent data
- ✅ Implement RBAC for storage resources
- ❌ Don't store credentials on volume

### Cost Optimization
- ✅ Use `reclaimPolicy: Delete` to clean up unused volumes
- ✅ Monitor and delete unused PVs/PVCs
- ✅ Use appropriate storage tier
- ✅ Set up lifecycle policies
- ❌ Don't leave abandoned volumes

### Backup & Recovery
- ✅ Regular backups of critical data
- ✅ Test restore procedures
- ✅ Use volume snapshots when supported
- ✅ Maintain backup outside cluster
- ❌ Don't skip backup for "temporary" data

---

## Troubleshooting

```bash
# Check Storage Classes
kubectl get storageclasses
kubectl describe sc standard

# Check PVs
kubectl get pv
kubectl describe pv my-pv

# Check PVCs
kubectl get pvc
kubectl describe pvc my-pvc

# Check binding
kubectl get pv,pvc

# Check pod volume mounting
kubectl describe pod pod-name
kubectl exec -it pod-name -- df -h

# Check CSI drivers
kubectl get daemonset -n kube-system | grep csi

# Check storage provisioner logs
kubectl logs -n kube-system -l app=provisioner

# Expand volume
kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'

# Check volume events
kubectl describe pvc my-pvc
# Look for events section
```

---

## Summary

- **Volumes**: Temporary storage for containers, deleted with Pod
- **PV & PVC**: Persistent cluster-level storage independent of Pods
- **Storage Classes**: Define storage types and provisioning parameters
- **Dynamic Provisioning**: Automatic PV creation when PVCs requested
- **CSI Drivers**: Standard interface allowing Kubernetes to use external storage systems

Understanding storage is critical for data persistence, backups, and stateful application management in Kubernetes!
