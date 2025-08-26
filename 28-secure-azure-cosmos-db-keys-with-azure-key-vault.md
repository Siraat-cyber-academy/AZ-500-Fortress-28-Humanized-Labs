# Secure Azure Cosmos DB Keys with Azure Key Vault

# 🔐 Lab: Secure Azure Cosmos DB Keys with Azure Key Vault (A → Z)

https://www.whizlabs.com/labs/secure-azure-cosmos-db-keys-with-azure-key-vault

## 🎯 Goal (One glance)

- Create **Key Vault**
- Create **Cosmos DB (SQL API)** + DB/Container
- Put **Cosmos keys/connection string** into **Key Vault (as secrets)**
- Give an **app’s Managed Identity** permission to read those secrets
- **Use Key Vault reference** (or SDK) so the app connects **without storing secrets in code**
- **Rotate keys** safely
- Add **hardening** (firewalls, private endpoints, logging, locks)

---

## 🧰 Prerequisites

- Azure subscription + Owner/Contributor on the resource group  
- Region: pick one (e.g., **West Europe**)  
- Names (you can copy/paste & change):
  - Resource Group: `rg-cosmos-kv-demo`
  - Key Vault: `kv-cosmos-demo-<unique>`
  - Cosmos DB Account: `cosmos-demo-<unique>`
  - App Service Plan: `asp-kv-demo`
  - App Service (Linux): `app-kv-demo-<unique>`
  - Database: `appdb`
  - Container: `items` (Partition Key: `/pk`)

> 💡 Names must be globally unique for Cosmos+App. Add your initials or random digits.

---

## 0) Cost & Clean-up Note

- Cosmos DB can incur cost if left running.  
- After testing, **delete** the resource group to clean everything.  
- Later we’ll add **Resource Locks** to prevent accidental deletion.  

---

## 1) Create a Resource Group

**Portal** → “Resource groups” → **Create**  
- Subscription: _your sub_  
- Resource group: `rg-cosmos-kv-demo`  
- Region: **West Europe**  
- Review + Create → **Create**  

---

## 2) Create a Key Vault

**Portal** → “Key vaults” → **Create**  
- RG: `rg-cosmos-kv-demo`  
- Name: `kv-cosmos-demo-<unique>`  
- Region: **West Europe**  
- Pricing: Standard  
- Access config: **RBAC** (recommended)  
- Soft delete: Enabled  
- Purge protection: **Enable**  
- Networking: Public (for now) → restrict later  
- Review + Create → **Create**

---

## 3) Create a Cosmos DB (SQL API)

**Portal** → “Azure Cosmos DB” → **Create**  
- API: **Core (SQL)**  
- RG: `rg-cosmos-kv-demo`  
- Name: `cosmos-demo-<unique>`  
- Location: West Europe  
- Capacity: Serverless (for demo)  
- TLS min: 1.2  
- Local auth: Enabled (for now)  
- Review + Create  

---

## 4) Database & Container

- Cosmos → **Data Explorer**  
- New Database: `appdb`  
- New Container:
  - Database: `appdb`  
  - Container: `items`  
  - Partition key: `/pk`  
  - Throughput: minimal for demo  

Sample Item:
```json
{
  "id": "1",
  "pk": "demo",
  "message": "Hello from Cosmos via Key Vault!"
}
```

---

## 5) Get Cosmos Keys

- Cosmos → **Settings → Keys**  
- Copy **Primary Connection String**  
- (We’ll store it in Key Vault)

---

## 6) Store in Key Vault

**Key Vault** → Secrets → **Generate/Import**  
- Name: `cosmos-connstr`  
- Value: paste connection string  
- Create  

---

## 7) Create an App with Managed Identity

**App Service Plan**
- Name: `asp-kv-demo`  
- OS: Linux  
- Free/B1 tier  

**App Service**
- Name: `app-kv-demo-<unique>`  
- Runtime: .NET / Node / Python  
- Enable **System-assigned Managed Identity**

---

## 8) Grant Key Vault Access

Key Vault → **Access control (IAM)**  
- Add role assignment → **Key Vault Secrets User**  
- Assign to: App `app-kv-demo-<unique>`  
- Save  

---

## 9) Use Key Vault in App

### Option A — Key Vault Reference
App Service → Configuration → App Setting:  
- Name: `COSMOS_CONNSTR`  
- Value:
  ```
  @Microsoft.KeyVault(SecretUri=https://kv-cosmos-demo-<unique>.vault.azure.net/secrets/cosmos-connstr/)
  ```
- Save → Restart app  

App code reads env var → `COSMOS_CONNSTR`.

### Option B — SDK Fetch
Use `DefaultAzureCredential` + `SecretClient` to fetch `cosmos-connstr` secret in code.

---

## 10) Quick Test
- App → Configuration → **Key Vault References** → Status ✔  
- Console → `printenv`  
- Deploy tiny sample app → fetch Cosmos item  

---

## 11) Key Rotation Plan

1. Store secrets: `cosmos-connstr-primary`, `cosmos-connstr-secondary`  
2. App starts with `secondary`  
3. Regenerate **primary** in Cosmos → update Key Vault secret  
4. Switch app to `primary`  
5. Regenerate **secondary** → update secret  
6. Repeat forever (ping-pong rotation)

---

## 12) Hardening

- **Cosmos networking**: Restrict → private endpoint  
- **Key Vault firewall**: Restrict → private endpoint  
- **Disable local auth**: Move to Azure AD RBAC  
- **Diagnostics**: Send to Log Analytics → Alerts  
- **Locks**: Protect critical resources  

---

## 13) Clean-up

- Remove locks  
- Delete RG `rg-cosmos-kv-demo`  

---

## 🧠 Troubleshooting

- ❌ Key Vault ref not resolved → check Managed Identity & RBAC role  
- ❌ 403 on secret → wrong role scope  
- ❌ App can’t reach Cosmos → check VNet/DNS  
- ❌ Rotation didn’t apply → avoid versioned SecretUri  

---

## 🌷 Why this Matters

Secrets don’t belong in code.  
They belong in a vault — protected, rotated, respected.  
That’s how you guard both **data** and **dignity**.  

---


✒️ **Closing Signature**  
✍️ Created & Curated by  
**Muhammad Naveed Ishaque (Eks2)**  
Content Creator | AI Writer | Narrative Simplifier  
With the inner voice of Eks2 — the whisper behind the work.  

🕊️ **Siraat AI Academy**  
_“The Straight Path — Empowering minds with clarity, illuminating paths with purpose.”_  
