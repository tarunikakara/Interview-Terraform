# ArgoCD: Infrastructure & Application Deployment Guide

**Table of Contents**
- [1. What is ArgoCD?](#1-what-is-argocd)
- [2. ArgoCD vs GitHub Actions](#2-argocd-vs-github-actions)
- [3. Infrastructure Pipeline](#3-infrastructure-pipeline)
- [4. Application Deployment Pipeline](#4-application-deployment-pipeline)
- [5. Real-World Troubleshooting](#5-real-world-troubleshooting)
- [6. Best Practices](#6-best-practices)

---

## 1. What is ArgoCD?

### Definition
**ArgoCD** is a **declarative, GitOps continuous delivery tool** for Kubernetes. It monitors a Git repository and automatically syncs the desired state to your Kubernetes cluster.

### Key Features
- **GitOps Model:** Everything is in Git (source of truth)
- **Declarative:** YAML manifests define what you want
- **Automated Sync:** Watches Git repo, deploys changes automatically
- **Multi-repo Support:** Multiple Git repos, branches, paths
- **Health Monitoring:** Tracks pod health, resource status
- **Rollback Support:** Git history = infrastructure history
- **Multi-cluster:** Deploy across multiple K8s clusters

### Why Use ArgoCD?
| Feature | Traditional CI/CD | ArgoCD |
|---------|------------------|--------|
| **Source of Truth** | Pipeline scripts | Git repository |
| **Deployment Trigger** | Webhook on code push | Git monitor (pull-based) |
| **Rollback Strategy** | Manual or scripted | Git revert (one command) |
| **Auditability** | Pipeline logs | Git commit history |
| **Disaster Recovery** | Recreate from backups | Git pull latest state |

---

## 2. ArgoCD vs GitHub Actions

### When to Use ArgoCD
✅ Kubernetes deployments  
✅ Infrastructure as Code (GitOps)  
✅ Multi-cluster deployments  
✅ Continuous deployment (auto-sync)  
✅ Environment management  

### When to Use GitHub Actions
✅ Pre-deployment (build, test, lint)  
✅ Terraform runs (infrastructure provisioning)  
✅ Non-K8s workflows  
✅ Custom integrations  

### Recommended Flow
```
GitHub Actions (CI) → Build & Test → Push Docker Image
                ↓
        ArgoCD (CD) → Monitor Git → Sync to K8s
```

---

## 3. Infrastructure Pipeline

### 3.1 Infrastructure as Code with ArgoCD

Infrastructure (VNet, Storage, Gateway) can be deployed via:
1. **Terraform + Helm:** Provision cloud resources
2. **ArgoCD:** Deploy to Kubernetes

### 3.2 Directory Structure

```
infrastructure/
├── apps/
│   ├── dev/
│   │   ├── vnet.yaml
│   │   ├── storage.yaml
│   │   ├── gateway.yaml
│   │   └── kustomization.yaml
│   ├── staging/
│   └── prod/
├── argocd-appsources/
│   ├── dev-appproject.yaml
│   ├── dev-application.yaml
│   ├── prod-application.yaml
└── helm-charts/
    ├── vnet-helm/
    ├── storage-helm/
    └── gateway-helm/
```

### 3.3 VNet (Virtual Network) Configuration

**File: `apps/dev/vnet.yaml`**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: vnet-config
  namespace: infra-dev
data:
  vnet-name: "vnet-prod-eastus"
  vnet-cidr: "10.0.0.0/16"
  region: "eastus"
  environment: "dev"
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: vnet-deployment
  namespace: infra-dev
resources:
  - vnet-config
```

**Note:** In real scenarios, use **Terraform** to create VNets in Azure/AWS, then reference outputs in K8s ConfigMaps.

### 3.4 Storage Containers Configuration

**File: `apps/dev/storage.yaml`**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: azure-storage-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  azureDisk:
    kind: Managed
    diskName: storage-prod-disk
    diskURI: /subscriptions/{subId}/resourceGroups/{rg}/providers/Microsoft.Compute/disks/storage-prod-disk

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-storage-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  volumeName: azure-storage-pv

---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-managed-premium
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  cachingMode: ReadOnly
```

### 3.5 Gateway (Application Gateway) Configuration

**File: `apps/dev/gateway.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-gateway-lb
  namespace: default
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "false"
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
    - name: https
      protocol: TCP
      port: 443
      targetPort: 8443
  sessionAffinity: ClientIP

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-rules
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: default
      ports:
        - protocol: TCP
          port: 8080
```

### 3.6 ArgoCD Application for Infrastructure

**File: `argocd-appsources/dev-application.yaml`**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: infra-deployment-dev
  namespace: argocd
spec:
  project: dev-project
  source:
    repoURL: https://github.com/tarunikakara/infrastructure-repo.git
    targetRevision: main
    path: apps/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: infra-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
  revisionHistoryLimit: 10
```

### 3.7 Sync Strategy for Infrastructure

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: dev-project
  namespace: argocd
spec:
  sourceRepos:
    - 'https://github.com/tarunikakara/*'
    - 'https://charts.helm.sh/stable'
  destinations:
    - namespace: 'infra-*'
      server: https://kubernetes.default.svc
  namespaceResourceBlacklist:
    - group: ''
      kind: 'ResourceQuota'
    - group: ''
      kind: 'LimitRange'
    - group: 'admissionregistration.k8s.io'
      kind: '*'
```

---

## 4. Application Deployment Pipeline

### 4.1 Multi-Language Support (Java, Python, .NET)

ArgoCD deploys any containerized application. Build step happens in CI (GitHub Actions), deployment in ArgoCD.

### 4.2 Java Application Deployment

**GitHub Actions CI: `build-java.yml`**

```yaml
name: Build Java App

on:
  push:
    branches: [main]
    paths:
      - "java-app/**"

jobs:
  build:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: java-app
    steps:
      - uses: actions/checkout@v4

      - name: Set up Java
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Build with Maven
        run: mvn clean package -DskipTests

      - name: Docker Build & Push
        uses: docker/build-push-action@v5
        with:
          context: java-app
          push: true
          tags: myregistry.azurecr.io/java-app:${{ github.sha }}
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Update Deployment Manifest
        run: |
          sed -i "s|myregistry.azurecr.io/java-app:.*|myregistry.azurecr.io/java-app:${{ github.sha }}|" deployment.yaml
          git config user.name "GitHub Action"
          git config user.email "action@github.com"
          git add deployment.yaml
          git commit -m "Update image to ${{ github.sha }}"
          git push
```

**ArgoCD Application: `apps/prod/java-app.yaml`**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: java-app-prod
  namespace: argocd
spec:
  project: prod-project
  source:
    repoURL: https://github.com/tarunikakara/app-deployment-repo.git
    targetRevision: main
    path: apps/prod/java-app
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: false  # Manual approval for prod
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
```

**Java App Deployment: `java-app/deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
  namespace: prod
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: java-app
  template:
    metadata:
      labels:
        app: java-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      serviceAccountName: java-app
      containers:
        - name: java-app
          image: myregistry.azurecr.io/java-app:latest
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          env:
            - name: JAVA_OPTS
              value: "-Xmx512m -Xms256m"
            - name: APP_ENV
              value: "production"
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: db-host
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: db-password
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 500m
              memory: 1Gi
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          volumeMounts:
            - name: app-config
              mountPath: /etc/app/config
      volumes:
        - name: app-config
          configMap:
            name: java-app-config

---
apiVersion: v1
kind: Service
metadata:
  name: java-app
  namespace: prod
spec:
  selector:
    app: java-app
  type: ClusterIP
  ports:
    - name: http
      port: 80
      targetPort: 8080
```

---

### 4.3 Python Application Deployment

**GitHub Actions CI: `build-python.yml`**

```yaml
name: Build Python App

on:
  push:
    branches: [main]
    paths:
      - "python-app/**"

jobs:
  build:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: python-app
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov black flake8

      - name: Lint with Flake8
        run: flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics

      - name: Format with Black
        run: black --check .

      - name: Test with Pytest
        run: pytest --cov=. --cov-report=xml

      - name: Upload Coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.xml

      - name: Docker Build & Push
        uses: docker/build-push-action@v5
        with:
          context: python-app
          push: true
          tags: myregistry.azurecr.io/python-app:${{ github.sha }}
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Update Deployment Manifest
        run: |
          sed -i "s|myregistry.azurecr.io/python-app:.*|myregistry.azurecr.io/python-app:${{ github.sha }}|" deployment.yaml
          git config user.name "GitHub Action"
          git config user.email "action@github.com"
          git add deployment.yaml
          git commit -m "Update image to ${{ github.sha }}"
          git push
```

**Python App Deployment: `python-app/deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-app
  namespace: prod
  labels:
    app: python-app
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: python-app
  template:
    metadata:
      labels:
        app: python-app
    spec:
      containers:
        - name: python-app
          image: myregistry.azurecr.io/python-app:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 5000
              name: http
          env:
            - name: FLASK_ENV
              value: "production"
            - name: WORKERS
              value: "4"
            - name: LOG_LEVEL
              value: "INFO"
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /health
              port: 5000
            initialDelaySeconds: 20
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 5000
            initialDelaySeconds: 10
            periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: python-app
  namespace: prod
spec:
  selector:
    app: python-app
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
```

---

### 4.4 .NET Application Deployment

**GitHub Actions CI: `build-dotnet.yml`**

```yaml
name: Build .NET App

on:
  push:
    branches: [main]
    paths:
      - "dotnet-app/**"

jobs:
  build:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: dotnet-app
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0'

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --configuration Release --no-restore

      - name: Test
        run: dotnet test --configuration Release --no-build --verbosity normal

      - name: Publish
        run: dotnet publish -c Release -o ./publish

      - name: Docker Build & Push
        uses: docker/build-push-action@v5
        with:
          context: dotnet-app
          push: true
          tags: myregistry.azurecr.io/dotnet-app:${{ github.sha }}
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Update Kustomization
        run: |
          cd kustomize
          kustomize edit set image myregistry.azurecr.io/dotnet-app:${{ github.sha }}
          git config user.name "GitHub Action"
          git config user.email "action@github.com"
          git add kustomization.yaml
          git commit -m "Update .NET app to ${{ github.sha }}"
          git push
```

**.NET App Deployment: `dotnet-app/deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dotnet-app
  namespace: prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: dotnet-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: dotnet-app
    spec:
      serviceAccountName: dotnet-app
      containers:
        - name: dotnet-app
          image: myregistry.azurecr.io/dotnet-app:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
              name: http
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: "Production"
            - name: ASPNETCORE_URLS
              value: "http://+:8080"
            - name: DOTNET_USE_POLLING_FILE_WATCHER
              value: "false"
          resources:
            requests:
              cpu: 200m
              memory: 512Mi
            limits:
              cpu: 500m
              memory: 1Gi
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: dotnet-app
  namespace: prod
spec:
  selector:
    app: dotnet-app
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
      name: http
```

---

## 5. Real-World Troubleshooting

### 5.1 Governance Issues

#### Issue 1: Unauthorized User Trying to Sync Application

**Symptom:**
```
rpc error: code = PermissionDenied desc = permission denied
User 'dev@company.com' cannot perform action 'applications/sync' on 'prod-app'
```

**Root Cause:**
RBAC not configured correctly. User doesn't have permission to access production applications.

**Solution:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: prod-project
  namespace: argocd
spec:
  sourceRepos:
    - 'https://github.com/company/*'
  destinations:
    - namespace: 'prod-*'
      server: https://kubernetes.default.svc
  roles:
    - name: prod-admins
      policies:
        - p, proj:prod-project:prod-admins, applications, *, prod-project/*, allow
      groups:
        - prod-admin-group
    - name: dev-read-only
      policies:
        - p, proj:prod-project:dev-read-only, applications, get, prod-project/*, allow
      groups:
        - dev-group
```

**Prevention:**
- Define RBAC roles per project
- Use groups from identity provider (Azure AD, GitHub Teams)
- Audit logs: `argocd admin logs`

---

#### Issue 2: Resource Quota Violations

**Symptom:**
```
pods is forbidden: exceeded quota
Error: quotas exceeded, refusing to create pod
```

**Root Cause:**
Namespace resource quota exceeded due to too many replicas or high resource requests.

**Solution:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: prod-quota
  namespace: prod
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "100"
    services.loadbalancers: "2"

---
apiVersion: v1
kind: LimitRange
metadata:
  name: prod-limits
  namespace: prod
spec:
  limits:
    - type: Pod
      max:
        cpu: "2"
        memory: "2Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
    - type: Container
      max:
        cpu: "1"
        memory: "1Gi"
      min:
        cpu: "10m"
        memory: "32Mi"
```

**Fix:**
1. Check current usage: `kubectl describe quota -n prod`
2. Reduce replicas or request limits
3. Scale down non-prod applications

---

#### Issue 3: Policy Violation - Disallowed Image Registry

**Symptom:**
```
admission webhook denied the request
image registry 'docker.io' is not in allowlist
```

**Root Cause:**
Pod security policy or image registry policy violation.

**Solution:**

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: allowedregistries
spec:
  crd:
    spec:
      names:
        kind: AllowedRegistries
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          image := container.image
          not startswith(image, "myregistry.azurecr.io/")
          msg := sprintf("Image %v must be from approved registry", [image])
        }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: AllowedRegistries
metadata:
  name: prod-registries
spec:
  match:
    namespaceSelector:
      matchLabels:
        env: prod
```

**Fix:**
- Update deployment to use approved registry
- `myregistry.azurecr.io/app:tag` instead of `docker.io/app:tag`

---

### 5.2 Deployment Issues

#### Issue 4: Image Pull Errors

**Symptom:**
```
Failed to pull image "myregistry.azurecr.io/java-app:latest"
ImagePullBackOff
```

**Root Cause:**
1. Image doesn't exist
2. Registry credentials missing
3. Image tag is wrong

**Diagnosis:**

```bash
# Check image in registry
az acr repository show --name myregistry --image java-app:latest

# Check ImagePullSecrets
kubectl get secrets -n prod
kubectl describe secret acr-secret -n prod

# Check pod events
kubectl describe pod <pod-name> -n prod
```

**Solution:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: acr-secret
  namespace: prod
type: kubernetes.io/dockercfg
data:
  .dockercfg: <base64-encoded-auth>

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
  namespace: prod
spec:
  template:
    spec:
      imagePullSecrets:
        - name: acr-secret
      containers:
        - name: java-app
          image: myregistry.azurecr.io/java-app:v1.2.3
          imagePullPolicy: IfNotPresent  # Use cached image if available
```

**Prevention:**
- Tag images with semantic versioning (v1.2.3, not latest)
- Automate credential rotation
- Scan images before deployment: `az acr scan`

---

#### Issue 5: CrashLoopBackOff - Application Won't Start

**Symptom:**
```
Pod is in CrashLoopBackOff
Last State: Terminated (ExitCode: 1)
```

**Root Cause:**
Application crashes due to:
- Missing environment variables
- Configuration errors
- Database connection failures
- Out of memory

**Debugging:**

```bash
# View logs
kubectl logs <pod-name> -n prod
kubectl logs <pod-name> -n prod --previous  # Previous crash

# Check events
kubectl describe pod <pod-name> -n prod

# SSH into pod (if debug enabled)
kubectl debug <pod-name> -it -n prod

# Check resource usage
kubectl top pod <pod-name> -n prod
```

**Solution Examples:**

**Missing ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: java-app-config
  namespace: prod
data:
  application.properties: |
    server.port=8080
    app.name=JavaApp
    logging.level.root=INFO

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
spec:
  template:
    spec:
      containers:
        - name: java-app
          volumeMounts:
            - name: config
              mountPath: /etc/config
      volumes:
        - name: config
          configMap:
            name: java-app-config
```

**Memory Issues:**
```yaml
spec:
  containers:
    - name: java-app
      resources:
        requests:
          memory: 512Mi
        limits:
          memory: 1Gi
      env:
        - name: JAVA_OPTS
          value: "-Xms256m -Xmx768m"  # Heap size
```

---

#### Issue 6: Sync Failed - Kustomize Build Error

**Symptom:**
```
Application out of sync
Sync Failed: error while building kustomization: ...
```

**Root Cause:**
Invalid kustomization.yaml or missing resource.

**Solution:**

**Before push, validate locally:**
```bash
# Test kustomize build
kustomize build apps/prod/java-app

# Test ArgoCD
argocd app create test-app \
  --repo https://github.com/company/app-repo \
  --path apps/prod/java-app \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace prod \
  --sync-policy automated \
  --dry-run
```

**Valid kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: prod

commonLabels:
  app: java-app
  version: v1.2.3

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

images:
  - name: java-app
    newTag: v1.2.3

patchesStrategicMerge:
  - patch-resources.yaml

vars:
  - name: IMAGE_REPO
    objref:
      kind: Deployment
      name: java-app
      apiVersion: apps/v1
```

---

### 5.3 Connectivity Issues

#### Issue 7: Service Cannot Connect to Database

**Symptom:**
```
Connection refused: unable to connect to database
App timeouts trying to reach db.prod.svc.cluster.local
```

**Root Cause:**
1. DNS resolution failing
2. Network policy blocking traffic
3. Service not exposing correct port
4. Database credentials wrong

**Diagnosis:**

```bash
# Test DNS resolution
kubectl run -it --rm debug --image=busybox:1.28 -- nslookup db.prod.svc.cluster.local

# Test connectivity
kubectl run -it --rm debug --image=busybox:1.28 -- nc -zv db.prod 5432

# Check NetworkPolicy
kubectl get networkpolicy -n prod
kubectl describe networkpolicy app-policy -n prod

# Check Service endpoints
kubectl get endpoints -n prod
kubectl describe svc database -n prod
```

**Solution:**

**NetworkPolicy allowing app to db communication:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-db
  namespace: prod
spec:
  podSelector:
    matchLabels:
      tier: db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: java-app
      ports:
        - protocol: TCP
          port: 5432

---
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: prod
spec:
  selector:
    tier: db
  ports:
    - port: 5432
      targetPort: 5432
      protocol: TCP
  type: ClusterIP
```

**ConfigMap with correct DB connection:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: java-app-config
  namespace: prod
data:
  application.properties: |
    spring.datasource.url=jdbc:postgresql://database.prod.svc.cluster.local:5432/appdb
    spring.datasource.username=appuser
    spring.jpa.hibernate.ddl-auto=validate
```

---

#### Issue 8: Ingress Not Routing Traffic Correctly

**Symptom:**
```
503 Service Unavailable
Ingress shows correct endpoints but traffic not reaching pods
```

**Root Cause:**
1. Selector mismatch
2. Service port doesn't match container port
3. Pod not ready
4. Ingress backend not pointing to correct service

**Solution:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: prod
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls-cert
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /java
            pathType: Prefix
            backend:
              service:
                name: java-app
                port:
                  number: 80  # Must match Service port
          - path: /python
            pathType: Prefix
            backend:
              service:
                name: python-app
                port:
                  number: 80
          - path: /dotnet
            pathType: Prefix
            backend:
              service:
                name: dotnet-app
                port:
                  number: 80

---
# Service must match pod labels
apiVersion: v1
kind: Service
metadata:
  name: java-app
  namespace: prod
spec:
  type: ClusterIP
  selector:
    app: java-app  # MUST match pod labels
  ports:
    - name: http
      port: 80         # Service port
      targetPort: 8080 # Container port
      protocol: TCP
```

**Verification:**

```bash
# Check Ingress status
kubectl get ingress -n prod
kubectl describe ingress app-ingress -n prod

# Check endpoints
kubectl get endpoints java-app -n prod

# Test from pod
kubectl exec -it $(kubectl get pods -l app=java-app -n prod -o name | head -1) -n prod -- wget http://java-app

# Check pod readiness
kubectl get pods -n prod -o wide
kubectl describe pod <pod-name> -n prod
```

---

### 5.4 Authorization Issues

#### Issue 9: Pod Cannot Access Kubernetes API

**Symptom:**
```
Unable to connect to the server: Unauthorized
error: Unauthorized
```

**Root Cause:**
1. ServiceAccount not created
2. RBAC role/rolebinding missing
3. Token mismatch

**Solution:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: java-app
  namespace: prod

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: java-app-role
  namespace: prod
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: java-app-binding
  namespace: prod
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: java-app-role
subjects:
  - kind: ServiceAccount
    name: java-app
    namespace: prod

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
  namespace: prod
spec:
  template:
    spec:
      serviceAccountName: java-app  # Use ServiceAccount
      containers:
        - name: java-app
          image: myregistry.azurecr.io/java-app:latest
```

---

#### Issue 10: Cross-Namespace Access Denied

**Symptom:**
```
Error from server (Forbidden): services is forbidden:
User "system:serviceaccount:monitoring:prometheus" cannot list services in the namespace "prod"
```

**Root Cause:**
ServiceAccount in one namespace can't access another namespace.

**Solution - ClusterRole with ClusterRoleBinding:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-read-all
rules:
  - apiGroups: [""]
    resources: ["services", "endpoints", "pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "daemonsets", "statefulsets"]
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-read-all
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus-read-all
subjects:
  - kind: ServiceAccount
    name: prometheus
    namespace: monitoring
```

**Verification:**

```bash
# Check what a ServiceAccount can do
kubectl auth can-i get pods --as=system:serviceaccount:prod:java-app -n prod

# Verify ServiceAccount token
kubectl get secret -n prod | grep java-app-token
```

---

### 5.5 ArgoCD-Specific Issues

#### Issue 11: ArgoCD Server Certificate Expired

**Symptom:**
```
x509: certificate has expired
unable to verify the certificate
```

**Solution:**

```bash
# Check certificate expiry
kubectl get secret argocd-server-tls -n argocd -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -text | grep -A 2 "Not After"

# Regenerate self-signed cert
kubectl delete secret argocd-server-tls -n argocd
kubectl rollout restart deployment/argocd-server -n argocd

# Or use cert-manager
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: argocd-server-tls
  namespace: argocd
spec:
  secretName: argocd-server-tls
  duration: 8760h
  issuerRef:
    name: selfsigned
    kind: Issuer
EOF
```

---

#### Issue 12: ArgoCD Cannot Access Private Git Repository

**Symptom:**
```
repository not accessible
authentication required
```

**Solution:**

```yaml
# Create SSH key in ArgoCD namespace
apiVersion: v1
kind: Secret
metadata:
  name: github-repo-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/company/private-repo.git
  password: <personal-access-token>  # For HTTPS
  username: not-used

---
# Or use SSH
apiVersion: v1
kind: Secret
metadata:
  name: github-repo-ssh
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: git@github.com:company/private-repo.git
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
```

**Test connectivity:**

```bash
# From ArgoCD repo-server pod
kubectl exec -it <repo-server-pod> -n argocd -- sh
ssh -T git@github.com
```

---

## 6. Best Practices

### 6.1 ArgoCD Configuration

#### Use ApplicationSet for Multi-Environment

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: infra-apps
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - name: dev
            namespace: dev
            server: https://dev-cluster:6443
          - name: staging
            namespace: staging
            server: https://staging-cluster:6443
          - name: prod
            namespace: prod
            server: https://prod-cluster:6443
  template:
    metadata:
      name: infra-deployment-{{name}}
    spec:
      project: {{name}}-project
      source:
        repoURL: https://github.com/company/infra-repo.git
        targetRevision: main
        path: apps/{{name}}
      destination:
        server: '{{server}}'
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
          allow_empty: false
```

---

### 6.2 Secrets Management

#### Use External Secrets Operator (ESO)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: azure-keyvault
  namespace: prod
spec:
  provider:
    azurekv:
      authType: ServiceAccount
      vaultUrl: https://mykeyvault.vault.azure.net
      tenantId: <tenant-id>

---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: prod
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: azure-keyvault
    kind: SecretStore
  target:
    name: db-secret
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: db-password
    - secretKey: username
      remoteRef:
        key: db-username
```

---

### 6.3 Monitoring & Alerts

#### ArgoCD Metrics + Prometheus

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-metrics
  namespace: argocd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-metrics
  endpoints:
    - port: metrics
      interval: 30s

---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: argocd-alerts
  namespace: argocd
spec:
  groups:
    - name: argocd.rules
      interval: 30s
      rules:
        - alert: ArgocdAppSyncFailed
          expr: argocd_app_sync_total{phase="Failed"} > 0
          for: 15m
          annotations:
            summary: "ArgoCD app sync failed"
```

---

### 6.4 GitOps Workflow

**Recommended Flow:**
1. Code change → GitHub PR
2. CI (GitHub Actions) → Build & Test
3. PR approval → Merge to main
4. ArgoCD detects change → Syncs to K8s
5. Application deployed automatically

**Never:**
- ❌ Use `kubectl apply` directly
- ❌ Manual edits in cluster
- ❌ Store secrets in Git (use External Secrets)
- ❌ Skip testing before deploy

---

## Quick Reference - Troubleshooting Checklist

| Issue | Diagnosis | Fix |
|-------|-----------|-----|
| **Pod won't start** | `kubectl logs`, `describe pod` | Fix config, secrets, resources |
| **Image pull fails** | Check registry credentials | Add imagePullSecrets |
| **No connectivity** | `kubectl exec ... nc -zv service` | Fix NetworkPolicy, DNS |
| **Auth denied** | Check RBAC, ServiceAccount | Add RoleBinding |
| **Sync failed** | `argocd app logs` | Validate YAML, check permissions |
| **Service unavailable** | Check endpoints, port mappings | Verify selector, port match |

---

**Created for:** Interview preparation - ArgoCD Infrastructure & Application Deployment  
**Last Updated:** April 1, 2026  
**Repository:** https://github.com/tarunikakara/Interview-Terraform.git
