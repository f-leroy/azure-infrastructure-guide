# App Service + Database Architecture on Azure

**Version**: 1.0
**Last updated**: 2026-03-18
**Category**: Workload / Web Application
**Cloud Adoption Framework alignment**: Application Platform

---

## Overview

The App Service + Database pattern is the most widely deployed architecture on Azure. It serves web applications, APIs, and backend services backed by a managed relational database. This architecture scales from a developer proof-of-concept to a fully secured enterprise deployment by layering network isolation, identity-based authentication, and centralized monitoring.

This document covers three deployment tiers -- Basic, Standard, and Premium -- each progressively adding security and resilience controls without changing the core application topology.

### When to use this architecture

- Web applications and REST APIs with a relational database backend.
- Internal line-of-business applications.
- SaaS products that need to scale compute independently from data.
- Workloads that do not require container orchestration (for containers, see the AKS Platform Architecture).

### Supported database engines

| Engine | Azure Service | Use case |
|---|---|---|
| PostgreSQL | Azure Database for PostgreSQL Flexible Server | Open-source applications, Django, Rails, Node.js |
| MySQL | Azure Database for MySQL Flexible Server | WordPress, PHP applications, legacy migrations |
| SQL Server | Azure SQL Database | .NET applications, enterprise workloads, SSRS/SSIS |

The architecture is identical regardless of the database engine. Only the Terraform module and connection string format change.

---

## Architecture Diagram

### Standard Tier (Production)

```
                    Internet
                       |
              +--------+--------+
              |   App Service   |
              |   (VNet         |
              |   integrated)   |
              +--------+--------+
                       |
            VNet Integration Subnet
              10.0.1.0/24
                       |
          +────────────+────────────+
          |            |            |
    +-----+-----+ +---+---+ +-----+-----+
    | Private    | | Private| | Private   |
    | Endpoint   | | Endpoint | Endpoint  |
    | PostgreSQL | | Key    | | Storage   |
    |            | | Vault  | | Account   |
    +-----+------+ +---+---+ +-----+-----+
          |            |            |
    PE Subnet 10.0.2.0/24
          |            |            |
    +-----+------+ +---+---+ +-----+-----+
    | PostgreSQL | | Key   | | Storage   |
    | Flexible   | | Vault | | Account   |
    | Server     | |       | |           |
    +------------+ +-------+ +-----------+

    +----------------+    +------------------+
    | Log Analytics  |    | Application      |
    | Workspace      |    | Insights         |
    +----------------+    +------------------+

    +----------------+
    | Managed        |
    | Identity       |
    | (User-Assigned)|
    +----------------+
```

---

## Components

### App Service Plan + App Service

The App Service Plan defines the compute tier. The App Service runs the application code.

| Property | Basic tier | Standard tier | Premium tier |
|---|---|---|---|
| Plan SKU | B1 | P1v3 | P2v3 |
| VNet integration | No | Yes | Yes |
| Custom domain | Optional | Yes | Yes |
| Deployment slots | No | 1 staging slot | 2+ slots |
| Auto-scale | No | Rule-based | Rule-based |
| Always On | No | Yes | Yes |
| Minimum TLS | 1.2 | 1.2 | 1.2 |

Key configuration:

```hcl
# Always enforced regardless of tier
https_only              = true
minimum_tls_version     = "1.2"
ftps_state              = "Disabled"
remote_debugging_enabled = false
```

### Database (PostgreSQL Flexible Server example)

| Property | Basic tier | Standard tier | Premium tier |
|---|---|---|---|
| SKU | Burstable B1ms | General Purpose D2ds_v5 | General Purpose D4ds_v5 |
| Storage | 32 GB | 128 GB | 256 GB |
| High availability | Disabled | Same-zone | Zone-redundant |
| Backup retention | 7 days | 14 days | 35 days |
| PITR | Yes | Yes | Yes |
| Public access | Enabled | Disabled | Disabled |
| Private Endpoint | No | Yes | Yes |
| SSL enforcement | Enabled | Enabled | Enabled |
| Microsoft Entra auth | No | Yes (via MI) | Yes (via MI) |

### Key Vault

Stores application secrets, database connection strings (in Basic tier), and TLS certificates.

| Property | All tiers |
|---|---|
| SKU | Standard |
| Soft delete | Enabled (90 days) |
| Purge protection | Enabled |
| Network ACLs | Allow all (Basic) / Private Endpoint only (Standard/Premium) |
| RBAC authorization | Enabled (no access policies) |

In Standard and Premium tiers, the application authenticates to Key Vault using its Managed Identity. No secrets are stored in app settings directly.

### Managed Identity (User-Assigned)

A single User-Assigned Managed Identity is created and assigned to the App Service. This identity is granted:

- **Key Vault Secrets User** role on the Key Vault.
- **Database authentication** via Microsoft Entra ID (PostgreSQL and SQL support this natively).
- **Storage Blob Data Contributor** on the Storage Account (if applicable).

This eliminates all credential management. No passwords in app settings, no connection string secrets in environment variables.

### Log Analytics Workspace + Application Insights

| Component | Purpose |
|---|---|
| Log Analytics Workspace | Central log sink for all resource diagnostic settings |
| Application Insights | APM for the App Service (request rates, failures, dependencies, custom telemetry) |

Application Insights is configured with workspace-based mode (not classic) and uses Managed Identity for authentication (not instrumentation key alone).

Diagnostic settings are configured on every resource:

- App Service: HTTP logs, app logs, platform logs.
- Database: Query performance, audit logs, connection logs.
- Key Vault: Audit events.
- Network: NSG flow logs (Standard/Premium).

### Private Endpoints

In Standard and Premium tiers, all data-plane access to PaaS services goes through Private Endpoints:

| Service | Private DNS Zone |
|---|---|
| PostgreSQL Flexible Server | `privatelink.postgres.database.azure.com` |
| MySQL Flexible Server | `privatelink.mysql.database.azure.com` |
| Azure SQL Database | `privatelink.database.windows.net` |
| Key Vault | `privatelink.vaultcore.azure.net` |
| Storage Account (blob) | `privatelink.blob.core.windows.net` |

Public access is disabled on all these services when Private Endpoints are in use.

### Virtual Network (Standard/Premium only)

| Subnet | CIDR | Purpose |
|---|---|---|
| App Integration | /24 | App Service VNet integration (delegated to Microsoft.Web/serverFarms) |
| Private Endpoints | /24 | PE NICs for database, Key Vault, Storage |
| Firewall (Premium only) | /26 | Azure Firewall |
| Bastion (Premium only) | /26 | Azure Bastion |

### Front Door + WAF (Premium only)

Azure Front Door provides global load balancing, SSL offloading, and Web Application Firewall (WAF) protection.

| Property | Premium tier |
|---|---|
| Front Door SKU | Standard |
| WAF policy | Prevention mode |
| Managed ruleset | Microsoft Default Rule Set 2.1 |
| Custom rules | Rate limiting, geo-filtering |
| Origin | App Service (private link origin) |
| Health probes | HTTPS, 30-second interval |

The App Service restricts inbound traffic to Front Door only using the `X-Azure-FDID` header validation and IP restriction to Front Door service tag.

### Backup Vault (Premium only)

| Property | Premium tier |
|---|---|
| Vault type | Backup Vault |
| Redundancy | Geo-redundant (GRS) |
| Blob backup | Operational + Vaulted |
| Soft delete | Enabled (14 days) |

---

## Three Tiers Compared

| Capability | Basic | Standard | Premium |
|---|---|---|---|
| **Target** | Dev/POC | Production | Enterprise/Regulated |
| **Public access** | Yes | No (PE only) | No (PE + Firewall) |
| **VNet** | None | Yes | Yes |
| **Private Endpoints** | None | All data services | All data services |
| **Azure Firewall** | None | None | Yes |
| **Azure Bastion** | None | None | Yes |
| **Front Door + WAF** | None | None | Yes |
| **Backup Vault** | None | None | Yes |
| **Database HA** | None | Same-zone | Zone-redundant |
| **Monitoring** | Log Analytics + App Insights | Full diagnostics | Full + extended retention |
| **Log retention** | 30 days | 90 days | 365 days |
| **Identity auth** | Connection string | Managed Identity | Managed Identity |
| **Budget alerts** | Yes | Yes | Yes |

---

## Security Considerations

### MCSB Control Mapping

| Control | Implementation | Tiers |
|---|---|---|
| NS-1 (Network segmentation) | VNet with dedicated subnets, NSG per subnet | Standard, Premium |
| NS-2 (Secure cloud services) | Private Endpoints on all PaaS, public access disabled | Standard, Premium |
| NS-3 (Edge firewall) | Azure Firewall with deny-by-default outbound rules | Premium |
| NS-6 (WAF) | Front Door WAF in Prevention mode | Premium |
| IM-3 (Application identities) | User-Assigned Managed Identity, no stored credentials | Standard, Premium |
| DP-3 (Encryption in transit) | TLS 1.2 enforced on App Service and database | All |
| DP-4 (Encryption at rest) | Platform-managed keys on database and storage | All |
| DP-6 (Secure key management) | Key Vault with RBAC, purge protection | All |
| DP-8 (Key repository security) | Soft delete 90 days, purge protection enabled | All |
| LT-3 (Security logging) | Diagnostic settings on all resources to Log Analytics | All |
| LT-5 (Centralized log analysis) | Single Log Analytics Workspace | All |
| BR-1 (Automated backup) | Database PITR + Backup Vault for blobs | All (Vault: Premium) |

### RGPD Alignment

| Article | Implementation |
|---|---|
| Art. 25 (Privacy by design) | Private Endpoints, encryption at rest and in transit, identity-based auth |
| Art. 32 (Security of processing) | TLS 1.2, Key Vault, Managed Identity, NSG, backup |
| Art. 33 (Breach notification) | Application Insights alerts, diagnostic settings, budget anomaly alerts |

### NIS2 Alignment

| Measure | Implementation |
|---|---|
| Art. 21(2)(b) Incident management | Log Analytics + Application Insights alerting |
| Art. 21(2)(c) Continuity and backup | Database PITR, Backup Vault (Premium), zone-redundant HA |
| Art. 21(2)(e) Secure development | Infrastructure as Code, deterministic generation, checkov validation |
| Art. 21(2)(h) Encryption | TLS 1.2 in transit, AES-256 at rest, Key Vault for secrets |
| Art. 21(2)(i) Access control | Managed Identity, RBAC, no stored credentials |
| Art. 21(2)(j) MFA and secure communications | Entra ID authentication, HTTPS-only |

### Application Security Checklist

- [ ] `WEBSITE_RUN_FROM_PACKAGE = 1` to make the filesystem read-only.
- [ ] FTPS disabled, remote debugging disabled.
- [ ] Health check endpoint configured for App Service health monitoring.
- [ ] Connection strings reference Key Vault, not hardcoded values.
- [ ] Database firewall denies all public access (Standard/Premium).
- [ ] App Service access restrictions allow only Front Door (Premium) or specific IPs.

---

## Terraform File Structure

```
appservice-database/
  main.tf                 # terraform block, providers, locals (naming, tags)
  resource_group.tf       # Resource group
  networking.tf           # VNet, subnets, NSGs, Private Endpoints, Private DNS Zones
  app.tf                  # App Service Plan, App Service, deployment slots
  database.tf             # PostgreSQL/MySQL/SQL Flexible Server
  keyvault.tf             # Key Vault, secrets, access configuration
  identity.tf             # User-Assigned Managed Identity, role assignments
  monitoring.tf           # Log Analytics, Application Insights, diagnostic settings
  frontdoor.tf            # Front Door + WAF policy (Premium only, count = 0 otherwise)
  security.tf             # Azure Firewall + Bastion (Premium only, count = 0 otherwise)
  backup.tf               # Backup Vault + policies (Premium only, count = 0 otherwise)
  finops.tf               # Budget alerts, auto-shutdown, storage lifecycle
  variables.tf            # All input variables with validation
  outputs.tf              # App URL, database FQDN, Key Vault URI, MI client ID
  terraform.tfvars        # Values for the selected tier
  backend.tf.example      # Remote state backend template
  .checkov.yaml           # Security exceptions with documented justifications
  README.md               # Stack documentation with deployment instructions
  COMPLIANCE.md           # Full compliance mapping
  manifest.yaml           # Stack metadata
```

### Tier selection in Terraform

A single `var.tier` variable controls resource provisioning:

```hcl
variable "tier" {
  type        = string
  description = "Deployment tier: basic, standard, or premium"
  validation {
    condition     = contains(["basic", "standard", "premium"], var.tier)
    error_message = "Tier must be basic, standard, or premium."
  }
}

locals {
  is_standard = var.tier == "standard" || var.tier == "premium"
  is_premium  = var.tier == "premium"
}
```

Resources that only exist in certain tiers use `count`:

```hcl
# Private Endpoints only in standard and premium
module "pe_database" {
  count  = local.is_standard ? 1 : 0
  source = "Azure/avm-res-network-privateendpoint/azurerm"
  # ...
}

# Firewall only in premium
module "firewall" {
  count  = local.is_premium ? 1 : 0
  source = "Azure/avm-res-network-azurefirewall/azurerm"
  # ...
}
```

---

## Cost Estimates

Estimates for West Europe region, single instance, as of early 2026.

### Basic Tier (~EUR 80-150/month)

| Component | Monthly Cost |
|---|---|
| App Service Plan (B1) | ~EUR 50 |
| PostgreSQL (Burstable B1ms, 32 GB) | ~EUR 25 |
| Key Vault (Standard, low operations) | ~EUR 1 |
| Log Analytics (1 GB/day) | ~EUR 20 |
| Application Insights (included in LA) | EUR 0 |
| Storage Account (LRS, minimal) | ~EUR 2 |
| **Total** | **~EUR 100/month** |

### Standard Tier (~EUR 400-700/month)

| Component | Monthly Cost |
|---|---|
| App Service Plan (P1v3) | ~EUR 130 |
| PostgreSQL (D2ds_v5, 128 GB, same-zone HA) | ~EUR 200 |
| Key Vault (Standard) | ~EUR 3 |
| Log Analytics (5 GB/day) | ~EUR 100 |
| Application Insights | EUR 0 (workspace-based) |
| VNet + Private Endpoints (3 PEs) | ~EUR 25 |
| Storage Account (LRS) | ~EUR 5 |
| Managed Identity | EUR 0 |
| **Total** | **~EUR 465/month** |

### Premium Tier (~EUR 2,000-4,000/month)

| Component | Monthly Cost |
|---|---|
| App Service Plan (P2v3) | ~EUR 260 |
| PostgreSQL (D4ds_v5, 256 GB, zone-redundant HA) | ~EUR 500 |
| Azure Firewall (Standard) | ~EUR 900 |
| Front Door (Standard) + WAF | ~EUR 250 |
| Bastion (Standard) | ~EUR 330 |
| Key Vault (Standard) | ~EUR 5 |
| Log Analytics (10 GB/day) | ~EUR 150 |
| Backup Vault + policies | ~EUR 30 |
| VNet + Private Endpoints (5 PEs) | ~EUR 35 |
| Storage Account (GRS) | ~EUR 15 |
| **Total** | **~EUR 2,475/month** |

> **Note**: The Premium tier cost is dominated by Azure Firewall and Bastion. In a hub-spoke architecture, these components are shared across all workloads and their cost is amortized. The per-workload cost in an existing hub-spoke environment drops to approximately EUR 1,100/month.

---

## Data Flow

### Request Flow (Standard Tier)

```
Client → HTTPS → App Service (VNet integrated)
  → Managed Identity auth → PostgreSQL (via Private Endpoint)
  → Managed Identity auth → Key Vault (via Private Endpoint)
  → Managed Identity auth → Storage (via Private Endpoint)
```

### Request Flow (Premium Tier)

```
Client → HTTPS → Front Door (WAF inspection)
  → Private Link → App Service (VNet integrated)
    → Managed Identity auth → PostgreSQL (via Private Endpoint)
    → Managed Identity auth → Key Vault (via Private Endpoint)
    → Azure Firewall → External API calls (outbound inspection)
```

### Monitoring Flow

```
App Service → Application Insights SDK → Log Analytics Workspace
All resources → Diagnostic Settings → Log Analytics Workspace
Budget → Cost alerts → Action Group → Email/Webhook
```

---

## References

- [Azure App Service documentation](https://learn.microsoft.com/en-us/azure/app-service/)
- [Azure Database for PostgreSQL Flexible Server](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/)
- [App Service networking features](https://learn.microsoft.com/en-us/azure/app-service/networking-features)
- [Tutorial: Connect to Azure databases from App Service with Managed Identity](https://learn.microsoft.com/en-us/azure/app-service/tutorial-connect-msi-azure-database)
- [Microsoft Cloud Security Benchmark v1](https://learn.microsoft.com/en-us/security/benchmark/azure/overview)

---

*Maintained by [Frederic Leroy](https://github.com/f-leroy) -- MCT, Azure Solutions Architect Expert*
