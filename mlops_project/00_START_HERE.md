# 🚀 End-to-End MLOps on Azure — Start Here

Welcome! This tutorial turns **one Jupyter notebook** (`predictive_maintenance.ipynb`) into a
**real, production-grade MLOps system on Azure** — the same way a company would run it.

You're about to take over this project, so the goal is not just "make it work" but **understand
why every piece exists.** Every step answers two questions:

1. **What am I doing?** (the exact clicks/commands/code)
2. **Why am I doing it?** (the reason a real team does it this way)

> 🧒 **Beginner promise:** I assume you know almost nothing about Azure, Docker, Kubernetes, or CI/CD.
> Each file is small, ordered, and builds on the last. Follow them **in number order.**

---

## 📺 The 30-second version of what we're building

We have a machine-learning model that predicts **when a jet engine will fail** (so maintenance can be
scheduled *before* a breakdown). Right now it lives in a notebook on a laptop. That's a problem:

- A notebook can't be trusted, tested, versioned, or deployed reliably.
- A real business needs the model **served as an API**, **retrained automatically**, and **promoted
  safely** from testing → production.

So we rebuild it as an **MLOps pipeline**: data goes to the cloud, pipelines process and train, the
model is registered and packaged into a container, the container is deployed to Kubernetes and exposed
as an API, and everything is automated with CI/CD across three environments (UAT → Pre-PROD → PROD).

---

## 🗺️ The architecture (in plain English)

Read this once. Don't worry if a word is new — every box has its own tutorial file.

```
            ┌──────────────────────────────────────────────────────────────┐
            │                     AZURE CLOUD                              │
            │                                                              │
  Raw data  │   ┌──────────┐   Kedro pipelines     ┌───────────────────┐   │
  (NASA     │   │  ADLS    │  (process, features,  │  Azure ML Studio  │   │
  engine ───┼──►│ (Data    │──► train, tune) ─────►│  - train model    │   │
  sensors)  │   │  Lake)   │◄── artifacts/graphs   │  - Model Registry │   │
            │   └──────────┘                       └─────────┬─────────┘   │
            │                                                │ package      │
            │                                                ▼              │
            │   ┌──────────┐   docker push   ┌──────────┐  deploy  ┌──────┐ │
            │   │ FastAPI  │───────────────► │   ACR    │────────► │ AKS  │ │
            │   │ (serves  │   (image)       │ (image   │          │(k8s) │ │
            │   │  model)  │                 │  store)  │          └───┬──┘ │
            │   └──────────┘                 └──────────┘              │    │
            │                                                          ▼    │
            │                                                  ┌────────────┐│
            │                                                  │   APIM     ││ ──► users call the API
            │                                                  │ (API gate) ││
            │                                                  └────────────┘│
            │                                                              │
            │   Everything above is automated by  AZURE DEVOPS (CI/CD)     │
            │   across 3 environments:  UAT ──► Pre-PROD ──► PROD          │
            └──────────────────────────────────────────────────────────────┘
```

**The journey of our data and model:**

| # | Stage | Tool | One-line "why" |
|---|-------|------|----------------|
| 1 | Store raw data in the cloud | **ADLS** (Azure Data Lake) | One central, scalable home for all data |
| 2 | Process + engineer features + train | **Kedro** pipelines | Turns messy notebook code into reusable, testable pipelines |
| 3 | Track training + register the model | **Azure ML** | Experiment tracking + a versioned model registry |
| 4 | Wrap model as an API | **FastAPI** | So apps can ask for predictions over HTTP |
| 5 | Package the API | **Docker → ACR** | Run identically anywhere; store the image safely |
| 6 | Run + scale the API | **AKS** (Kubernetes) | Auto-heal, auto-scale the running service |
| 7 | Expose + secure the API | **APIM** | A safe, managed front door for callers |
| 8 | Automate everything | **Azure DevOps** | Build/test/deploy on every code change, no manual steps |
| 9 | Keep it healthy & safe | **Evidently / SonarQube / Fortify / Mend** | Detect model drift, bad code, security holes |

---

## 📚 The tutorial files (follow in order)

| File | What you'll learn | Hands-on? |
|------|-------------------|-----------|
| `00_START_HERE.md` | This overview | Read |
| `01_understand_the_ml_project.md` | What the model actually does (the notebook explained) | Read |
| `02_mlops_concepts.md` | MLOps lifecycle + *why* each tool exists | Read |
| `03_project_structure.md` | Industrial-standard folder layout we'll build | Read |
| `04_azure_setup.md` | Create all Azure resources (RG, ADLS, AML, ACR, AKS, DevOps) | ✅ Do |
| `04b_credentials_and_env_setup.md` | Create `.env` + `credentials.yml` safely; SP + RBAC rights | ✅ Do |
| `05_data_to_adls.md` | Upload data to the data lake | ✅ Do |
| `06_kedro_pipelines.md` | Convert the notebook into Kedro pipelines | ✅ Do |
| `07_azure_ml_training.md` | Train in Azure ML + register the model | ✅ Do |
| `08_fastapi_docker_acr.md` | Build the API, containerize it, push to ACR | ✅ Do |
| `09_aks_apim_deployment.md` | Deploy to Kubernetes + expose via APIM | ✅ Do |
| `10_devops_repos_branching.md` | Set up repo + branching strategy | ✅ Do |
| `11_ci_pipelines.md` | Write the CI (build/test) pipelines | ✅ Do |
| `12_cd_release_pipelines.md` | Write the CD/release pipelines (UAT→PROD) | ✅ Do |
| `13_quality_security_monitoring.md` | Add code quality, security, drift monitoring | ✅ Do |
| `14_end_to_end_runbook.md` | Run the whole thing + checklist + glossary | ✅ Do |

---

## 🧰 What you need before starting (prerequisites)

Don't install everything now — each file tells you when you need a tool. But here's the full list so
you know what's coming:

**Accounts (free tiers are fine for learning):**
- An **Azure account** (portal.azure.com) — you already have one from the AZ-900 work. 👍
- An **Azure DevOps** organization (dev.azure.com) — free, sign in with the same account.

**Tools on your laptop (Windows):**
- **Python 3.10+** (we'll use a virtual environment)
- **Git** (version control)
- **Docker Desktop** (to build containers)
- **Azure CLI** (`az`) — to control Azure from the terminal
- **kubectl** — to control Kubernetes
- **VS Code** (recommended editor)

> 💡 **Why a virtual environment?** It keeps this project's Python packages separate from everything
> else on your machine, so versions never clash. We'll create one in `04_azure_setup.md`.

---

## 🧭 How to use this tutorial

1. **Go in order.** Later steps assume earlier ones are done.
2. **Read the "Why" boxes.** They're what turn you from "following steps" into "understanding the system."
3. **Do the ✅ checkpoints** at the end of each file before moving on.
4. **Keep names consistent.** We'll pick names (resource group, storage account, etc.) in file `04`
   and reuse them everywhere. Write them down.

When you're ready, open **`01_understand_the_ml_project.md`**. Let's go. 🙌
