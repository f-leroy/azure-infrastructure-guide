# Key Vault RBAC Mandatory — February 2027

> **Deadline: February 27, 2027**
> All Key Vault API versions prior to 2026-02-01 will be retired.
> New Key Vaults already default to RBAC authorization.

## What changes

Starting with API version 2026-02-01, new Azure Key Vaults default to **RBAC authorization** instead of access policies.

On **February 27, 2027**, all older API versions will stop working entirely. If your Terraform provider uses an older API version, Key Vault operations will fail.

## Impact

### Before (access policies)
```hcl
# Old model — access policies
resource "azurerm_key_vault" "main" {
  # ...
  access_policy {
    tenant_id = data.azurerm_client_config.current.tenant_id
    object_id = data.azurerm_client_config.current.object_id
    secret_permissions = ["Get", "List", "Set", "Delete"]
  }
}
```

### After (RBAC — recommended)
```hcl
# New model — RBAC
resource "azurerm_key_vault" "main" {
  # ...
  enable_rbac_authorization = true
  # No access_policy block needed
}

# Grant access via RBAC role assignment
resource "azurerm_role_assignment" "kv_secrets" {
  scope                = azurerm_key_vault.main.id
  role_definition_name = "Key Vault Secrets Officer"
  principal_id         = azurerm_user_assigned_identity.main.principal_id
}
```

## Why RBAC is better

| Feature | Access Policies | RBAC |
|---------|----------------|------|
| Granularity | Per-vault only | Per-key, per-secret, per-certificate |
| Auditing | Limited | Full Azure RBAC audit trail |
| Consistency | Unique to Key Vault | Same model as all Azure resources |
| Conditional Access | Not supported | Supported via Entra ID |
| Scale | Max 1,024 policies per vault | Unlimited via role assignments |
| Managed Identity | Manual policy creation | Standard role assignment |

## What to do now

1. Set `enable_rbac_authorization = true` on all new Key Vaults
2. Replace `access_policy` blocks with `azurerm_role_assignment` resources
3. Use built-in roles: `Key Vault Secrets Officer`, `Key Vault Crypto Officer`, `Key Vault Certificates Officer`
4. Test with `az keyvault show --name <name> --query properties.enableRbacAuthorization`

## References

- [Prepare for Key Vault API 2026-02-01 — Microsoft Learn](https://learn.microsoft.com/en-us/azure/key-vault/general/access-control-default)
- [Key Vault RBAC migration guide](https://digitalberg.com/azure-key-vault-transition-to-rbac-what-you-must-do-before-february-2027/)
- [Azure RBAC built-in roles for Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-guide)

---

*Maintained by [Frédéric Leroy](https://github.com/f-leroy) — MCT, Azure Solutions Architect Expert*
