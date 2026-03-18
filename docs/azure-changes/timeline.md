# Azure Breaking Changes Timeline — 2026-2027

> Last updated: March 18, 2026

## Already in effect

| Date | Change | Impact | Action |
|------|--------|--------|--------|
| Feb 3, 2026 | Storage: TLS 1.0/1.1 rejected | All storage connections must use TLS 1.2+ | Set `min_tls_version = "TLS1_2"` |
| Mar 2, 2026 | Log Analytics Agent (MMA) backend shutdown | MMA can no longer send data | Use Azure Monitor Agent (AMA) extensions |
| Sep 30, 2025 | Basic Public IP SKU blocked | Cannot create Basic PIPs | Use `sku = "Standard"` |
| Sep 30, 2025 | Basic Load Balancer SKU blocked | Cannot create Basic LBs | Use `sku = "Standard"` |

## March 2026

| Date | Change | Impact | Action |
|------|--------|--------|--------|
| **Mar 31** | **VNet default outbound access retired** | New VNets have no internet access | Add NAT Gateway, Firewall, or LB outbound rules |
| Mar 31 | VPN Gateway legacy SKUs retired | "Standard"/"HighPerformance" SKUs unavailable | Use `VpnGw1AZ` / `VpnGw2AZ` |
| Mar 31 | Recovery Services Vault classic alerts retired | Classic backup alerts stop working | Use Azure Monitor alerts |
| Mar 31 | HCP Terraform free tier EOL | Legacy free plan auto-migrated | No impact on local Terraform CLI |
| Mar 31 | Log Analytics Beta API retired | Beta API endpoints stop responding | Use GA API versions |
| Mar 31 | ML compute low-priority VMs deprecated | Cannot create low-priority clusters | Use dedicated or spot VMs |

## April–June 2026

| Date | Change | Impact | Action |
|------|--------|--------|--------|
| Apr 28 | Application Gateway v1 retired | All v1 gateways stopped and deleted | Use `Standard_v2` / `WAF_v2` SKU |
| Apr 30 | Node.js 20 end of support (Functions) | Node.js 20 unsupported in Azure Functions | Default to Node.js 22 |
| May 13 | Entra ID Conditional Access OIDC enforcement | Service principals may face MFA challenges | Review CA exemptions for CI/CD |
| May 31 | ACR Docker Content Trust (DCT) enrollment blocked | Cannot enable DCT on new registries | Use Notation/ORAS for image signing |
| Jun 30 | Azure SQL API 2014-04-01 retired | Legacy SQL API endpoints fail | Use `azurerm_mssql_*` resources (not `azurerm_sql_*`) |
| Jun 30 | Azure ML SDK v1 end of support | SDK v1 unsupported | Use SDK v2 |

## Q3–Q4 2026

| Date | Change | Impact | Action |
|------|--------|--------|--------|
| Sep 30 | Container Insights legacy auth retired | Workspace key auth stops working | Use Managed Identity for AKS monitoring |
| Sep 30 | Service Bus legacy SDKs retired | SBMP protocol removed | Document AMQP-only |
| Oct 1 | MFA mandatory for Azure Portal | All portal access requires MFA | Document in deployment guides |
| Oct 1 | AI Services: Personalizer, Metrics Advisor, Anomaly Detector retired | Services unavailable | Remove from stack options |
| Oct 28 | PIM v2 Beta API retired | PIM automation fails on old API | Migrate to PIM v3 API |
| Nov 10 | Functions .NET in-process model end of support | In-process unsupported | Recommend isolated worker model |

## 2027

| Date | Change | Impact | Action |
|------|--------|--------|--------|
| Feb 27 | Key Vault APIs < 2026-02-01 retired | Old API calls fail | Set `enable_rbac_authorization = true` explicitly |
| Mar 15 | WAF Configuration on App Gateway v2 retired | Inline WAF config stops working | Use `web_application_firewall_policy` resource |
| Mar 30 | Azure Cache for Redis Enterprise retired | Enterprise/Flash tiers unavailable | Migrate to Azure Managed Redis |
| Mar 31 | Azure Front Door classic retired | Classic FD unavailable | Use `azurerm_cdn_frontdoor_*` resources |
| Sep 30 | CDN Standard from Microsoft (classic) retired | Classic CDN unavailable | Migrate to Front Door Standard/Premium |

## 2028

| Date | Change | Impact | Action |
|------|--------|--------|--------|
| Mar 31 | ACR Docker Content Trust fully removed | DCT completely unavailable | Notation/ORAS |
| Sep 15 | Azure Disk Encryption (ADE) retired | Encrypted disks fail to unlock on reboot | Use `encryption_at_host_enabled = true` |
| Sep 30 | Azure Cache for Redis Basic/Standard/Premium retired | All Redis tiers migrated | Azure Managed Redis |

---

## Sources

- [Azure Service Retirement Workbook](https://learn.microsoft.com/en-us/azure/advisor/advisor-how-to-plan-migration-workloads-service-retirement)
- [Azure Updates](https://azure.microsoft.com/en-us/updates/)
- [AzureRM Provider Releases](https://github.com/hashicorp/terraform-provider-azurerm/releases)
- [Entra ID 2026 Changes](https://www.epcgroup.net/blog/microsoft-entra-id-changes-2026-admin-action-plan)

---

*Maintained by [Frédéric Leroy](https://github.com/f-leroy) — MCT, Azure Solutions Architect Expert*
