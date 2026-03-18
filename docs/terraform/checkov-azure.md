# Checkov on Azure — Getting to 0 Failed Checks

> A practical guide to fixing the most common Checkov failures in Azure Terraform code, with every fix explained.

## What Checkov Checks and Why It Matters

Checkov is an open-source static analysis tool by Bridgecrew (now Prisma Cloud / Palo Alto Networks) that scans Infrastructure as Code for security misconfigurations. It checks your Terraform code **before deployment**, catching problems that would otherwise become security findings in production.

### What Checkov catches

- Missing encryption settings (at rest, in transit)
- Public network exposure
- Missing logging and monitoring
- Overly permissive access controls
- Missing identity configurations
- Non-compliant resource configurations

### Why it matters for Azure

Azure services default to "open" in many cases. For example:

- Storage Accounts default to allowing public blob access
- SQL Servers default to allowing Azure-internal firewall rules
- Key Vaults default to access policies (not RBAC)
- App Services do not enforce HTTPS by default in Terraform

Checkov enforces the secure defaults that Azure does not.

---

## Installation and Basic Usage

### Installation

```bash
# In a Python virtual environment (recommended)
python3 -m venv .venv
source .venv/bin/activate
pip install checkov

# Verify
checkov --version
```

### Basic usage

```bash
# Scan current directory
checkov --directory .

# Scan with skips from .checkov.yaml
checkov --directory . --config-file .checkov.yaml

# Scan specific file
checkov --file main.tf

# Output as JSON (for CI/CD)
checkov --directory . --output json

# Compact output (fewer lines)
checkov --directory . --compact
```

### Understanding the output

```
Passed checks: 42
Failed checks: 3
Skipped checks: 2

Check: CKV_AZURE_35: "Ensure default network access rule for Storage Accounts is deny"
        FAILED for resource: azurerm_storage_account.main
        File: /storage.tf:1-15
        Guide: https://docs.prismacloud.io/en/enterprise-edition/policy-reference/...
```

Each failed check shows:

- **Check ID**: the unique identifier (e.g., `CKV_AZURE_35`)
- **Description**: what the check verifies
- **Resource**: which Terraform resource failed
- **File/lines**: where in your code
- **Guide**: link to the detailed explanation

---

## Top 10 Most Common Azure Terraform Failures

### 1. CKV_AZURE_35 — Storage Account: default network access not deny

**What it checks**: the Storage Account's default network access rule should be `Deny`, not `Allow`.

**Why it matters**: with the default `Allow`, anyone on the internet can attempt to access the storage account (subject to authentication). Setting it to `Deny` blocks all traffic except explicitly allowed networks.

**Failing code**:

```hcl
resource "azurerm_storage_account" "main" {
  name                     = "stmyappprdweu"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  # Missing: network_rules
}
```

**Fixed code**:

```hcl
resource "azurerm_storage_account" "main" {
  name                     = "stmyappprdweu"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "LRS"

  network_rules {
    default_action = "Deny"
    bypass         = ["AzureServices"]
    ip_rules       = []   # Add your allowed IPs here
  }
}
```

---

### 2. CKV_AZURE_59 — Storage Account: public access disabled

**What it checks**: `allow_nested_items_to_be_public` (formerly `allow_blob_public_access`) should be `false`.

**Why it matters**: when enabled, individual containers can be set to public access, exposing data to the internet without authentication.

**Failing code**:

```hcl
resource "azurerm_storage_account" "main" {
  # ... base config ...
  # Missing: allow_nested_items_to_be_public
}
```

**Fixed code**:

```hcl
resource "azurerm_storage_account" "main" {
  # ... base config ...
  allow_nested_items_to_be_public = false
}
```

---

### 3. CKV_AZURE_33 — Storage Account: queue service logging

**What it checks**: Storage Queue service should have logging enabled for read, write, and delete operations.

**Why it matters**: without logging, you have no audit trail for queue operations. Required for MCSB LT-3 (logging for investigation).

**Failing code**:

```hcl
resource "azurerm_storage_account" "main" {
  # ... base config ...
  # Missing: queue_properties logging
}
```

**Fixed code**:

```hcl
resource "azurerm_storage_account" "main" {
  # ... base config ...

  queue_properties {
    logging {
      delete                = true
      read                  = true
      write                 = true
      version               = "1.0"
      retention_policy_days = 90
    }
  }
}
```

> **Note**: If you are using AVM module `Azure/avm-res-storage-storageaccount/azurerm`, queue logging is configured through the module's diagnostic settings variable rather than inline.

---

### 4. CKV_AZURE_190 — Key Vault: purge protection

**What it checks**: Key Vault should have `purge_protection_enabled = true`.

**Why it matters**: without purge protection, a deleted Key Vault (and all its secrets, keys, certificates) can be permanently destroyed before the soft-delete retention period expires. This is a data loss risk and a compliance requirement for MCSB DP-8.

**Failing code**:

```hcl
resource "azurerm_key_vault" "main" {
  name                = "kv-myapp-prd-weu"
  # ... base config ...
  purge_protection_enabled = false  # or missing
}
```

**Fixed code**:

```hcl
resource "azurerm_key_vault" "main" {
  name                       = "kv-myapp-prd-weu"
  # ... base config ...
  purge_protection_enabled   = true
  soft_delete_retention_days = 90
}
```

> **Warning**: once purge protection is enabled, it cannot be disabled. This is by design. Plan accordingly for dev/test environments where you may want to recreate Key Vaults frequently.

---

### 5. CKV_AZURE_109 — Key Vault: RBAC authorization

**What it checks**: Key Vault should use `enable_rbac_authorization = true` instead of access policies.

**Why it matters**: RBAC authorization integrates with Azure's centralized identity model (Entra ID). Access policies are Key Vault-specific and harder to audit. MCSB IM-1 requires centralized identity management.

**Failing code**:

```hcl
resource "azurerm_key_vault" "main" {
  name                = "kv-myapp-prd-weu"
  # ... base config ...
  # Missing: enable_rbac_authorization (defaults to false)

  access_policy {
    tenant_id = data.azurerm_client_config.current.tenant_id
    object_id = "..."
    secret_permissions = ["Get", "List"]
  }
}
```

**Fixed code**:

```hcl
resource "azurerm_key_vault" "main" {
  name                       = "kv-myapp-prd-weu"
  # ... base config ...
  enable_rbac_authorization  = true
  # No access_policy blocks — use azurerm_role_assignment instead
}

resource "azurerm_role_assignment" "kv_secrets_reader" {
  scope                = azurerm_key_vault.main.id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = azurerm_user_assigned_identity.main.principal_id
}
```

---

### 6. CKV_AZURE_23 — SQL Server: auditing enabled

**What it checks**: SQL Server should have auditing enabled to a Storage Account or Log Analytics Workspace.

**Why it matters**: without audit logs, you cannot investigate security incidents or prove compliance. Required by MCSB LT-3, NIS2 Art. 21(2)(b), and DORA Art. 13.

**Failing code**:

```hcl
resource "azurerm_mssql_server" "main" {
  name                         = "sql-myapp-prd-weu"
  resource_group_name          = azurerm_resource_group.main.name
  location                     = azurerm_resource_group.main.location
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = random_password.sql.result
  # Missing: auditing policy
}
```

**Fixed code**:

```hcl
resource "azurerm_mssql_server" "main" {
  name                         = "sql-myapp-prd-weu"
  resource_group_name          = azurerm_resource_group.main.name
  location                     = azurerm_resource_group.main.location
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = random_password.sql.result
}

resource "azurerm_mssql_server_extended_auditing_policy" "main" {
  server_id              = azurerm_mssql_server.main.id
  log_monitoring_enabled = true
  # Sends audit logs to Log Analytics via diagnostic settings
}

# Or to a Storage Account:
resource "azurerm_mssql_server_extended_auditing_policy" "main" {
  server_id                  = azurerm_mssql_server.main.id
  storage_endpoint           = azurerm_storage_account.audit.primary_blob_endpoint
  storage_account_access_key = azurerm_storage_account.audit.primary_access_key
  retention_in_days          = 90
}
```

---

### 7. CKV_AZURE_24 — SQL Server: threat detection

**What it checks**: SQL Server should have Advanced Threat Protection (ATP) enabled.

**Why it matters**: ATP detects anomalous database activities like SQL injection, brute-force attacks, and data exfiltration. Required by MCSB LT-1 and NIS2 Art. 21(2)(b).

**Failing code**:

```hcl
resource "azurerm_mssql_server" "main" {
  # ... base config ...
  # Missing: threat detection policy
}
```

**Fixed code**:

```hcl
resource "azurerm_mssql_server_security_alert_policy" "main" {
  resource_group_name = azurerm_resource_group.main.name
  server_name         = azurerm_mssql_server.main.name
  state               = "Enabled"
  email_addresses     = [var.security_contact_email]
  retention_days      = 90
}
```

---

### 8. CKV_AZURE_47 — PostgreSQL: SSL enforcement

**What it checks**: PostgreSQL Flexible Server should enforce SSL connections.

**Why it matters**: without SSL enforcement, database connections can be intercepted (man-in-the-middle). Required by MCSB DP-3 (encryption in transit).

**Failing code**:

```hcl
resource "azurerm_postgresql_flexible_server" "main" {
  name                = "psql-myapp-prd-weu"
  # ... base config ...
  # Missing: SSL configuration
}
```

**Fixed code**:

```hcl
resource "azurerm_postgresql_flexible_server" "main" {
  name                = "psql-myapp-prd-weu"
  # ... base config ...
}

resource "azurerm_postgresql_flexible_server_configuration" "require_ssl" {
  server_id = azurerm_postgresql_flexible_server.main.id
  name      = "require_secure_transport"
  value     = "on"
}

resource "azurerm_postgresql_flexible_server_configuration" "tls_version" {
  server_id = azurerm_postgresql_flexible_server.main.id
  name      = "ssl_min_protocol_version"
  value     = "TLSv1.2"
}
```

---

### 9. CKV_TF_1 — Modules should use Git source with commit hash

**What it checks**: all Terraform module sources should use a Git URL with a pinned commit hash (`?ref=abc123`), not a registry source.

**Why it matters for most modules**: Git-pinned sources are immutable. Registry sources can theoretically be republished.

**Why we skip it for AVM**: Azure Verified Modules are published exclusively on the Terraform Registry, not as Git repositories. They are signed by HashiCorp and versioned. Requiring Git sources would mean not using AVM at all.

**The code that fails**:

```hcl
module "key_vault" {
  source  = "Azure/avm-res-keyvault-vault/azurerm"  # Registry source
  version = "~> 0.10"
  # ...
}
```

**Skip with justification** in `.checkov.yaml`:

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

### 10. CKV_AZURE_225 / CKV_AZURE_211 / CKV_AZURE_212 — App Service checks

These three checks apply to App Service and are often skipped in SMB contexts with documented justification.

**CKV_AZURE_225** — App Service should use a minimum TLS version of 1.3

- TLS 1.3 is not yet universally supported by all clients (older browsers, legacy integrations)
- TLS 1.2 remains the industry standard minimum
- For SMB deployments, TLS 1.2 is acceptable

**CKV_AZURE_211** — App Service should have detailed error messages disabled

- In development and staging, detailed errors are essential for debugging
- For SMB deployments without a dedicated security team, the trade-off is acceptable

**CKV_AZURE_212** — App Service should have failed request tracing enabled

- This check may conflict with data privacy requirements (RGPD Art. 5 — data minimization)
- For SMB deployments, enabling it selectively is more appropriate

**Skip with justification**:

```yaml
skip-check:
  - id: CKV_AZURE_225
    justification: >
      the infrastructure generator: TLS 1.2 enforced as minimum (industry standard).
      TLS 1.3 not universally supported by all client applications.
      SMB context — upgrade path documented in README.
  - id: CKV_AZURE_211
    justification: >
      the infrastructure generator: Detailed errors required for SMB teams without
      dedicated monitoring infrastructure. Disabled in premium tier.
  - id: CKV_AZURE_212
    justification: >
      the infrastructure generator: Failed request tracing configured via Application
      Insights diagnostic settings, not via App Service built-in.
      RGPD Art. 5 data minimization applies.
```

---

## The .checkov.yaml File

### Format

```yaml
# .checkov.yaml — Checkov configuration for Terraform stacks
# Every skip MUST include a justification. No silent skips.

framework:
  - terraform

compact: true
directory:
  - .

skip-check:
  - id: CKV_TF_1
    justification: >
      the infrastructure generator: AVM modules are sourced from Terraform Registry
      (registry.terraform.io/Azure), not Git repositories.
      Version pinning with ~> ensures reproducible builds.
      Registry modules are signed and verified by HashiCorp.
```

### Rules for skip justifications

1. **Every skip must have a justification** — no exceptions
2. **Prefix with "the infrastructure generator:"** — makes it clear this is a deliberate decision
3. **Explain the technical reason** — why the check does not apply or is mitigated
4. **State the alternative** — what compensating control is in place
5. **Never skip silently** — an unexplained skip is a security debt

### Common legitimate skips

| Check ID | Resource | Justification Pattern |
|----------|----------|--------------------|
| CKV_TF_1 | All AVM modules | Registry source with version pin |
| CKV_AZURE_225 | App Service | TLS 1.2 is sufficient for SMB |
| CKV_AZURE_211 | App Service | Needed for debugging in SMB context |
| CKV_AZURE_212 | App Service | Tracing via App Insights instead |
| CKV_AZURE_24 | SQL (brownfield) | DB admin pack — requires real Azure target |
| CKV_AZURE_23 | SQL (brownfield) | DB admin pack — requires real Azure target |

---

## CI/CD Integration

### GitHub Actions

```yaml
name: Checkov Security Scan

on:
  pull_request:
    paths:
      - '**.tf'
      - '.checkov.yaml'

jobs:
  checkov:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install Checkov
        run: pip install checkov

      - name: Run Checkov
        run: |
          checkov \
            --directory . \
            --config-file .checkov.yaml \
            --output cli \
            --output junitxml \
            --output-file-path console,results.xml \
            --compact

      - name: Upload results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: checkov-results
          path: results.xml
```

### Pre-commit hook

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/bridgecrewio/checkov
    rev: '3.2.0'
    hooks:
      - id: checkov
        args: ['--config-file', '.checkov.yaml', '--compact']
```

### Expected CI output

A passing Checkov run looks like this:

```
Passed checks: 47, Failed checks: 0, Skipped checks: 3

Skipped checks:
  CKV_TF_1: the infrastructure generator: AVM modules from Terraform Registry
  CKV_AZURE_225: the infrastructure generator: TLS 1.2 enforced as minimum
  CKV_AZURE_211: the infrastructure generator: Detailed errors for SMB debugging
```

The target is always **0 failed checks**. Skipped checks are acceptable when justified.

---

## Checkov Quick Reference Card

| Action | Command |
|--------|---------|
| Scan current directory | `checkov -d .` |
| Scan with config file | `checkov -d . --config-file .checkov.yaml` |
| Scan one file | `checkov -f main.tf` |
| List all Azure checks | `checkov --list --framework terraform --filter CKV_AZURE` |
| Run one specific check | `checkov -d . --check CKV_AZURE_35` |
| Skip specific checks inline | `checkov -d . --skip-check CKV_TF_1,CKV_AZURE_225` |
| JSON output | `checkov -d . --output json` |
| SARIF output (for GitHub) | `checkov -d . --output sarif` |
| Quiet mode (failures only) | `checkov -d . --quiet` |

---

## Further Reading

- [Checkov documentation](https://www.checkov.io/1.Welcome/What%20is%20Checkov.html)
- [Checkov Azure policy index](https://www.checkov.io/5.Policy%20Index/terraform.html)
- [MCSB v1 — Security benchmark](https://learn.microsoft.com/en-us/security/benchmark/azure/overview)
- [Bridgecrew by Prisma Cloud](https://www.prismacloud.io/cloud-code-security)

---

*Maintained by [Frederic Leroy](https://github.com/f-leroy) — MCT, Azure Solutions Architect Expert*
