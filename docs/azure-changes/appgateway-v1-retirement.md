# Application Gateway v1 Retirement — April 28, 2026

> **Deadline: April 28, 2026**
> All Application Gateway v1 deployments will be stopped and deleted by Azure.

## What changes

- No new v1 Application Gateways can be created (blocked since September 2024)
- All existing v1 gateways will be **stopped and deallocated** on April 28, 2026
- After this date, v1 gateways will be **deleted**
- Only v2 SKUs are supported: `Standard_v2` and `WAF_v2`

## v1 vs v2

| Feature | v1 | v2 |
|---------|----|----|
| Autoscaling | No | Yes |
| Zone redundancy | No | Yes |
| Static VIP | No | Yes |
| Performance | Lower | 5x faster TLS offload |
| Key Vault integration | No | Yes (managed identity) |
| Subnet size | /26 minimum | /24 recommended |

## Terraform

```hcl
module "application_gateway" {
  source  = "Azure/avm-res-network-applicationgateway/azurerm"
  version = "~> 0.3"

  name                = "agw-${local.name_prefix}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location

  sku = {
    name = "WAF_v2"    # NOT "WAF" (v1)
    tier = "WAF_v2"    # NOT "WAF" (v1)
  }

  enable_telemetry = false
}
```

## References

- [Application Gateway v1 retirement — Microsoft Learn](https://learn.microsoft.com/en-us/azure/application-gateway/v1-retirement)
- [Migrate from v1 to v2](https://learn.microsoft.com/en-us/azure/application-gateway/migrate-v1-v2)

---

*Maintained by [Frédéric Leroy](https://github.com/f-leroy) — MCT, Azure Solutions Architect Expert*
