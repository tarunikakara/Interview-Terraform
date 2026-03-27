# Kubernetes Scheduling & Node Management

## Table of Contents
1. [Overview](#overview)
2. [Manual Pod Scheduling](#manual-pod-scheduling)
3. [Taints & Tolerations](#taints--tolerations)
4. [Node Affinity](#node-affinity)
5. [Pod Affinity / Anti-affinity](#pod-affinity--anti-affinity)
6. [Resource Requests & Limits](#resource-requests--limits)
7. [Pod Priorities & Preemption](#pod-priorities--preemption)
8. [Scheduling Examples](#scheduling-examples)

---

## Overview

**Kubernetes Scheduling** determines which node a Pod runs on. The scheduler must balance:
- Resource availability
- Node constraints (taints)
- Pod requirements (affinity)
- System priorities
- Performance objectives

### Scheduling Decision Flow

```
Pod Created
    │
    ▼
┌─────────────────────────────────┐
│  Scheduler Evaluates:           │
│  ├─ Node availability           │
│  ├─ Resource capacity           │
│  ├─ Taints/Tolerations          │
│  ├─ Affinity rules              │
│  └─ Pod priorities              │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│  Filter Phase:                  │
│  Remove unsuitable nodes        │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│  Score Phase:                   │
│  Rank remaining nodes           │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│  Select highest-scoring node    │
│  Bind Pod to node               │
└─────────────────────────────────┘
```

### Scheduling Components

| Component | Role |
|-----------|------|
| **kube-scheduler** | Main scheduling controller |
| **Filter Plugins** | Remove unsuitable nodes |
| **Score Plugins** | Rank remaining nodes |
| **ExtenderConfig** | Custom scheduling logic |

---

## Manual Pod Scheduling

**Manual scheduling** bypasses the default scheduler and places Pods on specific nodes.

### Method 1: nodeName

Directly specify node name (highest priority):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: manual-pod
spec:
  nodeName: node-1              # Pod will go to node-1
  containers:
  - name: app
    image: myapp:latest
```

**Characteristics:**
- ✅ Pod goes directly to specified node
- ✅ Bypasses scheduler entirely
- ❌ No validation if node exists
- ❌ No resource checks
- Use case: Emergency fixes, testing

### Method 2: nodeSelector

Select nodes by labels:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: selector-pod
spec:
  nodeSelector:
    disk: "ssd"                # Pod goes to node with label disk=ssd
  containers:
  - name: app
    image: myapp:latest
```

**Setup:**
```bash
# Label nodes
kubectl label node node-1 disk=ssd
kubectl label node node-2 disk=hdd

# View labels
kubectl get nodes --show-labels

# Remove label
kubectl label node node-1 disk-
```

**Label Examples:**
```bash
# Compute type
kubectl label node node-1 workload=compute
kubectl label node node-2 workload=storage

# Hardware
kubectl label node node-1 disktype=ssd
kubectl label node node-2 disktype=hdd

# Zone/Region
kubectl label node node-1 zone=us-east-1a
kubectl label node node-2 zone=us-east-1b

# Custom
kubectl label node node-1 environment=production
kubectl label node node-2 environment=staging
```

**Using Multiple Labels:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-label-pod
spec:
  nodeSelector:
    disk: "ssd"
    cpu: "high"
  containers:
  - name: app
    image: myapp:latest
```

**Characteristics:**
- ✅ Simple label matching (AND logic)
- ✅ Scheduler validates label existence
- ❌ Limited to exact matching
- ❌ No OR/NOT logic
- Use case: Simple node selection

### Usage Pattern

```yaml
# Database nodes
---
apiVersion: v1
kind: Pod
metadata:
  name: database
spec:
  nodeSelector:
    type: database
  containers:
  - name: postgres
    image: postgres:13

---
# Web nodes
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  nodeSelector:
    type: web
  containers:
  - name: nginx
    image: nginx:latest
```

---

## Taints & Tolerations

**Taints** mark nodes as unsuitable for certain Pods. **Tolerations** allow Pods to tolerate (be scheduled on) tainted nodes.

### Taints Concept

```
Normal Node:
Any Pod can schedule
    │
    ├─ Pod 1 ✓
    ├─ Pod 2 ✓
    └─ Pod 3 ✓

Tainted Node (taint: key=value:NoSchedule):
Only Pods with matching toleration can schedule
    │
    ├─ Pod 1 ✗ (no toleration)
    ├─ Pod 2 ✓ (has toleration)
    └─ Pod 3 ✗ (no toleration)
```

### Taint Effects

| Effect | Meaning |
|--------|---------|
| **NoSchedule** | Don't schedule new Pods |
| **NoExecute** | Evict existing Pods without toleration |
| **PreferNoSchedule** | Try to avoid, but allow if necessary |

### Adding Taints to Nodes

```bash
# Add taint
kubectl taint node node-1 key=value:effect

# Examples
kubectl taint node node-1 workload=gpu:NoSchedule
kubectl taint node node-2 dedicated=compute:NoExecute
kubectl taint node node-3 type=database:PreferNoSchedule

# Remove taint
kubectl taint node node-1 workload-

# Remove all taints
kubectl taint node node-1 --all true
```

### Pod Tolerations

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  tolerations:
  # Match taints: workload=gpu:NoSchedule
  - key: workload
    operator: Equal
    value: gpu
    effect: NoSchedule
  
  containers:
  - name: app
    image: gpu-app:latest
```

### Toleration Fields

```yaml
tolerations:
- key: workload              # Taint key
  operator: Equal            # Equal or Exists
  value: gpu                 # Taint value (not required with Exists)
  effect: NoSchedule         # NoSchedule, NoExecute, PreferNoSchedule
  tolerationSeconds: 300     # Time to wait before evicting (NoExecute only)
```

### Toleration Operators

#### Equal (Exact Match)
```yaml
tolerations:
- key: workload
  operator: Equal
  value: gpu
  effect: NoSchedule
# Matches: workload=gpu:NoSchedule
```

#### Exists (Any Value)
```yaml
tolerations:
- key: workload
  operator: Exists           # Any value of 'workload'
  effect: NoSchedule
# Matches: workload=gpu:NoSchedule, workload=cpu:NoSchedule, etc.
```

#### No Key/Effect (Wildcard)
```yaml
tolerations:
- operator: Exists          # Tolerates all taints
# Matches any taint
```

### Real-World Examples

#### GPU Nodes
```bash
# Taint GPU nodes
kubectl taint node gpu-node-1 accelerator=gpu:NoSchedule
kubectl taint node gpu-node-2 accelerator=gpu:NoSchedule
```

```yaml
# Pod requiring GPU
apiVersion: v1
kind: Pod
metadata:
  name: ml-training
spec:
  tolerations:
  - key: accelerator
    operator: Equal
    value: gpu
    effect: NoSchedule
  containers:
  - name: training
    image: ml-framework:latest
    resources:
      limits:
        nvidia.com/gpu: 1
```

#### Dedicated Nodes
```bash
# Taint for dedicated workload
kubectl taint node compute-node-1 dedicated=compute:NoExecute
```

```yaml
# Only compute workload can run here
apiVersion: v1
kind: Pod
metadata:
  name: compute-job
spec:
  tolerations:
  - key: dedicated
    operator: Equal
    value: compute
    effect: NoExecute
    tolerationSeconds: 3600    # Grace period: 1 hour
  containers:
  - name: worker
    image: compute-worker:latest
```

#### System Nodes
```bash
# Prevent user pods on system nodes
kubectl taint node system-node-1 node-type=system:NoSchedule
```

---

## Node Affinity

**Node Affinity** provides more sophisticated node selection using expressions instead of simple labels.

### Node Affinity Types

#### 1. requiredDuringSchedulingIgnoredDuringExecution
Pod will only schedule if conditions met. If conditions become false during execution, Pod continues running.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: required-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disk
            operator: In
            values:
            - ssd
            - fast
  containers:
  - name: app
    image: myapp:latest
```

#### 2. preferredDuringSchedulingIgnoredDuringExecution
Pod prefers matching nodes but will schedule anyway if no match found.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: preferred-affinity
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - us-east-1a
  containers:
  - name: app
    image: myapp:latest
```

### Node Affinity Operators

| Operator | Meaning |
|----------|---------|
| **In** | Value is in list |
| **NotIn** | Value is NOT in list |
| **Exists** | Key exists (any value) |
| **DoesNotExist** | Key doesn't exist |
| **Gt** | Greater than |
| **Lt** | Less than |

### Practical Examples

#### Required Affinity (Multi-zone)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-zone-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        # OR logic between terms
        - matchExpressions:
          - key: zone
            operator: In
            values:
            - us-east-1a
            - us-east-1b
  containers:
  - name: app
    image: myapp:latest
```

#### Preferred Affinity (Scoring)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: preferred-pod
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      # Higher weight = higher priority
      - weight: 100
        preference:
          matchExpressions:
          - key: workload
            operator: In
            values:
            - web
      - weight: 50
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - us-east-1a
  containers:
  - name: app
    image: myapp:latest
```

#### Complex Affinity
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: complex-affinity
spec:
  affinity:
    nodeAffinity:
      # MUST run on nodes with these characteristics
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          # AND: All conditions must be true
          - key: disk
            operator: In
            values:
            - ssd
          - key: cpu
            operator: In
            values:
            - high
          - key: memory
            operator: Gt
            values:
            - "16"
      # PREFER these nodes
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - us-east-1a
```

---

## Pod Affinity / Anti-affinity

**Pod Affinity** schedules Pods near each other. **Pod Anti-affinity** schedules Pods away from each other.

### Pod Affinity (Co-location)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database-pod
spec:
  labels:
    role: database
  containers:
  - name: postgres
    image: postgres:13

---
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: role
            operator: In
            values:
            - database
        topologyKey: kubernetes.io/hostname  # Same node
  containers:
  - name: app
    image: myapp:latest
```

**Result:** App Pod runs on same node as database Pod

### Pod Anti-affinity (Separation)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod-1
spec:
  labels:
    tier: web
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: tier
            operator: In
            values:
            - web
        topologyKey: kubernetes.io/hostname  # Different nodes
  containers:
  - name: web
    image: nginx:latest
```

**Result:** Web Pods run on different nodes

### Topology Keys

```yaml
# Same node
topologyKey: kubernetes.io/hostname

# Same zone
topologyKey: topology.kubernetes.io/zone

# Same region
topologyKey: topology.kubernetes.io/region

# Custom topology
topologyKey: custom-topology/rack
```

### Pod Affinity Types

#### Required (Pod Anti-affinity)
```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchLabels:
        tier: frontend
    topologyKey: kubernetes.io/hostname
```
**Effect:** Pods must be on different nodes

#### Preferred (Pod Anti-affinity)
```yaml
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    podAffinityTerm:
      labelSelector:
        matchLabels:
          tier: frontend
      topologyKey: kubernetes.io/hostname
```
**Effect:** Pods prefer different nodes but can coexist

### Real-World Examples

#### Multi-Tier Application

```yaml
---
# Database Pod (Anti-affinity to other databases)
apiVersion: v1
kind: Pod
metadata:
  name: database
  labels:
    role: database
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              role: database
          topologyKey: kubernetes.io/hostname
  containers:
  - name: postgres
    image: postgres:13

---
# Backend Pod (Affinity to database)
apiVersion: v1
kind: Pod
metadata:
  name: backend
  labels:
    role: backend
spec:
  affinity:
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              role: database
          topologyKey: kubernetes.io/hostname
    # Anti-affinity to other backends
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 50
        podAffinityTerm:
          labelSelector:
            matchLabels:
              role: backend
          topologyKey: kubernetes.io/hostname
  containers:
  - name: api
    image: backend:latest

---
# Frontend Pod (Anti-affinity to other frontends)
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  labels:
    role: frontend
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              role: frontend
          topologyKey: kubernetes.io/hostname
  containers:
  - name: web
    image: frontend:latest
```

#### Multi-Zone High Availability

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      affinity:
        # Spread across zones
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: myapp
              topologyKey: topology.kubernetes.io/zone
        # Prefer high-memory nodes
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            preference:
              matchExpressions:
              - key: node-type
                operator: In
                values:
                - high-memory
      containers:
      - name: app
        image: myapp:latest
```

---

## Resource Requests & Limits

**Resource Requests** tell scheduler minimum resources needed. **Limits** cap maximum resource usage.

### Request vs Limit

```
Resources Spectrum:
0  ←────────────────────→ Unlimited

     ↑
    Request: Minimum guaranteed
     ↑
    Limit: Maximum allowed
     ↑
```

### Resource Types

| Resource | Unit | Purpose |
|----------|------|---------|
| **CPU** | cores or millicores (m) | Processing power |
| **Memory** | bytes (Ki, Mi, Gi) | RAM |
| **Ephemeral Storage** | bytes (Ki, Mi, Gi) | Temporary disk |
| **Custom** | arbitrary | GPU, accelerators |

### Setting Requests & Limits

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

**Explanation:**
```
CPU:
  requests: 250m (0.25 cores) → minimum guaranteed
  limits: 500m (0.5 cores) → throttled above this

Memory:
  requests: 64Mi → minimum guaranteed
  limits: 128Mi → Pod killed if exceeds
```

### CPU Units

```yaml
cpu: "1"       # 1 core
cpu: "1.5"     # 1.5 cores
cpu: "500m"    # 0.5 cores (millicores)
cpu: "100m"    # 0.1 cores
cpu: "10m"     # 0.01 cores
```

### Memory Units

```yaml
memory: "128Mi"    # 128 mebibytes
memory: "1Gi"      # 1 gibibyte
memory: "512M"     # 512 megabytes (decimal, not recommended)
memory: "1900Mi"   # 1900 mebibytes
```

### QoS Classes

Kubernetes automatically assigns QoS (Quality of Service) based on requests/limits:

#### 1. **Guaranteed** (No eviction)
Both requests and limits set, and equal:

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "500m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

#### 2. **Burstable** (Lower eviction priority)
Requests < Limits:

```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "256Mi"
    cpu: "1"
```

#### 3. **BestEffort** (High eviction priority)
No requests or limits:

```yaml
# No resources section
# Can use any available resources
```

### Node Capacity Allocation

```
Node with 4 CPU, 8Gi Memory
├─ Pod 1: requests 1 CPU, 2Gi → allocated
├─ Pod 2: requests 1 CPU, 2Gi → allocated
├─ Pod 3: requests 1 CPU, 2Gi → allocated
└─ Pod 4: requests 1 CPU, 2Gi → pending (insufficient)

Available: 0 CPU, 2Gi Memory (reserved)
```

### Practical Resource Sizing

```yaml
---
# Web tier (lightweight)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"

---
# Application tier (moderate)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      containers:
      - name: api
        image: api:latest
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"

---
# Database tier (heavy)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  template:
    spec:
      containers:
      - name: postgres
        image: postgres:13
        resources:
          requests:
            memory: "2Gi"
            cpu: "1"
          limits:
            memory: "4Gi"
            cpu: "2"
```

---

## Pod Priorities & Preemption

**Pod Priorities** determine scheduling order and enable preemption (evicting lower-priority Pods).

### PriorityClass Definition

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000                    # Higher value = higher priority
globalDefault: false           # Apply to all Pods if omitted priority
description: "For critical workloads"
```

### Priority Range

```
Value range: -2147483648 to 2147483647

Typical:
1000000: System critical
10000:   High priority business
1000:    Normal applications
100:     Low priority batch
1:       Minimal priority
```

### Using PriorityClass

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  priorityClassName: high-priority  # Reference PriorityClass
  containers:
  - name: app
    image: critical-app:latest
```

### Preemption Example

```yaml
---
# Low-priority batch job
apiVersion: v1
kind: Pod
metadata:
  name: batch-job
  namespace: default
spec:
  priorityClassName: low-priority
  containers:
  - name: batch
    image: batch-processor:latest
    resources:
      requests:
        memory: "2Gi"
        cpu: "1"

---
# High-priority business logic arrives
apiVersion: v1
kind: Pod
metadata:
  name: business-critical
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: critical-app:latest
    resources:
      requests:
        memory: "2Gi"
        cpu: "1"
```

**What happens:**
1. High-priority Pod needs resources
2. Cluster finds low-priority Pod using resources
3. Low-priority Pod evicted (graceful termination)
4. High-priority Pod scheduled

### PriorityClass Best Practices

```yaml
---
# System critical
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: system-critical
value: 1000000
globalDefault: false
preemptionPolicy: Never        # Can't be preempted

---
# Application critical
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: app-critical
value: 10000
preemptionPolicy: PreemptLowerPriority

---
# Normal workload
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: normal
value: 1000
preemptionPolicy: PreemptLowerPriority

---
# Batch jobs
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch
value: 100
preemptionPolicy: PreemptLowerPriority
```

### Pod Disruption Budgets (Protection)

Prevent priority-based eviction:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-pdb
spec:
  minAvailable: 2              # Always keep at least 2 Pods running
  selector:
    matchLabels:
      tier: critical
  unhealthyPodEvictionPolicy: AlwaysAllow

---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  maxUnavailable: 1            # Allow max 1 Pod to be disrupted
  selector:
    matchLabels:
      tier: web
```

---

## Scheduling Examples

### Example 1: Database Scheduling

```yaml
---
# Label database nodes
# kubectl label node postgres-node-1 workload=database

apiVersion: v1
kind: Pod
metadata:
  name: postgres-master
  labels:
    tier: database
    role: master
spec:
  # Must run on database nodes
  nodeSelector:
    workload: database
  
  # Never collocate master with replicas
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            tier: database
            role: replica
        topologyKey: kubernetes.io/hostname
  
  # Resource guarantee
  containers:
  - name: postgres
    image: postgres:13
    resources:
      requests:
        memory: "2Gi"
        cpu: "1"
      limits:
        memory: "4Gi"
        cpu: "2"
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql
  
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: postgres-pvc

---
# Replica instances
apiVersion: v1
kind: Pod
metadata:
  name: postgres-replica-1
  labels:
    tier: database
    role: replica
spec:
  nodeSelector:
    workload: database
  
  # Keep replicas on different nodes
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              tier: database
          topologyKey: kubernetes.io/hostname
  
  containers:
  - name: postgres
    image: postgres:13
    resources:
      requests:
        memory: "1Gi"
        cpu: "500m"
      limits:
        memory: "2Gi"
        cpu: "1"
```

### Example 2: GPU Workload Scheduling

```yaml
---
# Taint GPU nodes
# kubectl taint node gpu-node-1 accelerator=gpu:NoSchedule

apiVersion: v1
kind: Pod
metadata:
  name: ml-training
spec:
  # Only schedule on GPU nodes
  tolerations:
  - key: accelerator
    operator: Equal
    value: gpu
    effect: NoSchedule
  
  # Prefer high-memory nodes
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: memory-class
            operator: In
            values:
            - xlarge
  
  # Reserve resources
  containers:
  - name: training
    image: ml-framework:latest
    resources:
      requests:
        memory: "8Gi"
        cpu: "4"
        nvidia.com/gpu: "1"
      limits:
        memory: "16Gi"
        cpu: "8"
        nvidia.com/gpu: "1"
```

### Example 3: Multi-Zone High Availability

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-ha
spec:
  replicas: 6
  selector:
    matchLabels:
      app: web
      tier: frontend
  template:
    metadata:
      labels:
        app: web
        tier: frontend
    spec:
      affinity:
        # Spread across zones
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  tier: frontend
              topologyKey: topology.kubernetes.io/zone
        
        # Prefer different nodes too
          - weight: 50
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  tier: frontend
              topologyKey: kubernetes.io/hostname
        
        # Prefer compute-optimized nodes
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            preference:
              matchExpressions:
              - key: node-type
                operator: In
                values:
                - compute
      
      priorityClassName: high-priority
      
      containers:
      - name: web
        image: web-app:latest
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

### Example 4: Batch Job with Low Priority

```yaml
---
# Low priority for batch jobs
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-low
value: 10
preemptionPolicy: PreemptLowerPriority

---
# Batch job
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processing
spec:
  parallelism: 10
  completions: 100
  template:
    spec:
      # Low priority - can be preempted
      priorityClassName: batch-low
      
      # Can tolerate tainted nodes
      tolerations:
      - key: batch
        operator: Equal
        value: "true"
        effect: NoSchedule
      
      # Resource efficient
      containers:
      - name: processor
        image: batch-processor:latest
        resources:
          requests:
            memory: "100Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "250m"
      
      restartPolicy: OnFailure
```

---

## Best Practices

### Node Affinity
- ✅ Use requiredDuringScheduling for critical constraints
- ✅ Use preferred for performance optimization
- ✅ Combine with PodAntiAffinity for HA
- ❌ Don't over-constrain scheduling

### Taints & Tolerations
- ✅ Use for dedicated node pools (GPUs, memory)
- ✅ Use PreferNoSchedule for gradual migration
- ✅ Set tolerationSeconds for graceful eviction
- ❌ Don't use for general workload distribution

### Pod Affinity
- ✅ Use requiredDuringScheduling for coupled services
- ✅ Use antiAffinity with topologyKey for HA
- ✅ Consider performance impact
- ❌ Don't create scheduling deadlocks

### Resource Management
- ✅ Always set requests for production
- ✅ Set limits to prevent resource hogging
- ✅ Use QoS classes appropriately
- ✅ Monitor and adjust based on usage
- ❌ Don't set limits too low (causes eviction)
- ❌ Don't rely on bursting (unpredictable)

### Priorities
- ✅ Define clear priority hierarchy
- ✅ Use PodDisruptionBudgets for critical services
- ✅ Document priority policies
- ✅ Review regularly
- ❌ Don't make everything high-priority
- ❌ Don't allow unlimited preemption

---

## Troubleshooting

```bash
# Check schedulability
kubectl describe pod pod-name
# Look for "Events" section - scheduling issues shown here

# Check node resources
kubectl describe node node-name
# Shows allocated/requested resources

# Check available nodes for pod
kubectl get nodes -L kubernetes.io/hostname
kubectl get nodes --show-labels

# Check pod scheduling constraints
kubectl get pod pod-name -o yaml | grep -A 10 affinity

# Check priority classes
kubectl get priorityclasses

# Check taints
kubectl describe node node-name | grep Taints

# Check resource usage
kubectl top node
kubectl top pod

# Force reschedule
kubectl delete pod pod-name
# Deployment controller will reschedule

# Check scheduler logs
kubectl logs -n kube-system -l component=kube-scheduler
```

---

## Summary

- **Manual Scheduling**: nodeName and nodeSelector for direct placement
- **Taints & Tolerations**: Prevent or allow scheduling on specific nodes
- **Node Affinity**: Sophisticated node selection with expressions
- **Pod Affinity**: Co-locate or separate Pods based on other Pods
- **Resource Requests & Limits**: Guarantee and cap resources
- **Pod Priorities**: Determine scheduling order and preemption

Master scheduling to optimize cluster utilization, ensure high availability, and manage resource allocation effectively!
