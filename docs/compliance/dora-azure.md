# DORA — Azure Requirements for Financial Services

> **Regulation (EU) 2022/2554** — Digital Operational Resilience Act
> In force since January 17, 2025. Applies to 22,000+ financial entities and critical ICT providers.

## Overview

DORA establishes a unified framework for ICT risk management across the European financial sector. It applies to banks, insurers, investment firms, payment institutions, and crypto-asset service providers — as well as their critical ICT third-party service providers.

Microsoft was designated as a critical ICT provider in November 2025.

**DORA is lex specialis for the financial sector** — it overrides NIS2 for financial entities.

---

## Infrastructure-Relevant Articles

### Article 9 — Protection and Prevention

| Requirement | Azure Implementation | MCSB |
|---|---|---|
| Network segmentation | VNet + NSG + subnet isolation | NS-1 |
| Restrict access | Private Endpoints, disable public access | NS-2 |
| Perimeter protection | Azure Firewall with IDPS | NS-3, NS-4 |
| Encryption in transit | TLS 1.2+ on all resources | DP-3 |
| Encryption at rest | Platform keys minimum, CMK for sensitive | DP-4, DP-5 |
| Key management | Key Vault with purge protection | DP-6, DP-8 |
| Identity | User-assigned Managed Identity | IM-3 |
| Privileged access | Bastion for VM access, JIT | PA-2 |
| Least privilege | RBAC role assignments | PA-7 |

### Article 10 — Detection

| Requirement | Azure Implementation | MCSB |
|---|---|---|
| Anomaly detection | Log Analytics + alert rules | LT-1 |
| App monitoring | Application Insights | LT-1 |
| Metric alerting | Azure Monitor metric alerts | LT-1, IR-3 |
| Network detection | NSG flow logs + Network Watcher | LT-4 |
| Security events | Microsoft Defender for Cloud | LT-1 |
| Centralized analysis | Log Analytics as SIEM sink | LT-5 |

### Article 11 — Response and Recovery

| Requirement | Azure Implementation | MCSB |
|---|---|---|
| Business continuity | Zone-redundant deployments | BR-1 |
| DB recovery | Point-in-Time Restore (PITR) | BR-1 |
| Backup | Azure Backup Vault | BR-1, BR-2 |
| Backup protection | Soft delete, zone-redundant vault | BR-2 |
| Recovery testing | Documented procedures | BR-4 |

### Article 12 — Backup Policies

| Requirement | Azure Implementation | MCSB |
|---|---|---|
| DB backup | Automated with configurable retention | BR-1 |
| Storage protection | Blob versioning + soft delete | BR-1, BR-2 |
| Lifecycle management | Storage lifecycle policies | BR-1 |
| Key Vault backup | Soft delete 90 days + purge protection | DP-8 |
| Cross-region redundancy | GRS/ZRS for backup data | BR-2 |

### Article 13 — Logging

| Requirement | Azure Implementation | MCSB |
|---|---|---|
| Comprehensive logging | Diagnostic settings on ALL resources | LT-3 |
| Retention | 30d dev / 90d standard / 365d premium | LT-6 |
| Network logging | NSG flow logs, Firewall logs | LT-4 |
| Audit trail | Activity logs → Log Analytics | LT-3, LT-5 |
| Log integrity | Immutable storage (optional) | LT-6 |

### Article 17 — ICT Incident Management

| Requirement | Azure Implementation | MCSB |
|---|---|---|
| Incident detection | Azure Monitor alerts + action groups | IR-3 |
| Classification | Alert severity levels (0-4) | IR-3 |
| Notification | Action groups (email, SMS, webhook) | IR-3 |
| Automated response | Logic Apps (optional) | IR-1 |
| SIEM | Sentinel (recommended, not required) | LT-1, LT-5 |

> **Note:** Full Art. 17 compliance requires organizational processes beyond IaC. Azure Monitor provides the technical foundation.

### Articles 28-44 — Third-Party Risk

Out of scope for IaC. Requires contractual review, concentration risk assessment, and exit strategy documentation. See [Microsoft DORA compliance page](https://learn.microsoft.com/en-us/compliance/regulatory/offering-dora).

---

## Minimum Infrastructure Checklist

- [ ] VNet + NSG + Private Endpoints on all PaaS services
- [ ] Azure Firewall with IDPS
- [ ] TLS 1.2+ enforced everywhere
- [ ] Managed Identity (no credentials in code)
- [ ] Key Vault with purge protection, soft delete 90 days
- [ ] RBAC with least privilege
- [ ] Bastion for VM access
- [ ] Log Analytics + diagnostic settings on ALL resources
- [ ] Application Insights for app workloads
- [ ] Log retention: 90d minimum (365d recommended)
- [ ] Backup Vault with automated policies
- [ ] Database PITR enabled
- [ ] Zone-redundant deployments
- [ ] Alert rules + action groups
- [ ] Budget alerts

---

## References

- [DORA Full Text — EUR-Lex](https://eur-lex.europa.eu/eli/reg/2022/2554/oj)
- [Implementing Regulation (EU) 2024/2690](https://eur-lex.europa.eu/eli/reg_impl/2024/2690/oj)
- [Microsoft DORA Compliance](https://learn.microsoft.com/en-us/compliance/regulatory/offering-dora)
- [MCSB v1](https://learn.microsoft.com/en-us/security/benchmark/azure/overview)

---

*Maintained by [Frédéric Leroy](https://github.com/f-leroy) — MCT, Azure Solutions Architect Expert*
