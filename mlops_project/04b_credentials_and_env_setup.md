# 04b — Credentials & `.env` Setup (the right way to handle secrets)

> **Goal:** Create the two secret files this project needs — a `.env` (for your tools/scripts) and
> `conf/local/credentials.yml` (for Kedro/ADLS) — and **give them the correct rights** in two senses:
> *file permissions* (so they don't leak) and *Azure access rights / RBAC roles* (so they can only do
> what they need).

> 📍 **When to read this:** right after `04` (resources exist) and **before `06`** (Kedro needs the
> credentials). Ready-to-copy templates are in `tutorial/templates/`.

---

## 🤔 Why two files, and what's the difference? (the "why" first)

There are **two places** code runs, so there are **two places** secrets live:

| File | Used by | Lives where | Example secrets |
|------|---------|-------------|-----------------|
| `.env` | your terminal, Python scripts, Azure SDK (`07`), monitoring (`13`) | project root | subscription id, SP id/secret, ADLS name/key, resource names |
| `conf/local/credentials.yml` | **Kedro** specifically (it has its own credentials system) | `conf/local/` | the ADLS credentials Kedro's catalog references |

> 🧠 **Why not one file?** Kedro reads credentials from its *own* YAML (`conf/local/credentials.yml`) that
> the Data Catalog points to by name (`credentials: adls_creds`). Everything else (Azure SDK, CLI scripts)
> reads a standard `.env`. So we keep both — but they hold the **same underlying secret**, just in the
> format each tool expects. **Neither file is ever committed to git.**

---

## 🔑 The key idea: "giving the right" means TWO things

When you said "give the right," there are actually **two different rights** to set — and pros do both:

1. **File rights (permissions on disk)** → make sure the secret files are *git-ignored* and readable only
   by you. Stops accidental leaks.
2. **Azure rights (access control / RBAC roles)** → give the *identity* in the file only the permissions
   it needs — no more. This is **least privilege** (the RBAC idea from AZ-900).

We'll do both below.

---

## Part 1 — The simple way (account key) — fine for learning

Quickest to get going. We use the ADLS **account key** directly.

### Step 1.1 — Get your values
```powershell
# (reuse the $ variables from file 04, or re-paste them)
$SUB = az account show --query id -o tsv
$KEY = az storage account keys list --account-name $ADLS --resource-group $RG --query "[0].value" -o tsv
Write-Host "Subscription: $SUB"
Write-Host "ADLS account: $ADLS"
Write-Host "ADLS key:     $KEY"
```
> **Why grab these now?** These are the exact values you'll paste into the two files below. Copy them
> somewhere temporary (not into git!).

### Step 1.2 — Create the `.env` file
Copy `tutorial/templates/.env.example` to your project root as `.env` and fill it in:
```ini
# .env  — NEVER commit this file
AZURE_SUBSCRIPTION_ID=<your-sub-id>
AZURE_RESOURCE_GROUP=rg-predmaint-uat
AZURE_ML_WORKSPACE=mlw-predmaint-uat
ADLS_ACCOUNT_NAME=stpredmaintssuat01
ADLS_ACCOUNT_KEY=<your-adls-key>
ACR_NAME=acrpredmaintssuat01
AKS_NAME=aks-predmaint-uat
```

### Step 1.3 — Create the Kedro credentials file
Copy `tutorial/templates/credentials.yml.example` to `conf/local/credentials.yml`:
```yaml
# conf/local/credentials.yml  — NEVER commit this file
adls_creds:
  account_name: stpredmaintssuat01
  account_key: "<your-adls-key>"
```
> **Why does this name `adls_creds` matter?** Because `conf/base/catalog.yml` (file `06`) references it:
> `credentials: adls_creds`. The name in the catalog and the name here **must match exactly**, or Kedro
> can't find the credentials.

✅ That's the simple path working. Now let's make it **right** (production-grade).

---

## Part 2 — The right way (Service Principal + least-privilege RBAC)

Using the account key gives **full control of all storage** to anyone holding it — too much power. The
industrial standard is a **Service Principal (SP)**: a dedicated "robot identity" with *only* the roles
it needs. This is what the CI/CD pipelines use too.

### Step 2.1 — Create a Service Principal scoped to the resource group
```powershell
$SUB = az account show --query id -o tsv

az ad sp create-for-rbac `
  --name "sp-predmaint-uat" `
  --role "Contributor" `
  --scopes "/subscriptions/$SUB/resourceGroups/rg-predmaint-uat"
```
This prints something like:
```json
{
  "appId":    "11111111-....",   // = AZURE_CLIENT_ID
  "password": "abc~secret~xyz",  // = AZURE_CLIENT_SECRET  (shown ONCE — copy it now!)
  "tenant":   "22222222-...."    // = AZURE_TENANT_ID
}
```
> **What just happened?** Azure created a robot account that can manage things **only inside
> `rg-predmaint-uat`** — not your whole subscription. That `--scopes` is the "give the right" part: it
> *limits* where this identity has power.
> ⚠️ The `password` is shown **once**. If you lose it, you can't get it back — you'd reset it. Copy it
> immediately.

### Step 2.2 — Give it the *specific* data role on ADLS (least privilege)
`Contributor` lets it *manage* the storage account but not necessarily *read the data* (Azure separates
"manage the resource" from "read the bytes inside"). For reading/writing files, add the data role:
```powershell
az role assignment create `
  --assignee "<appId-from-step-2.1>" `
  --role "Storage Blob Data Contributor" `
  --scope "/subscriptions/$SUB/resourceGroups/rg-predmaint-uat/providers/Microsoft.Storage/storageAccounts/$ADLS"
```
> **Why a separate "Storage Blob Data Contributor" role?** This is a subtle but important Azure detail:
> *managing* a storage account (Contributor) ≠ *reading the data inside it* (Storage Blob Data
> Contributor). For Kedro to read/write your Parquet files, it needs the **data** role. Granting exactly
> this role = least privilege = "the right rights."

### Step 2.3 — Put the SP into `.env`
```ini
# .env  (Service Principal version)
AZURE_SUBSCRIPTION_ID=<your-sub-id>
AZURE_TENANT_ID=<tenant-from-step-2.1>
AZURE_CLIENT_ID=<appId-from-step-2.1>
AZURE_CLIENT_SECRET=<password-from-step-2.1>
AZURE_RESOURCE_GROUP=rg-predmaint-uat
AZURE_ML_WORKSPACE=mlw-predmaint-uat
ADLS_ACCOUNT_NAME=stpredmaintssuat01
ACR_NAME=acrpredmaintssuat01
AKS_NAME=aks-predmaint-uat
```
> 💡 **Bonus:** the Azure SDK's `DefaultAzureCredential` (file `07`) **automatically** uses
> `AZURE_TENANT_ID` + `AZURE_CLIENT_ID` + `AZURE_CLIENT_SECRET` from the environment. So setting these
> lets your training scripts authenticate as the SP with **no `az login`** — perfect for automation.

### Step 2.4 — Point Kedro at the SP (instead of the account key)
`conf/local/credentials.yml` (Service Principal version):
```yaml
adls_creds:
  account_name: stpredmaintssuat01
  tenant_id: <tenant-from-step-2.1>
  client_id: <appId-from-step-2.1>
  client_secret: <password-from-step-2.1>
```
> `adlfs` (the library from file `06`) understands these four fields and authenticates as the SP. No
> account key anywhere = if the file ever leaked, you can revoke just this SP instead of rotating the
> master storage key.

---

## Part 3 — Lock down the files (file permissions)

Two layers: **git** (most important) and **disk ACL** (extra).

### Step 3.1 — Make sure git ignores them (do this BEFORE any commit)
Your `.gitignore` (file `10`) must contain:
```gitignore
.env
conf/local/**
!conf/local/.gitkeep
*.pkl
```
Verify nothing secret is tracked:
```powershell
git status --ignored          # .env and conf/local/credentials.yml should appear under "Ignored files"
git check-ignore .env conf/local/credentials.yml   # should print both paths = they ARE ignored
```
> **Why is git the #1 concern?** Once a secret is committed and pushed, it's in history **forever** and
> bots scan public repos for keys within *minutes*. `git check-ignore` confirming both files = your
> safety proof. Do this before your first `git add`.

### Step 3.2 — Restrict who can read the files on Windows (optional, extra safety)
```powershell
icacls ".env" /inheritance:r /grant:r "$($env:USERNAME):(R,W)"
icacls "conf\local\credentials.yml" /inheritance:r /grant:r "$($env:USERNAME):(R,W)"
```
> **What this does:** removes inherited permissions and grants read/write to **only your user account**.
> So even other users on the machine can't open the secret files. This is the "give the right [file]
> rights" in the literal sense.

---

## Part 4 — Secrets in the CI/CD pipelines (the cloud side)

Your `.env` is for **your laptop**. The **pipelines** (files `11`–`12`) must NOT use your `.env` — they
get secrets from Azure DevOps securely:

1. **Service connection** (`azure-mlops-connection`) — the pipeline's identity to Azure (it's actually an
   SP behind the scenes, just like Part 2). Created in file `11`.
2. **Variable group** (`mlops-common`) — non-secret values (registry names, image name).
3. **Azure Key Vault + linked variable group** — *the right way* for pipeline secrets:
```powershell
# create a Key Vault and store the SP secret in it
az keyvault create --name "kv-predmaint-uat" --resource-group $RG --location $LOC
az keyvault secret set --vault-name "kv-predmaint-uat" --name "adls-key" --value $KEY
```
Then in Azure DevOps → **Pipelines → Library → + Variable group → Link secrets from an Azure Key Vault** →
select `kv-predmaint-uat`. Pipelines now read secrets from Key Vault at runtime — never stored in YAML.

> **Why Key Vault for pipelines?** The same secret-leak risk applies to pipeline logs and YAML. Key Vault
> keeps secrets out of source control *and* out of logs, and lets you rotate them centrally. This is the
> production-grade equivalent of your local `.env`.

> 🔁 **Per-environment secrets:** in the real architecture, **UAT and PROD each have their own SP, Key
> Vault, and variable group** (e.g. `kv-predmaint-prod`, `azure-mlops-connection-prod`). That's *why* the
> pipelines in file `12` reference `-prod` variable groups — each environment's identity can only touch
> its own resources. Least privilege, per environment.

---

## ✅ Checkpoint

- [ ] `.env` exists at project root with your values (account-key or SP version).
- [ ] `conf/local/credentials.yml` exists with matching `adls_creds`.
- [ ] `git check-ignore .env conf/local/credentials.yml` prints **both** paths (they're ignored).
- [ ] (Right way) A Service Principal `sp-predmaint-uat` exists with **Contributor** on the RG +
      **Storage Blob Data Contributor** on the storage account.
- [ ] You understand the two meanings of "rights": **file permissions** + **Azure RBAC roles**.

Next → continue to **`05_data_to_adls.md`** (or `06` if data is already uploaded). Your secrets are now
set up safely. 🔐
