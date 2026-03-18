# Azure Private Endpoints — The Complete Terraform Guide

> Everything you need to deploy Private Endpoints on Azure with Terraform, from architecture to DNS to cost.

## Why Private Endpoints Matter

### The compliance case

Private Endpoints are not optional in regulated environments. They are directly referenced by multiple compliance frameworks:

| Framework | Control | Requirement |
|-----------|---------|-------------|
| **MCSB v1** | NS-2 | "Secure cloud services with network controls" — Private Endpoints are the primary mechanism |
| **NIS2** | Art. 21(2)(e) | "Security in network and information systems acquisition, development and maintenance" |
| **NIS2** | Art. 21(2)(h) | "Policies and procedures regarding the use of cryptography and, where appropriate, encryption" |
| **DORA** | Art. 9 | "Protection and prevention" — network isolation for financial services |
| **RGPD** | Art. 32 | "Security of processing" — appropriate technical measures |

### The security case

Without Private Endpoints, PaaS services (databases, storage, key vaults) are accessed over the public internet, even from your own VNet. This means:

- Traffic leaves the Microsoft backbone and traverses public networks
- The service has a public IP visible to attackers
- Network-level attacks (DDoS, MitM) are possible
- Compliance auditors will flag it immediately

With Private Endpoints:

- Traffic stays on the Microsoft backbone via Azure Private Link
- The service gets a private IP in your VNet (e.g., `10.0.3.4`)
- No public IP exposure (when combined with `public_network_access_enabled = false`)
- NSG rules can restrict access to the private IP
- DNS resolution returns the private IP instead of the public IP

---

## Architecture

### Three components work together

A Private Endpoint deployment requires three resources that must be correctly wired:

```
                                        +-----------------------+
                                        |   Azure PaaS Service  |
                                        |   (e.g., PostgreSQL)  |
                                        |                       |
                                        |  public_network_access|
                                        |  = false              |
                                        +----------+------------+
                                                   |
                                          Private Link
                                                   |
+------------------+    +------------------+    +--+---------------+
|   Your VNet      |    | Private DNS Zone |    | Private Endpoint |
|                  |    |                  |    |                  |
|  +-----------+   |    | A record:        |    | NIC with         |
|  | Subnet    |   |    | psql-myapp...    |    | private IP       |
|  | (pe-snet) +---|--->| -> 10.0.3.4      |    | 10.0.3.4         |
|  +-----------+   |    |                  |    +------------------+
|                  |    +--------+---------+
|  +-----------+   |             |
|  | App Subnet|   |    VNet Link (auto-registration)
|  | 10.0.1.x  +---+-------------+
|  +-----------+   |
+------------------+
```

1. **Private Endpoint**: creates a network interface with a private IP in your subnet, connected to the target service via Private Link.
2. **Private DNS Zone**: holds the DNS record that maps the service's FQDN to the private IP.
3. **VNet Link**: connects the Private DNS Zone to your VNet so DNS queries resolve to the private IP.

### Subnet requirements

- The subnet used for Private Endpoints should be dedicated (no other resources)
- Minimum size: `/28` (16 IPs) for small deployments, `/24` (256 IPs) for enterprise
- NSG on the PE subnet is supported (since November 2023) but not required
- No delegation should be set on the PE subnet

---

## Private DNS Zone Names

Each Azure service has a specific Private DNS Zone name. Using the wrong zone name is the most common cause of "it deploys but does not work".

| Azure Service | Private DNS Zone Name | Subresource |
|---------------|----------------------|-------------|
| **App Service / Function App** | `privatelink.azurewebsites.net` | `sites` |
| **PostgreSQL Flexible Server** | `privatelink.postgres.database.azure.com` | `postgresqlServer` |
| **MySQL Flexible Server** | `privatelink.mysql.database.azure.com` | `mysqlServer` |
| **SQL Database** | `privatelink.database.windows.net` | `sqlServer` |
| **Cosmos DB (SQL API)** | `privatelink.documents.azure.com` | `Sql` |
| **Key Vault** | `privatelink.vaultcore.azure.net` | `vault` |
| **Storage Account (Blob)** | `privatelink.blob.core.windows.net` | `blob` |
| **Storage Account (File)** | `privatelink.file.core.windows.net` | `file` |
| **Storage Account (Table)** | `privatelink.table.core.windows.net` | `table` |
| **Storage Account (Queue)** | `privatelink.queue.core.windows.net` | `queue` |
| **Azure Container Registry** | `privatelink.azurecr.io` | `registry` |
| **Azure Cognitive Services** | `privatelink.cognitiveservices.azure.com` | `account` |
| **Azure OpenAI** | `privatelink.openai.azure.com` | `account` |
| **Azure Search** | `privatelink.search.windows.net` | `searchService` |
| **Event Hub** | `privatelink.servicebus.windows.net` | `namespace` |
| **Service Bus** | `privatelink.servicebus.windows.net` | `namespace` |
| **Redis Cache** | `privatelink.redis.cache.windows.net` | `redisCache` |
| **Azure Monitor** | `privatelink.monitor.azure.com` | `azuremonitor` |
| **Application Insights** | `privatelink.monitor.azure.com` | `azuremonitor` |
| **Log Analytics** | `privatelink.ods.opinsights.azure.com` | `azuremonitor` |

> **Source**: [Microsoft Learn — Azure Private Endpoint DNS configuration](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns)

---

## Terraform Implementation

### Generic Private Endpoint pattern

This pattern works for any Azure PaaS service. Replace the DNS zone name and subresource name for your target service.

```hcl
# networking.tf — Private Endpoint for PostgreSQL

# 1. Private DNS Zone
module "dns_zone_postgresql" {
  source  = "Azure/avm-res-network-privatednszone/azurerm"
  version = "~> 0.7"

  enable_telemetry = false

  domain_name         = "privatelink.postgres.database.azure.com"
  resource_group_name = module.resource_group.name
  tags                = local.common_tags

  virtual_network_links = {
    vnet_link = {
      vnetlinkname = "vnetlink-psql-${local.name_prefix}"
      vnetid       = module.vnet.resource_id
    }
  }
}

# 2. Private Endpoint
module "pe_postgresql" {
  source  = "Azure/avm-res-network-privateendpoint/azurerm"
  version = "~> 0.2"

  enable_telemetry = false

  name                = "pe-psql-${local.name_prefix}"
  location            = module.resource_group.resource_group_location
  resource_group_name = module.resource_group.name
  subnet_resource_id  = module.vnet.subnets["pe-snet"].resource_id

  private_connection_resource_id = module.postgresql.resource_id
  subresource_names              = ["postgresqlServer"]

  private_dns_zone_group = {
    zone_ids = [module.dns_zone_postgresql.resource_id]
  }

  tags = local.common_tags
}
```

### App Service with Private Endpoint

```hcl
# App Service Private Endpoint
module "dns_zone_appservice" {
  source  = "Azure/avm-res-network-privatednszone/azurerm"
  version = "~> 0.7"

  enable_telemetry = false

  domain_name         = "privatelink.azurewebsites.net"
  resource_group_name = module.resource_group.name
  tags                = local.common_tags

  virtual_network_links = {
    vnet_link = {
      vnetlinkname = "vnetlink-app-${local.name_prefix}"
      vnetid       = module.vnet.resource_id
    }
  }
}

module "pe_appservice" {
  source  = "Azure/avm-res-network-privateendpoint/azurerm"
  version = "~> 0.2"

  enable_telemetry = false

  name                = "pe-app-${local.name_prefix}"
  location            = module.resource_group.resource_group_location
  resource_group_name = module.resource_group.name
  subnet_resource_id  = module.vnet.subnets["pe-snet"].resource_id

  private_connection_resource_id = module.app_service.resource_id
  subresource_names              = ["sites"]

  private_dns_zone_group = {
    zone_ids = [module.dns_zone_appservice.resource_id]
  }

  tags = local.common_tags
}
```

### Key Vault with Private Endpoint

```hcl
module "dns_zone_keyvault" {
  source  = "Azure/avm-res-network-privatednszone/azurerm"
  version = "~> 0.7"

  enable_telemetry = false

  domain_name         = "privatelink.vaultcore.azure.net"
  resource_group_name = module.resource_group.name
  tags                = local.common_tags

  virtual_network_links = {
    vnet_link = {
      vnetlinkname = "vnetlink-kv-${local.name_prefix}"
      vnetid       = module.vnet.resource_id
    }
  }
}

module "pe_keyvault" {
  source  = "Azure/avm-res-network-privateendpoint/azurerm"
  version = "~> 0.2"

  enable_telemetry = false

  name                = "pe-kv-${local.name_prefix}"
  location            = module.resource_group.resource_group_location
  resource_group_name = module.resource_group.name
  subnet_resource_id  = module.vnet.subnets["pe-snet"].resource_id

  private_connection_resource_id = module.key_vault.resource_id
  subresource_names              = ["vault"]

  private_dns_zone_group = {
    zone_ids = [module.dns_zone_keyvault.resource_id]
  }

  tags = local.common_tags
}
```

### Storage Account with Private Endpoint (Blob)

```hcl
module "dns_zone_blob" {
  source  = "Azure/avm-res-network-privatednszone/azurerm"
  version = "~> 0.7"

  enable_telemetry = false

  domain_name         = "privatelink.blob.core.windows.net"
  resource_group_name = module.resource_group.name
  tags                = local.common_tags

  virtual_network_links = {
    vnet_link = {
      vnetlinkname = "vnetlink-blob-${local.name_prefix}"
      vnetid       = module.vnet.resource_id
    }
  }
}

module "pe_storage_blob" {
  source  = "Azure/avm-res-network-privateendpoint/azurerm"
  version = "~> 0.2"

  enable_telemetry = false

  name                = "pe-blob-${local.name_prefix}"
  location            = module.resource_group.resource_group_location
  resource_group_name = module.resource_group.name
  subnet_resource_id  = module.vnet.subnets["pe-snet"].resource_id

  private_connection_resource_id = module.storage_account.resource_id
  subresource_names              = ["blob"]

  private_dns_zone_group = {
    zone_ids = [module.dns_zone_blob.resource_id]
  }

  tags = local.common_tags
}
```

---

## Common Mistakes

### 1. Missing VNet Link on the DNS Zone

**Symptom**: PE deploys successfully, but your app cannot reach the service. DNS resolves to the public IP instead of the private IP.

**Cause**: the Private DNS Zone exists but is not linked to the VNet where your app runs.

**Fix**: always add `virtual_network_links` to the DNS zone module:

```hcl
virtual_network_links = {
  vnet_link = {
    vnetlinkname = "vnetlink-${local.name_prefix}"
    vnetid       = module.vnet.resource_id
  }
}
```

### 2. Wrong subnet for the Private Endpoint

**Symptom**: deployment fails with "subnet is not valid for private endpoint".

**Cause**: the subnet has a delegation (e.g., `Microsoft.Web/serverFarms` for App Service VNet integration). Delegated subnets cannot host Private Endpoints.

**Fix**: use a dedicated subnet with no delegation for Private Endpoints:

```hcl
subnets = {
  # Subnet for app VNet integration (delegated)
  app-snet = {
    address_prefix = "10.0.1.0/24"
    delegations = [{
      name = "webapp"
      service_delegation = {
        name = "Microsoft.Web/serverFarms"
      }
    }]
  }

  # Dedicated subnet for Private Endpoints (NO delegation)
  pe-snet = {
    address_prefix = "10.0.3.0/24"
    # No delegations here
  }
}
```

### 3. NSG blocking Private Endpoint traffic

**Symptom**: PE deploys, DNS resolves correctly, but connections time out.

**Cause**: NSG on the PE subnet has no inbound rule allowing traffic from the app subnet.

**Fix**: add an inbound rule on the PE subnet NSG:

```hcl
security_rules = [
  {
    name                       = "allow-app-to-pe"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "10.0.1.0/24"  # App subnet
    destination_address_prefix = "10.0.3.0/24"   # PE subnet
  }
]
```

### 4. Forgetting to disable public access

**Symptom**: PE works, but the service is still accessible from the internet. Compliance audit fails.

**Cause**: deploying a PE does not automatically disable public access. You must do it explicitly.

**Fix**: set `public_network_access_enabled = false` on the target service:

```hcl
module "postgresql" {
  # ...
  public_network_access_enabled = false  # Must be explicit
}
```

### 5. DNS zone name typo

**Symptom**: PE deploys, DNS zone exists, VNet link exists, but DNS resolves to the public IP.

**Cause**: the DNS zone name does not match the expected pattern. For example, using `privatelink.database.azure.com` (wrong) instead of `privatelink.postgres.database.azure.com` (correct) for PostgreSQL.

**Fix**: always copy the DNS zone name from the table above or from Microsoft documentation. Never type it from memory.

### 6. Multiple PEs sharing a DNS zone without planning

**Symptom**: works in one subscription, fails when you deploy a second stack.

**Cause**: two Private DNS Zones with the same name in the same VNet. Azure only allows one.

**Fix**: in hub-spoke architectures, deploy DNS zones in the hub and share them across spokes via VNet links. Never deploy the same DNS zone in multiple VNets without a plan.

---

## Hub vs Spoke PE Placement

### Centralized model (recommended for enterprise)

```
Hub VNet
  +-- Private DNS Zones (all zones here)
  +-- VNet Links to all spoke VNets

Spoke VNet 1
  +-- pe-snet (Private Endpoints for Spoke 1 services)
  +-- app-snet (workloads)

Spoke VNet 2
  +-- pe-snet (Private Endpoints for Spoke 2 services)
  +-- app-snet (workloads)
```

**Advantages**: single source of truth for DNS, no duplicate zones, easier management.

**Implementation**: deploy all Private DNS Zones in the hub resource group and add VNet links to each spoke.

### Decentralized model (simpler for small teams)

```
Spoke VNet 1
  +-- Private DNS Zones (for this spoke only)
  +-- pe-snet
  +-- app-snet

Spoke VNet 2
  +-- Private DNS Zones (for this spoke only)
  +-- pe-snet
  +-- app-snet
```

**Advantages**: each team manages their own DNS, no cross-subscription dependencies.

**Disadvantage**: duplicate zones, harder to manage at scale, no cross-spoke resolution.

---

## Limits and Quotas

| Limit | Default | Maximum (with support request) |
|-------|:---:|:---:|
| Private Endpoints per VNet | 1,000 | 5,000 (High Scale PE) |
| Private Endpoints per subnet | 1,000 | 1,000 |
| Private DNS Zones per subscription | 250 | 1,000 |
| VNet Links per Private DNS Zone | 1,000 | 1,000 |
| Auto-registration VNet Links per zone | 100 | 100 |
| A records per Private DNS Zone | 25,000 | 25,000 |

For most deployments, the default of 1,000 PEs per VNet is sufficient. If you are building a large enterprise platform with hundreds of PaaS services, request the High Scale PE feature through Azure Support.

---

## Cost

Private Endpoints are billed per hour and per GB of data processed:

| Component | Cost (West Europe, as of 2026) |
|-----------|---:|
| PE per hour | ~EUR 0.009 |
| PE per month (730 hours) | ~EUR 7 |
| Data processed (inbound) | EUR 0.00 |
| Data processed (outbound, first 1 PB) | ~EUR 0.009/GB |

**Typical monthly cost per PE**: approximately EUR 7/month for the PE itself, plus negligible data processing charges for most workloads.

**Cost optimization tips**:

- Consolidate services where possible (one Storage Account instead of three)
- In basic/dev tiers, consider whether public access with IP restrictions is acceptable (cheaper, less secure)
- Use VNet Service Endpoints for services that support them when compliance allows (free, but less secure than PE)

---

## Conditional Deployment by Tier

In a tiered deployment model, Private Endpoints should be deployed conditionally:

```hcl
variable "tier" {
  description = "Deployment tier: basic, standard, premium"
  type        = string
  default     = "standard"
}

locals {
  enable_private_endpoints = var.tier != "basic"
}

module "pe_postgresql" {
  source  = "Azure/avm-res-network-privateendpoint/azurerm"
  version = "~> 0.2"

  count = local.enable_private_endpoints ? 1 : 0

  # ... configuration ...
}
```

| Tier | Private Endpoints | Public Access | Justification |
|------|:-:|:-:|-------------|
| Basic | Disabled | Enabled with IP rules | Cost optimization for dev/test |
| Standard | Enabled | Disabled | Production security baseline |
| Premium | Enabled | Disabled | Full compliance (MCSB NS-2) |

---

## Further Reading

- [Microsoft Learn — What is Azure Private Link?](https://learn.microsoft.com/en-us/azure/private-link/private-link-overview)
- [Microsoft Learn — Private Endpoint DNS configuration](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns)
- [Microsoft Learn — Private Endpoint limits](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits#private-link-limits)
- [MCSB NS-2 — Secure cloud services with network controls](https://learn.microsoft.com/en-us/security/benchmark/azure/ns-2-secure-cloud-services-with-network-controls)

---

*Maintained by [Frederic Leroy](https://github.com/f-leroy) — MCT, Azure Solutions Architect Expert*
