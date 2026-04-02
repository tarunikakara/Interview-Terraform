# Azure Infrastructure & Terraform Interview Preparation Guide

## 📚 Table of Contents
1. [Azure Cloud - Real-Time Scenario Questions](#azure-cloud-scenario-questions)
2. [Terraform + IaC - Azure-Focused Questions](#terraform-iac-questions)
3. [GitHub Actions - CI/CD Concepts](#github-actions-cicd)
4. [Troubleshooting GitHub Actions Failures](#troubleshooting-failures)
5. [Terraform State - Deep Dive](#terraform-state-deepdive)

---

## Azure Cloud - Real-Time Scenario Questions {#azure-cloud-scenario-questions}

### 1.1 Creating Servers (VMs) in Azure

#### **Question 1: How do you design and provision a production-grade VM in Azure using best practices?**

**Key Points to Mention:**
- **Services involved:** Azure VM, Managed Disks, Network Interface Card (NIC), NSG, Public IP, Load Balancer
- **Design principles:**
  - Use **managed disks** (automatic backup, snapshots, encryption)
  - Assign **NSGs** for network security
  - Use **public IP** only if internet-facing, otherwise private IP
  - Enable **Azure Backup** for disaster recovery
- **Code structure:**
  ```hcl
  resource "azurerm_linux_virtual_machine" "web_vm" {
    name                = "prod-vm-01"
    resource_group_name = azurerm_resource_group.rg.name
    location            = azurerm_resource_group.rg.location
    size                = "Standard_B2s"
    
    admin_username = "azureuser"
    
    admin_ssh_key {
      username   = "azureuser"
      public_key = file("~/.ssh/id_rsa.pub")
    }
    
    os_disk {
      caching              = "ReadWrite"
      storage_account_type = "Premium_LRS"
    }
    
    source_image_reference {
      publisher = "Canonical"
      offer     = "0001-com-ubuntu-server-focal"
      sku       = "20_04-lts-gen2"
      version   = "latest"
    }
    
    tags = {
      Environment = "Production"
      ManagedBy   = "Terraform"
    }
  }
  ```

---

#### **Question 2: How do you choose between Availability Sets, Availability Zones, and Scale Sets for compute?**

**Decision Matrix:**

| Scenario | Solution | Why | SLA |
|----------|----------|-----|-----|
| **Single app, no HA** | Single VM | Simplest, cost-effective | 95% |
| **App needs HA, same region** | Availability Set (2-3 VMs) | Protects against hardware failure | 99.95% |
| **App needs zone-level resilience** | Availability Zones (across zones) | Protects entire zone failure | 99.99% |
| **Dynamic scaling needed** | Virtual Machine Scale Set (VMSS) | Auto-scale based on metrics | 99.95% |
| **Mission-critical, DR needed** | VMSS + multi-region | Highest resilience | 99.999% |

**Terraform Example - Availability Zones:**
```hcl
resource "azurerm_linux_virtual_machine" "vm_zone1" {
  name                 = "prod-vm-zone1"
  zone                 = "1"  # Availability Zone 1
  resource_group_name  = azurerm_resource_group.rg.name
  location             = azurerm_resource_group.rg.location
  # ... rest of config
}

resource "azurerm_linux_virtual_machine" "vm_zone2" {
  name                 = "prod-vm-zone2"
  zone                 = "2"  # Availability Zone 2
  resource_group_name  = azurerm_resource_group.rg.name
  location             = azurerm_resource_group.rg.location
  # ... rest of config
}
```

---

#### **Question 3: How would you automate VM creation using Terraform and deploy it via GitHub Actions?**

**Ideal Answer Structure:**
- Use **Terraform modules** for reusability (azurerm_linux_virtual_machine module)
- Store **IaC in GitHub** with folder structure: `infra/modules/compute/vm`, `infra/environments/prod`
- Use **GitHub Actions** to trigger `terraform plan` on PR, `terraform apply` on merge to main
- Implement **OIDC authentication** (no long-lived secrets)
- Use **environments** with manual approval gates for production
- **CI/CD workflow:**
  ```
  PR created → terraform plan → post on PR → review → merge → terraform apply → resources created
  ```

---

### 1.2 Providing Access to Servers

#### **Question 4: How do you securely provide SSH/RDP access to Azure VMs?**

**Security Approach (Priority Order):**

1. **Just-in-Time (JIT) VM Access** ⭐ (Most Secure)
   - Via **Microsoft Defender for Cloud**
   - SSH/RDP available only for approved time window (e.g., 2 hours)
   - Requires MFA, audit logs all access

2. **Azure Bastion** (Recommended for Regular Access)
   - Provides RDP/SSH over TLS (no public IP needed)
   - Bastion host lives in Azure, you connect via portal or CLI
   - All traffic encrypted, session logs available
   - Cost: ~$18-30/month

3. **NSG + Public IP** (For Dev/Test Only)
   - Allow port 22 (SSH) / 3389 (RDP) from specific IPs
   - Use strong passwords + Key-based auth
   - Not recommended for production

**Terraform + Bastion Example:**
```hcl
resource "azurerm_bastion_host" "bastion" {
  name                = "prod-bastion"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                 = "bastion-ip-config"
    subnet_id            = azurerm_subnet.bastion.id
    public_ip_address_id = azurerm_public_ip.bastion.id
  }
}

# NSG rule: Allow SSH only from Bastion subnet
resource "azurerm_network_security_group_rule" "allow_bastion_ssh" {
  name                        = "allow-bastion-ssh"
  priority                    = 100
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "22"
  source_address_prefix       = "10.1.1.0/24"  # Bastion subnet
  destination_address_prefix  = "*"
  resource_group_name         = azurerm_resource_group.rg.name
  network_security_group_name = azurerm_network_security_group.vms.name
}
```

---

#### **Question 5: How would you give a developer temporary access to a VM without exposing it to the internet?**

**Answer:**
- Use **JIT VM access** from Defender for Cloud → automatic time-based expiry
- Or use **Azure AD login** + RBAC roles (e.g., "Virtual Machine Administrator Login")
- **Never** expose VM to internet with open SSH/RDP
- Alternative: Use **Bastion** with RBAC (control who can access Bastion)

---

#### **Question 6: How do you manage access using RBAC vs local OS accounts?**

**RBAC Approach (Enterprise-Standard):**
- Users authenticate via **Azure AD**
- Assign **built-in roles:** "Virtual Machine Administrator Login", "Virtual Machine User Login", "Reader"
- All access logged in **Azure Activity Log**
- MFA enforced at Azure AD level
- Fine-grained control, audit-friendly

**Local OS Accounts (Legacy/Dev):**
- SSH keys or passwords on Linux, local users on Windows
- Hard to audit, difficult to revoke access
- ❌ Not recommended for production

**Terraform Example - RBAC:**
```hcl
resource "azurerm_role_assignment" "vm_admin" {
  scope              = azurerm_linux_virtual_machine.web_vm.id
  role_definition_name = "Virtual Machine Administrator Login"
  principal_id       = data.azuread_user.dev.object_id
}
```

---

### 1.3 Inbound and Outbound Rules (NSG / Firewall)

#### **Question 7: How do you design inbound and outbound rules for a web application hosted on Azure VMs?**

**Design Pattern:**

**Inbound:**
```
Internet (Port 443) 
  ↓ (HTTPS)
Azure Application Gateway (WAF - Web Application Firewall)
  ↓ 
NSG: Allow port 443 from App Gateway subnet
  ↓ 
VMs running web app
```

**Outbound:**
```
VMs 
  ↓ (Need to call external APIs, download updates)
NSG Outbound Rule: Allow 443 to Internet
  ↓ (Optional: Use Azure Firewall for centralized control)
Internet
```

**Terraform - NSG Example:**
```hcl
resource "azurerm_network_security_group" "web_tier" {
  name                = "nsg-web-tier"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  # Inbound: Allow HTTPS from internet
  security_rule {
    name                       = "AllowHTTPS"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  # Inbound: Allow SSH from Bastion only
  security_rule {
    name                       = "AllowSSHFromBastion"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "10.1.1.0/24"  # Bastion subnet
    destination_address_prefix = "*"
  }

  # Outbound: Allow all (default deny then add specific rules)
  security_rule {
    name                       = "AllowOutboundInternet"
    priority                   = 100
    direction                  = "Outbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}
```

---

#### **Question 8: What's the difference between NSG, Azure Firewall, and Application Gateway?**

| Tool | Layer | Use Case | Example |
|------|-------|----------|---------|
| **NSG** | L3/L4 (Network) | Subnet/NIC-level filtering | Allow port 443 to subnet |
| **Azure Firewall** | L3/L4/L7+ | Centralized egress control, threat detection | Block all outbound except AzureMonitor, Storage services |
| **App Gateway** | L7 (Application) | Load balancing, WAF, SSL termination, routing | Route /api → backend1, /images → backend2 |

**When to use each:**
- **NSG:** Every subnet/VM (mandatory, cheap)
- **Firewall:** Need centralized outbound control, threat protection
- **App Gateway:** Public-facing web apps needing WAF + load balancing

---

#### **Question 9: How would you lock down outbound traffic from a subnet?**

**Steps:**
1. Set **NSG default outbound rule to Deny**
2. Add explicit **Allow rules** only for required destinations
3. Use **service tags** instead of IP addresses

**Example:**
```hcl
resource "azurerm_network_security_group" "lockdown" {
  name                = "nsg-lockdown"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  # Deny all outbound by default
  security_rule {
    name                       = "DenyAllOutbound"
    priority                   = 4096  # Lowest priority default
    direction                  = "Outbound"
    access                     = "Deny"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  # Allow outbound to Azure Monitor (service tag)
  security_rule {
    name                       = "AllowAzureMonitor"
    priority                   = 100
    direction                  = "Outbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "*"
    destination_address_prefix = "AzureMonitor"
  }

  # Allow outbound to Storage (service tag)
  security_rule {
    name                       = "AllowStorage"
    priority                   = 110
    direction                  = "Outbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "*"
    destination_address_prefix = "Storage"
  }
}
```

---

## Terraform + IaC - Azure-Focused Questions {#terraform-iac-questions}

### 2.1 Designing IaC for Azure

#### **Question 10: How do you structure Terraform code for multi-environment Azure (dev/test/prod)?**

**Recommended Structure:**
```
infra/
├── modules/
│   ├── network/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── compute/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── storage/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── test/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       └── terraform.tfvars
├── .github/
│   └── workflows/
│       └── terraform.yml
└── README.md
```

**Benefits:**
- ✅ Reusable modules across environments
- ✅ Environment-specific values in `terraform.tfvars`
- ✅ CI/CD can target specific environments
- ✅ Easy to scale and maintain

**Example - Prod env main.tf:**
```hcl
terraform {
  required_version = ">= 1.5"
  
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"
    }
  }

  backend "azurerm" {
    resource_group_name  = "rg-tfstate-prod"
    storage_account_name = "tfstateprod01"
    container_name       = "tfstate"
    key                  = "prod.tfstate"
  }
}

provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
}

module "network" {
  source = "../../modules/network"
  
  environment         = "prod"
  resource_group_name = azurerm_resource_group.prod.name
  location            = azurerm_resource_group.prod.location
  vnet_cidr          = var.prod_vnet_cidr
}

module "compute" {
  source = "../../modules/compute"
  
  environment         = "prod"
  resource_group_name = azurerm_resource_group.prod.name
  location            = azurerm_resource_group.prod.location
  vm_size            = var.prod_vm_size
  vm_count           = 3  # HA
}
```

---

#### **Question 11: How do you manage remote state and state locking in Azure?**

**Remote State Backend Setup:**

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "tfstateprod123"
    container_name       = "tfstate"
    key                  = "infra-prod.tfstate"
  }
}
```

**Azure Storage Setup (Prerequisites):**
```bash
# Create resource group
az group create --name rg-tfstate --location eastus

# Create storage account
az storage account create \
  --name tfstateprod123 \
  --resource-group rg-tfstate \
  --location eastus \
  --sku Standard_LRS

# Create container
az storage container create \
  --name tfstate \
  --account-name tfstateprod123

# Enable versioning for automatic backups
az storage account blob-service-properties update \
  --account-name tfstateprod123 \
  --resource-group rg-tfstate \
  --enable-change-feed true \
  --enable-versioning true
```

**State Locking - How it works:**
- When `terraform apply` runs, it acquires a **lock** (stored in Azure Storage blob)
- If another `apply` tries to run → **blocked until lock is released**
- Prevents concurrent modifications → no state corruption
- Automatic unlock after completion

**Manual unlock (if stuck):**
```bash
terraform force-unlock <LOCK_ID>
```

---

#### **Question 12: How do you handle secrets (client IDs, client secrets) in Terraform?**

**❌ DON'T DO:**
```hcl
# Bad: Never hardcode secrets!
provider "azurerm" {
  client_id       = "12345-secret-id"
  client_secret   = "my-super-secret-password"
  tenant_id       = "67890"
  subscription_id = "abcdef"
}
```

**✅ DO THIS - Option 1: OIDC + Federated Credentials (Recommended):**

GitHub Actions + Azure OIDC (no secrets needed):
```hcl
provider "azurerm" {
  features {}
  
  use_oidc = true
  client_id       = var.client_id        # From GitHub secrets
  tenant_id       = var.tenant_id        # From GitHub secrets
  subscription_id = var.subscription_id  # From GitHub secrets
}
```

GitHub Actions workflow:
```yaml
- name: Azure Login (OIDC)
  uses: azure/login@v2
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

**✅ DO THIS - Option 2: Environment Variables:**
```bash
export ARM_CLIENT_ID="..."
export ARM_CLIENT_SECRET="..."  # Don't store in code!
export ARM_TENANT_ID="..."
export ARM_SUBSCRIPTION_ID="..."

terraform plan
```

**✅ DO THIS - Option 3: Azure Key Vault Reference:**
```hcl
# Pre-populate Terraform variables from Key Vault
variable "db_password" {
  description = "DB password from Key Vault"
  type        = string
  sensitive   = true
}

resource "azurerm_mssql_server" "db" {
  administrator_login_password = var.db_password
  # ...
}
```

Then pass value:
```bash
terraform apply -var="db_password=$(az keyvault secret show --name db-password --vault-name myprod --query value -o tsv)"
```

---

### 2.2 Terraform State – Troubleshooting & Backup

#### **Question 13: What can go wrong with a Terraform state file and how do you fix it?**

**Common Issues & Solutions:**

| Issue | Cause | Fix |
|-------|-------|-----|
| **State locked** | Previous run crashed/didn't unlock | `terraform force-unlock <LOCK_ID>` |
| **Drift** | Manual changes in Azure console | `terraform refresh` or `terraform import` |
| **Resource exists but state doesn't** | Manual Azure creation, state lost | `terraform import azurerm_resource_group.rg /subscriptions/xxx` |
| **State corrupted** | File edited, network interrupt | Restore from backup/remote backend |
| **Provider version mismatch** | Updated provider, old state format | `terraform init -upgrade` |
| **Resource deleted manually** | Someone deleted in portal | Re-create: `terraform apply` |

---

#### **Question 14: How do you recover from a corrupted or accidentally modified state?**

**Recovery Steps:**

1. **Check remote backend history:**
   ```bash
   # List all versions of state file in Azure Storage
   az storage blob version list \
     --account-name tfstateprod123 \
     --container-name tfstate \
     --name infra-prod.tfstate
   ```

2. **Restore from previous version:**
   ```bash
   az storage blob copy start \
     --account-name tfstateprod123 \
     --container-name tfstate \
     --name infra-prod.tfstate \
     --source-account-name tfstateprod123 \
     --source-container tfstate \
     --source-blob "infra-prod.tfstate?versionid=<VERSION_ID>"
   ```

3. **If no backup, rebuild from scratch:**
   ```bash
   # Delete state (careful!)
   terraform state rm azurerm_resource_group.rg
   terraform state rm azurerm_linux_virtual_machine.web_vm
   
   # Re-import existing resources
   terraform import azurerm_resource_group.rg /subscriptions/xxx
   terraform import azurerm_linux_virtual_machine.web_vm /subscriptions/xxx
   ```

---

#### **Question 15: How do you centralize and back up Terraform state in Azure?**

**Best Practice - Remote State + Backup:**

1. **Azure Storage as Backend:**
   - Centralized location (team access via RBAC)
   - Automatic locking for concurrency control
   - Encryption at rest (default)

2. **Enable Versioning (Auto-Backup):**
   ```bash
   az storage account blob-service-properties update \
     --account-name tfstateprod123 \
     --enable-versioning true
   ```

3. **Add Lifecycle Policy (Old versions cleanup):**
   ```bash
   az storage account management-policy create \
     --account-name tfstateprod123 \
     --resource-group rg-tfstate \
     --policy '{
       "rules": [{
         "name": "DeleteOldStateVersions",
         "definition": {
           "filters": {"blobTypes": ["blockBlob"]},
           "actions": {
             "version": {"delete": {"daysAfterCreationGreater": 90}}
           }
         }
       }]
     }'
   ```

4. **Monitor State Changes:**
   - Enable **Activity Log** alerts on storage account
   - Alert on `BlobWrite`, `BlobDelete` → Slack/email notification

5. **Disaster Recovery - Copy to Another Region:**
   ```bash
   # Cross-region replication
   az storage account update \
     --name tfstateprod123 \
     --resource-group rg-tfstate \
     --storage-redundancy GRS  # Geo-redundant storage
   ```

---

## GitHub Actions - CI/CD Concepts {#github-actions-cicd}

### 3.1 Why GitHub Actions? When to use it?

#### **Question 16: Why would you choose GitHub Actions over other CI/CD tools?**

**Advantages:**
- ✅ **Native GitHub integration** - PR checks, branch protection, code owners
- ✅ **Tight PR workflow** - terraform plan shows in PR comments automatically
- ✅ **Marketplace** - 15K+ pre-built actions (azure/login, hashicorp/setup-terraform, etc.)
- ✅ **Matrix builds** - Test across multiple OS/versions in parallel
- ✅ **OIDC support** - No long-lived service principal secrets
- ✅ **Free tier** - 2000 minutes/month for public repos
- ✅ **Reusable workflows** - DRY principle for infra + app CI/CD

**When NOT to use:**
- ❌ Very complex orchestration (consider **Azure DevOps Pipelines** instead)
- ❌ Need enterprise support SLA (consider Jenkins, GitLab CI)

---

#### **Question 17: How do you structure workflows for infra vs app deployments?**

**Infrastructure Workflow (terraform):**
```
PR → terraform plan → post comment → review → merge → terraform apply → deployed
```

**Application Workflow (build + deploy):**
```
PR → build → lint → test → build docker image → push registry → merge → deploy to app service
```

**Key Difference:**
- **Infra:** Human review of `terraform plan` output before apply
- **App:** Automated tests, security scans, automatic deploy (can use CD pattern)

---

#### **Question 18: How do you manage secrets and environments in GitHub Actions?**

**Secrets Hierarchy:**
- **Repo-level secrets** - Available to all workflows in repo
- **Environment-level secrets** - Specific to dev/test/prod environment
- **Organization-level secrets** - Shared across all repos (if on Pro/Enterprise)

**Example: Prod environment setup:**
```yaml
name: Deploy to Prod

on:
  push:
    branches: [main]

env:
  AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: prod  # Requires manual approval from CODEOWNERS
    steps:
      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}        # Prod-specific
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}        # Prod-specific
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy
        run: terraform apply -auto-approve
```

**Protection Rules for Prod:**
- Require approval from code owners
- Required reviewers: 2+ from DevOps team
- Dismiss on new push
- Restrict who can approve (only senior engineers)

---

### 3.2 CI/CD for Infrastructure (Terraform) – Sample Pipeline

#### **Question 19: How would you design a GitHub Actions pipeline to deploy Azure infra using Terraform?**

**Sample Workflow: `infra-ci-cd.yml`**

```yaml
name: Infrastructure CI/CD

on:
  push:
    branches:
      - main
    paths:
      - "infra/**"
      - ".github/workflows/infra-ci-cd.yml"
  pull_request:
    branches:
      - main
    paths:
      - "infra/**"

env:
  TERRAFORM_VERSION: 1.8.0

jobs:
  #====== PLAN JOB (runs on PR + push) ======
  terraform-plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infra
    
    permissions:
      contents: read
      pull-requests: write
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TERRAFORM_VERSION }}

      - name: Azure CLI Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform Init
        run: terraform init

      - name: Terraform Validate
        run: terraform validate -no-color

      - name: Terraform Plan
        id: plan
        run: |
          terraform plan -out=tfplan -no-color
          terraform show -no-color tfplan > plan.txt
        continue-on-error: true

      - name: Post Plan to PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('infra/plan.txt', 'utf8');
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## 📋 Terraform Plan\n\`\`\`\n${plan}\n\`\`\``
            });

      - name: Upload Plan Artifact
        uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: infra/tfplan
          retention-days: 5

      - name: Comment PR on Failure
        if: steps.plan.outcome == 'failure' && github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '❌ Terraform Plan failed. Check logs for details.'
            });

  #====== APPLY JOB (runs only on main push + approval) ======
  terraform-apply:
    name: Terraform Apply
    needs: terraform-plan
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    environment:
      name: production
      url: https://portal.azure.com
    
    defaults:
      run:
        working-directory: infra
    
    permissions:
      contents: read
      pull-requests: write

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TERRAFORM_VERSION }}

      - name: Download Plan
        uses: actions/download-artifact@v4
        with:
          name: tfplan
          path: infra

      - name: Azure CLI Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform Init
        run: terraform init

      - name: Terraform Apply
        run: terraform apply -auto-approve tfplan

      - name: Post Apply Status
        if: always()
        uses: actions/github-script@v7
        with:
          script: |
            const status = '${{ job.status }}' === 'success' ? '✅' : '❌';
            console.log(`${status} Infrastructure deployment completed`);
```

---

#### **Question 20: How do you separate plan and apply stages with approvals?**

**Key Pattern:**

1. **Plan stage** - Runs on all PRs and pushes (read-only, no modifications)
2. **Apply stage** - Runs only on main , requires **environment approval**
3. **GitHub Protection Rules** - Set review requirements before apply

**In workflow:**
```yaml
terraform-apply:
  needs: terraform-plan          # Must pass plan first
  if: github.ref == 'refs/heads/main'  # Only on main
  environment: production        # Requires approval
```

**In Repository Settings:**
- Go to **Settings → Environments → production**
- Add **Reviewers** (code owners)
- Enable **Required reviewers** (at least 2)
- Enable **Dismiss stale reviews**

---

### 3.3 CI/CD for Application Deployment – Sample Pipeline

#### **Question 21: How do you deploy an application to Azure (App Service/AKS) using GitHub Actions?**

**Sample Workflow - Node.js to App Service:**

```yaml
name: App CI/CD

on:
  push:
    branches: [main]
    paths:
      - "app/**"
      - ".github/workflows/app-ci-cd.yml"
  pull_request:
    branches: [main]
    paths:
      - "app/**"

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: app
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Install Dependencies
        run: npm ci

      - name: Run Tests
        run: npm test

      - name: Lint
        run: npm run lint

      - name: Build
        run: npm run build

      - name: Upload Build Artifact
        uses: actions/upload-artifact@v4
        with:
          name: build
          path: app/dist

  deploy:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    environment: production
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Download Build Artifact
        uses: actions/download-artifact@v4
        with:
          name: build
          path: build

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to App Service
        uses: azure/webapps-deploy@v3
        with:
          app-name: "my-prod-webapp"
          package: "./build"

      - name: Deployment Verification
        run: |
          az webapp show --name my-prod-webapp --resource-group rg-prod --query "{state:state, hostNames:hostNames}"
```

---

#### **Question 22: How do you handle blue/green or rolling deployments?**

**Blue/Green Strategy:**

```yaml
deploy:
  steps:
    # Deploy to "green" slot (inactive)
    - name: Deploy to Staging Slot
      uses: azure/webapps-deploy@v3
      with:
        app-name: my-prod-webapp
        slot-name: staging
        package: ./build

    # Run smoke tests on green
    - name: Smoke Tests
      run: |
        curl -f https://my-prod-webapp-staging.azurewebsites.net/health
        echo "✅ Green slot healthy"

    # Swap green → blue (instant cutover)
    - name: Swap Slots
      run: |
        az webapp deployment slot swap \
          --name my-prod-webapp \
          --resource-group rg-prod \
          --slot staging
        echo "✅ Swapped to production"
```

**Advantages:**
- ✅ Zero downtime
- ✅ Rollback in seconds (just swap back)
- ✅ Test before production traffic

---

## Troubleshooting GitHub Actions Failures {#troubleshooting-failures}

### Question 23: A GitHub Actions pipeline suddenly fails—how do you approach troubleshooting?

**Systematic Troubleshooting Approach:**

**Step 1 - Identify Failing Job/Step:**
- ❌ Check job run logs in GitHub Actions UI
- Look for **red X** on failing step
- Read error message carefully

**Step 2 - Classify the Issue:**

| Category | Signs | Fix |
|----------|-------|-----|
| **Auth Issues** | `401`, `403`, `Access Denied`, `Unauthorized` | ✓ Update secrets, verify OIDC setup, check RBAC |
| **Terraform Issues** | `terraform: not found`, plan/apply errors, syntax error | ✓ Run `terraform init -upgrade`, fix HCL, check provider |
| **Code Issues** | Build fails, tests fail, lint errors | ✓ Run locally, fix bugs, update dependencies |
| **Environment Issues** | `npm: not found`, Python version mismatch, disk full | ✓ Install tools, update runtime version, check limits |
| **Timeout** | Job cancelled after 6 hours | ✓ Optimize job, split into parallel jobs |

---

### Question 24: What are common failure causes in infra pipelines vs app pipelines?

**Infrastructure (Terraform) Failures:**

1. **Azure Auth Failure:**
   ```
   Error: Failed to get bearer token (OIDC error)
   ```
   Fix: Verify GitHub OIDC is configured, RBAC on subscription

2. **Backend Initialization Failure:**
   ```
   Error: Failed to read state file
   ```
   Fix: Check storage account access, state file locking issue

3. **Provider Version Mismatch:**
   ```
   Error: Unsupported Terraform version
   ```
   Fix: Run `terraform init -upgrade` in workflow

4. **Resource Already Exists:**
   ```
   Error: Storage Account 'xxx' already exists
   ```
   Fix: Check for manual creation, import into state: `terraform import`

5. **Insufficient Permissions:**
   ```
   Error: Insufficient privileges to complete the operation
   ```
   Fix: Add required Azure RBAC roles to service principal

---

**Application (Build/Deploy) Failures:**

1. **Dependency Resolution:**
   ```
   npm ERR! Cannot find module 'express'
   ```
   Fix: Run `npm ci` (clean install), check package.json, update package lock

2. **Test Failures:**
   ```
   ❌ 5 tests failed
   ```
   Fix: Run tests locally, debug, commit fix

3. **Build Optimization:**
   ```
   FATAL ERROR: CALL_AND_RETRY_LAST Allocation failed
   ```
   Fix: Increase Node memory: `NODE_OPTIONS: --max-old-space-size=4096`

4. **Deployment Failure:**
   ```
   Error: Failed to deploy to App Service
   ```
   Fix: Check app service is running, logs: `az webapp log tail`

5. **Secrets Not Found:**
   ```
   Error: CLIENT_ID not in environment
   ```
   Fix: Verify GitHub secrets exist, correct secret names

---

**Troubleshooting Commands:**

```bash
# Run terraform locally to replicate issue
terraform init
terraform plan

# Check storage lock
az storage blob lease break --account-name tfstate --container tfstate --blob prod.tfstate

# Monitor App Service
az webapp log tail --name myapp --resource-group mygroup

# Check service principal permissions
az role assignment list --assignee <PRINCIPAL_ID>
```

---

## Terraform State - Deep Dive {#terraform-state-deepdive}

### Question 25: Manual resource deletion – how to handle?

**Scenario:** Someone deleted a resource in Azure portal, but Terraform still thinks it exists.

**What happens next:**
```
terraform plan
→ Resource not found in Azure
→ Terraform shows: "destroy and recreate"
```

**Decision:**
1. **Option A - Let Terraform recreate it (Simple):**
   ```bash
   terraform apply
   # Resource recreated automatically
   ```

2. **Option B - Import manually (if resource existed before):**
   ```bash
   # Remove from state
   terraform state rm azurerm_storage_account.example

   # Re-import from Azure
   terraform import azurerm_storage_account.example /subscriptions/xxx/resourceGroups/rg1/providers/Microsoft.Storage/storageAccounts/mysa

   # Verify state restored
   terraform plan  # Should show no changes
   ```

**Prevention:**
- ✅ Only modify infrastructure via Terraform
- ✅ Use **Azure Policy** to prevent manual deletions
- ✅ Audit logs → alerts when manual changes detected

---

### Question 26: Moving resources between states (split monolith into modules)

**Scenario:** You have one large `main.tf`, want to split into reusable modules.

**Migration Steps:**

1. **Create module:**
   ```bash
   mkdir -p modules/storage
   # Move code to modules/storage/main.tf, variables.tf, outputs.tf
   ```

2. **Move state:**
   ```bash
   # Old location: root_module.azurerm_storage_account.mysa
   # New location: module.storage.azurerm_storage_account.mysa

   terraform state mv \
     'azurerm_storage_account.mysa' \
     'module.storage.azurerm_storage_account.mysa'
   ```

3. **Verify:**
   ```bash
   terraform plan  # Should show no changes
   ```

**Safe Pattern:**
```bash
# Dry-run first
terraform state mv -dry-run 'old.path' 'new.path'

# Then actual
terraform state mv 'old.path' 'new.path'

# Verify
terraform state list
```

---

### Question 27: Preventing state corruption from multiple engineers

**The Problem:** Engineer A running `apply`, Engineer B runs `apply` simultaneously → **state corruption**.

**Solution - Remote State + Locking:**

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "tfstateprod123"
    container_name       = "tfstate"
    key                  = "prod.tfstate"
  }
}
```

**How locking works:**
1. Engineer A runs `terraform apply` → **acquires lock**
2. Engineer B runs `terraform apply` → **blocked, waits for lock**
3. Engineer A finishes → **releases lock**
4. Engineer B's `apply` → **proceeds**

**Enforcement - Only Allow CI/CD:**

```yaml
# .gitignore
.terraform/
terraform.tfstate
terraform.tfstate.backup
```

**Policy in GitHub:**
- Require PR reviews (no direct pushes)
- All deploys via GitHub Actions (not local machines)
- Only main branch can trigger `apply`

**Team Guidelines:**
```
✅ DO: terraform plan locally, commit code, push PR
✅ DO: Wait for CI/CD to run terraform apply
❌ DON'T: Run terraform apply on your machine
❌ DON'T: Bypass CI/CD and apply directly
```

---

## Interview Cheat Sheet

### Quick Recall Items:

| Topic | Key Takeaway |
|-------|--------------|
| **JIT VM Access** | Most secure, auto time-based expiry |
| **Bastion** | SSH/RDP over TLS, no public IP |
| **NSG** | L3/L4 filtering, per-subnet |
| **Service Tags** | Use instead of IPs in NSG rules |
| **Availability Zones** | Multi-zone for 99.99% SLA |
| **Terraform Module** | Reusable code blocks for multi-env |
| **Remote State** | Azure Storage backend + locking |
| **OIDC** | No long-lived secrets, federated auth |
| **Blue/Green Deploy** | Zero-downtime, instant rollback |
| **terraform import** | Bring manual resources into state |
| **terraform state mv** | Move resources between modules/states |
| **terraform force-unlock** | Recover from stuck locks |

---

## Resources & Links

- [Azure Terraform Registry](https://registry.terraform.io/providers/hashicorp/azurerm/latest)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Terraform State Management](https://developer.hashicorp.com/terraform/language/state)
- [Azure Security Best Practices](https://learn.microsoft.com/en-us/azure/security/)

---

**Last Updated:** April 2026  
**Version:** 1.0

