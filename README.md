# Fabric Prefab

Prefabricated Microsoft Fabric workspace provisioning via REST API. Batch-create Lakehouses, Notebooks, Warehouses, Pipelines, and Dataflows in a single notebook run using Service Principal authentication.

## Why use this?

### Batch provisioning

Standing up a new Fabric environment typically means clicking through the portal dozens of times—create a Lakehouse, create another Lakehouse, create a Notebook, create a Pipeline, repeat. With Fabric Prefab, you define all items in code and create them in one execution. Need 3 Lakehouses, 10 Notebooks, a Warehouse, and 5 Pipelines? Uncomment the lines and run the notebook. Done.

### SPN ownership prevents production outages

**This is the bigger reason to use this tool.**

When a user creates Fabric items, that user becomes the **owner**. Fabric items use the owner's identity to access OneLake and other resources. This creates serious problems:

| Scenario | What happens |
|----------|--------------|
| Owner leaves the organization | Items stop working immediately |
| Owner doesn't sign in for 90 days | Items can stop working |
| Owner of a Warehouse doesn't sign in for 30 days | Warehouse stops working (Fabric requires owner sign-in every 30 days for security token refresh) |

Ownership transfer exists but is painful:
- Must be done item-by-item in most cases
- Some items (mirrored databases) don't support ownership transfer at all
- Pipeline child items (notebooks, etc.) must be transferred separately
- Connections using the previous owner's credentials break and must be manually reconfigured

**Items created by a Service Principal are owned by the SPN.** The SPN doesn't go on vacation, doesn't leave the company, and doesn't forget to sign in. The 30-day token refresh can be automated via API calls. Your production environment stays stable.

> **What is a Service Principal?**  
> A non-interactive identity for applications/automation. Unlike user accounts, SPNs don't require MFA, don't have expiring passwords (secrets do, but you control when), and can be scoped precisely.  
> Video explainer: [Service Principals in Microsoft Fabric](https://www.youtube.com/watch?v=PkLrKDW9gY8)

---

> **📋 New to Service Principals?**  
> Instructions for creating an SPN, setting up the required security group, and configuring tenant settings are at the bottom of this document. See:
> - [Creating a Service Principal](#creating-a-service-principal)
> - [Creating a Security Group](#creating-a-security-group-and-adding-the-spn)
> - [Limitations](#limitations)
> - [References](#references)

---

## Prerequisites

Before using this notebook, ensure the following are in place:

### 1. Service Principal with client secret
A registered application in Microsoft Entra ID. See [Creating a Service Principal](#creating-a-service-principal) below.

### 2. Security Group containing the SPN
**Required.** Fabric tenant settings restrict SPN access to members of specified security groups—you cannot enable the tenant setting for "the entire organization" and expect individual SPNs to work. The SPN must be in a security group that is explicitly allowed. See [Creating a Security Group](#creating-a-security-group-and-adding-the-spn) below.

### 3. Tenant setting enabled
A Fabric Administrator must enable **"Service principals can use Fabric APIs"** in Developer settings and add your security group. See [Configuring the Tenant Setting](#configuring-the-tenant-setting) below.

### 4. Workspace access
The SPN must have **Contributor** (minimum), **Member**, or **Admin** role on the target workspace:
1. Open the workspace in Fabric
2. Click **Manage access**
3. Click **+ Add people or groups**
4. Search for the SPN by name (it appears like a user)
5. Select **Contributor** role
6. Click **Add**

### 5. Credentials available
Have the following ready (stored in Azure Key Vault recommended):
- Tenant ID
- Application (Client) ID
- Client Secret

---

## Notebook Usage

### Supported Item Types

- Lakehouse (standard or schema-enabled)
- Notebook
- Warehouse
- Data Pipeline
- Dataflow Gen2

### Authentication Options

#### Option A: Key Vault (Recommended for production)

```python
import notebookutils.credentials as cred

KEY_VAULT_NAME = "your-keyvault-name"
KEY_VAULT_URL = f"https://{KEY_VAULT_NAME}.vault.azure.net/"

TENANT_ID = cred.getSecret(KEY_VAULT_URL, "aad-tenant-id")
SP_CLIENT_ID = cred.getSecret(KEY_VAULT_URL, "fabric-spn-client-id")
SP_CLIENT_SECRET = cred.getSecret(KEY_VAULT_URL, "fabric-spn-client-secret")
```

#### Option B: Manual Entry (Development only)

**⚠️ Never commit credentials to source control.**

```python
TENANT_ID = "your-tenant-id"
SP_CLIENT_ID = "your-client-id"
SP_CLIENT_SECRET = "your-client-secret"
```

### Creating Items

Uncomment the desired lines in the Execution section. Create as many items as you need in a single run:

```python
# Lakehouses (set enable_schemas=True for schema-enabled)
create_lakehouse(access_token, "lh_bronze", enable_schemas=True)
create_lakehouse(access_token, "lh_silver", enable_schemas=True)
create_lakehouse(access_token, "lh_gold", enable_schemas=True)

# Notebooks
create_notebook(access_token, "nb_100_ingest_raw")
create_notebook(access_token, "nb_200_transform")
create_notebook(access_token, "nb_300_load_warehouse")
create_notebook(access_token, "nb_900_run_all")

# Warehouses
create_warehouse(access_token, "wh_reporting")

# Pipelines
create_pipeline(access_token, "pl_100_orchestration")
create_pipeline(access_token, "pl_200_daily_refresh")

# Dataflows
create_dataflow_gen2(access_token, "df_transform_customers")
create_dataflow_gen2(access_token, "df_transform_orders")
```

All items are created in the current workspace (auto-detected from runtime context).

---

## Troubleshooting

| Error | Cause | Solution |
|-------|-------|----------|
| `401 Unauthorized` | Invalid or expired token | Verify credentials; regenerate client secret if expired |
| `403 Forbidden` | SPN lacks workspace access | Add SPN as Contributor to workspace |
| `PrincipalTypeNotSupported` | Tenant setting not enabled for SPNs | Enable "Service principals can use Fabric APIs" and add SPN's security group |
| `ItemDisplayNameAlreadyInUse` | Duplicate item name | Use a unique name |

---

## Creating a Service Principal

### Standard Naming Convention

`spn-global-fabric`

### Requirements

You must hold **Application Administrator** or **Owner** rights in the Azure tenant. If blocked, use the [Admin Request Template](#admin-request-template) below.

### Procedure

#### 1. Open Microsoft Entra ID
- Browse to https://portal.azure.com
- Select **Microsoft Entra ID**

#### 2. Register the Application
1. Navigate to **Manage** → **App registrations** → **+ New registration**
2. Name: `spn-global-fabric`
3. Supported account types: **Single tenant**
4. Click **Register**

#### 3. Verify the Service Principal
- In the app blade, locate **Managed application in local directory**
- If blank, click **Create service principal**

#### 4. Create a Client Secret
1. Navigate to **Certificates & secrets** → **+ New client secret**
2. Description: `fabric-access-key`
3. Expiration: 6–12 months
4. Click **Add**

**⚠️ Copy these values immediately (displayed only once):**
- Secret Value
- Secret ID
- Application (Client) ID
- Directory (Tenant) ID
- Secret Expiration Date
- SPN Name

Store all items securely in **Azure Key Vault**.

---

## Creating a Security Group and Adding the SPN

### 1. Create the Security Group
1. In Azure Portal, navigate to **Microsoft Entra ID** → **Groups**
2. Click **+ New group**
3. Group type: **Security**
4. Group name: `sg-fabric-spn-access` (or similar)
5. Click **Create**

### 2. Add the SPN to the Security Group
1. Open the security group you created
2. Navigate to **Members** → **+ Add members**
3. Search for your SPN by name (`spn-global-fabric`)
4. Select it and click **Select**

---

## Configuring the Tenant Setting

A **Fabric Administrator** must complete these steps in the [Fabric Admin Portal](https://app.fabric.microsoft.com/admin-portal/tenantSettings):

1. Navigate to **Admin Portal** → **Tenant Settings** → **Developer settings**
2. Locate **"Service principals can use Fabric APIs"**
3. Set to **Enabled**
4. Select **Specific security groups**
5. Add the security group containing your SPN (e.g., `sg-fabric-spn-access`)
6. Click **Apply**

> **Note:** As of mid-2025, Microsoft is splitting this into two settings:
> - *Service principal access to global APIs* — For creating workspaces, connections, deployment pipelines (disabled by default)
> - *Service principal access to permission-based APIs* — For CRUD operations on existing workspace items (enabled by default)
>
> Check your tenant settings if you encounter `PrincipalTypeNotSupported` errors.

---

## Admin Request Template

If you lack permissions to create the SPN yourself, send this to your Azure admin:

```
Subject: Request to Create SPN for Microsoft Fabric Integration

Hi [Azure Admin Team],

Please create a service principal named spn-global-fabric.

After creation, either:
1. Assign me Owner on the app registration, OR
2. Provide:
   • Application (Client) ID
   • Directory (Tenant) ID
   • Secret Value (only shown once)
   • Secret ID
   • Secret Expiration Date
   • SPN Name

Additionally, please:
3. Create a security group (e.g., sg-fabric-spn-access) and add the SPN as a member
4. Provide the security group name so it can be added to Fabric tenant settings

Purpose: Secure automation for Microsoft Fabric workloads.
```

---

## Limitations

Per [Microsoft documentation](https://learn.microsoft.com/en-us/fabric/data-warehouse/service-principals):

- SPNs cannot log into the Fabric portal interactively
- SPNs are not supported for GIT APIs
- SPNs cannot perform DCL operations (GRANT, REVOKE, DENY) in warehouses
- Some APIs still require user principal authentication—check the [API reference](https://learn.microsoft.com/en-us/rest/api/fabric/) for "Microsoft Entra supported identities"
- SPN tokens require renewal every 30 days (automate via OAuth 2.0 client-credentials flow or scheduled API calls)

---

## References

- [Fabric REST API Reference](https://learn.microsoft.com/en-us/rest/api/fabric/)
- [Identity Support for Fabric APIs](https://learn.microsoft.com/en-us/rest/api/fabric/articles/identity-support)
- [Service Principals in Fabric Data Warehouse](https://learn.microsoft.com/en-us/fabric/data-warehouse/service-principals)
- [Take Ownership of Fabric Items](https://learn.microsoft.com/en-us/fabric/fundamentals/item-ownership-take-over)
- [Change Ownership of Fabric Warehouse](https://learn.microsoft.com/en-us/fabric/data-warehouse/change-ownership)
- [Developer Tenant Settings](https://learn.microsoft.com/en-us/fabric/admin/service-admin-portal-developer)
- [Enable SPN for Admin APIs](https://learn.microsoft.com/en-us/fabric/admin/enable-service-principal-admin-apis)

---

## License

MIT
