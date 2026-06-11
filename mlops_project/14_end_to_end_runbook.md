# 14 — End-to-End Runbook, Troubleshooting & Handover

> **Goal:** Tie everything together. A single ordered checklist to run the whole system, a troubleshooting
> guide for common errors, a glossary, cost/cleanup commands, and what to demo when you **take over the
> project**.

---

## 🏁 The full run order (do it once, top to bottom)

This is the entire tutorial as a checklist. Tick each as you go.

**Setup (one-time):**
- [ ] `04` — Install tools, `az login`, create RG, ADLS, Azure ML, ACR, AKS, DevOps project, `.venv`.
- [ ] `05` — Download data, upload `raw/*.txt` to ADLS.

**Build the ML system:**
- [ ] `06` — Create Kedro project, catalog → ADLS, run `kedro run` → processed data + model + metrics + plot in ADLS.
- [ ] `07` — Submit Azure ML training job, log metrics, **register model** + bundle scaler.
- [ ] `08` — Build FastAPI, test locally, Dockerize, push `predmaint-api:v1` to ACR.
- [ ] `09` — `kubectl apply` deployment/service/hpa → API live on AKS → front with APIM.

**Automate it:**
- [ ] `10` — Git init, push to DevOps, create `develop`/`main`, branch policies.
- [ ] `11` — Create CI-1/CI-2/CI-3 pipelines (test → quality → security → build → push/sign).
- [ ] `12` — Create environments + approvals, CD-1/CD-2/CD-3 (UAT → Pre-PROD → PROD signed release).
- [ ] `13` — Wire SonarQube, Fortify/Mend into CI; set up Evidently drift monitoring.

**Prove it works:**
- [ ] Make a tiny change on a `feature/*` branch → PR to `develop` → watch CI run → merge → CD deploys to UAT.
- [ ] PR `develop → main` → watch CI-3 sign the image → approve CD-3 → confirm PROD updated.

> 🎉 If you can do that last loop, you have a **complete, automated, production-grade MLOps system.**

---

## 🔁 The "one breath" summary (memorize this for your handover)

> Raw NASA engine data lands in **ADLS** → **Kedro** pipelines clean it, engineer features, train a
> RandomForest, and write the model + metrics back to ADLS → **Azure ML** tracks the run and registers the
> approved model → we serve it with **FastAPI**, package it with **Docker**, store it in **ACR** → deploy
> to **AKS**, expose via **APIM** → **Azure DevOps CI/CD** automates the whole path across **UAT →
> Pre-PROD → PROD** with **signed images** and **approval gates** → **SonarQube/Fortify/Mend** guard code
> quality + security, and **Evidently** watches for drift to trigger retraining.

---

## 🧯 Troubleshooting (common beginner errors)

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `az login` opens but hangs | Browser/network | `az login --use-device-code` and follow the code flow |
| Kedro: `No credentials found for adls_creds` | `conf/local/credentials.yml` missing/typo | Recheck account name + key; ensure the file exists locally |
| Kedro: `abfs` path error | `adlfs` not installed | `pip install adlfs` |
| Docker build fails on `pip install` | Wrong/missing `requirements.txt` path in Dockerfile | Confirm `COPY api/requirements.txt .` matches your layout |
| AKS pods `ImagePullBackOff` | AKS can't pull from ACR | `az aks update -n $AKS -g $RG --attach-acr $ACR` |
| AKS pods `CrashLoopBackOff` | App errors on start (e.g. model file missing) | `kubectl logs <pod>` to see the Python error |
| Service `EXTERNAL-IP` stuck `<pending>` | LoadBalancer still provisioning | Wait 1–3 min; `kubectl get svc -w` |
| CI pipeline: "no hosted parallelism" | Free DevOps needs parallelism grant | Request free grant via Microsoft form, or use a self-hosted agent |
| CD can't deploy | Service connection lacks permission | Re-create the Azure RM service connection with Contributor role |
| `notation sign` not found | Signing tool not installed on agent | Add a step to install `notation`, or use ACR Tasks signing |

> **General debugging mindset:** read the **actual error message** (especially `kubectl logs` and pipeline
> logs), change **one thing at a time**, and confirm each layer works before moving up (local API →
> Docker → AKS → APIM).

---

## 💸 Cost control & daily cleanup

The expensive resources are **AKS** and **APIM**. To avoid surprise bills while learning:

**Pause at end of day:**
```powershell
az aks stop --name $AKS --resource-group $RG          # restart with: az aks start ...
az ml compute stop --name cpu-cluster -g $RG -w $MLW   # (auto-scales to 0 anyway)
```

**Nuke everything when fully done** (deletes the whole environment in one go — this is *why* we used one
Resource Group, file `04`):
```powershell
az group delete --name $RG --yes --no-wait
```
> ⚠️ This permanently deletes ADLS data, AKS, ACR images, and the ML workspace in that RG. Make sure
> you've saved anything you want first (e.g. download the model from the registry).

---

## 📚 Glossary (one line each)

| Term | Meaning |
|------|---------|
| **RUL** | Remaining Useful Life — cycles until the engine fails (what we predict) |
| **ADLS** | Azure Data Lake Storage — central cloud data home |
| **Kedro** | Framework that turns ML code into reusable pipelines (nodes + catalog) |
| **Node / Pipeline** | A function / a chain of functions wired by inputs & outputs |
| **Data Catalog** | Kedro YAML mapping dataset names → where they live (ADLS, etc.) |
| **Azure ML** | Managed ML platform: compute + experiment tracking + model registry |
| **Model Registry** | Versioned, approvable store of trained models |
| **MLflow** | Standard for logging ML runs (params, metrics, artifacts) |
| **FastAPI** | Python framework to serve the model as an HTTP API |
| **Docker / Image / Container** | Packaging tech / the package / a running instance of it |
| **ACR** | Azure Container Registry — private store of images (ACR001=release, ACR002=dev) |
| **AKS** | Azure Kubernetes Service — runs/scales/heals containers |
| **Pod / Deployment / Service / HPA** | running container / keeps N pods / stable address / autoscaler |
| **APIM** | API Management — secure managed gateway in front of the API |
| **CI / CD** | Continuous Integration (build+verify) / Continuous Deployment (ship) |
| **UAT / Pre-PROD / PROD** | Test / final-validation / live environments |
| **Image signing** | Cryptographic seal proving an image is trusted & untampered |
| **Drift** | When live data diverges from training data, hurting accuracy |
| **SonarQube / Fortify / Mend / Evidently** | code quality / code security / dependency security / drift monitoring |

---

## 🎤 Taking over the project — what to demo & know

When you present that you've "taken over" this project, be ready to:

1. **Draw the architecture** from memory (use the diagram in `00_START_HERE.md`).
2. **Explain the data journey** end to end (the "one breath" summary above).
3. **Show a live prediction** — call the APIM `/predict` URL and get an RUL.
4. **Trigger the pipeline** — push a feature branch and show CI/CD run through to UAT.
5. **Explain the release safety** — signed images, ACR001/ACR002 split, PROD approval gate.
6. **Explain how it stays healthy** — drift monitoring → retraining loop.

**Questions you should be able to answer:**
- *Why Kedro and not just the notebook?* → reusable, testable, swappable storage, runs anywhere.
- *Why register the model?* → versioning, approval, lineage; CD deploys approved versions.
- *Why two ACRs?* → trust separation; only signed images reach release/PROD.
- *Why three environments?* → prove changes safely before they hit real users.
- *Why monitor drift?* → models go stale as the world changes; retrain before failure.

---

## 🚀 Where to go next (after you're comfortable)

- Replace RandomForest with the **LSTM/Keras** approach (the notebook hints at deep learning) and
  compare metrics in Azure ML.
- Add **automated retraining** triggered by Evidently drift via an Azure ML pipeline.
- Add **blue/green or canary** deployments on AKS for even safer releases.
- Move secrets to **Azure Key Vault** + **Managed Identity** (remove all keys from config).

---

## ✅ Final checkpoint — you're done when…

- [ ] You ran the full checklist top to bottom at least once.
- [ ] A code change flows automatically: `feature → develop → UAT`, and `develop → main → PROD` (with approval).
- [ ] You can explain every box in the architecture diagram and *why* it's there.
- [ ] You cleaned up resources to control cost.

That's the whole project — from one notebook to a real Azure MLOps system. Congratulations. 🙌
You now understand not just *how* but *why* — which is exactly what taking over a project requires.
