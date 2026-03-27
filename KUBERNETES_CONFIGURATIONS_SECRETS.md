# Kubernetes Configurations & Secrets

## Table of Contents
1. [Overview](#overview)
2. [ConfigMaps](#configmaps)
3. [Secrets](#secrets)
4. [Environment Variables & Mounted Volumes](#environment-variables--mounted-volumes)
5. [Secure Handling of Secrets](#secure-handling-of-secrets)
6. [Best Practices](#best-practices)
7. [Examples & Patterns](#examples--patterns)

---

## Overview

**Configurations & Secrets** management in Kubernetes separates configuration data and sensitive information from application code. This enables applications to be portable and secure across different environments.

### Problem Solved

```
Without ConfigMaps/Secrets:
┌──────────────────────────────────┐
│   Hard-coded in application      │
│   ├─ Database URLs               │
│   ├─ API Keys                    │
│   ├─ Passwords                   │
│   └─ Environment-specific config │
└──────────────────────────────────┘
❌ Not portable
❌ Security risk
❌ Hard to change without rebuild

With ConfigMaps/Secrets:
┌──────────────────────────────────┐
│   External configuration         │
│   ├─ Dev config                  │
│   ├─ Prod config                 │
│   └─ Secrets encrypted           │
└──────────────────────────────────┘
✅ Portable
✅ Secure
✅ Easy to update
```

### Key Differences

| Aspect | ConfigMaps | Secrets |
|--------|-----------|---------|
| **Purpose** | Non-sensitive configuration | Sensitive data |
| **Size Limit** | 1MB | 1MB |
| **Encoding** | Plain text (readable) | Base64 encoded |
| **Security** | No encryption (stored as-is) | Can be encrypted at rest |
| **Access Control** | RBAC | RBAC + etcd encryption |
| **Use Cases** | Settings, URLs, flags | Passwords, tokens, keys |

---

## ConfigMaps

**ConfigMaps** store non-sensitive configuration data as key-value pairs. They decouple configuration from application code.

### ConfigMap Characteristics
- **Immutable** (by default, can be set mutable)
- **Size limited** to 1MB per ConfigMap
- **Namespace-scoped** (accessible within namespace)
- **Readable** (plain text, not encrypted)
- **Referenced** by name in Pods

### Creating ConfigMaps

#### Method 1: Imperative (Literal Key-Value)

```bash
kubectl create configmap app-config \
  --from-literal=app.name=myapp \
  --from-literal=app.version=1.0 \
  --from-literal=debug=true
```

**Resulting ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.name: myapp
  app.version: "1.0"
  debug: "true"
```

#### Method 2: From File

```bash
# Create from single file
kubectl create configmap app-config --from-file=app.properties

# Create from directory (all files become keys)
kubectl create configmap configs --from-file=/etc/config/

# Custom key name
kubectl create configmap app-config --from-file=myconfig=app.properties
```

#### Method 3: Declarative (YAML)

**Simple Key-Value:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: simple-config
data:
  app.name: myapp
  app.version: "2.0"
  debug: "false"
```

**Your ConfigMap Example:**

From your `configmap.yml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: taruni-configmap
data:
  index.html: |
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Sample HTML</title>
    </head>
    <body>
        <h1>Welcome to Kubernetes Config MAP with Taruni</h1>
    </body>
    </html>
```

**Multi-line Content:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  application.yaml: |
    server:
      port: 8080
      ssl: true
    database:
      host: db.example.com
      port: 5432
  settings.json: |
    {
      "theme": "dark",
      "language": "en",
      "features": {
        "darkMode": true,
        "cache": true
      }
    }
  nginx.conf: |
    server {
      listen 80;
      server_name localhost;
      root /usr/share/nginx/html;
      index index.html;
    }
```

### Using ConfigMaps in Pods

#### 1. Environment Variables

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_NAME: myapp
  APP_ENV: production
  APP_VERSION: "1.0"

---
apiVersion: v1
kind: Pod
metadata:
  name: app-with-env
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    # Single key from ConfigMap
    - name: APP_NAME
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_NAME
    
    # Single key with different env var name
    - name: ENVIRONMENT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV
    
    # Direct literal value
    - name: PORT
      value: "8080"
    
    # Pod metadata
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
```

#### 2. All ConfigMap as Environment Variables

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_URL: postgres://db:5432/mydb
  API_KEY: secretkey123
  LOG_LEVEL: info

---
apiVersion: v1
kind: Pod
metadata:
  name: app-with-all-env
spec:
  containers:
  - name: app
    image: myapp:latest
    envFrom:
    - configMapRef:
        name: app-config
    # All keys become environment variables:
    # DATABASE_URL=postgres://db:5432/mydb
    # API_KEY=secretkey123
    # LOG_LEVEL=info
```

#### 3. ConfigMap as Volume Mount

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: taruni-configmap
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <body>
    <h1>Welcome</h1>
    </body>
    </html>

---
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    volumeMounts:
    - name: web-content
      mountPath: /usr/share/nginx/html/
  volumes:
  - name: web-content
    configMap:
      name: taruni-configmap
      items:
      - key: index.html
        path: index.html
```

**Mounted File Structure:**
```
/usr/share/nginx/html/
└── index.html (content from ConfigMap)
```

#### 4. ConfigMap with Custom Permissions

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-with-perms
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
      defaultMode: 0644        # Read for all, write for owner
      items:
      - key: app.conf
        path: app.conf
        mode: 0600             # Only owner can read/write
```

### ConfigMap Immutability

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: immutable-config
data:
  key1: value1
immutable: true              # Cannot be modified after creation
```

**Behavior:**
```bash
# Try to edit
kubectl edit configmap immutable-config
# Error: ConfigMap is immutable

# Try to patch
kubectl patch configmap immutable-config --type merge -p '...'
# Error: ConfigMap is immutable

# Must delete and recreate
kubectl delete configmap immutable-config
kubectl create configmap immutable-config ...
```

### ConfigMap Size and Limitations

```yaml
# ConfigMaps limited to 1MB
# For larger files, use:
# - Persistent Volumes
# - Init containers to fetch from outside
# - Side-car containers with dynamic loading
```

---

## Secrets

**Secrets** store sensitive data like passwords, tokens, keys, and credentials. They are base64-encoded but can be encrypted at rest.

### Secret Types

#### 1. **Opaque** - Generic Secret (Default)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=          # admin (base64)
  password: cGFzc3dvcmQEVmMw # password123 (base64)
```

**Create from literal:**
```bash
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=password123
```

**Create from file:**
```bash
kubectl create secret generic db-credentials \
  --from-file=./username.txt \
  --from-file=./password.txt
```

#### 2. **Opaque** - stringData (Readable)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:                    # Auto base64-encoded
  username: admin
  password: mysecretpassword
  api-key: sk_live_1234567890
```

#### 3. **docker-registry** - Docker Config

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: docker-secret
type: docker-registry
data:
  .dockerconfigjson: eyJhdXRocyI6eyJteXJlZ2lzdHJ5LmNvbSI6eyJ...}}
```

**Create from Docker credentials:**
```bash
kubectl create secret docker-registry docker-secret \
  --docker-server=myregistry.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=user@example.com
```

**Use for pulling private images:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-image-pod
spec:
  imagePullSecrets:
  - name: docker-secret
  containers:
  - name: app
    image: myregistry.com/myapp:latest
```

#### 4. **kubernetes.io/basic-auth** - Basic Authentication

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth
type: kubernetes.io/basic-auth
stringData:
  username: admin
  password: supersecretpassword
```

#### 5. **kubernetes.io/ssh-auth** - SSH Authentication

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ssh-secret
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: LS0tLS1CRUdJTiBSU0EgUHJJVkFURSBLRVks... # SSH private key (base64)
```

#### 6. **kubernetes.io/tls** - TLS/SSL Certificate

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0t...  # Base64-encoded certificate
  tls.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0t...  # Base64-encoded private key
```

**Create from certificate files:**
```bash
kubectl create secret tls tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key
```

**Use in Ingress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - example.com
    secretName: tls-secret
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: web
            port:
              number: 80
```

#### 7. **kubernetes.io/service-account-token** - Service Account Token

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: serviceaccount-token
  annotations:
    kubernetes.io/service-account.name: myapp-sa
type: kubernetes.io/service-account-token
```

#### 8. **bootstrap.kubernetes.io/token** - Bootstrap Token

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-abcd1234
  namespace: kube-system
type: bootstrap.kubernetes.io/token
data:
  token-id: YWJjZA==         # abcd
  token-secret: ZWZnaDEyMzQ=  # efgh1234
  usage-bootstrap-authentication: dHJ1ZQ==
  usage-bootstrap-signing: dHJ1ZQ==
```

### Encoding & Decoding Secrets

```bash
# Encode value
echo -n "admin" | base64
# Output: YWRtaW4=

# Decode value
echo "YWRtaW4=" | base64 -d
# Output: admin

# Check secret content
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d

# Get all secret data
kubectl get secret db-credentials -o yaml
```

---

## Environment Variables & Mounted Volumes

### Method 1: Direct Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-var-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    # Direct value
    - name: APP_NAME
      value: "myapp"
    
    # From ConfigMap key
    - name: CONFIG_SETTING
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: setting1
    
    # From Secret key
    - name: DATABASE_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    
    # From Pod metadata
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    
    # From resource limits
    - name: MEMORY_LIMIT
      valueFrom:
        resourceFieldRef:
          containerName: app
          resource: limits.memory
```

### Method 2: All ConfigMap/Secret as Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envfrom-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    envFrom:
    # All ConfigMap keys become env vars
    - configMapRef:
        name: app-config
    
    # All Secret keys become env vars
    - secretRef:
        name: db-secret
    
    # With prefix
    - configMapRef:
        name: cache-config
        name: CACHE_
    
    # Still can add specific env vars
    env:
    - name: CUSTOM_VAR
      value: "custom-value"
```

### Method 3: Mounted Volumes (ConfigMap)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-volume-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:
      - key: app.properties
        path: app.properties
      - key: app.yaml
        path: app.yaml
```

**Mounted structure:**
```
/etc/config/
├── app.properties
└── app.yaml
```

### Method 4: Mounted Volumes (Secrets)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
      defaultMode: 0400       # Read-only for owner
      items:
      - key: username
        path: username
      - key: password
        path: password
```

**Mounted structure:**
```
/etc/secrets/
├── username
├── password
(permissions: -r-------- for root)
```

### Method 5: TLS Secrets in Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tls-pod
spec:
  containers:
  - name: app
    image: web-app:latest
    volumeMounts:
    - name: tls-certs
      mountPath: /etc/tls/certs
      readOnly: true
  volumes:
  - name: tls-certs
    secret:
      secretName: tls-secret
      items:
      - key: tls.crt
        path: cert.pem
      - key: tls.key
        path: key.pem
```

**Mounted files:**
```
/etc/tls/certs/
├── cert.pem      (from tls.crt)
├── key.pem       (from tls.key)
```

### Combining ConfigMap and Secrets

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: combined-config
spec:
  containers:
  - name: app
    image: myapp:latest
    
    # Environment variables from both
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: db-secret
    
    # Mount both as volumes
    volumeMounts:
    - name: config-files
      mountPath: /etc/config
    - name: secret-files
      mountPath: /etc/secrets
      readOnly: true
  
  volumes:
  - name: config-files
    configMap:
      name: app-config
  - name: secret-files
    secret:
      secretName: db-credentials
```

---

## Secure Handling of Secrets

### 1. Encryption at Rest

By default, Secrets are stored unencrypted in etcd. Enable encryption:

```yaml
# kube-apiserver encryption configuration
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}
```

### 2. RBAC for Secrets Access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
  resourceNames: ["specific-secret"]  # Only specific secret

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-secrets
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: secret-reader
subjects:
- kind: ServiceAccount
  name: myapp-sa
  namespace: default
```

### 3. Restrict Secret Visibility

```bash
# By default, secrets visible to authenticated users
# Prevent via RBAC:

# Deny all secret access
kubectl create role no-secrets --verb=get,list --resource=secrets --resource-name=""

# Allow only specific service account
kubectl create rolebinding secret-access \
  --clusterrole=secret-reader \
  --serviceaccount=default:myapp-sa
```

### 4. Use Secret External Systems

Instead of storing in Kubernetes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: external-secret-pod
spec:
  serviceAccountName: app-sa
  containers:
  - name: app
    image: myapp:latest
    # Fetch secret from external system (Vault, AWS Secrets Manager, etc)
    env:
    - name: VAULT_ADDR
      value: "https://vault.example.com"
    - name: VAULT_TOKEN
      valueFrom:
        secretKeyRef:
          name: vault-token
          key: token
    command:
    - sh
    - -c
    - |
      # Init container fetches secrets from Vault
      curl -H "X-Vault-Token: $VAULT_TOKEN" \
        $VAULT_ADDR/v1/secret/data/myapp > /etc/secrets/config
      exec myapp
```

### 5. Audit Logging

```yaml
# Enable audit logs for secret access
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
--audit-log-path=/var/log/kubernetes/audit.log

# Audit policy targeting secrets
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  resources: ["secrets"]
  omitStages:
  - RequestReceived
```

### 6. Secret Rotation

```yaml
# ConfigMap with rotation annotation
apiVersion: v1
kind: Secret
metadata:
  name: rotated-secret
  annotations:
    rotation.example.com/interval: "30d"
    rotation.example.com/last-rotated: "2026-01-15"
stringData:
  api-key: new-api-key-value
```

### 7. Minimize Secret Scope

```yaml
# Only mount needed secrets
apiVersion: v1
kind: Pod
metadata:
  name: minimal-secrets
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    # Mount only needed secrets
    - name: db-pass
      mountPath: /etc/secrets/db
      readOnly: true
  volumes:
  - name: db-pass
    secret:
      secretName: db-credentials
      items:
      - key: password               # Only password, not username
        path: password
```

### 8. Secret Cleanup on Pod Termination

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  terminationGracePeriodSeconds: 30
  containers:
  - name: app
    image: myapp:latest
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "rm -rf /etc/secrets/*"]
  volumes:
  - name: secrets
    secret:
      secretName: app-secret
      defaultMode: 0400
```

---

## Best Practices

### ConfigMaps
- ✅ Use for non-sensitive configuration only
- ✅ Mount as volumes for large config files
- ✅ Use envFrom for simple key-value pairs
- ✅ Set immutable: true for production configs
- ✅ Name ConfigMaps descriptively
- ❌ Don't store secrets in ConfigMaps
- ❌ Don't modify ConfigMaps in running pods

### Secrets
- ✅ Always encrypt at rest
- ✅ Use dedicated secret management systems (Vault, AWS Secrets Manager)
- ✅ Rotate secrets regularly
- ✅ Use stringData for readability
- ✅ Mount as volumes with restrictive permissions
- ✅ Use RBAC to limit access
- ✅ Enable audit logging for secret access
- ❌ Don't commit secrets to Git
- ❌ Don't log secrets
- ❌ Don't pass secrets as command arguments

### Access Control
- ✅ Use ServiceAccounts for applications
- ✅ Apply least privilege RBAC
- ✅ Separate ConfigMaps/Secrets by environment
- ✅ Use namespace isolation
- ✅ Regular access reviews
- ❌ Don't use default service account

### Secret Visibility
- ✅ Use external secret management
- ✅ Implement secret rotation
- ✅ Limit secret access to specific pods
- ✅ Monitor secret access
- ✅ Clean up old/unused secrets
- ❌ Don't expose secrets in logs
- ❌ Don't use weak encryption

---

## Examples & Patterns

### Pattern 1: Database Credentials

**ConfigMap with non-sensitive config:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  host: postgres.default.svc.cluster.local
  port: "5432"
  database: myapp_db
  max-connections: "100"
```

**Secret with sensitive credentials:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: app_user
  password: SecurePassword123!
```

**Using in Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        env:
        # Non-sensitive from ConfigMap
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: host
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: port
        - name: DB_NAME
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: database
        
        # Sensitive from Secret
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
```

### Pattern 2: Multi-Environment Config

```yaml
---
# Development ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-dev
data:
  debug: "true"
  log-level: "DEBUG"
  api-timeout: "5"

---
# Production ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-prod
data:
  debug: "false"
  log-level: "ERROR"
  api-timeout: "30"

---
# Dev Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-dev
spec:
  template:
    spec:
      containers:
      - name: app
        envFrom:
        - configMapRef:
            name: app-config-dev

---
# Prod Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-prod
spec:
  template:
    spec:
      containers:
      - name: app
        envFrom:
        - configMapRef:
            name: app-config-prod
```

### Pattern 3: Configuration with Init Container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-init-pod
spec:
  initContainers:
  - name: config-init
    image: busybox:1.28
    command:
    - sh
    - -c
    - |
      cp /config/*.conf /config-volume/
      chmod 644 /config-volume/*
    volumeMounts:
    - name: config-source
      mountPath: /config
    - name: config-volume
      mountPath: /config-volume
  
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: config-volume
      mountPath: /etc/app/config
  
  volumes:
  - name: config-source
    configMap:
      name: app-config
  - name: config-volume
    emptyDir: {}
```

### Pattern 4: TLS Configuration for Ingress

```yaml
---
# Generate TLS secret
apiVersion: v1
kind: Secret
metadata:
  name: api-tls
type: kubernetes.io/tls
stringData:
  tls.crt: |
    -----BEGIN CERTIFICATE-----
    MIIBkTCB+wIJAKHHCgVZEOqGMA0GCSqGSIb3DQEBBQUAMBMxETAPBgNVBAMMCHdl
    ...
    -----END CERTIFICATE-----
  tls.key: |
    -----BEGIN PRIVATE KEY-----
    MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQDW...
    ...
    -----END PRIVATE KEY-----

---
# Ingress using TLS secret
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 8443
```

### Pattern 5: Private Docker Registry Access

```yaml
---
# Create pull secret for private registry
apiVersion: v1
kind: Secret
metadata:
  name: private-registry
type: docker-registry
stringData:
  .dockerconfigjson: |
    {
      "auths": {
        "myregistry.com": {
          "username": "myuser",
          "password": "mypassword",
          "auth": "bXl1c2VyOm15cGFzc3dvcmQ="
        }
      }
    }

---
# Deployment using private registry
apiVersion: apps/v1
kind: Deployment
metadata:
  name: private-app
spec:
  template:
    spec:
      imagePullSecrets:
      - name: private-registry
      containers:
      - name: app
        image: myregistry.com/private-app:latest
```

---

## Commands Reference

```bash
# ConfigMaps
kubectl create configmap cm-name --from-literal=key=value
kubectl create configmap cm-name --from-file=./config.yaml
kubectl get configmaps
kubectl describe configmap cm-name
kubectl edit configmap cm-name
kubectl delete configmap cm-name

# Secrets
kubectl create secret generic secret-name --from-literal=key=value
kubectl create secret generic secret-name --from-file=./secret.txt
kubectl create secret docker-registry docker-secret --docker-server=...
kubectl create secret tls tls-secret --cert=cert.pem --key=key.pem
kubectl get secrets
kubectl describe secret secret-name
kubectl get secret secret-name -o yaml           # View encoded data
kubectl get secret secret-name -o jsonpath='{.data.key}' | base64 -d

# Decode secrets
echo 'base64-encoded-value' | base64 -d

# Check secret usage
kubectl get pods -o json | grep -A5 valueFrom
```

---

## Troubleshooting

```bash
# Secret not visible
# Check RBAC permissions
kubectl auth can-i get secrets

# ConfigMap not mounted
# Check volume mount points
kubectl describe pod pod-name
kubectl exec pod-name -- ls -la /etc/config

# Check environment variables
kubectl exec pod-name -- env | grep DB_

# View ConfigMap/Secret content
kubectl get cm cm-name -o yaml
kubectl get secret secret-name -o yaml

# Check secret encoding
kubectl get secret secret-name -o jsonpath='{.data.password}' | base64 -d

# Pod can't start due to missing Secret
kubectl describe pod pod-name       # Check events
kubectl get secrets                  # Verify secret exists
```

---

## Summary

- **ConfigMaps**: Non-sensitive configuration, plain text, 1MB limit
- **Secrets**: Sensitive data, base64-encoded, can be encrypted at rest
- **Environment Variables**: Simple key-value access via env vars
- **Mounted Volumes**: File-based access via filesystem mounts
- **Secure Handling**: Encryption, RBAC, external systems, rotation, auditing

Properly managing configuration and secrets is critical for security, portability, and maintainability of Kubernetes applications!
