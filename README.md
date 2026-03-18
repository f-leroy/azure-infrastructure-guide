# Azure Infrastructure Guide

> A curated, practitioner-maintained guide to building production-ready Azure infrastructure with Terraform.
> Covers breaking changes, compliance frameworks, security best practices, and architecture patterns.

[![Azure](https://img.shields.io/badge/Microsoft%20Azure-0078D4?style=flat&logo=microsoftazure&logoColor=white)](https://azure.microsoft.com)
[![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=flat&logo=terraform&logoColor=white)](https://www.terraform.io)
[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)

---

## Why this guide?

Azure changes fast. Services get retired, security defaults shift, compliance regulations stack up. Keeping infrastructure up-to-date requires constant vigilance.

This guide is maintained by a **Microsoft Certified Trainer** and **Azure Solutions Architect Expert** who actively builds production infrastructure for clients. Everything here is verified against real deployments — not copied from marketing docs.

---

## Contents

### [Azure Breaking Changes 2026-2027](docs/azure-changes/)

Critical changes every Azure architect must know about:

- [March 2026 — VNet Default Outbound Access Retirement](docs/azure-changes/vnet-outbound-retirement.md)
- [March 2026 — VPN Gateway Legacy SKUs Retirement](docs/azure-changes/vpn-gateway-skus.md)
- [April 2026 — Application Gateway v1 Retirement](docs/azure-changes/appgateway-v1-retirement.md)
- [February 2027 — Key Vault RBAC Mandatory](docs/azure-changes/keyvault-rbac-mandatory.md)
- [Full Timeline](docs/azure-changes/timeline.md)

### [Compliance Frameworks for Azure](docs/compliance/)

Infrastructure-level mapping for EU regulations and security benchmarks:

- [NIS2 — What Your Azure Infrastructure Must Include](docs/compliance/nis2-azure.md)
- [DORA — Azure Requirements for Financial Services](docs/compliance/dora-azure.md)
- [RGPD/GDPR — Infrastructure Controls](docs/compliance/rgpd-azure.md)
- [MCSB v1 — Quick Reference for Azure Architects](docs/compliance/mcsb-quick-reference.md)

### [Terraform Best Practices for Azure](docs/terraform/)

Patterns and anti-patterns from building 90+ production stacks:

- [Azure Verified Modules (AVM) — What You Need to Know](docs/terraform/avm-guide.md)
- [Private Endpoints — The Complete Terraform Guide](docs/terraform/private-endpoints.md)
- [Checkov on Azure — Getting to 0 Failed Checks](docs/terraform/checkov-azure.md)
- [AzureRM Provider v4 — What Changed](docs/terraform/azurerm-v4.md)

### [Architecture Best Practices](docs/best-practices/)

Production-ready patterns for common Azure workloads:

- [Security Baseline — Every Stack Needs This](docs/best-practices/security-baseline.md)
- [Monitoring Baseline — Diagnostic Settings Done Right](docs/best-practices/monitoring-baseline.md)
- [FinOps — Budget Alerts and Cost Control](docs/best-practices/finops-baseline.md)
- [CAF Naming Convention — Practical Guide](docs/best-practices/caf-naming.md)

### [Architecture Diagrams](docs/architecture/)

Reference architectures for common Azure patterns:

- [Hub-Spoke Network](docs/architecture/hub-spoke.md)
- [App Service + Database](docs/architecture/appservice-database.md)
- [AKS Platform](docs/architecture/aks-platform.md)

---

## About the author

**Frédéric Leroy** — Microsoft Certified Trainer, Azure Solutions Architect Expert

Certifications: MCT · Azure Solutions Architect Expert · Azure Administrator · Azure Database Administrator · Azure AI Fundamentals · AWS Cloud Practitioner · Oracle Certified Associate

Building [AethronOps](https://aethronops.com) — a platform that generates production-ready Azure Terraform stacks with built-in compliance documentation for 10 frameworks.

---

## Contributing

Found an error? Have a suggestion? Open an issue or submit a PR.

## License

This work is licensed under [Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/).
You are free to share and adapt this material with appropriate credit.
