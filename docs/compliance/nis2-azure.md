# NIS2 — What Your Azure Infrastructure Must Include

> NIS2 Directive (EU 2022/2555) — Article 21(2) security measures mapped to Azure infrastructure controls.
> ~160,000 entities affected across the EU. France transposition expected ~July 2026.

## Article 21(2) — Security Measures

NIS2 requires "appropriate and proportionate technical, operational and organisational measures" in 10 areas. Here's what each means for your Azure infrastructure:

### (a) Risk analysis and information system security policies

| Requirement | Azure implementation |
|-------------|---------------------|
| Risk assessment | Threat modeling process (organizational) |
| Security policies | Azure Policy, RBAC, Conditional Access |
| **Terraform** | `azurerm_policy_assignment`, governance stacks |

### (b) Incident handling

| Requirement | Azure implementation |
|-------------|---------------------|
| Detect incidents | Log Analytics, Application Insights, Azure Monitor alerts |
| Respond to incidents | Alert rules → Action Groups → Logic Apps |
| **Terraform** | Diagnostic settings on all resources, alert rules, action groups |
| **MCSB mapping** | LT-1, LT-3, LT-5, IR-1, IR-3 |

### (c) Business continuity, backup management, disaster recovery

| Requirement | Azure implementation |
|-------------|---------------------|
| Backup | Backup Vault, database PITR, Storage versioning |
| DR | Zone-redundant resources, geo-replication |
| **Terraform** | `backup.tf` with Backup Vault, retention policies |
| **MCSB mapping** | BR-1, BR-2, BR-3 |

### (d) Supply chain security

| Requirement | Azure implementation |
|-------------|---------------------|
| Vendor assessment | Organizational process |
| Software integrity | ACR with image signing (Notation) |
| **Note** | Primarily organizational — infrastructure supports with audit trails |

### (e) Security in network and information systems acquisition, development, maintenance

| Requirement | Azure implementation |
|-------------|---------------------|
| Secure by default | Private Endpoints, TLS 1.2, Managed Identity |
| Vulnerability handling | Defender for Cloud, Checkov |
| **Terraform** | Security-hardened templates, `.checkov.yaml` |
| **MCSB mapping** | NS-1, NS-2, DP-3, DP-4, IM-3 |

### (f) Policies and procedures to assess the effectiveness of security measures

| Requirement | Azure implementation |
|-------------|---------------------|
| Security assessment | Defender Secure Score, compliance dashboards |
| **Terraform** | Monitoring stack with Log Analytics + dashboards |

### (g) Basic cyber hygiene practices and cybersecurity training

| Requirement | Azure implementation |
|-------------|---------------------|
| Hygiene | Organizational process |
| **Note** | Not implementable via IaC |

### (h) Policies and procedures regarding the use of cryptography and encryption

| Requirement | Azure implementation |
|-------------|---------------------|
| Encryption at rest | Platform-managed keys (default), CMK (premium) |
| Encryption in transit | TLS 1.2+, HTTPS-only |
| Key management | Key Vault with purge protection |
| **Terraform** | Key Vault module, TLS settings on all resources |
| **MCSB mapping** | DP-3, DP-4, DP-5, DP-6, DP-8 |

### (i) Human resources security, access control policies, asset management

| Requirement | Azure implementation |
|-------------|---------------------|
| Access control | RBAC, Managed Identity, PIM/JIT |
| Asset management | Resource Groups, tags, Azure Policy |
| **Terraform** | `identity.tf` with Managed Identity, RBAC assignments |
| **MCSB mapping** | IM-3, PA-2, PA-7 |

### (j) Use of MFA, secured communications

| Requirement | Azure implementation |
|-------------|---------------------|
| MFA | Entra ID MFA (organizational) |
| Secured communications | VPN, ExpressRoute, Private Endpoints |
| **Terraform** | VPN Gateway, Private Endpoints in `networking.tf` |
| **MCSB mapping** | NS-2, NS-9 |

## Minimum Azure infrastructure for NIS2

At minimum, a NIS2-aligned Azure deployment should include:

```
✅ Virtual Network with NSG                  → Art. 21(2)(e)
✅ Private Endpoints (public access off)     → Art. 21(2)(e)(h)
✅ Managed Identity (no credentials)         → Art. 21(2)(i)
✅ Key Vault (purge protection, soft delete) → Art. 21(2)(h)
✅ Log Analytics + diagnostic settings       → Art. 21(2)(b)
✅ Application Insights                      → Art. 21(2)(b)
✅ Backup Vault with retention policy        → Art. 21(2)(c)
✅ TLS 1.2+ on all resources                → Art. 21(2)(h)
✅ Budget alerts                             → Art. 21(2)(a)
```

## References

- [NIS2 Directive Full Text (EUR-Lex)](https://eur-lex.europa.eu/eli/dir/2022/2555)
- [NIS2 Implementing Regulation (EU) 2024/2690](https://eur-lex.europa.eu/eli/reg_impl/2024/2690)
- [ENISA Technical Implementation Guidance v1.0](https://www.enisa.europa.eu/publications/nis2-technical-implementation-guidance)
- [Azure NIS2 Compliance Guide](https://azure.microsoft.com/en-us/blog/leverage-microsoft-azure-tools-to-navigate-nis2-compliance/)

---

*Maintained by [Frédéric Leroy](https://github.com/f-leroy) — MCT, Azure Solutions Architect Expert*
