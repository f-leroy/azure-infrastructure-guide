# Azure Verified Modules (AVM) — What You Need to Know

> A practical guide to using Microsoft's official Terraform modules for Azure infrastructure.

## What is AVM?

Azure Verified Modules (AVM) is Microsoft's initiative to provide **official, maintained, and tested Terraform modules** for Azure resources. Launched in 2023 and reaching maturity in 2024-2025, AVM replaces the fragmented ecosystem of community modules with a single, standardized set of modules that follow Microsoft's own best practices.

### Why Microsoft created AVM

Before AVM, the Azure Terraform ecosystem suffered from:

- **Fragmentation**: dozens of community modules for the same resource, each with different interfaces
- **Security gaps**: community modules rarely enforced security defaults (TLS 1.2, encryption at rest, diagnostic settings)
- **Maintenance burden**: modules abandoned when maintainers moved on
- **Inconsistency**: no standard naming for inputs/outputs across modules

AVM solves these problems by providing modules that are:

| Property | Detail |
|----------|--------|
| **Owned by Microsoft** | Maintained by Azure product teams and Microsoft employees |
| **Security-first** | TLS 1.2, encryption at rest, and diagnostic settings are defaults |
| **Tested** | Automated integration tests run against real Azure subscriptions |
| **Versioned** | Semantic versioning with changelogs |
| **Standardized** | Consistent interface patterns across all modules |
| **Published on Terraform Registry** | Easy to discover and consume |

### AVM module types

AVM publishes two kinds of modules:

- **Resource modules** (`avm-res-*`): wrap a single Azure resource with security defaults. Example: `avm-res-keyvault-vault`.
- **Pattern modules** (`avm-ptn-*`): compose multiple resource modules into an architecture. Example: `avm-ptn-hubnetworking`.

For most production work, **resource modules** give you the right balance of control and safety.

---

## How to Find AVM Modules

### Terraform Registry

All AVM modules live under the `Azure` namespace on the Terraform Registry:

```
https://registry.terraform.io/namespaces/Azure
```

Search for modules using the pattern:

```
Azure/avm-res-{resource-provider}-{resource-type}/azurerm
```

Examples:

| Azure Resource | AVM Module Name |
|----------------|----------------|
| Key Vault | `Azure/avm-res-keyvault-vault/azurerm` |
| Virtual Network | `Azure/avm-res-network-virtualnetwork/azurerm` |
| PostgreSQL Flexible | `Azure/avm-res-dbforpostgresql-flexibleserver/azurerm` |
| Storage Account | `Azure/avm-res-storage-storageaccount/azurerm` |
| App Service | `Azure/avm-res-web-site/azurerm` |

### GitHub repositories

Each module has a corresponding GitHub repository:

```
https://github.com/Azure/terraform-azurerm-avm-res-keyvault-vault
```

The GitHub repo contains:

- Full source code
- Examples in `/examples/`
- Integration tests in `/tests/`
- Changelog and version tags

### Quick discovery commands

```bash
# Search Terraform Registry for AVM resource modules
terraform-mcp-server search_modules "Azure avm-res"

# Get details for a specific module
terraform-mcp-server get_module_details "Azure/avm-res-keyvault-vault/azurerm"
```

---

## Key Gotchas

### 1. Always disable telemetry

Every AVM module includes a `enable_telemetry` variable that defaults to `true`. This sends usage data to Microsoft. **Always set it to `false`** for production use:

```hcl
module "key_vault" {
  source  = "Azure/avm-res-keyvault-vault/azurerm"
  version = "~> 0.10"

  enable_telemetry = false  # ALWAYS

  name                = "kv-myapp-prd-weu"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  tenant_id           = data.azurerm_client_config.current.tenant_id
}
```

### 2. parent_id vs resource_group_name — the critical difference

This is **the most confusing part of AVM**. Different modules use different patterns for specifying where a resource lives. There is no consistency, and getting this wrong produces cryptic errors.

| Module | Uses `parent_id` | Uses `resource_group_name` | Notes |
|--------|:-:|:-:|-------|
| **virtual-network** | Yes | No | `parent_id` = resource group ID |
| **bastion** | Yes | No | `parent_id` = resource group ID |
| **nat-gateway** | Yes | No | `parent_id` = resource group ID |
| **log-analytics** | No | Yes | Standard `resource_group_name` |
| **firewall** | No | Yes | Uses `resource_group_name` + `firewall_` prefix on all inputs |
| **key-vault** | No | Yes | Standard `resource_group_name` |
| **storage-account** | No | Yes | Standard `resource_group_name` |
| **postgresql-flexible** | No | Yes | Standard `resource_group_name` |

#### Example: virtual-network uses parent_id

```hcl
module "vnet" {
  source  = "Azure/avm-res-network-virtualnetwork/azurerm"
  version = "~> 0.8"

  enable_telemetry = false

  name      = "vnet-myapp-prd-weu"
  location  = azurerm_resource_group.main.location

  # parent_id = the resource group ID, NOT the name
  resource_group_id = azurerm_resource_group.main.id

  address_space = ["10.0.0.0/16"]
}
```

#### Example: log-analytics uses resource_group_name

```hcl
module "log_analytics" {
  source  = "Azure/avm-res-operationalinsights-workspace/azurerm"
  version = "~> 0.4"

  enable_telemetry = false

  name                = "log-myapp-prd-weu"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name  # name, NOT id
}
```

#### Example: firewall uses resource_group_name + firewall_ prefix

```hcl
module "firewall" {
  source  = "Azure/avm-res-network-azurefirewall/azurerm"
  version = "~> 0.3"

  enable_telemetry = false

  # All inputs prefixed with firewall_
  firewall_name               = "fw-myapp-prd-weu"
  resource_group_name         = azurerm_resource_group.main.name
  firewall_sku_name           = "AZFW_VNet"
  firewall_sku_tier           = "Standard"
  firewall_policy_id          = azurerm_firewall_policy.main.id
}
```

**Rule of thumb**: always check the module's `variables.tf` on GitHub before using it. Never assume the interface matches another AVM module.

### 3. The azapi provider is always required

AVM modules internally use the `azapi` provider for certain operations. Even if your code does not directly reference `azapi`, you must declare it:

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.14"
    }
    azapi = {
      source  = "azure/azapi"
      version = "~> 2.4"  # ALWAYS required for AVM
    }
  }
}
```

Failing to include `azapi` results in errors like:

```
Error: Missing required provider
  provider registry.terraform.io/azure/azapi was not found
```

### 4. random provider for Key Vault and Storage Account

Key Vault names must be globally unique (3-24 characters). Storage Account names must be globally unique (3-24 characters, lowercase alphanumeric only). Use the `random` provider to generate suffixes:

```hcl
terraform {
  required_providers {
    # ... azurerm, azapi ...
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"  # Required for KV and Storage
    }
  }
}

resource "random_string" "suffix" {
  length  = 4
  special = false
  upper   = false
}

# Storage Account: no hyphens, max 24 chars
locals {
  storage_name = "st${local.name_slug}${random_string.suffix.result}"
}
```

---

## Version Pinning Best Practices

### Use pessimistic constraint operator

Always pin AVM modules with `~>` (pessimistic constraint):

```hcl
module "key_vault" {
  source  = "Azure/avm-res-keyvault-vault/azurerm"
  version = "~> 0.10"   # Allows 0.10.x but not 0.11.0
  # ...
}
```

| Constraint | Meaning | Recommended? |
|-----------|---------|:---:|
| `"0.10.2"` | Exact version only | No — misses patches |
| `"~> 0.10"` | `>= 0.10.0` and `< 0.11.0` | Yes |
| `">= 0.10"` | Any version from 0.10 up | No — breaks on major changes |

### Why ~> is the right choice for AVM

AVM modules follow semver. During the `0.x` phase (most modules today):

- **Patch** (`0.10.1` to `0.10.2`): bug fixes only
- **Minor** (`0.10.x` to `0.11.0`): may include breaking changes

Using `~> 0.10` gives you bug fixes while protecting against breaking changes.

### Lock file discipline

Always commit `.terraform.lock.hcl` to your repository:

```bash
terraform init -upgrade    # Update lock file
git add .terraform.lock.hcl
git commit -m "chore: update terraform lock file"
```

---

## Checkov and AVM: Skip CKV_TF_1

Checkov check `CKV_TF_1` requires that all modules be sourced from Git with a commit hash. AVM modules are published on the Terraform Registry, not Git. This check will always fail for AVM modules.

**Skip it with a documented justification** in `.checkov.yaml`:

```yaml
skip-check:
  - id: CKV_TF_1
    justification: >
      the infrastructure generator: AVM modules are sourced from Terraform Registry
      (registry.terraform.io/Azure), not Git repositories.
      Version pinning with ~> ensures reproducible builds.
      Registry modules are signed and verified by HashiCorp.
```

---

## Custom Module vs AVM Module

### Lines of code comparison

| Resource | Custom Module | AVM Module | Savings |
|----------|:---:|:---:|:---:|
| Key Vault with RBAC, purge protection, diagnostics | ~120 lines | ~15 lines | 87% |
| VNet with subnets, NSGs, service endpoints | ~200 lines | ~30 lines | 85% |
| PostgreSQL Flexible with HA, backup, TLS | ~180 lines | ~25 lines | 86% |
| Storage Account with encryption, lifecycle, PE | ~150 lines | ~20 lines | 87% |
| App Service with slots, identity, TLS | ~160 lines | ~25 lines | 84% |

### Security defaults comparison

| Security Feature | Custom Module | AVM Module |
|-----------------|:---:|:---:|
| TLS 1.2 minimum | Must configure | Default |
| Encryption at rest | Must configure | Default |
| Diagnostic settings | Must add | Built-in variable |
| Managed Identity support | Must add | Built-in variable |
| Network ACLs | Must add | Built-in variable |
| Purge protection (KV) | Must configure | Default |
| HTTPS only (Storage) | Must configure | Default |

### When to use custom resources instead of AVM

- The AVM module does not exist yet for your resource type
- You need a very thin wrapper with minimal inputs
- The AVM module has a bug that blocks your use case (file an issue first)
- You need behavior that the AVM module explicitly does not support

---

## Most Common AVM Modules Reference

The table below lists the most frequently used AVM resource modules. Always verify the latest version on the Terraform Registry before using.

| Module | Source | Version | Category |
|--------|--------|---------|----------|
| Resource Group | `Azure/avm-res-resources-resourcegroup/azurerm` | `~> 0.2` | Foundation |
| Virtual Network | `Azure/avm-res-network-virtualnetwork/azurerm` | `~> 0.8` | Networking |
| NSG | `Azure/avm-res-network-networksecuritygroup/azurerm` | `~> 0.5` | Networking |
| Private Endpoint | `Azure/avm-res-network-privateendpoint/azurerm` | `~> 0.2` | Networking |
| Private DNS Zone | `Azure/avm-res-network-privatednszone/azurerm` | `~> 0.7` | Networking |
| NAT Gateway | `Azure/avm-res-network-natgateway/azurerm` | `~> 0.4` | Networking |
| Public IP | `Azure/avm-res-network-publicipaddress/azurerm` | `~> 0.2` | Networking |
| Route Table | `Azure/avm-res-network-routetable/azurerm` | `~> 0.5` | Networking |
| Azure Firewall | `Azure/avm-res-network-azurefirewall/azurerm` | `~> 0.3` | Security |
| Bastion | `Azure/avm-res-network-bastionhost/azurerm` | `~> 0.5` | Security |
| Key Vault | `Azure/avm-res-keyvault-vault/azurerm` | `~> 0.10` | Security |
| Managed Identity | `Azure/avm-res-managedidentity-userassignedidentity/azurerm` | `~> 0.4` | Identity |
| Log Analytics | `Azure/avm-res-operationalinsights-workspace/azurerm` | `~> 0.4` | Monitoring |
| Application Insights | `Azure/avm-res-insights-component/azurerm` | `~> 0.2` | Monitoring |
| Storage Account | `Azure/avm-res-storage-storageaccount/azurerm` | `~> 0.6` | Storage |
| App Service Plan | `Azure/avm-res-web-serverfarm/azurerm` | `~> 0.4` | Compute |
| App Service | `Azure/avm-res-web-site/azurerm` | `~> 0.15` | Compute |
| PostgreSQL Flexible | `Azure/avm-res-dbforpostgresql-flexibleserver/azurerm` | `~> 0.4` | Database |
| MySQL Flexible | `Azure/avm-res-dbformysql-flexibleserver/azurerm` | `~> 0.5` | Database |
| SQL Database | `Azure/avm-res-sql-server/azurerm` | `~> 0.2` | Database |
| Front Door | `Azure/avm-res-cdn-profile/azurerm` | `~> 0.7` | CDN/WAF |
| Backup Vault | `Azure/avm-res-dataprotection-backupvault/azurerm` | `~> 0.3` | Backup |
| Virtual Machine | `Azure/avm-res-compute-virtualmachine/azurerm` | `~> 0.18` | Compute |
| Container Registry | `Azure/avm-res-containerregistry-registry/azurerm` | `~> 0.6` | Containers |
| AKS | `Azure/avm-res-containerservice-managedcluster/azurerm` | `~> 0.4` | Containers |

> **Note**: versions listed are approximate as of early 2026. Always check the Terraform Registry or use the Terraform Registry MCP server to verify the latest version before deployment.

---

## Complete Example: Minimal Production Stack

```hcl
# main.tf
terraform {
  required_version = ">= 1.9"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.14"
    }
    azapi = {
      source  = "azure/azapi"
      version = "~> 2.4"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}

provider "azurerm" {
  features {}
  resource_provider_registrations = "none"
}

provider "azapi" {}

locals {
  name_prefix = lower(join("-", compact([var.project, var.environment, var.region])))
  common_tags = {
    environment = var.environment
    project     = var.project
    managed_by  = "terraform"
    iac_source  = "terraform"
  }
}

# resource_group.tf
module "resource_group" {
  source  = "Azure/avm-res-resources-resourcegroup/azurerm"
  version = "~> 0.2"

  enable_telemetry = false

  name     = "rg-${local.name_prefix}"
  location = var.region
  tags     = local.common_tags
}

# keyvault.tf
resource "random_string" "kv_suffix" {
  length  = 4
  special = false
  upper   = false
}

module "key_vault" {
  source  = "Azure/avm-res-keyvault-vault/azurerm"
  version = "~> 0.10"

  enable_telemetry = false

  name                          = "kv-${local.name_prefix}-${random_string.kv_suffix.result}"
  resource_group_name           = module.resource_group.name
  location                      = module.resource_group.resource_group_location
  tenant_id                     = data.azurerm_client_config.current.tenant_id
  purge_protection_enabled      = true
  soft_delete_retention_days    = 90
  enable_rbac_authorization     = true

  tags = local.common_tags
}

# monitoring.tf
module "log_analytics" {
  source  = "Azure/avm-res-operationalinsights-workspace/azurerm"
  version = "~> 0.4"

  enable_telemetry = false

  name                = "log-${local.name_prefix}"
  resource_group_name = module.resource_group.name
  location            = module.resource_group.resource_group_location
  retention_in_days   = 90

  tags = local.common_tags
}
```

---

## Further Reading

- [AVM official site](https://azure.github.io/Azure-Verified-Modules/)
- [AVM module index](https://azure.github.io/Azure-Verified-Modules/indexes/terraform/)
- [Terraform Registry — Azure namespace](https://registry.terraform.io/namespaces/Azure)
- [AVM contribution guide](https://azure.github.io/Azure-Verified-Modules/contributing/)

---

*Maintained by [Frederic Leroy](https://github.com/f-leroy) — MCT, Azure Solutions Architect Expert*
