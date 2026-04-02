# GitHub Actions: Infrastructure & Application Deployment Pipeline

## Table of Contents
1. [GitHub Actions Fundamentals](#github-actions-fundamentals)
2. [Infrastructure Pipeline (IaC)](#infrastructure-pipeline-iac)
3. [Application Deployment Pipelines](#application-deployment-pipelines)
4. [Real-Time Troubleshooting Guide](#real-time-troubleshooting-guide)
5. [Best Practices](#best-practices)
6. [Quick Reference](#quick-reference)

---

## GitHub Actions Fundamentals

### What is GitHub Actions?
GitHub Actions is a **CI/CD platform** built directly into GitHub that automates:
- **Building** code
- **Testing** code
- **Deploying** infrastructure and applications
- **Publishing** artifacts

### Key Concepts
- **Workflow:** Automated process defined in YAML (.github/workflows/)
- **Trigger:** When workflow runs (push, pull_request, schedule)
- **Job:** Set of steps running in parallel or sequence
- **Step:** Individual task (checkout, build, test, deploy)
- **Action:** Reusable unit of code (checkout, setup-node, etc.)
- **Runner:** Machine that executes the workflow (ubuntu-latest, windows-latest)

### GitHub Actions vs ArgoCD
| Aspect | GitHub Actions | ArgoCD |
|--------|----------------|---------|
| **Purpose** | CI/CD pipeline automation | GitOps CD for Kubernetes |
| **Trigger** | Webhooks, schedule, manual | Git sync, manual |
| **Best For** | Build, test, infra provisioning | Continuous deployment to K8s |
| **Stateless** | Each run is independent | Continuous reconciliation |
| **Use Case** | Terraform apply, Docker build | App deployment to prod K8s |

---

## Infrastructure Pipeline (IaC)

### Directory Structure
```
.github/
├── workflows/
│   └── infrastructure-pipeline.yml
infra/
├── terraform/
│   ├── main.tf         # VNet, Subnets
│   ├── storage.tf      # Storage accounts, containers
│   ├── networking.tf   # Gateway, NSG, routes
│   ├── variables.tf
│   ├── outputs.tf
│   └── terraform.tfvars
├── environments/
│   ├── dev.tfvars
│   ├── staging.tfvars
│   └── prod.tfvars
└── scripts/
    └── validate.sh
```

### Infrastructure Components

#### 1. VNet & Subnet Configuration

**`infra/terraform/main.tf`:**
```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "tfstateprod123"
    container_name       = "tfstate"
    key                  = "infra-${var.environment}.tfstate"
  }
}

provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
}

# Resource Group
resource "azurerm_resource_group" "main" {
  name     = "rg-${var.environment}-${var.location}-001"
  location = var.location

  tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# Virtual Network
resource "azurerm_virtual_network" "main" {
  name                = "vnet-${var.environment}-001"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  address_space       = var.vnet_address_space

  tags = {
    Environment = var.environment
  }
}

# Subnets
resource "azurerm_subnet" "app" {
  name                 = "subnet-app-${var.environment}"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = var.app_subnet_prefix
  
  delegation {
    name = "app-service"
    service_delegation {
      name = "Microsoft.Web/serverFarms"
    }
  }
}

resource "azurerm_subnet" "database" {
  name                 = "subnet-db-${var.environment}"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = var.db_subnet_prefix
}

resource "azurerm_subnet" "gateway" {
  name                 = "GatewaySubnet"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = var.gateway_subnet_prefix
}
```

#### 2. Storage Containers

**`infra/terraform/storage.tf`:**
```hcl
# Storage Account
resource "azurerm_storage_account" "main" {
  name                     = "sa${var.environment}${var.location}001"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "GRS"
  https_traffic_only_enabled = true
  min_tls_version          = "TLS1_2"

  tags = {
    Environment = var.environment
  }
}

# Storage Container for App Data
resource "azurerm_storage_container" "app_data" {
  name                  = "app-data"
  storage_account_name  = azurerm_storage_account.main.name
  container_access_type = "private"
}

# Storage Container for Backups
resource "azurerm_storage_container" "backups" {
  name                  = "backups"
  storage_account_name  = azurerm_storage_account.main.name
  container_access_type = "private"
}

# Storage Container for Logs
resource "azurerm_storage_container" "logs" {
  name                  = "logs"
  storage_account_name  = azurerm_storage_account.main.name
  container_access_type = "private"
}

# Storage Account Firewall Rules
resource "azurerm_storage_account_network_rules" "main" {
  storage_account_id         = azurerm_storage_account.main.id
  default_action             = "Deny"
  bypass                     = ["AzureServices", "Logging"]
  virtual_network_subnet_ids = [azurerm_subnet.app.id]
  ip_rules                   = var.allowed_ips
}
```

#### 3. Gateway & Networking

**`infra/terraform/networking.tf`:**
```hcl
# Application Gateway
resource "azurerm_public_ip" "gateway" {
  name                = "pip-appgw-${var.environment}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  allocation_method   = "Static"
  sku                 = "Standard"
}

resource "azurerm_application_gateway" "main" {
  name                = "appgw-${var.environment}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  sku {
    name     = "Standard_v2"
    tier     = "Standard_v2"
    capacity = var.environment == "prod" ? 3 : 1
  }

  gateway_ip_configuration {
    name      = "gateway-ip-config"
    subnet_id = azurerm_subnet.gateway.id
  }

  frontend_port {
    name = "http"
    port = 80
  }

  frontend_port {
    name = "https"
    port = 443
  }

  frontend_ip_configuration {
    name                 = "frontend-ip"
    public_ip_address_id = azurerm_public_ip.gateway.id
  }

  backend_address_pool {
    name = "backend-pool"
  }

  backend_http_settings {
    name                  = "http-settings"
    cookie_based_affinity = "Disabled"
    port                  = 80
    protocol              = "Http"
    request_timeout       = 20
    match_type            = "StatusCode"
    match_values          = ["200", "201"]
  }

  http_listener {
    name                           = "http-listener"
    frontend_ip_configuration_name = "frontend-ip"
    frontend_port_name             = "http"
    protocol                       = "Http"
  }

  request_routing_rule {
    name                       = "routing-rule"
    rule_type                  = "Basic"
    http_listener_name         = "http-listener"
    backend_address_pool_name  = "backend-pool"
    backend_http_settings_name = "http-settings"
    priority                   = 100
  }
}

# Network Security Group for App Subnet
resource "azurerm_network_security_group" "app" {
  name                = "nsg-app-${var.environment}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  security_rule {
    name                       = "allow-http"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "80"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "allow-https"
    priority                   = 101
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "allow-ssh"
    priority                   = 102
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "10.0.0.0/8"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "deny-all-other-inbound"
    priority                   = 200
    direction                  = "Inbound"
    access                     = "Deny"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

# Associate NSG with Subnet
resource "azurerm_subnet_network_security_group_association" "app" {
  subnet_id                 = azurerm_subnet.app.id
  network_security_group_id = azurerm_network_security_group.app.id
}
```

#### 4. Variables

**`infra/terraform/variables.tf`:**
```hcl
variable "subscription_id" {
  description = "Azure subscription ID"
  type        = string
}

variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "location" {
  description = "Azure region"
  type        = string
  default     = "eastus"
}

variable "vnet_address_space" {
  description = "VNet address space"
  type        = list(string)
  default     = ["10.0.0.0/16"]
}

variable "app_subnet_prefix" {
  description = "App subnet CIDR"
  type        = list(string)
  default     = ["10.0.1.0/24"]
}

variable "db_subnet_prefix" {
  description = "Database subnet CIDR"
  type        = list(string)
  default     = ["10.0.2.0/24"]
}

variable "gateway_subnet_prefix" {
  description = "Gateway subnet CIDR"
  type        = list(string)
  default     = ["10.0.3.0/24"]
}

variable "allowed_ips" {
  description = "IPs allowed to access storage"
  type        = list(string)
  default     = []
}
```

### GitHub Actions Infrastructure Pipeline

**`.github/workflows/infrastructure-pipeline.yml`:**
```yaml
name: Infrastructure CI/CD

on:
  push:
    branches: [ main, develop ]
    paths:
      - "infra/terraform/**"
      - ".github/workflows/infrastructure-pipeline.yml"
  pull_request:
    branches: [ main, develop ]
    paths:
      - "infra/terraform/**"
  workflow_dispatch:
    inputs:
      environment:
        description: "Environment to deploy"
        required: true
        default: "dev"
        type: choice
        options:
          - dev
          - staging
          - prod

env:
  TF_VERSION: 1.8.0
  ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
  ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
  ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
  ARM_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}

jobs:
  determine-environment:
    name: Determine Environment
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.env.outputs.environment }}
    steps:
      - name: Determine target environment
        id: env
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "environment=prod" >> $GITHUB_OUTPUT
          elif [[ "${{ github.ref }}" == "refs/heads/develop" ]]; then
            echo "environment=staging" >> $GITHUB_OUTPUT
          else
            echo "environment=dev" >> $GITHUB_OUTPUT
          fi

  validate:
    name: Validate Terraform
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infra/terraform
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: Terraform Validate
        run: terraform validate

      - name: Run TFLint
        uses: terraform-linters/setup-tflint@v4
      - run: tflint --init
      - run: tflint

  plan:
    name: Plan Infrastructure
    needs: [determine-environment, validate]
    runs-on: ubuntu-latest
    environment: ${{ needs.determine-environment.outputs.environment }}
    defaults:
      run:
        working-directory: infra/terraform
    outputs:
      plan_file: ${{ steps.plan.outputs.plan_file }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        run: terraform init -backend-config="key=infra-${{ needs.determine-environment.outputs.environment }}.tfstate"

      - name: Terraform Plan
        id: plan
        run: |
          terraform plan \
            -var-file="../../environments/${{ needs.determine-environment.outputs.environment }}.tfvars" \
            -out=tfplan_${{ needs.determine-environment.outputs.environment }}.binary
          echo "plan_file=tfplan_${{ needs.determine-environment.outputs.environment }}.binary" >> $GITHUB_OUTPUT

      - name: Upload Plan Artifact
        uses: actions/upload-artifact@v4
        with:
          name: tfplan-${{ needs.determine-environment.outputs.environment }}
          path: infra/terraform/tfplan_${{ needs.determine-environment.outputs.environment }}.binary
          retention-days: 1

      - name: Comment on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '✅ Terraform plan completed. Review changes and approve for apply.'
            })

  approve:
    name: Require Approval
    needs: plan
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    steps:
      - name: Waiting for approval
        run: echo "Deployment requires manual approval in GitHub"

  apply:
    name: Apply Infrastructure
    needs: [plan, determine-environment]
    runs-on: ubuntu-latest
    environment: ${{ needs.determine-environment.outputs.environment }}
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop' || github.event_name == 'workflow_dispatch'
    defaults:
      run:
        working-directory: infra/terraform
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Download Plan Artifact
        uses: actions/download-artifact@v4
        with:
          name: tfplan-${{ needs.determine-environment.outputs.environment }}
          path: infra/terraform

      - name: Terraform Init
        run: terraform init -backend-config="key=infra-${{ needs.determine-environment.outputs.environment }}.tfstate"

      - name: Terraform Apply
        run: |
          terraform apply \
            -auto-approve \
            tfplan_${{ needs.determine-environment.outputs.environment }}.binary

      - name: Capture Outputs
        id: outputs
        run: |
          echo "vnet_id=$(terraform output -raw vnet_id)" >> $GITHUB_OUTPUT
          echo "app_gateway_ip=$(terraform output -raw app_gateway_public_ip)" >> $GITHUB_OUTPUT

      - name: Notify Slack
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Infrastructure Deployment - ${{ needs.determine-environment.outputs.environment }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "Status: ${{ job.status }}\nEnvironment: ${{ needs.determine-environment.outputs.environment }}\nVNet ID: ${{ steps.outputs.outputs.vnet_id }}\nGateway IP: ${{ steps.outputs.outputs.app_gateway_ip }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## Application Deployment Pipelines

### 1. Java Application Pipeline

**`.github/workflows/deploy-java-app.yml`:**
```yaml
name: Deploy Java Application

on:
  push:
    branches: [ main, develop ]
    paths:
      - "apps/java/**"
  pull_request:
    branches: [ main, develop ]
    paths:
      - "apps/java/**"

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/java-app
  JAVA_VERSION: "17"

jobs:
  build:
    name: Build & Test Java App
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    defaults:
      run:
        working-directory: apps/java
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: "temurin"
          cache: maven

      - name: Run Maven tests
        run: mvn clean test

      - name: Code coverage with JaCoCo
        run: mvn verify

      - name: Build JAR
        run: mvn clean package -DskipTests

      - name: Upload coverage reports
        uses: codecov/codecov-action@v4
        with:
          files: ./target/site/jacoco/jacoco.xml
          flags: java-app

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: java-app-jar
          path: apps/java/target/*.jar

  build-image:
    name: Build Docker Image
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: ./apps/java
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-dev:
    name: Deploy to Dev
    needs: build-image
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: dev
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: "v1.28.0"

      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Get AKS credentials
        run: |
          az aks get-credentials \
            --resource-group rg-dev-eastus-001 \
            --name aks-dev-001 \
            --overwrite-existing

      - name: Create image pull secret
        run: |
          kubectl create secret docker-registry ghcr-secret \
            --docker-server=${{ env.REGISTRY }} \
            --docker-username=${{ github.actor }} \
            --docker-password=${{ secrets.GITHUB_TOKEN }} \
            --docker-email=${{ github.actor }}@users.noreply.github.com \
            --dry-run=client -o yaml | kubectl apply -f -

      - name: Deploy with Helm
        run: |
          helm upgrade --install java-app ./helm/java-app \
            --namespace dev \
            --values ./helm/values-dev.yaml \
            --set image.tag=${{ needs.build-image.outputs.image-tag }} \
            --wait \
            --timeout 5m

      - name: Verify deployment
        run: |
          kubectl rollout status deployment/java-app -n dev --timeout=5m

  deploy-prod:
    name: Deploy to Production
    needs: build-image
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: 
      name: prod
      url: https://java-app-prod.example.com
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup kubectl
        uses: azure/setup-kubectl@v3

      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Get AKS credentials
        run: |
          az aks get-credentials \
            --resource-group rg-prod-eastus-001 \
            --name aks-prod-001

      - name: Create image pull secret
        run: |
          kubectl create secret docker-registry ghcr-secret \
            --docker-server=${{ env.REGISTRY }} \
            --docker-username=${{ github.actor }} \
            --docker-password=${{ secrets.GITHUB_TOKEN }} \
            -n prod \
            --dry-run=client -o yaml | kubectl apply -f -

      - name: Deploy with Blue/Green strategy
        run: |
          # Deploy new version to blue deployment
          helm upgrade --install java-app-blue ./helm/java-app \
            --namespace prod \
            --values ./helm/values-prod.yaml \
            --set image.tag=${{ needs.build-image.outputs.image-tag }} \
            --set deployment.name=java-app-blue \
            --wait

          # Run smoke tests
          kubectl run smoke-test \
            --image=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build-image.outputs.image-tag }} \
            --namespace prod \
            --attach \
            --rm \
            --restart=Never \
            -- /app/healthcheck.sh

          # Switch traffic to new version
          kubectl patch service java-app -p '{"spec":{"selector":{"deployment":"java-app-blue"}}}' -n prod

      - name: Rollback on failure
        if: failure()
        run: |
          kubectl patch service java-app -p '{"spec":{"selector":{"deployment":"java-app-green"}}}' -n prod
          echo "Deployment failed, rolled back to previous version"
```

### 2. Python Application Pipeline

**`.github/workflows/deploy-python-app.yml`:**
```yaml
name: Deploy Python Application

on:
  push:
    branches: [ main, develop ]
    paths:
      - "apps/python/**"
  pull_request:
    branches: [ main, develop ]
    paths:
      - "apps/python/**"

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/python-app
  PYTHON_VERSION: "3.11"

jobs:
  test:
    name: Test Python App
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: apps/python
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: "pip"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest pytest-cov black flake8 mypy

      - name: Lint with flake8
        run: |
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
          flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics

      - name: Format check with black
        run: black --check .

      - name: Type check with mypy
        run: mypy src/

      - name: Run unit tests
        run: pytest tests/ -v --cov=src/ --cov-report=xml

      - name: Upload coverage reports
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage.xml
          flags: python-app

  build-image:
    name: Build Docker Image
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: ./apps/python
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

  deploy-dev:
    name: Deploy to Dev
    needs: build-image
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: dev
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Get AKS credentials
        run: |
          az aks get-credentials \
            --resource-group rg-dev-eastus-001 \
            --name aks-dev-001

      - name: Deploy with kubectl
        run: |
          kubectl set image deployment/python-app \
            python-app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build-image.outputs.image-tag }} \
            -n dev \
            --record

      - name: Wait for rollout
        run: kubectl rollout status deployment/python-app -n dev

  deploy-prod:
    name: Deploy to Production
    needs: build-image
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: prod
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Get AKS credentials
        run: |
          az aks get-credentials \
            --resource-group rg-prod-eastus-001 \
            --name aks-prod-001

      - name: Run pre-deployment tests
        run: |
          kubectl run pre-deploy-test \
            --image=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build-image.outputs.image-tag }} \
            -n prod \
            --rm \
            --attach \
            --restart=Never

      - name: Deploy with Helm (Rolling deployment)
        run: |
          helm upgrade python-app ./helm/python-app \
            --namespace prod \
            --values ./helm/values-prod.yaml \
            --set image.tag=${{ needs.build-image.outputs.image-tag }} \
            --set strategy.type=RollingUpdate \
            --set strategy.rollingUpdate.maxSurge=1 \
            --set strategy.rollingUpdate.maxUnavailable=0 \
            --wait

      - name: Verify deployment health
        run: |
          kubectl get pods -n prod -l app=python-app
          kubectl top pod -n prod -l app=python-app
```

### 3. .NET Application Pipeline

**`.github/workflows/deploy-dotnet-app.yml`:**
```yaml
name: Deploy .NET Application

on:
  push:
    branches: [ main, develop ]
    paths:
      - "apps/dotnet/**"
  pull_request:
    branches: [ main, develop ]
    paths:
      - "apps/dotnet/**"

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/dotnet-app
  DOTNET_VERSION: "8.0"

jobs:
  build:
    name: Build & Test .NET App
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    defaults:
      run:
        working-directory: apps/dotnet
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Restore dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build --configuration Release --no-restore

      - name: Run unit tests
        run: dotnet test --configuration Release --no-build --verbosity normal --collect:"XPlat Code Coverage"

      - name: Upload coverage reports
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage.xml
          flags: dotnet-app

      - name: Publish
        run: dotnet publish --configuration Release --output ./publish

      - name: Upload publish artifact
        uses: actions/upload-artifact@v4
        with:
          name: dotnet-app-publish
          path: apps/dotnet/publish/

  build-image:
    name: Build Docker Image
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: ./apps/dotnet
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

  deploy-dev:
    name: Deploy to Dev
    needs: build-image
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: dev
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Get AKS credentials
        run: |
          az aks get-credentials \
            --resource-group rg-dev-eastus-001 \
            --name aks-dev-001

      - name: Deploy application
        run: |
          kubectl set image deployment/dotnet-app \
            dotnet-app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build-image.outputs.image-tag }} \
            -n dev

      - name: Wait for deployment
        run: kubectl rollout status deployment/dotnet-app -n dev --timeout=5m

  deploy-prod:
    name: Deploy to Production
    needs: build-image
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: prod
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Get AKS credentials
        run: |
          az aks get-credentials \
            --resource-group rg-prod-eastus-001 \
            --name aks-prod-001

      - name: Canary deployment (10% traffic)
        run: |
          helm upgrade --install dotnet-app-canary ./helm/dotnet-app \
            --namespace prod \
            --set replicas=1 \
            --set weight=10 \
            --set image.tag=${{ needs.build-image.outputs.image-tag }} \
            --wait

      - name: Run canary tests
        run: |
          for i in {1..10}; do
            kubectl run canary-test-$i \
              --image=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build-image.outputs.image-tag }} \
              -n prod \
              --rm \
              --attach \
              --restart=Never
          done

      - name: Full production deployment
        run: |
          helm upgrade --install dotnet-app ./helm/dotnet-app \
            --namespace prod \
            --values ./helm/values-prod.yaml \
            --set image.tag=${{ needs.build-image.outputs.image-tag }} \
            --set replicas=3 \
            --wait
```

---

## Real-Time Troubleshooting Guide

### 1. Governance Issues

#### Issue 1.1: Subscription or Resource Group Access Denied

**Error Message:**
```
Error: Error building AzureRM Client: client_id has no access to subscription
```

**Root Cause:**
- Service Principal lacks permissions
- Subscription ID mismatch
- Service Principal assigned wrong role

**Resolution Steps:**
```bash
# 1. Verify the service principal has contributor role
az role assignment list \
  --assignee <SERVICE_PRINCIPAL_ID> \
  --scope /subscriptions/<SUBSCRIPTION_ID>

# 2. If missing, add the role
az role assignment create \
  --assignee <SERVICE_PRINCIPAL_ID> \
  --role "Contributor" \
  --scope /subscriptions/<SUBSCRIPTION_ID>

# 3. Verify in GitHub Actions secrets
# Check AZURE_SUBSCRIPTION_ID, AZURE_CLIENT_ID, AZURE_TENANT_ID

# 4. Test Azure login
az login --service-principal \
  -u $AZURE_CLIENT_ID \
  -p $AZURE_CLIENT_SECRET \
  --tenant $AZURE_TENANT_ID
```

**Prevention:**
- Use least-privilege IAM (specific roles, not Contributor)
- Rotate service principal credentials regularly
- Use OIDC federated credentials instead of secrets

---

#### Issue 1.2: Resource Quota Exceeded

**Error Message:**
```
Error: creating Application Gateway: Code="InvalidTemplateDeployment" Message="The template deployment failed with error: 'The current subscription does not have enough capacity to complete the request for the requested quantity in the current regions.'"
```

**Root Cause:**
- Regional quota limits exceeded
- Too many resources deployed simultaneously

**Resolution:**
```bash
# 1. Check current usage
az compute vm list -g <resource-group> --query "length(@)"
az network public-ip list -g <resource-group> --query "length(@)"

# 2. Check quota limits
az compute vm list-usage -l eastus --query "[?name=='Total Regional vCPUs']"

# 3. Options:
# Option A: Deploy to different region
terraform apply -var="location=westus"

# Option B: Delete unused resources
az resource delete -g <resource-group> --name <resource-name> --resource-type <type>

# Option C: Request quota increase from Azure portal
```

---

#### Issue 1.3: Resource Policy Violation

**Error Message:**
```
Error: Code="RequestDisallowedByPolicy" Message="Resource 'xxx' was disallowed by policy 'Deny-Public-IP-Addresses'"
```

**Root Cause:**
- Azure Policy denies public IPs, managed disks, etc.
- Governance policies enforce compliance

**Resolution:**
```bash
# 1. List active policies
az policy assignment list -g <resource-group>

# 2. View policy details
az policy assignment show \
  --name "Deny-Public-IP-Addresses" \
  -g <resource-group>

# 3. Update Terraform to comply
# Instead of public IP, use private IP + Azure Bastion
resource "azurerm_private_dns_zone" "internal" {
  name = "internal.example.com"
}

# 4. Request policy exception (if needed)
# Contact governance team for exemption
```

---

### 2. Deployment Issues

#### Issue 2.1: Docker Image Pull Failed

**Error Message:**
```
ImagePullBackOff: Failed to pull image "ghcr.io/myapp:v1": rpc error: code = Unknown desc = Error response from daemon: pull access denied
```

**Root Cause:**
- Kubernetes secret for registry not created
- Secret in wrong namespace
- Registry credentials expired

**Resolution:**
```bash
# 1. Create/verify image pull secret
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=<github-username> \
  --docker-password=<github-token> \
  -n prod

# 2. Add to service account
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets": [{"name": "ghcr-secret"}]}' \
  -n prod

# 3. Verify secret exists
kubectl get secrets -n prod

# 4. Debug pod
kubectl describe pod <pod-name> -n prod
kubectl logs <pod-name> -n prod
```

**Prevention in GitHub Actions:**
```yaml
- name: Create image pull secret
  run: |
    kubectl create secret docker-registry ghcr-secret \
      --docker-server=ghcr.io \
      --docker-username=${{ github.actor }} \
      --docker-password=${{ secrets.GITHUB_TOKEN }} \
      --dry-run=client -o yaml | kubectl apply -f -
```

---

#### Issue 2.2: CrashLoopBackOff

**Error Message:**
```
NAME          READY   STATUS             RESTARTS   AGE
java-app-xyz  0/1     CrashLoopBackOff   5          2m15s
```

**Root Cause:**
- Application crashed on startup
- Missing environment variables
- Resource limits too low
- Health check failing

**Resolution:**
```bash
# 1. View pod logs
kubectl logs <pod-name> -n prod --previous
kubectl logs <pod-name> -n prod --tail=100 -f

# 2. Describe pod to see events
kubectl describe pod <pod-name> -n prod

# 3. Common fixes:

# Fix A: Check environment variables
kubectl set env deployment/java-app \
  DATABASE_URL=jdbc:mysql://db:3306/myapp \
  -n prod

# Fix B: Review resource limits
kubectl set resources deployment/java-app \
  --limits=memory=512Mi,cpu=500m \
  --requests=memory=256Mi,cpu=250m \
  -n prod

# Fix C: Disable or fix health checks temporarily
kubectl patch deployment java-app \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"java-app","livenessProbe":null}]}}}}' \
  -n prod

# 4. Redeploy
kubectl rollout restart deployment/java-app -n prod
```

**GitHub Actions to mitigate:**
```yaml
- name: Run integration tests before deploy
  run: |
    docker run --rm \
      -e DATABASE_URL=${{ secrets.DATABASE_URL }} \
      -e LOG_LEVEL=DEBUG \
      ghcr.io/myapp:${{ github.sha }} \
      ./healthcheck.sh
```

---

#### Issue 2.3: Helm Apply Fails - Chart Validation Error

**Error Message:**
```
Error: chart requires kubeVersion: >= 1.24.0, current version is 1.23.x
Error: unable to build kubernetes objects from release manifest
```

**Root Cause:**
- Helm chart version incompatible with K8s cluster
- Invalid values in helm values file
- CRD not installed

**Resolution:**
```bash
# 1. Check K8s version
kubectl version --short

# 2. Validate Helm chart
helm lint ./helm/java-app

# 3. Dry-run deployment
helm install java-app ./helm/java-app \
  --namespace prod \
  --values ./helm/values-prod.yaml \
  --dry-run \
  --debug

# 4. Check for missing CRDs
kubectl get crd

# 5. If CRD missing, install first
kubectl apply -f ./helm/crds/

# 6. Deploy with correct K8s version targeting
helm install java-app ./helm/java-app \
  --namespace prod \
  --kubeVersion 1.23.x
```

---

### 3. Connectivity Issues

#### Issue 3.1: Pod Cannot Reach Database

**Error Message:**
```
Connection refused: Unable to connect to database at postgres.db.svc.cluster.local:5432
```

**Root Cause:**
- DNS resolution failure
- Network policy blocking traffic
- Database service not running
- Wrong connection string

**Resolution:**
```bash
# 1. Verify database pod is running
kubectl get pods -n db
kubectl describe pod postgres-0 -n db

# 2. Test DNS resolution from app pod
kubectl exec -it <app-pod> -n prod -- nslookup postgres.db.svc.cluster.local
kubectl exec -it <app-pod> -n prod -- ping postgres.db.svc.cluster.local

# 3. Check network policies blocking traffic
kubectl get networkpolicies -n db
kubectl get networkpolicies -n prod

# 4. Review NetworkPolicy to allow traffic
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-db
  namespace: db
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: prod
      ports:
        - protocol: TCP
          port: 5432
EOF

# 5. Check connection directly from pod
kubectl exec -it <app-pod> -n prod -- \
  nc -zv postgres.db.svc.cluster.local 5432

# 6. Verify service configuration
kubectl get svc postgres -n db
kubectl get endpoints postgres -n db
```

---

#### Issue 3.2: Ingress Not Routing Traffic

**Error Message:**
```
curl: (7) Failed to connect to myapp.example.com
DNS lookup error: no such host
```

**Root Cause:**
- Ingress controller not running
- DNS not pointing to ingress IP
- Ingress rule misconfigured
- TLS certificate missing

**Resolution:**
```bash
# 1. Check ingress controller
kubectl get pods -n ingress-nginx

# 2. Get ingress details
kubectl get ingress -n prod
kubectl describe ingress java-app-ingress -n prod

# 3. Get ingress IP
kubectl get ingress java-app-ingress -n prod -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# 4. Verify DNS points to ingress IP
nslookup myapp.example.com

# 5. Test direct access using ingress IP
curl -H "Host: myapp.example.com" http://<ingress-ip>

# 6. Check backend service
kubectl get svc java-app -n prod
kubectl get endpoints java-app -n prod

# 7. Verify TLS certificate if HTTPS
kubectl get certificates -n prod
kubectl describe certificate java-app-cert -n prod

# 8. Example working ingress config
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: java-app-ingress
  namespace: prod
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: java-app-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: java-app
                port:
                  number: 8080
EOF
```

---

### 4. Authorization Issues

#### Issue 4.1: ServiceAccount Lacks RBAC Permissions

**Error Message:**
```
error: jobs.batch is forbidden: User "system:serviceaccount:prod:default" cannot create resource "jobs" in API group "batch" in the namespace "prod"
```

**Root Cause:**
- ServiceAccount missing required Role/ClusterRole binding
- Role doesn't grant required permissions
- Wrong namespace

**Resolution:**
```bash
# 1. Check current RBAC for ServiceAccount
kubectl get rolebindings -n prod
kubectl get clusterrolebindings

# 2. Check what permissions serviceaccount has
kubectl auth can-i create jobs \
  --as=system:serviceaccount:prod:default \
  -n prod

# 3. Create Role with required permissions
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: prod
  name: app-deployer
rules:
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["create", "get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods", "pods/logs"]
    verbs: ["get", "list"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "update"]
EOF

# 4. Bind Role to ServiceAccount
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: prod
  name: app-deployer-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-deployer
subjects:
  - kind: ServiceAccount
    name: default
    namespace: prod
EOF

# 5. Verify permissions now granted
kubectl auth can-i create jobs \
  --as=system:serviceaccount:prod:default \
  -n prod
```

---

#### Issue 4.2: GitHub Actions Cannot Access Azure Resources

**Error Message:**
```
ERROR: AADSTS700016: Application with identifier 'xxx' was not found in the directory 'yyy'. This can happen if the application has not been installed by the administrator of the tenant or consented to by any user in the tenant.
```

**Root Cause:**
- Service Principal not registered in Azure AD
- OIDC federated credentials not configured
- Expired or wrong secrets

**Resolution:**
```bash
# 1. Verify Service Principal exists
az ad sp list --all --query "[?appDisplayName=='<app-name>']"

# 2. Set up OIDC Federated Credentials (recommended over secrets)
az identity federated-credential create \
  --name "github-actions" \
  --identity-name "<managed-identity-name>" \
  --issuer "https://token.actions.githubusercontent.com" \
  --subject "repo:<owner>/<repo>:ref:refs/heads/main" \
  --resource-group "<resource-group>"

# 3. Add to GitHub Actions secrets (if using secrets instead)
AZURE_CLIENT_ID=$(az ad sp show --id <client-id> --query appId -o tsv)
AZURE_CLIENT_SECRET=$(az ad sp credential reset --id <client-id> --query password -o tsv)
AZURE_TENANT_ID=$(az account show --query tenantId -o tsv)
AZURE_SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# 4. Add to GitHub repo Settings → Secrets → Actions

# 5. Use in workflow:
env:
  ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
  ARM_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
  ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
  ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

---

### 5. GitHub Actions Workflow Failures

#### Issue 5.1: Terraform Plan Job Fails on PR

**Error:**
```
Error: error acquiring the state lock
Code: BlobAlreadyLeased
```

**Root Cause:**
- Previous terraform run didn't complete properly
- State file is locked
- Multiple PRs trying to plan simultaneously

**Resolution:**
```bash
# 1. Check lock status in Azure Storage
az storage blob lease break \
  --account-name "tfstateprod123" \
  --container-name "tfstate" \
  --blob-name "infra-prod.tfstate.lock"

# 2. Force unlock from Terraform (use with caution)
terraform force-unlock <LOCK_ID>

# 3. Add concurrency control to workflow
jobs:
  plan:
    concurrency:
      group: terraform-${{ matrix.environment }}
      cancel-in-progress: false
```

---

#### Issue 5.2: Docker Build Exceeds 6-hour Timeout

**Error:**
```
Error: The job was cancelled because it took longer than 6 hours
```

**Root Cause:**
- Slow compilation or tests
- Large dependencies
- Network issues

**Solution:**
```yaml
# 1. Use Docker layer caching
- name: Build Docker image
  uses: docker/build-push-action@v5
  with:
    context: ./apps/java
    cache-from: type=gha
    cache-to: type=gha,mode=max

# 2. Parallel jobs in matrix
strategy:
  matrix:
    app: [java-app, python-app, dotnet-app]

# 3. Increase runner timeout
timeout-minutes: 360  # 6 hours
```

---

#### Issue 5.3: Secret Not Available in Workflow

**Error:**
```
Error: The referenced secret does not exist
${{ secrets.DATABASE_PASSWORD }} is null
```

**Root Cause:**
- Secret not created in repo/org/environment
- Wrong secret name
- Missing environment configuration

**Resolution:**
```bash
# 1. Create secret in GitHub
# Settings → Secrets and variables → Actions → New repository secret

# 2. Use correct reference
env:
  DB_PASSWORD: ${{ secrets.DATABASE_PASSWORD }}

# 3. For environment-specific secrets
environment: prod  # Links to prod environment secrets
env:
  DB_PASSWORD: ${{ secrets.DB_PASSWORD }}

# 4. Verify secret exists (in workflow logs will show ***)
- name: Verify secrets
  run: |
    if [ -z "${{ secrets.DATABASE_PASSWORD }}" ]; then
      echo "ERROR: DATABASE_PASSWORD secret not set"
      exit 1
    fi
```

---

### Troubleshooting Checklist

| Category | Check | Command |
|----------|-------|---------|
| **Authentication** | Azure login | `az account show` |
| | GitHub token | `gh auth status` |
| | Kubernetes access | `kubectl auth can-i get pods` |
| **Infrastructure** | Terraform state | `terraform state list` |
| | Resource quotas | `az compute vm list-usage` |
| | VNet connectivity | `kubectl exec -it <pod> -- ping <service>` |
| **Application** | Pod status | `kubectl get pods -n prod` |
| | Pod logs | `kubectl logs <pod> -n prod` |
| | Service endpoints | `kubectl get endpoints -n prod` |
| **CI/CD** | Workflow status | GitHub → Actions tab |
| | Runner logs | View in GitHub Actions UI |
| | Image pull | `kubectl describe pod <pod>` |

---

## Best Practices

### 1. Infrastructure as Code
- ✅ Store all IaC in Git (Terraform, Helm charts)
- ✅ Use modules for reusability
- ✅ Separate environments with tfvars files
- ✅ Enable state locking and versioning
- ✅ Use remote backends (Azure Storage, Terraform Cloud)

### 2. CI/CD Pipeline Design
- ✅ Separate plan and apply stages
- ✅ Require manual approval for production
- ✅ Use environment protection rules
- ✅ Implement status checks on PRs
- ✅ Store secrets securely (GitHub Secrets, Azure Key Vault)

### 3. Application Deployment
- ✅ Use containers for consistency
- ✅ Implement health checks (readiness/liveness probes)
- ✅ Use rolling updates or blue/green deployments
- ✅ Implement automated rollbacks on failure
- ✅ Monitor deployments with alerts

### 4. Security
- ✅ Use OIDC federated credentials instead of secrets
- ✅ Rotate credentials regularly
- ✅ Use least-privilege IAM roles
- ✅ Encrypt state files at rest and in transit
- ✅ Audit all infrastructure changes

### 5. Observability
- ✅ Log workflow executions
- ✅ Monitor application deployments
- ✅ Alert on critical failures
- ✅ Track infrastructure drift
- ✅ Use distributed tracing for applications

---

## Quick Reference

### Common Commands

**GitHub Actions:**
```bash
# View workflow run
gh run view <run-id>

# Re-run failed jobs
gh run rerun <run-id>

# View logs
gh run view <run-id> --log
```

**Terraform:**
```bash
# Initialize backend
terraform init

# Validate configuration
terraform validate

# Preview changes
terraform plan -out=tfplan

# Apply changes
terraform apply tfplan

# Destroy resources
terraform destroy

# Lock resources
terraform lock
```

**Kubernetes:**
```bash
# Deploy application
kubectl apply -f deployment.yaml

# Check rollout
kubectl rollout status deployment/<name>

# View logs
kubectl logs <pod> -f

# Debug pod
kubectl describe pod <pod>
kubectl exec -it <pod> -- /bin/bash
```

**Azure CLI:**
```bash
# Login
az login

# Set subscription
az account set --subscription <id>

# Get resource info
az resource list --query "[?contains(name, '<pattern>')]"

# Delete resource
az resource delete --ids <resource-id>
```

---

## Interview Preparation Summary

### Key Topics to Master
1. **Infrastructure Pipelines** - Terraform + GitHub Actions
2. **Application Deployment** - Multi-language support (Java, Python, .NET)
3. **Troubleshooting** - Systematic approach to diagnosis
4. **Security** - RBAC, secrets management, OIDC
5. **Kubernetes** - Deployments, services, networking, RBAC

### Interview Questions to Prepare
- How do you structure Terraform code for multi-environment deployment?
- What's the difference between Helm apply and kubectl apply?
- How do you handle secrets securely in CI/CD pipelines?
- What causes CrashLoopBackOff and how do you debug it?
- Design a zero-downtime deployment strategy
- How do you implement RBAC for multi-tenant clusters?

### Practice Exercises
1. Create a complete infrastructure pipeline from scratch
2. Deploy 3 different language applications to Kubernetes
3. Troubleshoot a failing deployment in 5 minutes
4. Design a disaster recovery plan using Terraform
5. Implement blue/green deployment with Helm
