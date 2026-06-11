# 04 — Azure Setup (create all the cloud resources)

> **Goal:** Install the tools and create every Azure resource we'll need: a Resource Group, ADLS, an
> Azure ML workspace, a Container Registry (ACR), a Kubernetes cluster (AKS), and an Azure DevOps
> project. We'll do it with the **Azure CLI** so it's repeatable and copy-paste friendly.

> ⏱️ **Time:** ~45–60 min (AKS creation alone takes ~10 min — that's normal).
> 💰 **Cost note:** AKS and APIM cost money while running. For learning, **delete or stop resources when
> done each day** (we show how in file `14`). Use the smallest sizes given here.

---

## Part A — Install your tools (one-time)

Open **PowerShell** and install these. (If you already have some, skip them.)

```powershell
# Azure CLI — controls Azure from the terminal
winget install -e --id Microsoft.AzureCLI

# Git — version control
winget install -e --id Git.Git

# Python 3.10+ — to run Kedro/FastAPI locally
winget install -e --id Python.Python.3.11

# Docker Desktop — to build containers
winget install -e --id Docker.DockerDesktop

# kubectl — controls Kubernetes/AKS
winget install -e --id Kubernetes.kubectl

# VS Code — editor (recommended)
winget install -e --id Microsoft.VisualStudioCode
```

> **Why the Azure CLI instead of just clicking in the portal?** Clicking is fine for one resource, but
> it's not repeatable — you can't copy a click. CLI commands are **documentation + automation in one**:
> anyone can re-run them to rebuild the exact same setup. This is the professional habit.

**Close and reopen PowerShell** after installing (so the new commands are found), then verify:
```powershell
az version
python --version
docker --version
kubectl version --client
git --version
```

---

## Part B — Log in and pick your subscription

```powershell
az login          # opens a browser; sign in with your Azure account
az account show   # confirms which subscription you're using
```

> **Why "subscription"?** Remember from AZ-900: a subscription is your **billing + access boundary**.
> Everything we create gets billed to it. `az account show` confirms you're spending in the right place.

If you have multiple subscriptions, set the right one:
```powershell
az account set --subscription "<your-subscription-name-or-id>"
```

---

## 📝 Part C — Decide your names (write these down!)

We'll reuse these everywhere. **Storage and ACR names must be globally unique and lowercase, no dashes.**
Pick your own unique suffix (e.g. your initials + numbers). Example using `ss01`:

```powershell
# --- EDIT THESE, then paste the whole block into PowerShell to set variables ---
$LOC      = "centralindia"              # Azure region close to you
$RG       = "rg-predmaint-uat"          # Resource Group
$ADLS     = "stpredmaintssuat01"        # ADLS storage account (unique, lowercase, no dashes)
$MLW      = "mlw-predmaint-uat"         # Azure ML workspace
$ACR      = "acrpredmaintssuat01"       # Container Registry (unique, lowercase, no dashes)
$AKS      = "aks-predmaint-uat"         # Kubernetes cluster
```

> **Why set variables?** So the commands below "just work" without you editing each line. Keep this
> PowerShell window open for the whole file. (If you close it, paste the block again.)

> 🧱 **Why one Resource Group?** A Resource Group is a **folder for related resources** (AZ-900 again!).
> Putting this whole environment in one RG means we can **delete everything in one command** when we're
> done — no leftover charges. In a real company you'd have one RG per environment (uat/preprod/prod).

---

## Part D — Create the Resource Group

```powershell
az group create --name $RG --location $LOC
```
> This is the container everything else goes into. Cheap (free) and instant.

---

## Part E — Create ADLS (the data lake)

ADLS Gen2 is just a **Storage Account with "hierarchical namespace" turned on** (that's the flag that
makes it a true data *lake* with real folders, not just flat blobs).

```powershell
az storage account create `
  --name $ADLS `
  --resource-group $RG `
  --location $LOC `
  --sku Standard_LRS `
  --kind StorageV2 `
  --hierarchical-namespace true
```

Create a **container** (top-level folder) for our project data:
```powershell
$KEY = az storage account keys list --account-name $ADLS --resource-group $RG --query "[0].value" -o tsv

az storage fs create `
  --name "predictive-maintenance" `
  --account-name $ADLS `
  --account-key $KEY
```

> **Why `Standard_LRS`?** Cheapest redundancy (3 copies in one datacenter) — fine for learning. Recall
> from AZ-900: production might use ZRS/GRS for higher resilience.
> **Why hierarchical namespace?** It gives real directories (`raw/`, `processed/`, `models/`) so big-data
> tools and Kedro can organize and read data efficiently.

We'll actually upload data in **file `05`**.

---

## Part F — Create the Azure ML workspace

```powershell
# Install the ML extension for the CLI (one-time)
az extension add --name ml

az ml workspace create `
  --name $MLW `
  --resource-group $RG `
  --location $LOC
```

> **Why a workspace?** It's the **home for all your ML work** — experiments, registered models, compute,
> and endpoints. When we train in file `07`, every run gets logged here, and the best model is stored in
> this workspace's **Model Registry**. Creating it also auto-creates a few helper resources (a key vault,
> app insights, a storage account) — that's expected.

---

## Part G — Create the Container Registry (ACR)

```powershell
az acr create `
  --name $ACR `
  --resource-group $RG `
  --sku Basic `
  --admin-enabled true
```

> **Why ACR?** It's the **private, secure store for our Docker images** (file `08`). AKS pulls images
> from here to run them. `--sku Basic` is the cheapest tier (fine for learning). In the real CI/CD flow
> you have two registries — **ACR002** (dev/build images) and **ACR001** (signed release images) — which
> we cover in files `10`–`12`. For now one registry is enough to learn the flow.

---

## Part H — Create the Kubernetes cluster (AKS)

⏳ This takes ~10 minutes. Use a small node count to save cost.

```powershell
az aks create `
  --name $AKS `
  --resource-group $RG `
  --node-count 1 `
  --node-vm-size Standard_B2s `
  --generate-ssh-keys `
  --attach-acr $ACR
```

Then connect your local `kubectl` to it:
```powershell
az aks get-credentials --name $AKS --resource-group $RG
kubectl get nodes        # should list 1 node in "Ready" state
```

> **Why `--attach-acr`?** This step quietly grants AKS permission to **pull images from your ACR**.
> Without it, AKS couldn't download your container and you'd get "ImagePullBackOff" errors later. Doing
> it now saves a painful debugging session.
> **Why `Standard_B2s` / 1 node?** Smallest viable cluster = lowest cost for learning. Production would
> use bigger nodes and more of them (and a VM Scale Set behind the scenes — AZ-900 concept!).

---

## Part I — Create the Azure DevOps project

Azure DevOps is separate from the Azure portal. Two ways:

**Option 1 — Portal (easiest first time):**
1. Go to **https://dev.azure.com** and sign in with your Azure account.
2. Click **New organization** (accept defaults) → then **New project**.
3. Name it `predictive-maintenance-mlops`, set visibility **Private**, click **Create**.

**Option 2 — CLI:**
```powershell
az extension add --name azure-devops
az devops configure --defaults organization=https://dev.azure.com/<your-org>
az devops project create --name "predictive-maintenance-mlops"
```

> **Why Azure DevOps?** This is where the **code (Repos)** and the **automation (Pipelines)** live. Every
> push will trigger build/test/deploy. We set up repos and branching in file `10`. Creating the project
> now means it's ready when we get there.

---

## Part J — Create your local Python environment

Back in your project folder, make an isolated Python environment:
```powershell
cd "C:\Users\User\Documents\code_base\Azure Export Form Beginner\project_mlops"
python -m venv .venv
.\.venv\Scripts\Activate.ps1     # you should see (.venv) appear in your prompt
python -m pip install --upgrade pip
```

> **Why a virtual environment (`.venv`)?** It keeps *this project's* package versions separate from your
> system Python and other projects. No version clashes, and you can recreate it exactly from
> `requirements.txt`. Pros never install project packages globally.

> If `Activate.ps1` is blocked, run once:
> `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` then try again.

---

## ✅ Checkpoint — verify everything exists

```powershell
az resource list --resource-group $RG --output table
```
You should see your **storage account, ML workspace, ACR, and AKS** (plus a few auto-created helpers).
Also confirm:
- [ ] `kubectl get nodes` shows 1 Ready node
- [ ] You can open your Azure DevOps project at dev.azure.com
- [ ] `(.venv)` shows in your PowerShell prompt

> 💸 **Before you stop for the day:** AKS bills while running. To pause it:
> `az aks stop --name $AKS --resource-group $RG` (restart later with `az aks start ...`).

Next → **`05_data_to_adls.md`** — get the engine data into the data lake. 📦
