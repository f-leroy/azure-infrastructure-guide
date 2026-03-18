# Azure Infrastructure Guide

> A curated, practitioner-maintained guide to building production-ready Azure infrastructure with Terraform.
> Covers breaking changes, compliance frameworks, security best practices, and architecture patterns.

[![Azure](https://img.shields.io/badge/Microsoft%20Azure-0078D4?style=flat&logo=microsoftazure&logoColor=white)](https://azure.microsoft.com)
[![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=flat&logo=terraform&logoColor=white)](https://www.terraform.io)
[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)

---

## Why this guide?

Azure changes fast. Services get retired, security defaults shift, compliance regulations stack up. Keeping infrastructure up-to-date requires constant vigilance.

This guide is maintained by a **Microsoft Certified Trainer** and **Azure Solutions Architect Expert** who actively builds production infrastructure. Everything here is verified against real deployments.

---

## Azure Breaking Changes 2026-2027

Critical changes every Azure architect must know about:

- [Full Timeline — All changes at a glance](docs/azure-changes/timeline.md)
- [VNet Default Outbound Access Retirement (March 2026)](docs/azure-changes/vnet-outbound-retirement.md)
- [VPN Gateway Legacy SKUs Retirement (March 2026)](docs/azure-changes/vpn-gateway-skus.md)
- [Application Gateway v1 Retirement (April 2026)](docs/azure-changes/appgateway-v1-retirement.md)
- [Key Vault RBAC Mandatory (February 2027)](docs/azure-changes/keyvault-rbac-mandatory.md)

## Compliance Frameworks for Azure

Infrastructure-level mapping for EU regulations and security benchmarks:

- [NIS2 — What Your Azure Infrastructure Must Include](docs/compliance/nis2-azure.md)
- [DORA — Azure Requirements for Financial Services](docs/compliance/dora-azure.md)
- [RGPD/GDPR — Infrastructure Controls](docs/compliance/rgpd-azure.md)
- [MCSB v1 — Quick Reference for Azure Architects](docs/compliance/mcsb-quick-reference.md)

## Terraform Best Practices

Patterns and lessons from building 90+ production stacks:

- [Azure Verified Modules (AVM) — What You Need to Know](docs/terraform/avm-guide.md)
- [Private Endpoints — The Complete Terraform Guide](docs/terraform/private-endpoints.md)
- [Checkov on Azure — Getting to 0 Failed Checks](docs/terraform/checkov-azure.md)
- [AzureRM Provider v4 — What Changed](docs/terraform/azurerm-v4.md)

## Best Practices

Production-ready patterns for common Azure workloads:

- [Security Baseline — Every Stack Needs This](docs/best-practices/security-baseline.md)
- [Monitoring Baseline — Diagnostic Settings Done Right](docs/best-practices/monitoring-baseline.md)
- [FinOps — Budget Alerts and Cost Control](docs/best-practices/finops-baseline.md)
- [CAF Naming Convention — Practical Guide](docs/best-practices/caf-naming.md)

## Architecture Patterns

Reference architectures for common Azure deployments:

- [Hub-Spoke Network](docs/architecture/hub-spoke.md)
- [App Service + Database](docs/architecture/appservice-database.md)
- [AKS Platform](docs/architecture/aks-platform.md)

---

## About the author

**Frédéric Leroy** — Microsoft Certified Trainer, Azure Solutions Architect Expert

Building [AethronOps](https://aethronops.com) — a platform that generates production-ready Azure Terraform stacks with built-in compliance documentation.

## Contributing

Found an error? Open an issue or submit a PR.

## License

[Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/)
