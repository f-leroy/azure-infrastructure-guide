# AKS Platform Architecture on Azure

**Version**: 1.0
**Last updated**: 2026-03-18
**Category**: Workload / Container Platform
**Cloud Adoption Framework alignment**: Application Platform, Containers

---

## Overview

Azure Kubernetes Service (AKS) is a managed Kubernetes platform that abstracts the control plane and lets teams focus on deploying containerized workloads. This document describes three deployment patterns -- public cluster, private cluster, and private cluster with Azure Firewall -- and the supporting infrastructure required for production readiness.

AKS is the right choice when you need container orchestration, microservice isolation, advanced scaling (HPA, KEDA), service mesh, or multi-team workload scheduling. For simpler containerized applications, consider Azure Container Apps instead.

### Deployment Patterns

| Pattern | API Server Access | Outbound | Use case |
|---|---|---|---|
| **Public cluster** | Public endpoint + authorized IPs | LoadBalancer (SNAT) | Dev/staging, small teams |
| **Private cluster** | Private endpoint only | LoadBalancer (SNAT) | Production, compliance |
| **Private + Firewall** | Private endpoint only | UserDefinedRouting via Azure Firewall | Enterprise, regulated industries |

---

## Architecture Diagram

### Private Cluster with Azure Firewall (Enterprise)

```
                    Internet
                       |
              +--------+--------+
              |  Azure Firewall  |
              |  (Hub VNet)      |
              +--------+--------+
                       |
                  UDR 0.0.0.0/0
                       |
    +──────────────────────────────────────+
    |        AKS VNet (10.1.0.0/16)        |
    |                                      |
    |  +--------------------------------+  |
    |  | AKS Node Subnet               |  |
    |  | 10.1.0.0/22  (1019 usable IPs) |  |
    |  |                                |  |
    |  |  System Pool    User Pool      |  |
    |  |  (3 nodes)      (3-10 nodes)   |  |
    |  +--------------------------------+  |
    |                                      |
    |  +--------------------------------+  |
    |  | AKS API Server Subnet         |  |
    |  | 10.1.4.0/28                   |  |
    |  | (Private Endpoint for API)    |  |
    |  +--------------------------------+  |
    |                                      |
    |  +--------------------------------+  |
    |  | Private Endpoints Subnet       |  |
    |  | 10.1.5.0/24                   |  |
    |  | (ACR, Key Vault, Storage PEs) |  |
    |  +--------------------------------+  |
    |                                      |
    |  +--------------------------------+  |
    |  | Internal LB Subnet (optional)  |  |
    |  | 10.1.6.0/24                   |  |
    |  +--------------------------------+  |
    +──────────────────────────────────────+

    Supporting services (connected via PE):

    +----------+  +----------+  +---------------+
    | Azure    |  | Key      |  | Log Analytics |
    | Container|  | Vault    |  | Workspace     |
    | Registry |  |          |  | + Container   |
    | (Premium)|  |          |  |   Insights    |
    +----------+  +----------+  +---------------+
```

---

## Components

### AKS Cluster

| Property | Public | Private | Private + Firewall |
|---|---|---|---|
| API server | Public + authorized IPs | Private endpoint | Private endpoint |
| Private cluster | No | Yes | Yes |
| Outbound type | `loadBalancer` | `loadBalancer` | `userDefinedRouting` |
| Network plugin | Azure CNI Overlay | Azure CNI Overlay | Azure CNI Overlay |
| Network policy | Calico | Calico | Calico |
| SKU tier | Free | Standard | Standard |
| Kubernetes version | Current stable | Current stable | Current stable |
| Automatic upgrades | `patch` | `patch` | `patch` |

**Kubernetes Version Policy**:

AKS supports N-2 minor versions from the latest GA release. As of early 2026, the current stable versions are 1.30.x, 1.31.x, and 1.32.x. AKS also offers Long Term Support (LTS) for version 1.32 and select future versions, extending community support by one year with Microsoft-managed CVE patches.

Recommended strategy:
- **Production**: Use the latest N-1 stable version with `automatic_upgrade_channel = "patch"`.
- **Development**: Use the latest stable version to test upcoming changes.
- **Regulated**: Use LTS versions for maximum stability window.

### Node Pools

| Pool | VM Size | Count | Purpose |
|---|---|---|---|
| System | Standard_D4ds_v5 | 3 (fixed) | CoreDNS, konnectivity, metrics-server |
| User (general) | Standard_D4ds_v5 | 3-10 (autoscale) | Application workloads |
| User (compute) | Standard_F8s_v2 | 0-5 (autoscale) | CPU-intensive workloads (optional) |

Key node pool settings:
- System pool: `CriticalAddonsOnly=true:NoSchedule` taint to prevent application pods from scheduling.
- All pools: Availability Zones 1, 2, 3 for zone-redundant deployment.
- Ephemeral OS disk (`os_disk_type = "Ephemeral"`) for faster node operations.
- Max pods per node: 50 (Azure CNI Overlay default).

### Azure Container Registry (ACR)

| Property | Value |
|---|---|
| SKU | Premium (required for Private Endpoint and geo-replication) |
| Admin user | Disabled (`admin_enabled = false`) |
| Anonymous pull | Disabled |
| Network access | Private Endpoint only (Standard/Premium tiers) |
| Authentication | AKS kubelet identity with AcrPull role |
| Content trust | Enabled for production images |
| Geo-replication | Optional, for multi-region deployments |

AKS authenticates to ACR using its kubelet managed identity. No Docker credentials are stored in the cluster.

### Key Vault

Secrets, TLS certificates, and encryption keys are stored in Key Vault and accessed by pods via:

1. **Secrets Store CSI Driver** (recommended): Mounts Key Vault secrets as volumes in pods.
2. **Workload Identity** + application SDK: Pods authenticate directly to Key Vault using federated tokens.

| Property | Value |
|---|---|
| SKU | Standard |
| Soft delete | 90 days |
| Purge protection | Enabled |
| RBAC mode | Enabled |
| Network | Private Endpoint |

### Log Analytics + Container Insights

Container Insights provides monitoring for the AKS cluster using a managed identity-based data collection rule (DCR). The legacy OMS Agent approach is deprecated.

| Component | Purpose |
|---|---|
| Log Analytics Workspace | Central log sink |
| Container Insights (managed identity mode) | Pod logs, node metrics, Kubernetes events |
| Prometheus (Azure Monitor managed) | Metrics collection for Grafana dashboards |
| Azure Managed Grafana | Visualization (optional) |

Data collected:
- Container stdout/stderr logs.
- Kubernetes events (pod scheduling, failures, scaling).
- Node-level metrics (CPU, memory, disk, network).
- Kubernetes API audit logs (optional, high volume).

**Cost control**: Configure `ContainerLogV2` schema and set log filtering rules to exclude noisy namespaces (e.g., `kube-system` informational logs).

---

## Networking Deep Dive

### Azure CNI Overlay vs Azure CNI vs Kubenet

| Feature | Azure CNI Overlay | Azure CNI | Kubenet |
|---|---|---|---|
| Pod IPs from | Overlay network (private) | VNet subnet | Bridge network |
| VNet IP consumption | Nodes only | Nodes + pods | Nodes only |
| Pod-to-VNet routing | NAT through node | Direct | NAT through node |
| Network policies | Calico, Azure NPM | Calico, Azure NPM, Cilium | Calico |
| Max pods/node | 250 | 250 | 110 |
| Windows node pools | Yes | Yes | No |

**Recommendation**: Azure CNI Overlay for most deployments. It provides the scalability of kubenet with the feature set of Azure CNI, without consuming VNet IPs for every pod.

### Subnet Sizing

| Subnet | CIDR | Usable IPs | Calculation |
|---|---|---|---|
| AKS Nodes | /22 | 1,019 | Max 200 nodes (includes autoscale headroom) |
| API Server (private) | /28 | 11 | API server PE (Azure manages this) |
| Private Endpoints | /24 | 251 | ACR, Key Vault, Storage, database PEs |
| Internal LB | /24 | 251 | Internal LoadBalancer services |

**Azure CNI Overlay advantage**: Since pod IPs come from a private overlay (default 10.244.0.0/16), you do not need to size the node subnet for pods. A /22 is generous for nodes alone.

### Private Cluster

In a private cluster, the API server is exposed only through a Private Endpoint in the AKS VNet. This means:

- `kubectl` commands must be run from a network that can reach the Private Endpoint (VPN, Bastion + jump box, or Azure DevOps self-hosted agent in the VNet).
- CI/CD pipelines need a self-hosted runner/agent with VNet connectivity.
- DNS resolution for `*.privatelink.<region>.azmk8s.io` must work from the calling network.

### Outbound Traffic Patterns

#### LoadBalancer (default)

AKS provisions an Azure Load Balancer with a public IP for outbound SNAT. Simple but provides no outbound traffic inspection.

```
Pod → Node → AKS Load Balancer (SNAT) → Internet
```

#### UserDefinedRouting (enterprise)

All outbound traffic is routed through Azure Firewall in a hub VNet via UDR. The firewall controls which FQDNs and ports are accessible.

```
Pod → Node → UDR 0.0.0.0/0 → Azure Firewall (Hub) → Internet
```

Required Azure Firewall application rules for AKS:

| FQDN / Pattern | Port | Purpose |
|---|---|---|
| `*.hcp.<region>.azmk8s.io` | 443 | AKS API server |
| `mcr.microsoft.com` | 443 | Microsoft Container Registry |
| `*.data.mcr.microsoft.com` | 443 | MCR data endpoint |
| `management.azure.com` | 443 | Azure management API |
| `login.microsoftonline.com` | 443 | Entra ID authentication |
| `packages.microsoft.com` | 443 | Microsoft packages |
| `acs-mirror.azureedge.net` | 443 | AKS required packages |
| `*.docker.io` | 443 | Docker Hub (if needed) |
| `dc.services.visualstudio.com` | 443 | Container Insights telemetry |

---

## Identity Architecture

### Cluster-Level Identity

| Identity | Type | Purpose |
|---|---|---|
| Cluster identity | System-Assigned MI | AKS control plane operations (VNet, LB, disks) |
| Kubelet identity | User-Assigned MI | Node-to-ACR authentication, disk encryption |
| Ingress MI | User-Assigned MI | App Gateway Ingress Controller (if used) |

### Pod-Level Identity (Workload Identity)

Azure AD Workload Identity is the recommended approach for pod-to-Azure-service authentication. It replaces the deprecated AAD Pod Identity v1.

How it works:

1. A Kubernetes ServiceAccount is annotated with the Azure User-Assigned MI client ID.
2. A Federated Identity Credential links the Kubernetes OIDC issuer to the Azure MI.
3. Pods using that ServiceAccount receive a projected token that Azure services accept.

```yaml
# Kubernetes ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  annotations:
    azure.workload.identity/client-id: "<managed-identity-client-id>"
```

```hcl
# Terraform: Federated Identity Credential
resource "azurerm_federated_identity_credential" "my_app" {
  name                = "my-app-k8s"
  resource_group_name = azurerm_resource_group.main.name
  parent_id           = azurerm_user_assigned_identity.my_app.id
  audience            = ["api://AzureADTokenExchange"]
  issuer              = azurerm_kubernetes_cluster.main.oidc_issuer_url
  subject             = "system:serviceaccount:my-namespace:my-app"
}
```

### Azure RBAC for Kubernetes

Instead of managing Kubernetes ClusterRoleBindings manually, Azure RBAC for Kubernetes authorization allows managing access through Azure role assignments:

| Azure Role | Kubernetes equivalent | Use case |
|---|---|---|
| Azure Kubernetes Service RBAC Reader | view ClusterRole | Read-only access for operators |
| Azure Kubernetes Service RBAC Writer | edit ClusterRole | Deploy workloads |
| Azure Kubernetes Service RBAC Admin | admin ClusterRole | Namespace administration |
| Azure Kubernetes Service RBAC Cluster Admin | cluster-admin | Full cluster control |

Enable with `azure_rbac_enabled = true` and `local_account_disabled = true` to enforce Entra ID authentication only.

---

## Monitoring Architecture

### Container Insights (Managed Identity Mode)

Container Insights is configured using a Data Collection Rule (DCR) and Data Collection Endpoint (DCE), authenticated via managed identity. This replaces the legacy `omsagent` DaemonSet.

```hcl
resource "azurerm_monitor_data_collection_rule" "aks" {
  name                = "dcr-${local.name_prefix}-aks"
  resource_group_name = azurerm_resource_group.main.name
  location            = var.location
  kind                = "Linux"

  destinations {
    log_analytics {
      workspace_resource_id = module.log_analytics.resource_id
      name                  = "la-destination"
    }
  }

  data_flow {
    streams      = ["Microsoft-ContainerLogV2"]
    destinations = ["la-destination"]
  }

  data_sources {
    extension {
      name           = "ContainerInsightsExtension"
      extension_name = "ContainerInsights"
      streams        = ["Microsoft-ContainerLogV2"]
    }
  }
}
```

### Recommended Alerts

| Alert | Condition | Severity |
|---|---|---|
| Node CPU > 80% | Avg over 5 min | Warning (Sev 2) |
| Node memory > 80% | Avg over 5 min | Warning (Sev 2) |
| Pod restart count > 5 | Sum over 15 min | Warning (Sev 2) |
| Node NotReady | Any node NotReady > 5 min | Critical (Sev 1) |
| OOM killed containers | Count > 0 over 5 min | Warning (Sev 2) |
| Persistent volume usage > 80% | Avg over 15 min | Warning (Sev 2) |
| API server latency > 1s | P99 over 5 min | Warning (Sev 2) |
| Failed pod scheduling | Count > 0 over 5 min | Informational (Sev 3) |

---

## Kubernetes Version Support

### Current Support Matrix (early 2026)

| Version | Status | End of support |
|---|---|---|
| 1.32.x | Current (GA) | ~March 2027 (community), ~March 2028 (LTS) |
| 1.31.x | Supported (N-1) | ~December 2026 |
| 1.30.x | Supported (N-2) | ~September 2026 |
| 1.29.x | End of life | Ended |

### Long Term Support (LTS)

AKS LTS extends support for select Kubernetes versions by one year beyond community end-of-life. During LTS, Microsoft provides:

- Security patches (CVEs) for Kubernetes components.
- Bug fixes for AKS-specific components.
- No new features or minor version bumps.

LTS requires AKS Standard or Premium tier. It is designed for workloads that cannot upgrade frequently due to certification, testing, or regulatory requirements.

### Upgrade Strategy

```hcl
resource "azurerm_kubernetes_cluster" "main" {
  # ...
  automatic_upgrade_channel       = "patch"    # Auto-apply patch versions
  node_os_upgrade_channel         = "NodeImage" # Auto-update node OS images
  maintenance_window {
    allowed {
      day   = "Sunday"
      hours = [2, 6]  # UTC
    }
  }
}
```

---

## Security Considerations

### MCSB Control Mapping

| Control | Implementation |
|---|---|
| NS-1 (Network segmentation) | Dedicated VNet, node/PE/LB subnets, Calico network policies |
| NS-2 (Secure cloud services) | Private Endpoints for ACR, Key Vault, database; private cluster |
| NS-3 (Edge firewall) | Azure Firewall with UDR for all outbound (enterprise pattern) |
| IM-3 (Application identities) | Workload Identity for pods, kubelet MI for ACR, no stored credentials |
| PA-7 (Least privilege) | Azure RBAC for Kubernetes, namespace-scoped roles, local accounts disabled |
| DP-3 (Encryption in transit) | mTLS via service mesh (optional), TLS ingress, HTTPS-only registry |
| DP-4 (Encryption at rest) | AKS managed disk encryption, etcd encryption at rest |
| DP-6 (Secure key management) | Key Vault + Secrets Store CSI Driver |
| LT-3 (Security logging) | Container Insights, Kubernetes audit logs, diagnostic settings |
| LT-5 (Centralized logging) | Log Analytics Workspace for all cluster and infrastructure logs |
| ES-1 (Endpoint protection) | Microsoft Defender for Containers |

### Additional Security Controls

| Control | Configuration |
|---|---|
| Pod Security Standards | `restricted` profile enforced via admission controller |
| Image scanning | Defender for Containers scans images in ACR and at runtime |
| Network policies | Calico policies for namespace isolation (deny-all default, allow explicit) |
| Secrets encryption | Secrets Store CSI Driver; never store secrets in Kubernetes Secrets directly |
| Node access | No SSH access to nodes; use `kubectl debug node/` for troubleshooting |
| API server access | Private endpoint + authorized IP ranges (no public access in production) |
| Local accounts | Disabled (`local_account_disabled = true`); Entra ID only |
| Disk encryption | Host-based encryption for temp disks and cached data |

### RGPD Alignment

| Article | Implementation |
|---|---|
| Art. 25 (Privacy by design) | Private cluster, workload identity, encryption, namespace isolation |
| Art. 32 (Security of processing) | Defender for Containers, network policies, pod security standards |
| Art. 33 (Breach notification) | Container Insights alerts, Defender for Containers runtime protection |

---

## Terraform File Structure

```
aks-platform/
  main.tf                 # terraform block, providers, locals (naming, tags)
  resource_group.tf       # Resource group
  networking.tf           # VNet, subnets, NSGs, route tables, Private Endpoints
  aks.tf                  # AKS cluster, node pools, maintenance window
  identity.tf             # Managed Identities (kubelet, workload), federated credentials
  keyvault.tf             # Key Vault for cluster secrets and certificates
  monitoring.tf           # Log Analytics, Container Insights DCR, diagnostic settings
  security.tf             # Azure Firewall rules for AKS (enterprise, count-gated)
  storage.tf              # Storage Account for persistent volumes (if needed)
  variables.tf            # All input variables with validation
  outputs.tf              # Cluster name, FQDN, kubelet identity, OIDC issuer URL
  terraform.tfvars        # Environment-specific values
  backend.tf.example      # Remote state backend template
  finops.tf               # Budget alerts, node pool auto-shutdown (dev)
  .checkov.yaml           # Security exceptions with documented justifications
  README.md               # Stack documentation
  COMPLIANCE.md           # Full compliance mapping
  manifest.yaml           # Stack metadata
```

### Key Terraform patterns

**aks.tf** contains the AKS cluster and node pool definitions:

```hcl
module "aks" {
  source  = "Azure/avm-res-containerservice-managedcluster/azurerm"
  version = "~> 0.4"

  name                = "aks-${local.name_prefix}"
  resource_group_name = module.resource_group.name
  location            = var.location

  kubernetes_version = var.kubernetes_version
  sku_tier           = var.aks_sku_tier  # "Free" or "Standard"

  # Private cluster
  private_cluster_enabled             = var.tier != "basic"
  private_cluster_public_fqdn_enabled = false
  private_dns_zone_id                 = var.tier != "basic" ? "System" : null

  # Network
  network_profile = {
    network_plugin      = "azure"
    network_plugin_mode = "overlay"
    network_policy      = "calico"
    outbound_type       = local.is_premium ? "userDefinedRouting" : "loadBalancer"
    pod_cidr            = "10.244.0.0/16"
    service_cidr        = "10.245.0.0/16"
    dns_service_ip      = "10.245.0.10"
  }

  # Identity
  identity_type           = "UserAssigned"
  kubelet_identity_type   = "UserAssigned"
  local_account_disabled  = true
  azure_rbac_enabled      = true

  # Monitoring
  oms_agent_enabled                  = false  # Use managed identity Container Insights
  monitor_metrics_enabled            = true

  enable_telemetry = false

  tags = local.common_tags
}
```

**networking.tf** contains VNet, subnets, and NSGs. The AKS node subnet has an NSG but does NOT have a route table applied by Terraform when using Azure CNI Overlay with `loadBalancer` outbound -- AKS manages its own routes. For `userDefinedRouting`, a UDR pointing to the Azure Firewall is applied.

---

## Cost Estimates

Estimates for West Europe region, as of early 2026.

### Public Cluster / Dev (~EUR 400-600/month)

| Component | Monthly Cost |
|---|---|
| AKS (Free tier) | EUR 0 |
| System pool (3x D2ds_v5) | ~EUR 220 |
| User pool (2x D2ds_v5) | ~EUR 150 |
| ACR (Basic) | ~EUR 5 |
| Key Vault (Standard) | ~EUR 3 |
| Log Analytics (3 GB/day) | ~EUR 60 |
| Load Balancer (Basic) | ~EUR 0 |
| **Total** | **~EUR 440/month** |

### Private Cluster / Production (~EUR 1,200-2,000/month)

| Component | Monthly Cost |
|---|---|
| AKS (Standard tier) | ~EUR 60 |
| System pool (3x D4ds_v5) | ~EUR 440 |
| User pool (3x D4ds_v5) | ~EUR 440 |
| ACR (Premium) | ~EUR 130 |
| Key Vault (Standard) | ~EUR 5 |
| Log Analytics (10 GB/day) | ~EUR 150 |
| VNet + Private Endpoints (4 PEs) | ~EUR 30 |
| Load Balancer (Standard) | ~EUR 20 |
| **Total** | **~EUR 1,275/month** |

### Private + Firewall / Enterprise (~EUR 3,000-5,000/month)

| Component | Monthly Cost |
|---|---|
| AKS (Standard tier) | ~EUR 60 |
| System pool (3x D4ds_v5) | ~EUR 440 |
| User pool (5x D4ds_v5, autoscale) | ~EUR 740 |
| Compute pool (2x F8s_v2, optional) | ~EUR 350 |
| ACR (Premium, geo-replicated) | ~EUR 260 |
| Azure Firewall (Standard, shared) | ~EUR 900 (amortized) |
| Bastion (Standard, shared) | ~EUR 330 (amortized) |
| Key Vault (Standard) | ~EUR 5 |
| Log Analytics (20 GB/day) | ~EUR 200 |
| Defender for Containers | ~EUR 60 |
| VNet + Private Endpoints (6 PEs) | ~EUR 40 |
| Load Balancer (Standard) | ~EUR 20 |
| **Total** | **~EUR 3,405/month** |

> **Note**: In a hub-spoke environment, Azure Firewall and Bastion costs are shared across all workloads. The AKS-specific cost in an existing hub is approximately EUR 2,175/month for the enterprise pattern. Autoscaling node pools significantly affect cost -- the estimates above use average utilization.

---

## Operational Recommendations

### Day 2 Operations

| Task | Frequency | Method |
|---|---|---|
| Kubernetes patch upgrades | Automatic (patch channel) | AKS automatic upgrade |
| Kubernetes minor upgrades | Every 3-4 months | Planned maintenance window |
| Node OS updates | Weekly | NodeImage upgrade channel |
| ACR image scanning | Continuous | Defender for Containers |
| Certificate rotation | Automatic | AKS managed (90-day rotation) |
| Log review | Daily | Log Analytics saved queries, workbooks |
| Cost review | Weekly | Azure Cost Management + budget alerts |

### Scaling Guidelines

| Dimension | Mechanism | Trigger |
|---|---|---|
| Pod horizontal scaling | HPA (Horizontal Pod Autoscaler) | CPU/memory/custom metrics |
| Pod vertical scaling | VPA (Vertical Pod Autoscaler) | Resource recommendation |
| Node scaling | Cluster Autoscaler | Pending pods unable to schedule |
| Event-driven scaling | KEDA | Queue depth, event count, custom triggers |

---

## References

- [AKS documentation](https://learn.microsoft.com/en-us/azure/aks/)
- [AKS baseline architecture](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks/baseline-aks)
- [AKS private cluster](https://learn.microsoft.com/en-us/azure/aks/private-clusters)
- [AKS network concepts](https://learn.microsoft.com/en-us/azure/aks/concepts-network)
- [Azure CNI Overlay](https://learn.microsoft.com/en-us/azure/aks/azure-cni-overlay)
- [Workload Identity on AKS](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview)
- [AKS monitoring with managed identity](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-onboard)
- [Microsoft Cloud Security Benchmark v1](https://learn.microsoft.com/en-us/security/benchmark/azure/overview)

---

*Maintained by [Frederic Leroy](https://github.com/f-leroy) -- MCT, Azure Solutions Architect Expert*
