# Kubernetes Helm & Packaging + Autoscaling

## Table of Contents
1. [Helm & Packaging](#helm--packaging)
2. [What is Helm](#what-is-helm)
3. [Helm Charts](#helm-charts)
4. [Values and Templates](#values-and-templates)
5. [Installing and Upgrading](#installing-and-upgrading)
6. [Autoscaling](#autoscaling)
7. [Horizontal Pod Autoscaler (HPA)](#horizontal-pod-autoscaler-hpa)
8. [Vertical Pod Autoscaler (VPA)](#vertical-pod-autoscaler-vpa)
9. [Cluster Autoscaler](#cluster-autoscaler)
10. [Best Practices](#best-practices)

---

# Helm & Packaging

## What is Helm

**Helm** is the package manager for Kubernetes. It helps deploy and manage applications on Kubernetes clusters.

### Why Helm?

Problem without Helm:
```yaml
# Manual deployment requires multiple files
deployment.yaml
service.yaml
configmap.yaml
secret.yaml
pvc.yaml
ingress.yaml
# Plus environment-specific versions:
deployment-prod.yaml
deployment-staging.yaml
deployment-dev.yaml
```

Solution with Helm:
```
Single Helm Chart
├── Production: helm install myapp ./chart --values values-prod.yaml
├── Staging: helm install myapp ./chart --values values-staging.yaml
└── Development: helm install myapp ./chart --values values-dev.yaml
```

### Helm Architecture

```
┌──────────────────────────┐
│    Helm Command Line     │
│     (helm install,       │
│      helm upgrade,       │
│      helm uninstall)     │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│  Helm Engine             │
│  - Parse Chart           │
│  - Merge Values          │
│  - Render Templates      │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│  Rendered Manifests      │
│  (YAML files)            │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│  kubectl apply           │
│  (Deploy to cluster)     │
└──────────────────────────┘
```

### Key Concepts

| Concept | Purpose |
|---------|---------|
| **Chart** | Package with all K8s resources |
| **Release** | Running instance of a Chart |
| **Values** | Configuration data for Chart |
| **Template** | K8s manifest with variables |
| **Repository** | Collection of Charts |

---

## Helm Charts

### Chart Directory Structure

```
my-app-chart/
├── Chart.yaml              # Chart metadata
├── values.yaml              # Default values
├── values-prod.yaml         # Production values
├── charts/                  # Sub-charts (dependencies)
├── templates/               # K8s manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl         # Helper templates
│   └── NOTES.txt            # Post-install notes
├── README.md
└── .helmignore              # Files to ignore
```

### Chart.yaml

```yaml
apiVersion: v2
name: my-app
description: My application Helm chart
type: application
version: 1.0.0              # Chart version
appVersion: "1.0"           # Application version

# Metadata
keywords:
  - myapp
  - kubernetes
home: https://github.com/myorg/my-app
sources:
  - https://github.com/myorg/my-app
maintainers:
  - name: John Doe
    email: john@example.com

# Dependencies
dependencies:
  - name: postgresql
    version: "11.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

### Creating a Chart

```bash
# Create new chart structure
helm create my-app-chart

# This generates default chart with example templates
cd my-app-chart
ls -la

# Chart structure is created automatically
```

### Chart Validation

```bash
# Validate chart syntax
helm lint my-app-chart

# Output example:
# ==> Linting my-app-chart
# [INFO] Chart.yaml: icon is recommended
# 1 chart(s) linted, 0 chart(s) failed
```

### Packaging Chart

```bash
# Create .tgz archive
helm package my-app-chart
# Creates: my-app-chart-1.0.0.tgz

# Upload to repository
helm repo index .
# Creates index.yaml for repository
```

---

## Values and Templates

### Values Files

#### Default Values (values.yaml)

```yaml
# Default values for my-app.
# This is a YAML-formatted file.

# Replica count
replicaCount: 2

image:
  repository: myapp
  pullPolicy: IfNotPresent
  tag: "1.0.0"

nameOverride: ""
fullnameOverride: ""

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  host: example.com
  path: /
  tls:
    enabled: true
    secretName: example-tls

# Resource limits
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

# Autoscaling
autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

# Environment variables
env:
  LOG_LEVEL: "INFO"
  DEBUG: "false"

# Database
database:
  enabled: false
  host: postgres.default
  port: 5432
  name: myapp

# Node selector
nodeSelector: {}

# Tolerations
tolerations: []

# Affinity
affinity: {}
```

#### Environment-Specific Values

**values-prod.yaml**
```yaml
replicaCount: 5

image:
  tag: "1.0.0"
  pullPolicy: Always

resources:
  limits:
    cpu: 2000m
    memory: 2Gi
  requests:
    cpu: 1000m
    memory: 1Gi

autoscaling:
  enabled: true
  minReplicas: 5
  maxReplicas: 50
  targetCPUUtilizationPercentage: 70

env:
  LOG_LEVEL: "WARN"
  DEBUG: "false"

database:
  enabled: true
  host: postgres-prod.production
  port: 5432
  name: myapp_prod
```

**values-dev.yaml**
```yaml
replicaCount: 1

image:
  tag: "latest"
  pullPolicy: Always

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false

env:
  LOG_LEVEL: "DEBUG"
  DEBUG: "true"

database:
  enabled: false
```

### Helm Templates

#### Deployment Template (templates/deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "my-app.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.service.targetPort }}
          protocol: TCP
        
        # Environment variables from values
        env:
        {{- range $key, $value := .Values.env }}
        - name: {{ $key }}
          value: {{ $value | quote }}
        {{- end }}
        
        # Database config if enabled
        {{- if .Values.database.enabled }}
        - name: DB_HOST
          value: {{ .Values.database.host }}
        - name: DB_PORT
          value: {{ .Values.database.port | quote }}
        - name: DB_NAME
          value: {{ .Values.database.name }}
        {{- end }}
        
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
        
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
      
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

#### Service Template (templates/service.yaml)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "my-app.selectorLabels" . | nindent 4 }}
```

#### Ingress Template (templates/ingress.yaml)

```yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls.enabled }}
  tls:
    - hosts:
        - {{ .Values.ingress.host }}
      secretName: {{ .Values.ingress.tls.secretName }}
  {{- end }}
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: {{ .Values.ingress.path }}
            pathType: Prefix
            backend:
              service:
                name: {{ include "my-app.fullname" . }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
```

#### Helper Templates (_helpers.tpl)

```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "my-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "my-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "my-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "my-app.labels" -}}
helm.sh/chart: {{ include "my-app.chart" . }}
{{ include "my-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "my-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "my-app.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "my-app.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

### Template Functions

| Function | Purpose |
|----------|---------|
| **include** | Include one template in another |
| **default** | Use default value if not set |
| **toYaml** | Convert to YAML |
| **tpl** | Render string as template |
| **range** | Loop over values |
| **if/else** | Conditional rendering |
| **quote** | Quote string |
| **upper/lower** | Change case |
| **nindent** | Indent with newline |

---

## Installing and Upgrading

### Helm Repositories

```bash
# Add Helm repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add stable https://charts.helm.sh/stable

# List repositories
helm repo list

# Update repositories
helm repo update

# Search for chart
helm search repo postgresql

# Search with version filter
helm search repo postgresql --versions
```

### Installing Applications

#### Basic Installation

```bash
# Install from repository
helm install my-release bitnami/postgresql

# Install from local chart
helm install my-app ./my-app-chart

# Install with custom values
helm install my-app ./my-app-chart \
  --values values-prod.yaml

# Install with override values
helm install my-app ./my-app-chart \
  --set replicas=5 \
  --set image.tag=2.0.0 \
  --set ingress.enabled=true

# Install in specific namespace
helm install my-app ./my-app-chart \
  -n production \
  --create-namespace

# Install with timeout
helm install my-app ./my-app-chart \
  --timeout 10m
```

#### Dry-Run and Preview

```bash
# Validate without installing
helm install my-app ./my-app-chart --dry-run

# Generated manifests
helm install my-app ./my-app-chart --dry-run --debug

# See what would be installed
helm template my-app ./my-app-chart --values values-prod.yaml
```

### Upgrading Releases

```bash
# Upgrade to new version
helm upgrade my-app ./my-app-chart

# Upgrade with new values
helm upgrade my-app ./my-app-chart \
  --values values-prod.yaml

# Upgrade specific value
helm upgrade my-app ./my-app-chart \
  --set image.tag=2.0.0

# Upgrade from repository
helm upgrade my-app bitnami/postgresql

# Upgrade keeping previous version (for rollback)
helm upgrade my-app ./my-app-chart \
  --install                    # Install if not exists

# Upgrade with history
helm upgrade my-app ./my-app-chart \
  --wait                       # Wait for ready
```

### Rollback and History

```bash
# View release history
helm history my-app

# Historical values
helm get values my-app --revision 1

# Rollback to previous version
helm rollback my-app

# Rollback to specific revision
helm rollback my-app 1

# View current values
helm get values my-app

# View current manifests
helm get manifest my-app
```

### Release Management

```bash
# List releases
helm list

# List in all namespaces
helm list -A

# Get release details
helm status my-app

# Get release info
helm get all my-app

# Uninstall release
helm uninstall my-app

# Uninstall with history
helm uninstall my-app --keep-history
```

---

# Autoscaling

## Overview

Kubernetes provides three autoscaling mechanisms:

```
┌─────────────────────────────────────────────┐
│         Kubernetes Autoscaling              │
├─────────────────────────────────────────────┤
│                                             │
│  HPA (Horizontal Pod Autoscaler)            │
│  ├─ Scale pods up/down                      │
│  └─ Based on metrics (CPU, memory, custom)  │
│                                             │
│  VPA (Vertical Pod Autoscaler)              │
│  ├─ Adjust pod resource requests            │
│  └─ Based on actual usage                   │
│                                             │
│  Cluster Autoscaler                         │
│  ├─ Add/remove nodes                        │
│  └─ Based on pod scheduling needs           │
│                                             │
└─────────────────────────────────────────────┘
```

---

## Horizontal Pod Autoscaler (HPA)

**HPA** automatically scales the number of pod replicas based on observed metrics.

### Prerequisites

```bash
# Metrics-Server must be installed
kubectl get deployment metrics-server -n kube-system

# If not present:
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### HPA Based on CPU

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app              # Scale this deployment
  
  minReplicas: 2              # Minimum pods
  maxReplicas: 10             # Maximum pods
  
  metrics:
  # Scale based on CPU utilization
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # 70% CPU usage
  
  # Scale based on memory utilization
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80  # 80% memory usage
  
  # Scale down behavior
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
      policies:
      - type: Percent
        value: 50              # Scale down by 50% max
        periodSeconds: 60
```

### HPA Based on Custom Metrics

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa-custom
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  
  minReplicas: 1
  maxReplicas: 100
  
  metrics:
  # Custom metric: requests per pod
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1k"    # 1000 requests/sec per pod
  
  # Container resource metric
  - type: ContainerResource
    containerResource:
      container: app
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
```

### HPA Commands

```bash
# Create HPA
kubectl autoscale deployment my-app \
  --min=2 --max=10 \
  --cpu-percent=70

# View HPA
kubectl get hpa
kubectl get hpa -o wide

# Describe HPA
kubectl describe hpa app-hpa

# View HPA events
kubectl get events | grep HorizontalPodAutoscaler

# Delete HPA
kubectl delete hpa app-hpa

# Monitor autoscaling
kubectl get hpa app-hpa --watch
```

### HPA Scaling Behavior

```
Initial: 2 replicas
        │
        ▼
CPU: 85% (above 70% target)
        │
        ▼
Scale up: 2 → 4 (double)
        │
        ▼
Wait: 3 min before next scale
        │
        ▼
CPU: 30% (below 70% target)
        │
        ▼
Scale down: 4 → 3 → 2
```

---

## Vertical Pod Autoscaler (VPA)

**VPA** recommends or automatically adjusts CPU and memory requests based on actual usage.

### Installation

```bash
# Clone VPA repository
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler

# Install VPA
./hack/vpa-up.sh

# Verify installation
kubectl get deployment -n kube-system | grep vpa
```

### VPA Modes

| Mode | Behavior |
|------|----------|
| **Off** | Only provides recommendations |
| **Initial** | Set at pod creation, no updates |
| **Recreate** | Update and recreate pod |
| **Auto** | Recreate if possible, else run in background |

### VPA Configuration

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: my-app
  
  updatePolicy:
    updateMode: "Auto"        # Off, Initial, Recreate, Auto
  
  resourcePolicy:
    containerPolicies:
    - containerName: "*"      # All containers
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2
        memory: 2Gi
      # Control scaling speed
      controlledResources:
      - cpu
      - memory
      controlledValues: RequestsAndLimits
```

### VPA with Static Limits

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 100m
        memory: 128Mi

---
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-vpa
spec:
  targetRef:
    apiVersion: "v1"
    kind: Pod
    name: app-pod
  
  updatePolicy:
    updateMode: "Off"          # Only provide recommendations
  
  resourcePolicy:
    containerPolicies:
    - containerName: app
      controlledValues: RequestsOnly  # Only adjust requests
```

### VPA Commands

```bash
# View VPA recommendations
kubectl describe vpa my-app-vpa

# View VPA status
kubectl get vpa

# View pods managed by VPA
kubectl get pods -L verticalpodautoscaler

# Check recommended resources
kubectl get vpa my-app-vpa -o yaml | grep -A 10 "recommendation"

# View VPA controller logs
kubectl logs -n kube-system -l app=vpa-recommender
```

---

## Cluster Autoscaler

**Cluster Autoscaler** automatically scales the number of nodes in the cluster based on pod scheduling needs.

### Installation (AWS EKS Example)

```bash
# Create IAM policy for cluster autoscaler
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/iam_policy.json

aws iam create-policy \
  --policy-name AmazonEKSClusterAutoscalerPolicy \
  --policy-document file://iam_policy.json

# Create service account
kubectl create serviceaccount cluster-autoscaler -n kube-system

# Attach IAM role to service account (IRSA)
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --name=cluster-autoscaler \
  --namespace=kube-system \
  --attach-policy-arn=arn:aws:iam::AWS_ACCOUNT:policy/AmazonEKSClusterAutoscalerPolicy

# Install Helm chart
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  -n kube-system \
  --set autoDiscovery.clusterName=my-cluster
```

### Cluster Autoscaler Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: cluster-autoscaler
        image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.24.0
        command:
        - ./cluster-autoscaler
        - --cloud-provider=aws
        - --nodes=2:10:aws-nodes    # min:max:ASG-name
        - --scale-down-enabled=true
        - --skip-nodes-with-local-storage=false
        - --balance-similar-node-groups
        - --balancing-ignore-label=node.kubernetes.io/exclude-from-external-load-balancers
```

### Node Scaling Behavior

```
Pod creation
    │
    ▼
Pod pending (not scheduled)
    │
    ▼
Cluster Autoscaler detects
    │
    ▼
Add node to ASG
    │
    ▼
New node joins cluster
    │
    ▼
Pod scheduled on new node
    │
    ▼

(Low CPU/Memory utilization for 10 min)
    │
    ▼
Drain node
    │
    ▼
Remove node from ASG
```

### Autoscaler Annotations

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  annotations:
    # Prevent pod eviction during scale-down
    cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
spec:
  containers:
  - name: app
    image: myapp:latest
```

---

## Integrating HPA, VPA, and Cluster Autoscaler

### Complete Autoscaling Setup

```yaml
---
# Deployment with base resources
apiVersion: apps/v1
kind: Deployment
metadata:
  name: complete-auto-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: complete-auto-app
  template:
    metadata:
      labels:
        app: complete-auto-app
    spec:
      containers:
      - name: app
        image: myapp:latest
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi

---
# Horizontal Pod Autoscaler (scale pods)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: complete-auto-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: complete-auto-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80

---
# Vertical Pod Autoscaler (adjust resources)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: complete-auto-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: complete-auto-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        cpu: 50m
        memory: 64Mi
      maxAllowed:
        cpu: 2
        memory: 2Gi
```

---

## Best Practices

### Helm Best Practices

- ✅ Use semantic versioning for charts (Chart.yaml)
- ✅ Make values configurable for all deployable options
- ✅ Use helper templates to reduce duplication
- ✅ Include resource requests and limits in values
- ✅ Document values with comments
- ✅ Test charts with multiple value combinations
- ✅ Use namespace in templates for flexibility
- ✅ Version charts for backward compatibility
- ❌ Don't hardcode values in templates
- ❌ Don't use latest image tag in production

### HPA Best Practices

- ✅ Set resource requests (HPA uses these for calculations)
- ✅ Use multiple metrics (CPU + memory)
- ✅ Set appropriate min/max replicas
- ✅ Test scaling behavior before production
- ✅ Use behavior policies for stable scaling
- ✅ Monitor HPA events for scaling triggers
- ✅ Combine with Pod Disruption Budgets
- ✅ Use custom metrics for application-specific logic
- ❌ Don't scale to zero (set minReplicas >= 1)
- ❌ Don't set very aggressive scale-up policies

### VPA Best Practices

- ✅ Use "Off" mode first to see recommendations
- ✅ Start with "Initial" to avoid pod disruption
- ✅ Use with HPA cautiously (can conflict)
- ✅ Set min/max resource limits
- ✅ Monitor VPA recommendations
- ✅ Use for right-sizing existing deployments
- ✅ Don't combine VPA scale mode with HPA on same metric
- ❌ Don't use both HPA and VPA in "Auto" mode on CPU
- ❌ Don't set very low minAllowed limits

### Cluster Autoscaler Best Practices

- ✅ Set appropriate min/max node counts
- ✅ Use multiple node pools for different workload types
- ✅ Tag nodes for discovery
- ✅ Enable scale-down for cost optimization
- ✅ Monitor node utilization and scaling events
- ✅ Use pod disruption budgets to maintain availability
- ✅ Test during maintenance windows first
- ❌ Don't disable scale-down permanently
- ❌ Don't create pods with local storage (they won't evict)

### Combined Autoscaling Example

```yaml
---
# Service with three-tier autoscaling
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-api
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: production-api
      tier: api
  template:
    metadata:
      labels:
        app: production-api
        tier: api
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: production-api
              topologyKey: kubernetes.io/hostname
      
      containers:
      - name: api
        image: production-api:v1.0.0
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 1Gi

---
# HPA: Scale pods 3 to 50 based on CPU/Memory
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: production-api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: production-api
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30

---
# VPA: Optimize resource requests
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: production-api-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: production-api
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: api
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2
        memory: 2Gi
      controlledResources:
      - cpu
      - memory
      controlledValues: RequestsAndLimits

---
# Pod Disruption Budget: Ensure availability
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: production-api-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: production-api
```

---

## Summary

**Helm & Packaging**:
- Helm simplifies Kubernetes application management
- Charts provide templating and reusability
- Values enable environment-specific configuration
- Install/upgrade easily manage application lifecycle

**Autoscaling**:
- **HPA**: Scale pods based on metrics (CPU, memory, custom)
- **VPA**: Optimize resource requests based on usage
- **Cluster Autoscaler**: Add/remove nodes based on demand
- Combine all three for optimal resource utilization and cost efficiency

Effective packaging and autoscaling ensure efficient, scalable Kubernetes deployments!
