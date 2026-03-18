# AzureRM Provider v4 — What Changed

> A comprehensive guide to the breaking changes in AzureRM v4, how to migrate, and what to expect next.

## Overview

The AzureRM Terraform provider v4.0 was released in September 2024. It is a major version bump with breaking changes that affect nearly every Azure Terraform codebase. This guide covers every significant change, with before/after code examples.

### Timeline

| Event | Date |
|-------|------|
| v4.0.0 GA release | September 2024 |
| v3.x last release | September 2024 (maintenance mode) |
| v3.x end of support | Expected mid-2025 |
| v5.x | No announced date (see "v5 Outlook" section) |

### How to upgrade

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.14"  # Use latest 4.x
    }
  }
}

provider "azurerm" {
  features {}
  resource_provider_registrations = "none"
}
```

Then run:

```bash
terraform init -upgrade
terraform plan  # Review ALL changes before applying
```

---

## Key Breaking Changes

### 1. azurerm_sql_* removed — use azurerm_mssql_*

The old `azurerm_sql_*` resources (deprecated since v3.x) are completely removed in v4. All SQL Database resources now use the `azurerm_mssql_*` prefix.

| Removed (v3) | Replacement (v4) |
|--------------|-----------------|
| `azurerm_sql_server` | `azurerm_mssql_server` |
| `azurerm_sql_database` | `azurerm_mssql_database` |
| `azurerm_sql_firewall_rule` | `azurerm_mssql_firewall_rule` |
| `azurerm_sql_virtual_network_rule` | `azurerm_mssql_virtual_network_rule` |
| `azurerm_sql_elasticpool` | `azurerm_mssql_elasticpool` |
| `azurerm_sql_failover_group` | `azurerm_mssql_failover_group` |
| `azurerm_sql_active_directory_administrator` | Inline in `azurerm_mssql_server` |

**Before (v3)**:

```hcl
resource "azurerm_sql_server" "main" {
  name                         = "sql-myapp-prd"
  resource_group_name          = azurerm_resource_group.main.name
  location                     = azurerm_resource_group.main.location
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = var.sql_password
}

resource "azurerm_sql_database" "main" {
  name                = "db-myapp"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  server_name         = azurerm_sql_server.main.name
  edition             = "Standard"
  requested_service_objective_name = "S0"
}
```

**After (v4)**:

```hcl
resource "azurerm_mssql_server" "main" {
  name                         = "sql-myapp-prd"
  resource_group_name          = azurerm_resource_group.main.name
  location                     = azurerm_resource_group.main.location
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = var.sql_password

  minimum_tls_version = "1.2"
}

resource "azurerm_mssql_database" "main" {
  name      = "db-myapp"
  server_id = azurerm_mssql_server.main.id
  sku_name  = "S0"
}
```

Key differences in the `mssql` variants:

- `azurerm_mssql_database` uses `server_id` instead of `server_name` + `resource_group_name` + `location`
- SKU is specified as `sku_name` instead of `edition` + `requested_service_objective_name`
- `minimum_tls_version` is a direct attribute on the server

**Migration path**: use `terraform state mv` to move existing resources:

```bash
terraform state mv azurerm_sql_server.main azurerm_mssql_server.main
terraform state mv azurerm_sql_database.main azurerm_mssql_database.main
```

---

### 2. resource_provider_registrations replaces skip_provider_registration

The `skip_provider_registration` boolean is removed. It is replaced by `resource_provider_registrations` which gives more granular control.

**Before (v3)**:

```hcl
provider "azurerm" {
  features {}
  skip_provider_registration = true
}
```

**After (v4)**:

```hcl
provider "azurerm" {
  features {}
  resource_provider_registrations = "none"  # Equivalent to skip = true
}
```

Available values:

| Value | Behavior | Use Case |
|-------|----------|----------|
| `"all"` | Registers all known resource providers (default in v3) | Full admin permissions |
| `"core"` | Registers only providers needed by your config | Recommended for most cases |
| `"extended"` | Registers core + commonly used providers | Broad deployments |
| `"none"` | Registers nothing (equivalent to old `skip = true`) | CI/CD with pre-registered providers |

**Recommendation**: use `"none"` in CI/CD pipelines and `"core"` for interactive use:

```hcl
provider "azurerm" {
  features {}
  resource_provider_registrations = "none"
}
```

---

### 3. enable_* renamed to *_enabled

In v4, several boolean properties were renamed from `enable_X` to `X_enabled` for consistency with the Azure API naming conventions.

| Before (v3) | After (v4) | Resource |
|------------|-----------|----------|
| `enable_https_traffic_only` | `https_traffic_only_enabled` | `azurerm_storage_account` |
| `enable_non_ssl_port` | `non_ssl_port_enabled` | `azurerm_redis_cache` |
| `enable_blob_encryption` | Removed (always true in v4) | `azurerm_storage_account` |
| `enable_file_encryption` | Removed (always true in v4) | `azurerm_storage_account` |
| `enable_automatic_failover` | `automatic_failover_enabled` | `azurerm_cosmosdb_account` |
| `enable_free_tier` | `free_tier_enabled` | `azurerm_cosmosdb_account` |
| `enable_multiple_write_locations` | `multiple_write_locations_enabled` | `azurerm_cosmosdb_account` |
| `enable_disk_encryption` | `disk_encryption_enabled` | `azurerm_kusto_cluster` |
| `enable_purge` | `purge_enabled` | `azurerm_kusto_cluster` |
| `enable_streaming_ingest` | `streaming_ingest_enabled` | `azurerm_kusto_cluster` |

**Before (v3)**:

```hcl
resource "azurerm_storage_account" "main" {
  name                      = "stmyappprdweu"
  resource_group_name       = azurerm_resource_group.main.name
  location                  = azurerm_resource_group.main.location
  account_tier              = "Standard"
  account_replication_type  = "LRS"
  enable_https_traffic_only = true
}
```

**After (v4)**:

```hcl
resource "azurerm_storage_account" "main" {
  name                          = "stmyappprdweu"
  resource_group_name           = azurerm_resource_group.main.name
  location                      = azurerm_resource_group.main.location
  account_tier                  = "Standard"
  account_replication_type      = "LRS"
  https_traffic_only_enabled    = true  # Renamed
}
```

> **Note**: some `enable_*` properties were removed entirely because they are now always enabled (like blob and file encryption). If your code sets these, simply remove the lines.

---

### 4. Provider-defined functions

AzureRM v4 introduces **provider-defined functions** that can be called in Terraform configurations. These are available with Terraform 1.8+ and provide utility operations that were previously impossible or required workarounds.

#### normalise_resource_id

Normalizes Azure resource IDs to a consistent casing format. Azure resource IDs are case-insensitive in the API but case-sensitive in Terraform state, which causes drift issues.

```hcl
locals {
  # Normalize a resource ID to prevent case-sensitivity drift
  normalized_id = provider::azurerm::normalise_resource_id(
    "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/MyRG/providers/Microsoft.Compute/virtualMachines/myVM"
  )
  # Result: /subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/MyRG/providers/Microsoft.Compute/virtualMachines/myVM
}
```

**Use case**: when importing existing resources or referencing resources created outside Terraform, the ID casing may not match what the provider expects. This function prevents false drift detection.

#### parse_resource_id

Parses an Azure resource ID into its component parts:

```hcl
locals {
  parsed = provider::azurerm::parse_resource_id(
    "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/myVNet"
  )
  # parsed.subscription_id = "00000000-0000-0000-0000-000000000000"
  # parsed.resource_group_name = "myRG"
  # parsed.resource_type = "Microsoft.Network/virtualNetworks"
  # parsed.resource_name = "myVNet"
}
```

**Use case**: extract components from resource IDs without string manipulation (`split`, `element` hacks).

#### Requires Terraform 1.8+

Provider-defined functions require Terraform 1.8 or later:

```hcl
terraform {
  required_version = ">= 1.8"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.14"
    }
  }
}
```

---

### 5. Other notable changes

#### features block simplification

Several `features` sub-blocks were removed or simplified:

**Before (v3)** — common pattern to prevent accidental deletion:

```hcl
provider "azurerm" {
  features {
    key_vault {
      purge_soft_delete_on_destroy    = false
      recover_soft_deleted_key_vaults = true
    }
    resource_group {
      prevent_deletion_if_contains_resources = true
    }
  }
}
```

**After (v4)** — some of these are now defaults or removed:

```hcl
provider "azurerm" {
  features {}
  resource_provider_registrations = "none"
}
```

Check the [v4 upgrade guide](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/guides/4.0-upgrade-guide) for the full list of changed `features` blocks.

#### azurerm_virtual_machine deprecated

The old `azurerm_virtual_machine` resource (which handled both Linux and Windows VMs) is fully deprecated. Use the specific resources:

| Deprecated | Replacement |
|-----------|-------------|
| `azurerm_virtual_machine` (Linux) | `azurerm_linux_virtual_machine` |
| `azurerm_virtual_machine` (Windows) | `azurerm_windows_virtual_machine` |

#### Container Registry: admin_enabled removed

The `admin_enabled` attribute on `azurerm_container_registry` now defaults to `false` and emits a warning if set to `true`. This aligns with the security best practice of using Managed Identity or service principals instead of admin credentials.

#### Storage Account: min_tls_version default

In v4, `min_tls_version` defaults to `TLS1_2` (it was `TLS1_0` in early v3). This is a non-breaking change that improves security posture by default.

#### Cosmos DB account: consistency_policy required

The `consistency_policy` block is now required on `azurerm_cosmosdb_account`. Previously it defaulted to `Session`. You must now explicitly set it:

```hcl
resource "azurerm_cosmosdb_account" "main" {
  # ...
  consistency_policy {
    consistency_level = "Session"
  }
}
```

---

## Required Provider Block — Complete Example

```hcl
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
```

---

## Migration Checklist

Use this checklist when upgrading from v3 to v4:

| Step | Action | Status |
|:---:|--------|:---:|
| 1 | Update provider version constraint to `~> 4.0` | |
| 2 | Replace `skip_provider_registration` with `resource_provider_registrations` | |
| 3 | Replace all `azurerm_sql_*` with `azurerm_mssql_*` | |
| 4 | Rename `enable_*` properties to `*_enabled` | |
| 5 | Remove `enable_blob_encryption` and `enable_file_encryption` (always true) | |
| 6 | Review `features {}` block for removed sub-blocks | |
| 7 | Replace `azurerm_virtual_machine` with Linux/Windows specific resources | |
| 8 | Add explicit `consistency_policy` to Cosmos DB accounts | |
| 9 | Run `terraform init -upgrade` | |
| 10 | Run `terraform plan` and review every change | |
| 11 | Use `terraform state mv` for renamed resources | |
| 12 | Run Checkov to verify security posture | |
| 13 | Test in staging before production | |

### Automated search for affected code

```bash
# Find all deprecated azurerm_sql_* resources
grep -rn "azurerm_sql_" --include="*.tf" .

# Find enable_* properties that need renaming
grep -rn "enable_https_traffic_only\|enable_non_ssl_port\|enable_automatic_failover\|enable_free_tier" --include="*.tf" .

# Find skip_provider_registration
grep -rn "skip_provider_registration" --include="*.tf" .

# Find old azurerm_virtual_machine
grep -rn "azurerm_virtual_machine\"" --include="*.tf" . | grep -v "linux\|windows\|scale_set"
```

---

## v5 Outlook

As of early 2026, there is **no announced release date for AzureRM v5**. However, the HashiCorp team has indicated that breaking changes are accumulating for the next major version. Here is what is known and expected:

### Confirmed or strongly expected changes

| Change | Status | Impact |
|--------|--------|--------|
| Remove remaining deprecated resources | Expected | Medium |
| Default `public_network_access_enabled = false` | Discussed | High |
| Remove `features` block entirely | Discussed | Low |
| More provider-defined functions | In progress | Low (additive) |
| Require Terraform 1.9+ | Expected | Low |

### What this means for your code

1. **Pin to `~> 4.0`** — this protects you from v5 breaking changes when it drops
2. **Stop using deprecated resources now** — they will be removed in v5
3. **Set `public_network_access_enabled = false` explicitly** — if it becomes the default in v5, your code is already compliant
4. **Use AVM modules** — AVM modules are updated for each major provider version, so they absorb the migration work for you

### Tracking v5

- [AzureRM provider changelog](https://github.com/hashicorp/terraform-provider-azurerm/blob/main/CHANGELOG.md)
- [AzureRM v5.0 milestone](https://github.com/hashicorp/terraform-provider-azurerm/milestone/232) (if created)
- [HashiCorp Discuss — Azure provider](https://discuss.hashicorp.com/c/terraform-providers/tf-azure/33)

---

## Common Migration Errors and Fixes

### Error: "skip_provider_registration" is not a valid argument

```
Error: Unsupported argument
  on main.tf line 5, in provider "azurerm":
   5:   skip_provider_registration = true
```

**Fix**: replace with `resource_provider_registrations = "none"`.

### Error: "azurerm_sql_server" resource type not found

```
Error: Invalid resource type
  on database.tf line 1:
   1: resource "azurerm_sql_server" "main" {
```

**Fix**: replace with `azurerm_mssql_server` and update all attribute names (see section above).

### Error: "enable_https_traffic_only" is not expected here

```
Error: Unsupported argument
  on storage.tf line 8:
   8:   enable_https_traffic_only = true
```

**Fix**: rename to `https_traffic_only_enabled = true`.

### State drift after upgrade

If `terraform plan` shows resources being destroyed and recreated after the upgrade:

1. Check if the resource type was renamed (use `terraform state mv`)
2. Check if default values changed (explicitly set the old default)
3. Check if attribute names changed (update your code to match)

```bash
# Move state for renamed resources
terraform state mv azurerm_sql_server.main azurerm_mssql_server.main

# List all resources in state
terraform state list

# Show a specific resource's state
terraform state show azurerm_mssql_server.main
```

---

## Further Reading

- [AzureRM v4 upgrade guide (official)](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/guides/4.0-upgrade-guide)
- [AzureRM provider changelog](https://github.com/hashicorp/terraform-provider-azurerm/blob/main/CHANGELOG.md)
- [Terraform provider-defined functions](https://developer.hashicorp.com/terraform/plugin/framework/functions)
- [AzureRM provider GitHub](https://github.com/hashicorp/terraform-provider-azurerm)

---

*Maintained by [Frederic Leroy](https://github.com/f-leroy) — MCT, Azure Solutions Architect Expert*
