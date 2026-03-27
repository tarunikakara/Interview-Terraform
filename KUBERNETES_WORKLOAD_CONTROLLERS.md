# Kubernetes Workload Controllers

## Table of Contents
1. [Overview](#overview)
2. [Deployments](#deployments)
3. [ReplicaSets](#replicasets)
4. [DaemonSets](#daemonsets)
5. [StatefulSets](#statefulsets)
6. [Jobs & CronJobs](#jobs--cronjobs)
7. [Comparison Table](#comparison-table)

---

## Overview

**Workload Controllers** are Kubernetes objects that manage and ensure the desired state of application workloads. They create, update, and manage Pods based on specifications you define.

### Key Concepts
- **Desired State**: What you want (e.g., 3 replicas running)
- **Current State**: What actually exists
- **Reconciliation**: Controller continuously ensures current state matches desired state

---

## Deployments

### What is a Deployment?

A **Deployment** is the most common way to run applications in Kubernetes. It manages:
- **Multiple replicas** of your application
- **Rolling updates** without downtime
- **Rollback** to previous versions
- **Self-healing** (replacing failed Pods)

### Key Features

| Feature | Description |
|---------|------------|
| **Replicas** | Number of Pod copies to maintain |
| **Rolling Update** | Gradually replace old Pods with new ones |
| **Rollback** | Revert to previous version if needed |
| **Revision History** | Keep history of updates for rollback |
| **Selector** | Identify which Pods belong to this Deployment |
| **Strategy** | How to update (RollingUpdate or Recreate) |

### Basic Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

### Your Deployment Configuration

From your `deployment.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-taruni
spec:
  replicas: 6
  revisionHistoryLimit: 3              # Keep last 3 versions for rollback
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate                # Gradual update strategy
    rollingUpdate:
      maxSurge: 2                       # Max 2 extra Pods during update
      maxUnavailable: 0                 # Keep all running (zero downtime)
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

**What this does:**
- Maintains 6 nginx replicas
- Uses rolling updates (2 new Pods created, 2 old removed at a time)
- Zero downtime deployment (maxUnavailable: 0)
- Keeps 3 old versions for rollback

### Rolling Update Strategy

```
Initial State (6 Pods with v1.0):
Pod-1(v1.0), Pod-2(v1.0), Pod-3(v1.0), Pod-4(v1.0), Pod-5(v1.0), Pod-6(v1.0)

Step 1 (maxSurge: 2, create new):
Pod-1(v1.0), Pod-2(v1.0), Pod-3(v1.0), Pod-4(v1.0), Pod-5(v1.0), Pod-6(v1.0),
Pod-7(v2.0), Pod-8(v2.0)

Step 2 (maxUnavailable: 0, remove old):
Pod-3(v1.0), Pod-4(v1.0), Pod-5(v1.0), Pod-6(v1.0),
Pod-7(v2.0), Pod-8(v2.0), Pod-9(v2.0), Pod-10(v2.0)

Final State (6 Pods with v2.0):
Pod-9(v2.0), Pod-10(v2.0), Pod-11(v2.0), Pod-12(v2.0), Pod-13(v2.0), Pod-14(v2.0)
```

### Deployment Update Strategies

#### 1. **RollingUpdate** (Default)
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1               # Max 1 extra Pod during update
    maxUnavailable: 1         # Max 1 Pod unavailable during update
```
**Use case:** Minimal downtime, gradual rollout
- Zero downtime or controlled downtime
- Can have temporary excess resources

#### 2. **Recreate**
```yaml
strategy:
  type: Recreate
```
**Use case:** Applications that can't run multiple versions
- All old Pods terminated, immediately replaced with new
- Brief downtime period
- No temporary excess resources

### Deployment Rollback

```bash
# View deployment history
kubectl rollout history deployment/deployment-taruni

# Check specific revision details
kubectl rollout history deployment/deployment-taruni --revision=2

# Rollback to previous version
kubectl rollout undo deployment/deployment-taruni

# Rollback to specific revision
kubectl rollout undo deployment/deployment-taruni --to-revision=2

# Pause rollout (for inspection)
kubectl rollout pause deployment/deployment-taruni

# Resume rollout
kubectl rollout resume deployment/deployment-taruni

# Check rollout status
kubectl rollout status deployment/deployment-taruni
```

### Updating a Deployment

```bash
# Method 1: Edit the deployment
kubectl edit deployment/deployment-taruni

# Method 2: Patch specific field
kubectl set image deployment/deployment-taruni nginx=nginx:1.16.0

# Method 3: Apply updated YAML
kubectl apply -f deployment.yml
```

---

## ReplicaSets

### What is a ReplicaSet?

A **ReplicaSet** ensures that a specified number of Pod replicas are running at all times. It's the low-level controller that Deployments use internally.

### Key Characteristics
- **Replica Management**: Maintains exact number of running Pods
- **Self-Healing**: Replaces failed or deleted Pods
- **Selector-Based**: Uses label selectors to identify Pods
- **Usually Not Created Directly**: Deployments manage ReplicaSets for you

### Your ReplicaSet Configuration

From your `replicaset.yml`:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: arun-rs
spec:
  replicas: 10
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

**What this does:**
- Maintains exactly 10 running nginx Pods
- If a Pod dies, automatically creates a replacement
- If a Pod is added externally, it's removed to maintain 10
- All Pods have label `app: nginx`

### ReplicaSet vs Deployment

| Aspect | ReplicaSet | Deployment |
|--------|-----------|-----------|
| **Purpose** | Manage Pod replicas | Manage ReplicaSets & updates |
| **Updates** | Not supported | Supports rolling updates |
| **Rollback** | Not supported | Full rollback support |
| **Usage** | Rarely used directly | Standard way to deploy apps |

### When to Use ReplicaSet

Directly use ReplicaSets only when you need:
- Simple static replication without updates
- Complete control over Pod management
- Integration with other custom controllers

**Better practice**: Use Deployments instead (they manage ReplicaSets internally)

### How Deployment Manages ReplicaSets

When you create a Deployment:
1. Deployment creates a ReplicaSet
2. ReplicaSet creates and manages Pods
3. On update, Deployment creates a new ReplicaSet
4. Old ReplicaSet scaled down, new one scaled up

```bash
# See ReplicaSets created by Deployments
kubectl get replicasets

# View the hierarchy
kubectl describe deployment/deployment-taruni
# Shows which ReplicaSet it's using
```

---

## DaemonSets

### What is a DaemonSet?

A **DaemonSet** ensures that a copy of a Pod runs on every node in the cluster (or nodes matching specific selectors).

### Key Characteristics
- **One Pod per Node**: Automatically created on new nodes
- **System Services**: Perfect for system daemons
- **Node-Specific**: Can target specific nodes with selectors
- **Auto-Cleanup**: Removed when nodes are deleted

### Use Cases

| Use Case | Example |
|----------|---------|
| **Monitoring** | Prometheus Node Exporter on all nodes |
| **Logging** | Log aggregators (Fluentd, Logstash) on all nodes |
| **Networking** | Network plugins, service mesh proxies |
| **System Management** | System upgrade daemons |
| **Security** | Security scanners on all nodes |

### Basic DaemonSet Example

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
      - name: exporter
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
        volumeMounts:
        - name: proc
          mountPath: /proc
        - name: sys
          mountPath: /sys
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
```

### DaemonSet with Node Selector

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-daemon
spec:
  selector:
    matchLabels:
      app: monitor
  template:
    metadata:
      labels:
        app: monitor
    spec:
      # Only run on nodes with label: disktype=ssd
      nodeSelector:
        disktype: ssd
      
      containers:
      - name: monitor
        image: monitoring-agent:latest
```

```bash
# Label a node for DaemonSet targeting
kubectl label node node-1 disktype=ssd
```

### DaemonSet with Taints & Tolerations

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: system-monitor
spec:
  selector:
    matchLabels:
      app: sys-monitor
  template:
    metadata:
      labels:
        app: sys-monitor
    spec:
      # Tolerate tainted nodes (e.g., master nodes)
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      
      containers:
      - name: monitor
        image: monitor:latest
```

### Managing DaemonSets

```bash
# View all DaemonSets
kubectl get daemonsets

# Describe a DaemonSet
kubectl describe daemonset/node-exporter

# Update a DaemonSet image
kubectl set image daemonset/node-exporter exporter=prom/node-exporter:v1.0.0

# Delete a DaemonSet
kubectl delete daemonset/node-exporter
```

### DaemonSet Pod Lifecycle

```
New Node in Cluster
        ↓
DaemonSet Controller Detects It
        ↓
Creates Pod on New Node
        ↓
Pod Monitors/Services Node
        ↓
Node Removed or Drained
        ↓
DaemonSet Pod Deleted Automatically
```

---

## StatefulSets

### What is a StatefulSet?

A **StatefulSet** manages stateful applications requiring:
- **Stable identities**: predictable Pod names and hostnames
- **Persistent storage**: each Pod has its own PVC
- **Ordered deployment**: Pods created/updated in order
- **Network identity**: stable DNS names

### Key Characteristics

| Feature | Description |
|---------|------------|
| **Pod Identity** | Pod names like `mysql-0`, `mysql-1`, `mysql-2` (ordinal) |
| **Persistent Storage** | Each Pod gets own PVC via volumeClaimTemplates |
| **Ordered Deployment** | Pods created sequentially (0 → 1 → 2) |
| **Ordered Termination** | Pods deleted in reverse order (2 → 1 → 0) |
| **Headless Service** | Required for stable DNS names |
| **Network Identity** | DNS: `pod-name.service-name.namespace.svc.cluster.local` |

### Use Cases

| Application | Why StatefulSet |
|-------------|-----------------|
| **Databases** (MySQL, PostgreSQL) | Each instance needs separate data |
| **Message Queues** (RabbitMQ, Kafka) | Preserving queue state |
| **Search Engines** (Elasticsearch) | Persistent indices per node |
| **Cache Clusters** (Redis, Memcached) | Consistent data across cluster |
| **Distributed Systems** | Ordinal Pod names for coordination |

### Your StatefulSet Configuration

From your `statefullset-pvc.yml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-statefulset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-statefulset
  serviceName: my-statefulset-service    # Headless service (required)
  template:
    metadata:
      labels:
        app: my-statefulset
    spec:
      containers:
      - name: my-container
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: my-volume
          mountPath: /usr/share/nginx/html/
      volumes:
      - name: my-volume
        persistentVolumeClaim:
          claimName: my-pvc
  
  # Automatically create PVCs for each Pod
  volumeClaimTemplates:
  - metadata:
      name: my-volume-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi

---
apiVersion: v1
kind: Service
metadata:
  name: my-statefulset-service
spec:
  # Important: Must be Headless Service for StatefulSet
  clusterIP: None
  selector:
    app: my-statefulset
  ports:
  - port: 80
    targetPort: 80
```

**What this does:**
- Creates 3 replicas: `my-statefulset-0`, `my-statefulset-1`, `my-statefulset-2`
- Each Pod gets its own PVC: `my-volume-data-my-statefulset-0`, `my-volume-data-my-statefulset-1`, etc.
- Stable DNS names: `my-statefulset-0.my-statefulset-service.default.svc.cluster.local`
- Data persists across Pod restarts

### Headless Service (Required)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  clusterIP: None                    # Makes it headless
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
```

Without `clusterIP: None`:
- StatefulSet won't have stable DNS names
- Pods can't reliably discover each other
- Pod identity is lost

### StatefulSet with ConfigMap & PVC

From your `statefullset-config.yml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-statefulset-config
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-statefulset-01
  serviceName: my-service-01
  template:
    metadata:
      labels:
        app: my-statefulset-01
    spec:
      containers:
      - name: my-container-01
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: my-volume
          mountPath: /usr/share/nginx/html/
      volumes:
      - name: my-volume
        configMap:
          name: taruni-configmap
  
  volumeClaimTemplates:
  - metadata:
      name: my-volume-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

### StatefulSet Deployment Flow

```
Step 1: Create my-statefulset-0 Pod
         ↓
Step 2: Pod ready? → Yes
         ↓
Step 3: Create my-statefulset-1 Pod
         ↓
Step 4: Pod ready? → Yes
         ↓
Step 5: Create my-statefulset-2 Pod
         ↓
Step 6: All Complete
```

**Termination Flow (Reverse Order):**
```
Step 1: Delete my-statefulset-2
         ↓
Step 2: Pod terminated? → Yes
         ↓
Step 3: Delete my-statefulset-1
         ↓
Step 4: Pod terminated? → Yes
         ↓
Step 5: Delete my-statefulset-0
         ↓
Step 6: All Cleaned Up
```

### StatefulSet Pod Identity

```bash
# Pod names follow pattern: <name>-<ordinal>
# Example Pod names for replicas: 3
my-statefulset-0
my-statefulset-1
my-statefulset-2

# Persistent Volume Claims
my-volume-data-my-statefulset-0
my-volume-data-my-statefulset-1
my-volume-data-my-statefulset-2

# DNS Names
my-statefulset-0.my-statefulset-service.default.svc.cluster.local
my-statefulset-1.my-statefulset-service.default.svc.cluster.local
my-statefulset-2.my-statefulset-service.default.svc.cluster.local

# All Pods DNS name
my-statefulset.my-statefulset-service.default.svc.cluster.local
```

### Managing StatefulSets

```bash
# View StatefulSets
kubectl get statefulsets

# Describe
kubectl describe statefulset/my-statefulset

# Update image
kubectl set image statefulset/my-statefulset my-container=nginx:1.20

# Scale (ordered)
kubectl scale statefulset/my-statefulset --replicas=5

# Check Pod ordinal
kubectl get pods -o wide | grep my-statefulset

# Delete StatefulSet (pods persist by default)
kubectl delete statefulset/my-statefulset

# Delete StatefulSet and Pods
kubectl delete statefulset/my-statefulset --cascade=foreground

# Delete StatefulSet but keep Pods
kubectl delete statefulset/my-statefulset --cascade=orphan
```

### Ordered Pod Management

```bash
# Pods created in order (sequential)
kubectl get pods -o wide

# Pod 0 starts first
# Pod 0 becomes ready
# Pod 1 starts
# Pod 1 becomes ready
# Pod 2 starts
# Pod 2 becomes ready
```

---

## Jobs & CronJobs

### What are Jobs?

A **Job** creates one or more Pods and ensures they complete successfully. Unlike Deployments/ReplicaSets (which run continuously), Jobs run to completion.

### Key Characteristics
- **Run to Completion**: Pod runs until it exits with success
- **One-Time or Batch**: Execute task and terminate
- **Retry Policy**: Restart failed Pods automatically
- **Parallelism**: Run multiple Pods in parallel
- **Completions**: How many Pods must complete successfully

### Use Cases

| Use Case | Example |
|----------|---------|
| **Batch Processing** | Process data batch, then exit |
| **Database Migrations** | Run migrations on startup |
| **Backup Tasks** | Backup database to storage |
| **Report Generation** | Generate and export reports |
| **Cleanup Tasks** | Cleanup old data/logs |
| **One-time Setup** | Initialize database, load data |

### Basic Job Example

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-job
spec:
  # Backoff limit for retries
  backoffLimit: 3
  
  # Time limit before terminating
  activeDeadlineSeconds: 3600
  
  template:
    spec:
      # Don't restart on completion
      restartPolicy: Never
      containers:
      - name: backup
        image: backup-tool:latest
        command: ["sh", "-c", "mysqldump -u root -p mydb > /backup/mydb.sql"]
        volumeMounts:
        - name: backup-storage
          mountPath: /backup
      volumes:
      - name: backup-storage
        persistentVolumeClaim:
          claimName: backup-pvc
```

### Job with Parallelism and Completions

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-processing-job
spec:
  parallelism: 3          # Run 3 Pods in parallel
  completions: 10         # Need 10 successful completions
  backoffLimit: 2
  activeDeadlineSeconds: 3600
  
  template:
    spec:
      restartPolicy: OnFailure  # Restart if failed
      containers:
      - name: processor
        image: batch-processor:latest
        command: ["python", "process.py", "$(ITEM_ID)"]
        env:
        - name: ITEM_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

**Execution Pattern:**
```
Start 3 Pods in parallel
Pod 1 completes → Start Pod 4
Pod 2 completes → Start Pod 5
Pod 3 completes → Start Pod 6
... continue until 10 completions
```

### CronJob

A **CronJob** runs Jobs on a schedule, like a cron scheduler in Linux.

### Basic CronJob Example

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  # Cron schedule format
  schedule: "0 2 * * *"              # Daily at 2 AM UTC
  
  # Keep last 3 successful jobs
  successfulJobsHistoryLimit: 3
  
  # Keep last 1 failed job for debugging
  failedJobsHistoryLimit: 1
  
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: backup-sa
          containers:
          - name: backup
            image: backup-tool:latest
            command: ["sh", "-c", "mysqldump -u root -p mydb > /backup/backup-$(date +%Y%m%d).sql"]
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
          restartPolicy: OnFailure
```

### CronJob Schedule Format

```
┌─────────── minute (0 - 59)
│ ┌─────────── hour (0 - 23)
│ │ ┌─────────── day of month (1 - 31)
│ │ │ ┌─────────── month (1 - 12)
│ │ │ │ ┌─────────── day of week (0 - 6) (Sunday to Saturday)
│ │ │ │ │
│ │ │ │ │
* * * * *
```

### CronJob Schedule Examples

| Schedule | Meaning |
|----------|---------|
| `0 0 * * *` | Daily at midnight |
| `0 2 * * *` | Daily at 2 AM |
| `0 */4 * * *` | Every 4 hours |
| `0 9 * * 1-5` | Weekdays at 9 AM |
| `0 9 1 * *` | 1st of month at 9 AM |
| `*/15 * * * *` | Every 15 minutes |
| `0 0 * * 0` | Weekly (Sunday midnight) |
| `30 2 * * *` | Daily at 2:30 AM |

### Managing Jobs and CronJobs

```bash
# View Jobs
kubectl get jobs

# View Job details and logs
kubectl describe job/backup-job
kubectl logs job/backup-job

# Delete Job (also deletes Pods)
kubectl delete job/backup-job

# Delete Job but keep Pods
kubectl delete job/backup-job --cascade=orphan

# View CronJobs
kubectl get cronjobs

# View CronJob schedule and last run
kubectl describe cronjob/daily-backup
kubectl get cronjob/daily-backup

# Check Job runs from CronJob
kubectl get jobs -l cronjob=daily-backup

# Manually trigger CronJob
kubectl create job manual-backup --from=cronjob/daily-backup

# Suspend/Resume CronJob
kubectl patch cronjob daily-backup -p '{"spec":{"suspend":true}}'
kubectl patch cronjob daily-backup -p '{"spec":{"suspend":false}}'
```

### Job Conditions

```bash
# Check Job status
kubectl describe job/backup-job

# Output shows conditions:
# Type         Status  Reason
# Complete     True    Succeeded
# Failed       False
```

### Parallel Job Example: Image Processing

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: image-resize-job
spec:
  parallelism: 5           # Process 5 images in parallel
  completions: 100         # Process 100 images total
  backoffLimit: 3
  
  template:
    metadata:
      labels:
        app: image-processor
    spec:
      restartPolicy: OnFailure
      containers:
      - name: processor
        image: imagemagick:latest
        command: ["sh", "-c"]
        args:
        - |
          # Process image from input directory
          # Job index provided as environment variable
          for img in /input/images/*; do
            convert "$img" -resize 800x600 "/output/$(basename $img)"
          done
```

---

## Comparison Table

### All Workload Controllers at a Glance

| Feature | Deployment | ReplicaSet | DaemonSet | StatefulSet | Job | CronJob |
|---------|-----------|-----------|-----------|------------|-----|---------|
| **Purpose** | General app management | Pod replication | System daemon | Stateful apps | Batch processing | Scheduled tasks |
| **Replicas** | Multiple | Multiple | One per node | Multiple | 1+ (optional) | 1+ per schedule |
| **Pod Identity** | Random names | Random names | Random names | Stable ordinal | Temporary | Temporary |
| **Persistent Storage** | Optional | Optional | No | Per-Pod PVC | Optional | Optional |
| **Updates** | Rolling/Recreate | Not supported | Rolling | Ordered | N/A | Via Job |
| **Lifecycle** | Continuous | Continuous | Continuous | Continuous | Completion | Completion |
| **DNS Names** | Via Service | Via Service | Via Service | Stable FQDN | Temporary | Temporary |
| **Typical Use** | Web servers | Rarely used | Logging, monitoring | Databases, queues | Backups, migrations | Scheduled tasks |
| **Node Affinity** | Optional | Optional | All nodes | Optional | Optional | Optional |

### Decision Tree: Which Workload Controller?

```
START
  ├─ Need multiple instances of same app?
  │   ├─ YES → Need to update/version app?
  │   │   ├─ YES → DEPLOYMENT ✓
  │   │   └─ NO → ReplicaSet (rarely used directly)
  │   └─ NO → Go to next question
  │
  ├─ Need one instance per node?
  │   ├─ YES → DAEMONSET ✓
  │   └─ NO → Go to next question
  │
  ├─ Need stable identities & persistent storage?
  │   ├─ YES → STATEFULSET ✓
  │   └─ NO → Go to next question
  │
  ├─ Need to run once and complete?
  │   ├─ YES → JOB ✓
  │   └─ NO → Go to next question
  │
  └─ Need to run on schedule?
      ├─ YES → CRONJOB ✓
      └─ NO → Review requirements
```

---

## Practical Examples

### Example 1: Web Application Stack

**Deployment** for stateless web server:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: app
        image: myapp:2.0
        ports:
        - containerPort: 8080
```

### Example 2: Database with Storage

**StatefulSet** for database:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-db
spec:
  replicas: 3
  serviceName: postgres-service
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
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### Example 3: Monitoring on All Nodes

**DaemonSet** for monitoring agent:
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: prometheus-node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      tolerations:
      - effect: NoSchedule
        operator: Exists
      containers:
      - name: exporter
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
```

### Example 4: Daily Database Backup

**CronJob** for scheduled backup:
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: backup-sa
          containers:
          - name: backup
            image: backup-image:latest
            command: ["mysqldump", "-u", "root", "-p$MYSQL_ROOT_PASSWORD", "mydb"]
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
          restartPolicy: OnFailure
```

---

## Best Practices

### Deployments
- ✅ Always use Deployments for running applications
- ✅ Set resource requests and limits
- ✅ Use health probes (liveness, readiness)
- ✅ Implement graceful shutdown
- ✅ Version your Docker images
- ❌ Avoid creating ReplicaSets directly

### DaemonSets
- ✅ Use for system-level monitoring and logging
- ✅ Use node selectors for targeted deployments
- ✅ Use tolerations for system nodes
- ✅ Keep resource usage light
- ❌ Don't use for application scaling

### StatefulSets
- ✅ Always use a headless Service
- ✅ Use volumeClaimTemplates for persistence
- ✅ Implement proper shutdown handlers
- ✅ Use init containers for initialization
- ❌ Don't scale StatefulSets frequently
- ❌ Never delete StatefulSet with cascade=foreground in production

### Jobs & CronJobs
- ✅ Set appropriate backoffLimit
- ✅ Use activeDeadlineSeconds to prevent hung jobs
- ✅ Monitor job completion and logs
- ✅ Clean up old job history
- ✅ Set reasonable cron schedules
- ❌ Don't create thousands of parallel Pods

---

## Troubleshooting

```bash
# Check Deployment status
kubectl get deployment -o wide
kubectl describe deployment/deployment-taruni

# View replica counts
kubectl get replicasets

# Check Pod status
kubectl get pods -o wide
kubectl logs <pod-name>
kubectl describe pod <pod-name>

# Check StatefulSet ordering
kubectl get statefulset
kubectl get pods -o wide | grep <statefulset-name>

# Check Job completion
kubectl get jobs
kubectl describe job/<job-name>
kubectl logs job/<job-name>

# Check CronJob history
kubectl get cronjobs
kubectl get jobs -l cronjob=<cronjob-name>
```

---

## Summary

- **Deployments**: General-purpose app management with rolling updates ✓
- **ReplicaSets**: Pod replication engine (used by Deployments) ✓
- **DaemonSets**: One Pod per node for system services ✓
- **StatefulSets**: Stateful apps with stable identities and storage ✓
- **Jobs**: Batch processing and one-time tasks ✓
- **CronJobs**: Scheduled jobs like cron ✓

Each controller serves a specific purpose in Kubernetes workload orchestration. Choose the right controller based on your application requirements!
