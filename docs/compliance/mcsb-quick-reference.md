# MCSB v1 — Quick Reference for Azure Architects

> Microsoft Cloud Security Benchmark v1 — verified control IDs and their scope.
> Use this as a cheat sheet when mapping infrastructure controls to compliance requirements.

## Network Security (NS)

| ID | Control | What it covers | Azure services |
|----|---------|---------------|----------------|
| NS-1 | Network segmentation | VNet, NSG, subnets | Virtual Network, NSG |
| NS-2 | Secure cloud services with network controls | Private Endpoints, disable public access | Private Endpoint, service firewalls |
| NS-3 | Deploy firewall at network edge | Azure Firewall, UDR | Azure Firewall, Route Table |
| NS-4 | Deploy IDS/IPS | Firewall IDPS, Defender for Endpoint | Azure Firewall Premium |
| NS-5 | Deploy DDoS protection | DDoS Protection Standard | DDoS Protection Plan |
| NS-6 | Deploy WAF for web applications | Front Door WAF, App Gateway WAF | WAF Policy |
| NS-7 | Simplify network security configuration | Firewall Manager, Adaptive Hardening | Azure Firewall Manager |
| NS-8 | Detect/disable insecure services | Obsolete protocols | TLS version enforcement |
| NS-9 | Private network connectivity | VPN, ExpressRoute | VPN Gateway, ExpressRoute |
| NS-10 | DNS security | Private DNS, Defender for DNS | Private DNS Zone |

### Common mistakes
- **NS-2 ≠ NS-3** — NS-2 is Private Endpoints, NS-3 is Firewall. Most frequent error.
- **NS-4 ≠ NS-5** — NS-4 is IDS/IPS, NS-5 is DDoS. Don't mix them.
- **NS-7 ≠ DDoS** — NS-7 is about simplifying config, not DDoS L7.

## Identity Management (IM)

| ID | Control | What it covers |
|----|---------|---------------|
| IM-1 | Centralized identity | Azure AD / Entra ID |
| IM-2 | Protect identity systems | Entra ID Protection |
| IM-3 | Manage application identities | Managed Identity (not credentials) |
| IM-4 to IM-9 | SSO, MFA, Conditional Access | Organizational (out of IaC scope) |

### Common mistake
- **IM-2 ≠ IM-3** — IM-2 is protecting the identity platform, IM-3 is using Managed Identity for apps.

## Privileged Access (PA)

| ID | Control | What it covers |
|----|---------|---------------|
| PA-2 | No standing access | JIT, PIM (Bastion for VMs) |
| PA-7 | Least privilege | RBAC role assignments |

## Data Protection (DP)

| ID | Control | What it covers |
|----|---------|---------------|
| DP-1 | Discover/classify sensitive data + encryption at rest | Data classification, AES-256 |
| DP-2 | Monitor anomalies on sensitive data | Audit logs |
| DP-3 | Encryption in transit | TLS 1.2+, HTTPS |
| DP-4 | Encryption at rest by default | TDE, platform-managed keys |
| DP-5 | Customer-managed keys (CMK) | Key Vault CMK |
| DP-6 | Secure key management | Key Vault |
| DP-7 | Secure certificate management | Key Vault certificates |
| DP-8 | Security of key/certificate repository | Purge protection, soft delete |

## Logging & Threat Detection (LT)

| ID | Control | What it covers |
|----|---------|---------------|
| LT-1 | Threat detection | Defender, logs |
| LT-2 | Identity threat detection | Entra ID Protection (NOT Log Analytics) |
| LT-3 | Logging for investigation | Diagnostic settings |
| LT-4 | Network logging | NSG flow logs, Firewall logs |
| LT-5 | Centralize log analysis | SIEM, Log Analytics |
| LT-6 | Log retention | Retention configuration |
| LT-7 | Time synchronization | NTP sources |

### Common mistake
- **LT-2 ≠ Log Analytics** — LT-2 is specifically Entra ID Protection for identity threats.

## Backup & Recovery (BR)

| ID | Control | What it covers |
|----|---------|---------------|
| BR-1 | Automated backups | Backup Vault, PITR |
| BR-2 | Protect backup data | Encryption, soft delete |
| BR-3 | Monitor backups | Backup alerts |
| BR-4 | Test restorations | Restore testing (process) |

## Other

| ID | Control | What it covers |
|----|---------|---------------|
| ES-1 | EDR | Azure Monitor Agent, Defender for Containers |
| DS-1 | Threat modeling | Process, not a tool |
| AM-2 | Use only approved services | Inventory |
| IR-1 | Incident response preparation | IR plan (process) |
| IR-3 | Create incidents from alerts | Alert-to-incident workflow |

### Common mistake
- **DS-1 ≠ Defender for Containers** — DS-1 is a threat modeling PROCESS, not a product.

## References

- [MCSB v1 — Microsoft Learn](https://learn.microsoft.com/en-us/security/benchmark/azure/overview)
- [MCSB v2 Preview (November 2025)](https://learn.microsoft.com/en-us/security/benchmark/azure/overview-v2) — Not yet GA, v1 IDs remain valid

---

*Maintained by [Frédéric Leroy](https://github.com/f-leroy) — MCT, Azure Solutions Architect Expert*
