# RGPD/GDPR — Infrastructure Controls on Azure

> **Regulation (EU) 2016/679** — General Data Protection Regulation
> In force since May 25, 2018. This document covers the infrastructure-relevant articles only.

## Relevant Articles

| Article | Topic | Infrastructure? |
|---------|-------|:---:|
| Art. 5 | Data minimization, storage limitation | Yes |
| Art. 25 | Privacy by design and by default | Yes |
| Art. 30 | Records of processing (audit logs) | Yes |
| Art. 32 | Security of processing | Yes |
| Art. 33 | Breach notification (72h) | Yes |
| Art. 35 | DPIA | No (organizational process) |

---

## Article 5 — Principles

| Principle | Azure Implementation | MCSB |
|-----------|---------------------|------|
| Data minimization | Deploy in EU regions only | DP-1 |
| Storage limitation | Storage lifecycle policies, auto-delete | LT-6 |
| Integrity & confidentiality | Encryption at rest + in transit | DP-3, DP-4 |
| Accountability | Resource tagging (`data_classification`, `owner`) | — |

## Article 25 — Privacy by Design

| Measure | Azure Implementation | MCSB |
|---------|---------------------|------|
| Network isolation | Private Endpoints, public access off | NS-2 |
| No credentials | User-Assigned Managed Identity | IM-3 |
| Encryption default | Platform-managed encryption | DP-4 |
| Least privilege | RBAC minimal role assignments | PA-7 |
| Secrets management | Key Vault (purge protection, soft delete) | DP-6, DP-8 |
| Network segmentation | NSG with deny-all default | NS-1 |

## Article 30 — Records of Processing

| Measure | Azure Implementation | MCSB |
|---------|---------------------|------|
| Operation logging | Azure Activity Log | LT-3 |
| Data plane audit | Diagnostic settings → Log Analytics | LT-3 |
| Centralized logs | Log Analytics Workspace | LT-5 |
| Retention | 30d dev / 90d standard / 365d premium | LT-6 |
| Immutable trail | Storage with immutability policies | LT-6 |

## Article 32 — Security of Processing

| Clause | Azure Implementation | MCSB |
|--------|---------------------|------|
| (a) Encryption at rest | Platform keys / CMK via Key Vault | DP-4, DP-5 |
| (a) Encryption in transit | TLS 1.2+, HTTPS-only | DP-3 |
| (b) Confidentiality | Private Endpoints + NSG | NS-2 |
| (b) Integrity | Key Vault purge protection | DP-8 |
| (b) Availability | Zone-redundant deployments | BR-1 |
| (c) Restore capability | Backup Vault, PITR, Storage soft delete | BR-1, BR-2 |
| (d) Test effectiveness | Checkov + Defender Secure Score | — |

## Article 33 — Breach Notification (72h)

| Measure | Azure Implementation | MCSB |
|---------|---------------------|------|
| Threat detection | Defender for Cloud | LT-1 |
| Anomaly alerting | Azure Monitor alerts | LT-1 |
| Real-time notification | Action Groups (email, SMS, webhook) | IR-3 |
| Forensic data | Log Analytics (90-365d retention) | LT-3, LT-5 |
| Network forensics | NSG Flow Logs | LT-4 |

---

## Minimum Infrastructure Checklist

### Network
- [ ] Private Endpoints on all PaaS services
- [ ] Public access disabled
- [ ] NSG with deny-all default
- [ ] TLS 1.2+ enforced
- [ ] EU regions only

### Identity
- [ ] Managed Identity (no credentials in code)
- [ ] Key Vault with purge protection + soft delete 90 days
- [ ] RBAC least privilege

### Monitoring
- [ ] Log Analytics + diagnostic settings on all resources
- [ ] Application Insights for web apps
- [ ] Alert rules + action groups
- [ ] Log retention: 90d minimum

### Backup
- [ ] Backup Vault with automated policies
- [ ] Database PITR enabled
- [ ] Storage soft delete + versioning

### Documentation
- [ ] Resources tagged: `data_classification`, `environment`, `owner`
- [ ] Infrastructure validated with Checkov

---

## References

- [RGPD Full Text — EUR-Lex](https://eur-lex.europa.eu/eli/reg/2016/679/oj)
- [Azure GDPR Compliance](https://learn.microsoft.com/en-us/compliance/regulatory/gdpr)
- [CNIL — Guide RGPD développeurs](https://www.cnil.fr/fr/guide-rgpd-du-developpeur)
- [MCSB v1](https://learn.microsoft.com/en-us/security/benchmark/azure/overview)

---

*Maintained by [Frédéric Leroy](https://github.com/f-leroy) — MCT, Azure Solutions Architect Expert*
