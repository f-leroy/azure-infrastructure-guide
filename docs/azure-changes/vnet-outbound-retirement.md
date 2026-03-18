# VNet Default Outbound Access Retirement — March 31, 2026

> **Deadline: March 31, 2026**
> After this date, new Azure Virtual Networks will NOT have default outbound internet access.

## What changes

Before March 31, 2026, every VM in an Azure VNet could reach the internet via "default outbound access" — an implicit SNAT provided by Azure at no cost.

After March 31, 2026, this implicit access is **removed** for all newly created VNets. VMs and VNet-integrated PaaS services will have **no internet connectivity** unless you explicitly provide an outbound method.

**This does NOT affect existing VNets** — only new deployments.

## What breaks without action

| Resource type | What breaks |
|---------------|------------|
| Virtual Machines | No OS updates (`apt update`, `yum update`), no package installs, no external API calls |
| App Service (VNet-integrated) | Cannot pull dependencies, cannot reach external APIs |
| Function Apps (VNet-integrated) | Cannot pull packages, cannot reach external services |
| AKS nodes | Cannot pull container images from Docker Hub or external registries |
| ML Workspaces | Cannot download models, packages, or datasets |
| Any resource needing internet | No outbound connectivity |

## Solutions

| Method | Cost | Best for |
|--------|------|----------|
| **NAT Gateway** | ~€32/month + data processing | Most workloads — simple, scalable, auditable |
| **Azure Firewall** | ~€900/month | Enterprise — when you need L7 filtering, IDPS, URL filtering |
| **Load Balancer** (Standard, with outbound rules) | ~€18/month | When you already have a LB for inbound traffic |
| **Public IP on VM** | ~€3.50/month per IP | Single VMs only — not recommended for production |

## Terraform implementation

### NAT Gateway (recommended)

```hcl
resource "azurerm_public_ip" "natgw" {
  name                = "pip-natgw-${local.name_prefix}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  allocation_method   = "Static"
  sku                 = "Standard"
  zones               = ["1", "2", "3"]
}

module "nat_gateway" {
  source  = "Azure/avm-res-network-natgateway/azurerm"
  version = "~> 0.3"

  name                = "ng-${local.name_prefix}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  public_ips = {
    pip1 = {
      resource_id = azurerm_public_ip.natgw.id
    }
  }

  subnet_associations = {
    default = {
      resource_id = module.virtual_network.subnets["default"].resource_id
    }
  }

  enable_telemetry = false
}
```

### Key points

- NAT Gateway must be associated to each subnet that needs outbound
- NAT Gateway + Private Endpoints = best practice (PE for Azure services, NAT GW for internet)
- NAT Gateway is zone-redundant when using a Standard SKU Public IP with all zones
- Idle timeout default: 4 minutes (configurable up to 120 minutes)

## How to check if you're affected

Run this Azure Resource Graph query to find VMs using default outbound:

```kusto
resources
| where type == "microsoft.compute/virtualmachines"
| where properties.networkProfile.networkInterfaces[0].properties.ipConfigurations[0].properties.publicIPAddress == ""
| project name, resourceGroup, subscriptionId
```

Or use the [Azure Portal link](https://portal.azure.com/#view/HubsExtension/ArgQueryBlade/query/resources%0A%7C%20where%20type%20%3D%3D%20%22microsoft.compute%2Fvirtualmachines%22%0A%7C%20where%20properties.networkProfile.networkInterfaces%5B0%5D.properties.ipConfigurations%5B0%5D.properties.publicIPAddress%20%3D%3D%20%22%22%0A%7C%20project%20name%2C%20resourceGroup%2C%20subscriptionId).

## References

- [Default outbound access for VMs in Azure — Microsoft Learn](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/default-outbound-access)
- [NAT Gateway documentation](https://learn.microsoft.com/en-us/azure/nat-gateway/nat-overview)
- [AVM NAT Gateway module](https://registry.terraform.io/modules/Azure/avm-res-network-natgateway/azurerm/latest)

---

*Maintained by [Frédéric Leroy](https://github.com/f-leroy) — MCT, Azure Solutions Architect Expert*
