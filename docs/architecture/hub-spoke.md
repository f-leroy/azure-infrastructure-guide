# Hub-Spoke Network Architecture on Azure

**Version**: 1.0
**Last updated**: 2026-03-18
**Category**: Platform / Networking
**Cloud Adoption Framework alignment**: Network Topology and Connectivity

---

## Overview

The hub-spoke topology is the recommended network architecture for Azure workloads that require centralized control over security, routing, and shared services. A single **hub** Virtual Network acts as the central point of connectivity, while multiple **spoke** VNets host individual workloads and peer back to the hub.

This pattern is fundamental to Azure Landing Zones and provides clear separation between platform services (managed by a central team) and workload services (managed by application teams).

### When to use hub-spoke

- You need centralized firewall inspection for all outbound and east-west traffic.
- Multiple workload teams share common services (DNS, monitoring, identity) but require network isolation.
- Regulatory or compliance requirements mandate traffic inspection and logging at a network level.
- You plan to scale to multiple subscriptions with consistent networking governance.

### When NOT to use hub-spoke

- Single-application deployments where VNet integration + Private Endpoints is sufficient.
- Proof-of-concept environments where cost and simplicity outweigh governance.
- Scenarios where Azure Virtual WAN is preferred (large-scale branch connectivity, SD-WAN integration).

---

## Architecture Diagram

```
                          On-premises
                              |
                         VPN Gateway /
                        ExpressRoute
                              |
            +─────────────────────────────────────+
            |           Hub VNet (10.0.0.0/16)    |
            |                                     |
            |  +-----------+   +--------------+   |
            |  | Azure     |   | Bastion      |   |
            |  | Firewall  |   | (AzureBastion|   |
            |  | Subnet    |   |  Subnet)     |   |
            |  | 10.0.1.0  |   | 10.0.2.0/26  |   |
            |  |   /26     |   +--------------+   |
            |  +-----------+                      |
            |                                     |
            |  +-----------+   +--------------+   |
            |  | GatewaySubnet| | DNS Private  |   |
            |  | 10.0.0.0/27 | | Resolver     |   |
            |  +-----------+   | 10.0.3.0/28  |   |
            |                  +--------------+   |
            |                                     |
            |  +-------------------------------+  |
            |  | Shared Services Subnet        |  |
            |  | 10.0.4.0/24                   |  |
            |  | (Log Analytics, Key Vault,    |  |
            |  |  Private DNS Zones)           |  |
            |  +-------------------------------+  |
            +───────────┬──────────┬──────────────+
                        |          |
              VNet Peering      VNet Peering
                        |          |
         +--------------+--+  +--+--------------+
         | Spoke 1         |  | Spoke 2         |
         | 10.1.0.0/16     |  | 10.2.0.0/16     |
         |                 |  |                 |
         | App Subnet      |  | App Subnet      |
         | 10.1.1.0/24     |  | 10.2.1.0/24     |
         |                 |  |                 |
         | Data Subnet     |  | Data Subnet     |
         | 10.1.2.0/24     |  | 10.2.2.0/24     |
         |                 |  |                 |
         | PE Subnet       |  | PE Subnet       |
         | 10.1.3.0/24     |  | 10.2.3.0/24     |
         +-----------------+  +-----------------+
```

---

## Hub Components

### Azure Firewall

Azure Firewall is deployed in the hub to inspect and control all traffic flowing between spokes, to the internet, and to on-premises networks. It serves as the single egress point.

| Property | Value |
|---|---|
| SKU | Standard (most workloads) or Premium (TLS inspection, IDPS) |
| Subnet | `AzureFirewallSubnet`, minimum /26 |
| Availability Zones | Zone-redundant deployment recommended for production |
| DNS Proxy | Enabled, to forward DNS queries from spokes |
| Threat Intelligence | Alert and Deny mode in production |

**Firewall rules are organized in Rule Collection Groups**:

- **Network rules**: Allow spoke-to-spoke traffic on specific ports, allow outbound to Azure services.
- **Application rules**: Allow outbound HTTPS to approved FQDNs (Windows Update, package registries, Azure management endpoints).
- **DNAT rules**: Inbound port forwarding if required (rare in hub-spoke; prefer Front Door or Application Gateway).

### Azure Bastion

Bastion provides secure RDP/SSH access to VMs in any peered spoke without exposing public IPs on VMs.

| Property | Value |
|---|---|
| SKU | Standard (for IP-based connection to peered VNets) |
| Subnet | `AzureBastionSubnet`, minimum /26 |
| Features | Native client support, shareable links, file transfer |

Bastion Standard SKU can connect to VMs in peered spoke VNets using their private IP address. This eliminates the need to deploy Bastion in every spoke.

### VPN Gateway / ExpressRoute

Hybrid connectivity from on-premises to Azure.

| Option | Use case | Bandwidth |
|---|---|---|
| VPN Gateway (VpnGw2AZ) | Small to medium offices, encrypted over internet | Up to 1.25 Gbps |
| ExpressRoute | Enterprise, dedicated private circuit | 50 Mbps to 100 Gbps |
| Both (coexistence) | ExpressRoute primary + VPN failover | Combined |

The gateway is deployed in `GatewaySubnet` (minimum /27). Route propagation from the gateway to spokes is handled via VNet peering with `allow_gateway_transit` on the hub and `use_remote_gateways` on each spoke.

### DNS Private Resolver

Azure DNS Private Resolver enables DNS forwarding between Azure and on-premises networks without deploying custom DNS VMs.

| Component | Purpose |
|---|---|
| Inbound endpoint | On-premises resolves Azure Private DNS Zones |
| Outbound endpoint | Azure resolves on-premises DNS domains |
| Forwarding rules | Map specific domains to on-premises DNS servers |

The resolver is deployed in its own subnet (minimum /28) in the hub VNet. All Private DNS Zones are linked to the hub VNet, and spokes use the hub's DNS configuration.

### Shared Services

The hub hosts centralized services consumed by all spokes:

- **Log Analytics Workspace**: Central logging destination for all diagnostic settings.
- **Key Vault**: Shared secrets and certificates (platform-level).
- **Private DNS Zones**: Linked to the hub and resolved via Azure Firewall DNS Proxy or DNS Private Resolver.

---

## Spoke Components

Each spoke is an isolated VNet hosting a single workload or application tier.

### Spoke Virtual Network

- Address space: /16 per spoke (e.g., 10.1.0.0/16, 10.2.0.0/16).
- Peered to hub with `allow_forwarded_traffic = true` and `use_remote_gateways = true`.
- No direct peering between spokes (all inter-spoke traffic routes through the hub firewall).

### Route Table (UDR)

Every spoke subnet has a User-Defined Route that forces all traffic through the Azure Firewall:

```
Address Prefix    Next Hop Type       Next Hop IP
0.0.0.0/0         VirtualAppliance    10.0.1.4 (Firewall private IP)
```

`bgp_route_propagation_enabled` is set to `false` on spoke route tables to prevent gateway routes from bypassing the firewall.

### Network Security Groups

Every subnet in a spoke has an NSG. Even though the firewall inspects traffic at L3-L7, NSGs provide defense-in-depth at the subnet level:

- Default deny all inbound from internet.
- Allow only required ports between subnets.
- Allow outbound to hub firewall only.

---

## Traffic Flows

### Spoke to Internet

```
Spoke App Subnet → UDR → Hub Azure Firewall → Internet
```

The firewall evaluates network rules, then application rules. Only approved FQDNs and ports are allowed outbound.

### Spoke to Spoke

```
Spoke 1 → UDR → Hub Azure Firewall → Spoke 2
```

The firewall must have a network rule allowing the specific spoke-to-spoke communication. This provides full visibility and control over east-west traffic.

### Spoke to On-premises

```
Spoke → UDR → Hub Azure Firewall → VPN Gateway/ExpressRoute → On-premises
```

Traffic flows through the firewall before reaching the gateway, ensuring inspection of all hybrid traffic.

### On-premises to Spoke

```
On-premises → VPN/ER Gateway → Hub → Firewall → Spoke
```

Gateway route propagation combined with UDRs ensures return traffic also passes through the firewall.

### DNS Resolution Flow

```
Spoke VM → Azure DNS (168.63.129.16) → Firewall DNS Proxy → Private DNS Zone (in hub)
                                                           → DNS Private Resolver → On-premises DNS
```

---

## DNS Architecture with Private DNS Zones

All Azure Private DNS Zones are created in the hub and linked to the hub VNet. Spoke VNets resolve these zones through:

1. **Azure Firewall DNS Proxy** (recommended): The firewall acts as DNS forwarder for all spokes. Spoke VNets use the firewall's private IP as custom DNS server.
2. **DNS Private Resolver**: For hybrid scenarios requiring bi-directional DNS resolution.

Common Private DNS Zones hosted in the hub:

| Zone | Service |
|---|---|
| `privatelink.database.windows.net` | SQL Database |
| `privatelink.postgres.database.azure.com` | PostgreSQL Flexible Server |
| `privatelink.mysql.database.azure.com` | MySQL Flexible Server |
| `privatelink.vaultcore.azure.net` | Key Vault |
| `privatelink.blob.core.windows.net` | Storage Blob |
| `privatelink.azurewebsites.net` | App Service |
| `privatelink.azurecr.io` | Container Registry |
| `privatelink.search.windows.net` | Azure AI Search |
| `privatelink.openai.azure.com` | Azure OpenAI |
| `privatelink.cognitiveservices.azure.com` | Cognitive Services |

---

## Security Considerations

### Network Security (MCSB)

| Control | Implementation |
|---|---|
| NS-1 (Network segmentation) | Hub-spoke topology with isolated VNets, NSG on every subnet |
| NS-2 (Secure cloud services) | Private Endpoints for all PaaS services |
| NS-3 (Edge firewall) | Azure Firewall in hub with deny-by-default rules |
| NS-4 (IDS/IPS) | Azure Firewall Premium with IDPS in Alert+Deny mode |
| NS-5 (DDoS protection) | DDoS Protection Plan associated with hub VNet |
| NS-10 (DNS security) | Private DNS Zones, DNS Private Resolver, no public DNS exposure |

### Identity and Access (MCSB)

| Control | Implementation |
|---|---|
| IM-3 (Application identities) | Managed Identity for all Azure resources |
| PA-7 (Least privilege) | RBAC assignments scoped to resource groups, not subscriptions |

### Logging (MCSB)

| Control | Implementation |
|---|---|
| LT-3 (Security logging) | Diagnostic settings on Firewall, Bastion, NSG to Log Analytics |
| LT-4 (Network logging) | NSG Flow Logs, Firewall structured logs |
| LT-5 (Centralized log analysis) | Single Log Analytics Workspace in hub |

### Additional Security Measures

- **DDoS Protection Plan**: Associated with the hub VNet. All peered spoke VNets inherit protection.
- **Network Watcher**: Enabled in every region for NSG Flow Logs, connection troubleshooting, and packet capture.
- **Azure Policy**: Enforce NSG on subnets, deny public IP creation, enforce UDR association.

---

## Subnet Sizing Recommendations

### Hub VNet (10.0.0.0/16)

| Subnet | CIDR | Usable IPs | Purpose |
|---|---|---|---|
| GatewaySubnet | /27 | 27 | VPN/ExpressRoute gateway (Azure requirement) |
| AzureFirewallSubnet | /26 | 59 | Azure Firewall (minimum /26 required) |
| AzureFirewallManagementSubnet | /26 | 59 | Forced tunneling (if needed) |
| AzureBastionSubnet | /26 | 59 | Azure Bastion (minimum /26 required) |
| DNS Resolver Inbound | /28 | 11 | DNS Private Resolver inbound endpoint |
| DNS Resolver Outbound | /28 | 11 | DNS Private Resolver outbound endpoint |
| Shared Services | /24 | 251 | Private Endpoints for shared services |

### Spoke VNet (10.x.0.0/16)

| Subnet | CIDR | Usable IPs | Purpose |
|---|---|---|---|
| Application | /24 | 251 | App Services, VMs, containers |
| Data | /24 | 251 | Databases, storage (when subnet-delegated) |
| Private Endpoints | /24 | 251 | PE NICs (plan 1 IP per PE per service) |
| AKS Nodes (if applicable) | /22 | 1019 | AKS node pool (plan 30 IPs per node with Azure CNI) |

**Key rules**:
- Azure reserves 5 IPs per subnet (first 4 + last).
- AzureFirewallSubnet and AzureBastionSubnet names are fixed by Azure; they cannot be renamed.
- Never share the Private Endpoints subnet with application workloads.

---

## Terraform File Structure

```
hub-spoke/
  main.tf                 # terraform block, providers, locals (naming, tags)
  resource_group.tf       # Hub resource group
  networking.tf           # Hub VNet, subnets, peerings, NSGs, route tables
  security.tf             # Azure Firewall, Firewall Policy, Bastion
  identity.tf             # User-Assigned Managed Identity
  keyvault.tf             # Shared Key Vault
  monitoring.tf           # Log Analytics Workspace, diagnostic settings
  dnsresolver.tf          # DNS Private Resolver, forwarding rules
  variables.tf            # All input variables
  outputs.tf              # Hub VNet ID, Firewall private IP, Log Analytics ID
  terraform.tfvars        # Environment-specific values
  backend.tf.example      # Remote state configuration template
  finops.tf               # Budget alerts, cost anomaly detection
  .checkov.yaml           # Security scan exceptions with justifications
  README.md               # Stack documentation
  COMPLIANCE.md           # Compliance mapping (CAF, MCSB, RGPD, NIS2)
  manifest.yaml           # Stack metadata
```

### Key Terraform patterns

**main.tf** contains only providers and locals:

```hcl
terraform {
  required_version = ">= 1.9"
  required_providers {
    azurerm = { source = "hashicorp/azurerm", version = "~> 4.14" }
    azapi   = { source = "azure/azapi",       version = "~> 2.4" }
  }
}

locals {
  name_prefix = lower(join("-", compact([var.org_code, var.business_unit, var.project, var.env, var.region])))
  common_tags = {
    environment = var.env
    project     = var.project
    managed_by  = "terraform"
    iac_source  = "terraform"
  }
}
```

**networking.tf** contains VNet, subnets, peerings, NSGs, and route tables. No Azure Firewall or Bastion resources (those go in security.tf).

**security.tf** contains Azure Firewall, Firewall Policy, Rule Collection Groups, and Bastion Host. Firewall inputs use the `firewall_` prefix as required by the AVM module.

---

## Cost Estimates

### Basic Hub (without Firewall)

For development or non-production environments where centralized firewall inspection is not required. Traffic between spokes uses NSGs for security.

| Component | Monthly Cost (West Europe) |
|---|---|
| VPN Gateway (VpnGw1AZ) | ~EUR 140 |
| Bastion (Basic SKU) | ~EUR 140 |
| Log Analytics (5 GB/day) | ~EUR 100 |
| Key Vault (Standard) | ~EUR 5 |
| VNet + Peering | ~EUR 10 |
| **Total** | **~EUR 395/month** |

### Production Hub (with Firewall)

| Component | Monthly Cost (West Europe) |
|---|---|
| Azure Firewall (Standard) | ~EUR 900 |
| VPN Gateway (VpnGw2AZ) | ~EUR 280 |
| Bastion (Standard SKU) | ~EUR 330 |
| DDoS Protection Plan | ~EUR 2,700 (covers all VNets) |
| Log Analytics (20 GB/day) | ~EUR 200 |
| DNS Private Resolver | ~EUR 150 |
| Key Vault (Standard) | ~EUR 5 |
| VNet + Peering | ~EUR 10 |
| **Total** | **~EUR 4,575/month** |

### Enterprise Hub (with Firewall Premium)

| Component | Monthly Cost (West Europe) |
|---|---|
| Azure Firewall (Premium) | ~EUR 1,400 |
| ExpressRoute (1 Gbps) | ~EUR 400+ (circuit fee, varies by provider) |
| Bastion (Standard SKU) | ~EUR 330 |
| DDoS Protection Plan | ~EUR 2,700 |
| Log Analytics (50 GB/day) | ~EUR 350 |
| DNS Private Resolver | ~EUR 150 |
| Key Vault (Premium, HSM) | ~EUR 30 |
| VNet + Peering | ~EUR 15 |
| **Total** | **~EUR 5,375+/month** |

> **Note**: DDoS Protection Plan is the single most expensive item but covers all VNets in the subscription. If your organization already has a DDoS plan, it does not need to be duplicated. Costs are estimates based on West Europe pricing as of early 2026 and will vary by region and usage.

---

## References

- [Azure hub-spoke topology](https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/hub-spoke)
- [Azure Firewall documentation](https://learn.microsoft.com/en-us/azure/firewall/overview)
- [Azure DNS Private Resolver](https://learn.microsoft.com/en-us/azure/dns/dns-private-resolver-overview)
- [Azure Landing Zones - Network topology](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/network-topology-and-connectivity)
- [Microsoft Cloud Security Benchmark v1](https://learn.microsoft.com/en-us/security/benchmark/azure/overview)

---

*Maintained by [Frederic Leroy](https://github.com/f-leroy) -- MCT, Azure Solutions Architect Expert*
