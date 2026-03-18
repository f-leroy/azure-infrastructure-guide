# VPN Gateway Legacy SKUs Retirement — March 31, 2026

> **Deadline: March 31, 2026**
> Legacy "Standard" and "HighPerformance" VPN Gateway SKUs will be deprecated.

## What changes

The legacy VPN Gateway SKU names are retired. You must use the new `VpnGw*` naming:

| Legacy SKU | New SKU | Bandwidth |
|-----------|---------|-----------|
| Standard | **VpnGw1** / **VpnGw1AZ** | 650 Mbps |
| HighPerformance | **VpnGw2** / **VpnGw2AZ** | 1 Gbps |

The `AZ` suffix indicates zone-redundant (recommended for production).

## Terraform

```hcl
resource "azurerm_virtual_network_gateway" "vpn" {
  name                = "vpng-${local.name_prefix}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  type                = "Vpn"
  vpn_type            = "RouteBased"

  # Use new SKU names
  sku = "VpnGw1AZ"  # NOT "Standard"

  ip_configuration {
    public_ip_address_id = azurerm_public_ip.vpn.id
    subnet_id            = module.virtual_network.subnets["GatewaySubnet"].resource_id
  }
}
```

## References

- [VPN Gateway legacy SKUs — Microsoft Learn](https://learn.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-about-skus-legacy)

---

*Maintained by [Frédéric Leroy](https://github.com/f-leroy) — MCT, Azure Solutions Architect Expert*
