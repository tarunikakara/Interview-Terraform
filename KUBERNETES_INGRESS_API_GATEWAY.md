# Kubernetes Ingress & API Gateway

## Table of Contents
1. [Overview](#overview)
2. [Ingress Fundamentals](#ingress-fundamentals)
3. [Ingress Controller vs Ingress Resource](#ingress-controller-vs-ingress-resource)
4. [Path-based Routing](#path-based-routing)
5. [Host-based Routing](#host-based-routing)
6. [TLS Termination](#tls-termination)
7. [Popular Ingress Controllers](#popular-ingress-controllers)
8. [API Gateway Patterns](#api-gateway-patterns)
9. [Best Practices](#best-practices)

---

## Overview

**Ingress** exposes HTTP(S) routes from outside the cluster to services inside the cluster. It provides load balancing, SSL/TLS termination, and name-based virtual hosting.

### Traffic Flow

```
┌─────────────────────────────────────────┐
│       External Users (Internet)         │
└──────────────────┬──────────────────────┘
                   │ (HTTP/HTTPS)
                   ▼
        ┌──────────────────────┐
        │ Ingress Controller   │ ← Watches Ingress resources
        │ (NGINX, Traefik)     │   Configures routing
        └──────────┬───────────┘
                   │
        ┌──────────┴─────────────┐
        │                        │
        ▼                        ▼
   Service A              Service B
(backend-api)          (frontend-web)
        │                        │
    ┌───┴──┐             ┌───────┴───┐
    │      │             │           │
Pod1  Pod2 Pod3       Pod1        Pod2
```

### Ingress vs LoadBalancer vs NodePort

| Type | Scope | Use Case |
|------|-------|----------|
| **NodePort** | Pod-to-Node | Direct Pod access, dev/test |
| **LoadBalancer** | HTTP/TCP | Single service external access |
| **Ingress** | HTTP/HTTPS | Multiple services, domain-based routing |

---

## Ingress Fundamentals

### Ingress Resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: basic-ingress
  namespace: default
spec:
  rules:
  # Host-based rule
  - host: example.com
    http:
      paths:
      # Path-based rule
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      
      - path: /web
        pathType: Exact
        backend:
          service:
            name: web-service
            port:
              number: 3000
```

### Path Types

| Path Type | Matching | Example |
|-----------|----------|---------|
| **Exact** | Exact string match | `/api` matches `/api` only |
| **Prefix** | Prefix match | `/api` matches `/api`, `/api/v1`, `/api/users` |
| **ImplementationSpecific** | Controller-dependent | Depends on ingress controller |

### Ingress Lifecycle

```
1. User creates Ingress resource (YAML)
        │
        ▼
2. Ingress Controller watches for changes
        │
        ▼
3. Controller reads Ingress spec
        │
        ▼
4. Controller generates controller-specific config
        ├─ NGINX: nginx.conf
        ├─ Traefik: Dynamic config
        ├─ Istio: VirtualService + DestinationRule
        │
        ▼
5. Controller deploys/updates configuration
        │
        ▼
6. External traffic routes through controller
        │
        ▼
7. Traffic reaches backend service → Pod
```

### Ingress Class (v1.19+)

```yaml
# Define ingress class
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
spec:
  controller: nginx.org/ingress-controller

---
# Use specific ingress class
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  ingressClassName: nginx  # Specify which controller to use
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

---

## Ingress Controller vs Ingress Resource

### Ingress Resource

**What**: Kubernetes API object defining routing rules

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 80
```

**Characteristics**:
- Declarative configuration
- Stored in etcd
- API server validates syntax
- Platform-agnostic specification
- Does NOT implement routing by itself

### Ingress Controller

**What**: Actual software component implementing the routing

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-ingress
  template:
    metadata:
      labels:
        app: nginx-ingress
    spec:
      containers:
      - name: nginx-ingress-controller
        image: nginx/nginx-ingress:latest
        args:
        - -ingress-class=nginx
        - -watch-namespace=
        ports:
        - containerPort: 80
          name: http
        - containerPort: 443
          name: https
```

**Characteristics**:
- Imperative implementation
- Runs as Pod/Deployment
- Watches Ingress resources
- Generates controller-specific config:
  - NGINX: nginx.conf
  - Traefik: Dynamic routing rules
  - Istio: VirtualService
- Actually accepts and routes traffic

### Relationship Diagram

```
┌─────────────────────────────────────────┐
│     Ingress Resource (Specification)    │
│  - Routes                               │
│  - TLS config                           │
│  - Service backends                     │
└──────────────────┬──────────────────────┘
                   │ (read by)
                   ▼
┌─────────────────────────────────────────┐
│   Ingress Controller (Implementation)   │
│  - Watches Ingress resources            │
│  - Generates config                     │
│  - Listens on port 80/443               │
│  - Routes traffic to services           │
└─────────────────────────────────────────┘
```

### Common Ingress Controllers

| Controller | Use Case | Config Language |
|----------|----------|-----------------|
| **NGINX Ingress** | General purpose, widely used | nginx.conf |
| **Traefik** | Cloud-native, dynamic | TOML/YAML |
| **Istio Gateway** | Service mesh, advanced routing | VirtualService |
| **HAProxy** | High-performance | HAProxy config |
| **Kong** | API gateway, plugins | Plugin system |

---

## Path-based Routing

**Path-based routing** directs requests to different backends based on URL path.

### Basic Path Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-routing
spec:
  rules:
  - host: example.com
    http:
      paths:
      # /api/* → api-gateway service
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-gateway
            port:
              number: 8080
      
      # /admin/* → admin-panel service
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-panel
            port:
              number: 3000
      
      # / → web-frontend service
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-frontend
            port:
              number: 80
```

### Path Rewriting (NGINX Annotation)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-rewrite-ingress
  annotations:
    # Rewrite path before routing
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: example.com
    http:
      paths:
      # Request /api/v1/users → /users (to backend)
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      
      # Request /static/(.*) → /(.*) (to cdn service)
      - path: /static(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: cdn-service
            port:
              number: 80
```

### Regular Expression Paths

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: regex-path-ingress
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      # Match /v1/users, /v2/products, etc.
      - path: /v[0-9]+
        pathType: Prefix
        backend:
          service:
            name: versioned-api
            port:
              number: 8080
      
      # Match /users/123, /products/456, etc.
      - path: /([a-z]+)/([0-9]+)
        pathType: Prefix
        backend:
          service:
            name: resource-service
            port:
              number: 8080
```

### Multiple Services with Path Routing

```yaml
---
# Deployment 1: API Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
      version: v1
  template:
    metadata:
      labels:
        app: api
        version: v1
    spec:
      containers:
      - name: api
        image: api:v1.0
        ports:
        - containerPort: 8080

---
# Deployment 2: Web Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: web-frontend:latest
        ports:
        - containerPort: 3000

---
# Service 1: API
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - port: 8080
    targetPort: 8080

---
# Service 2: Web
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 3000
    targetPort: 3000

---
# Ingress with path routing
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-service-ingress
spec:
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 3000
```

---

## Host-based Routing

**Host-based routing** directs requests to different backends based on domain/hostname.

### Basic Host Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-routing
spec:
  rules:
  # api.example.com → api-service
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
  
  # web.example.com → web-service
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  
  # admin.example.com → admin-service
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 3000
```

### VirtualHost Pattern

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: virtualhost-ingress
spec:
  tls:
  - hosts:
    - api.example.com
    - web.example.com
    - admin.example.com
    secretName: wildcard-tls
  
  rules:
  # Production
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-prod
            port:
              number: 8080
  
  # Staging
  - host: api-staging.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-staging
            port:
              number: 8080
  
  # Development
  - host: api-dev.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-dev
            port:
              number: 8080
```

### Wildcard Hosts

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wildcard-host-ingress
spec:
  rules:
  # *.example.com → wildcard-service
  - host: "*.example.com"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wildcard-service
            port:
              number: 80
  
  # Default rule (no host match)
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: default-service
            port:
              number: 80
```

### Combined Host and Path Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: combined-routing
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      # api.example.com/v1/* → api-v1-service
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1-service
            port:
              number: 8080
      
      # api.example.com/v2/* → api-v2-service
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-service
            port:
              number: 8080
  
  - host: web.example.com
    http:
      paths:
      # web.example.com/blog/* → blog-service
      - path: /blog
        pathType: Prefix
        backend:
          service:
            name: blog-service
            port:
              number: 3000
      
      # web.example.com/store/* → ecommerce-service
      - path: /store
        pathType: Prefix
        backend:
          service:
            name: ecommerce-service
            port:
              number: 3000
```

---

## TLS Termination

**TLS Termination** means the ingress controller handles encryption/decryption, backends communicate via plain HTTP.

### TLS Concepts

```
┌─────────────────────────────────────┐
│   Client (Browser)                  │
└────────────────┬────────────────────┘
                 │ HTTPS (Encrypted)
                 ▼
      ┌──────────────────────┐
      │ Ingress Controller   │ ← TLS Termination here
      │ (Port 443)           │   Decrypts HTTPS
      └──────────┬───────────┘
                 │ HTTP (Plaintext)
    ┌────────────┴────────────┐
    ▼                         ▼
Service A                 Service B
(port 80)                (port 80)
```

### TLS Certificate Setup

#### Method 1: Self-signed Certificate

```bash
# Generate private key
openssl genrsa -out tls.key 2048

# Generate self-signed certificate
openssl req -new -x509 -key tls.key -out tls.crt -days 365

# Create Kubernetes secret
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key
```

#### Method 2: Using cert-manager (Automated)

```yaml
---
# Install cert-manager first:
# helm repo add jetstack https://charts.jetstack.io
# helm install cert-manager jetstack/cert-manager

# ClusterIssuer for Let's Encrypt
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx

---
# Ingress with auto-generated TLS certificate
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-auto-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - example.com
    secretName: example-tls
  
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### TLS Configuration in Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  # TLS configuration
  tls:
  # Certificate 1: example.com
  - hosts:
    - example.com
    - www.example.com
    secretName: example-tls
  
  # Certificate 2: api.example.com
  - hosts:
    - api.example.com
    secretName: api-tls
  
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

### TLS Secret Format

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  # Base64 encoded certificate and key
  tls.crt: |
    -----BEGIN CERTIFICATE-----
    MIIDXTCCAkWgAwIBAgIJAJ...
    ...
    -----END CERTIFICATE-----
  tls.key: |
    -----BEGIN PRIVATE KEY-----
    MIIEvQIBADANBgkqhkiG9w0BAQE...
    ...
    -----END PRIVATE KEY-----
```

### HTTP to HTTPS Redirect

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: https-redirect-ingress
  annotations:
    # NGINX: Force HTTPS
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    # Traefik: Redirect HTTP to HTTPS
    traefik.ingress.kubernetes.io/redirect-entry-point: https
    traefik.ingress.kubernetes.io/redirect-permanent: "true"
spec:
  tls:
  - hosts:
    - example.com
    secretName: example-tls
  
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

---

## Popular Ingress Controllers

### 1. NGINX Ingress Controller

**Official NGINX Ingress Controller** maintained by NGINX/Kubernetes community.

#### Installation

```bash
# Using Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

#### NGINX Ingress Example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    # Request/response manipulation
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/add-base-url: "true"
    
    # Security
    nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.2 TLSv1.3"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-connections: "10"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    
    # Custom headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "Server: MyServer";
      more_set_headers "X-Frame-Options: SAMEORIGIN";
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

#### Features
- ✅ Wide community support
- ✅ Excellent documentation
- ✅ Rich annotations for customization
- ✅ High performance
- ✅ Production-tested

---

### 2. Traefik Ingress Controller

**Cloud-native proxy** with dynamic configuration and middlewares.

#### Installation

```bash
# Using Helm
helm repo add traefik https://traefik.github.io/charts
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace
```

#### Traefik Ingress Example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: traefik-ingress
  annotations:
    # Middleware: rate limiting
    traefik.ingress.kubernetes.io/middleware: "ratelimit@kubernetescrd"
spec:
  ingressClassName: traefik
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80

---
# Traefik Middleware (custom resource)
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: ratelimit
spec:
  rateLimit:
    average: 100
    burst: 200
```

#### Features
- ✅ Dynamic configuration (no restart needed)
- ✅ Native Kubernetes integration
- ✅ Middleware system for extensibility
- ✅ Built-in dashboard
- ✅ Service mesh integration (Traefik Mesh)

---

### 3. Istio Gateway Controller

**Service mesh with advanced traffic management.**

#### Installation

```bash
# Install Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.x.x
./bin/istioctl install --set profile=demo -y
```

#### Istio Gateway Example

```yaml
---
# Gateway: Entry point
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "example.com"
    - "*.example.com"
  
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "example.com"
    tls:
      mode: SIMPLE
      credentialName: example-tls
    hosts:
    - "*.example.com"

---
# VirtualService: Routing rules
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-virtualservice
spec:
  hosts:
  - "example.com"
  gateways:
  - my-gateway
  http:
  # Route 1: /api to api-service
  - match:
    - uri:
        prefix: /api
    route:
    - destination:
        host: api-service
        port:
          number: 8080
  
  # Route 2: /web to web-service
  - match:
    - uri:
        prefix: /web
    route:
    - destination:
        host: web-service
        port:
          number: 80
  
  # Route 3: default
  - route:
    - destination:
        host: default-service
        port:
          number: 80
```

#### Features
- ✅ Advanced traffic management (canary, A/B testing)
- ✅ Service mesh capabilities
- ✅ Distributed tracing, metrics
- ✅ Mutual TLS by default
- ✅ Traffic policies and load balancing
- ❌ Steeper learning curve

---

### Controller Comparison

| Feature | NGINX | Traefik | Istio |
|---------|-------|---------|-------|
| **Setup Complexity** | Low | Low | Medium-High |
| **Performance** | Excellent | Good | Good |
| **Community** | Large | Growing | Large |
| **Middleware/Plugins** | Annotations | Native | Advanced |
| **Service Mesh** | No | Optional | Yes |
| **Learning Curve** | Gradual | Moderate | Steep |
| **Best For** | General use | Cloud-native | Microservices |

---

## API Gateway Patterns

### API Gateway as Facade

```yaml
---
# External Gateway (Ingress)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-gateway
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-gateway-service
            port:
              number: 8080

---
# API Gateway Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
      - name: gateway
        image: api-gateway:latest
        ports:
        - containerPort: 8080
        env:
        - name: BACKEND_SERVICES
          value: "auth-service:8080,user-service:8080,product-service:8080"

---
# API Gateway Service
apiVersion: v1
kind: Service
metadata:
  name: api-gateway-service
spec:
  selector:
    app: api-gateway
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 8080
```

### API Gateway with Authentication

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-api-gateway
  annotations:
    # NGINX: Basic Auth
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required'
    
    # OAuth2 proxy
    nginx.ingress.kubernetes.io/auth-url: "http://oauth2-proxy/oauth2/auth"
    nginx.ingress.kubernetes.io/auth-signin: "http://oauth2-proxy/oauth2/start?rd=$scheme://$host$request_uri"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

### API Gateway with Rate Limiting

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ratelimit-api-gateway
  annotations:
    # NGINX: Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "20"
    nginx.ingress.kubernetes.io/limit-whitelist: "10.0.0.0/8"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

---

## Best Practices

### Design
- ✅ Separate ingress controllers by namespace for isolation
- ✅ Use IngressClass to support multiple controllers
- ✅ Implement path-based routing for microservices
- ✅ Use host-based routing for multi-tenant applications
- ✅ Enable TLS for all production traffic

### Security
- ✅ Always use HTTPS/TLS with valid certificates
- ✅ Implement authentication/authorization at gateway
- ✅ Use rate limiting to prevent abuse
- ✅ Enable CORS carefully and restrictively
- ✅ Validate all headers and inputs
- ❌ Don't expose internal services unnecessarily
- ❌ Don't pass secrets in URLs

### Performance
- ✅ Use appropriate backend connection timeouts
- ✅ Implement caching for static content
- ✅ Use gzip compression for responses
- ✅ Monitor ingress controller performance
- ✅ Scale ingress controllers horizontally

### Maintenance
- ✅ Use health checks on backends
- ✅ Implement graceful shutdowns
- ✅ Monitor certificate expiration
- ✅ Test failover scenarios
- ✅ Version your Ingress resources

### Example: Production-Ready Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.2 TLSv1.3"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://example.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE"
    nginx.ingress.kubernetes.io/cors-allow-headers: "Content-Type, Authorization"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    - web.example.com
    secretName: production-tls
  
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1-service
            port:
              number: 8080
      
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-service
            port:
              number: 8080
  
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

---

## Troubleshooting

```bash
# Check ingress status
kubectl get ingress
kubectl describe ingress my-ingress

# View ingress controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Test URL resolution
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl http://api-service:8080/

# Check service endpoints
kubectl get endpoints api-service
kubectl describe service api-service

# Verify TLS certificate
kubectl get secret example-tls -o yaml
openssl s_client -connect example.com:443

# Check ingress class
kubectl get ingressclass

# Verify backend connectivity
kubectl port-forward service/api-service 8080:8080
curl http://localhost:8080

# Monitor ingress controller
kubectl top pod -n ingress-nginx
kubectl get events -n ingress-nginx --sort-by='.lastTimestamp'
```

---

## Summary

- **Ingress Resource**: Declarative specification of routing rules
- **Ingress Controller**: Actual implementation that routes traffic
- **Path-based Routing**: Direct requests based on URL path
- **Host-based Routing**: Direct requests based on domain/hostname
- **TLS Termination**: Ingress handles encryption, backends use HTTP
- **Popular Controllers**: NGINX (general), Traefik (cloud-native), Istio (service mesh)
- **API Gateway Pattern**: Centralized entry point for multiple services

Kubernetes Ingress enables flexible, scalable external traffic management for your cluster!
