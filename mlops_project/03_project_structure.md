# 03 — Industrial-Standard Project Structure

> **Goal:** See the *exact* folder layout we'll build, and understand *why* professionals organize
> projects this way. A good structure is the difference between a project you can maintain and one that
> becomes a nightmare. We build this gradually in later files — this is the map.

---

## 🤔 Why structure matters (the "why" first)

A notebook is **one file** where everything is mixed: data loading, cleaning, training, plotting. That's
fine for exploring, terrible for production because:

- You can't **test** one part without running everything.
- You can't **reuse** the training step in a different pipeline.
- Two people can't work on it without colliding.
- Nobody can tell what depends on what.

Professionals split a project into **clear, single-purpose folders** with predictable names. New team
members (like future-you taking over) can find anything in seconds.

> 🧠 **Golden rule:** *Separate concerns.* Data code, pipeline code, API code, infrastructure code, and
> CI/CD config each live in their own place. Change one without breaking the others.

---

## 🗂️ The full layout we'll build

We use a **Kedro project** as the backbone (it gives us a proven, opinionated structure for the ML part),
and wrap it with folders for the API, Docker, Kubernetes, and CI/CD.

```
predictive-maintenance-mlops/
│
├── conf/                          # ⚙️ Configuration (NO code, NO secrets in git)
│   ├── base/                      #    Shared defaults
│   │   ├── catalog.yml            #    Kedro Data Catalog — where every dataset lives (incl. ADLS)
│   │   ├── parameters.yml         #    Tunable values (model depth, test split, etc.)
│   │   └── logging.yml
│   └── local/                     #    Secrets/credentials — git-ignored! (ADLS keys, etc.)
│       └── credentials.yml
│
├── data/                          # 📦 Local data layers (Kedro convention; real data lives in ADLS)
│   ├── 01_raw/                    #    Raw input, never edited
│   ├── 02_intermediate/           #    Cleaned data
│   ├── 03_primary/                #    Feature tables
│   ├── 06_models/                 #    Trained model files
│   └── 08_reporting/              #    Metrics, plots (feature importance, etc.)
│
├── src/
│   └── predictive_maintenance/
│       ├── pipelines/             # 🔧 The heart: each ML stage is its own pipeline
│       │   ├── data_processing/   #    load + clean + compute RUL
│       │   │   ├── nodes.py        #      the functions (pure Python)
│       │   │   └── pipeline.py     #      wires nodes into a pipeline
│       │   ├── feature_engineering/ #  scaling + feature selection
│       │   ├── model_training/    #    train Random Forest
│       │   └── model_tuning/      #    GridSearch hyper-parameter tuning
│       ├── pipeline_registry.py   #    registers all pipelines
│       └── settings.py
│
├── api/                           # 🌐 The FastAPI serving app (separate from training code)
│   ├── main.py                    #    the /predict endpoint
│   ├── schema.py                  #    request/response models
│   └── requirements.txt
│
├── docker/                        # 🐳 Containerization
│   ├── Dockerfile                 #    how to package the API into an image
│   └── .dockerignore
│
├── k8s/                           # ☸️ Kubernetes manifests (how AKS runs the container)
│   ├── deployment.yml
│   ├── service.yml
│   └── hpa.yml                    #    horizontal pod autoscaler
│
├── azure-pipelines/               # 🔁 CI/CD pipeline definitions (Azure DevOps)
│   ├── ci/                        #    build/test/scan pipelines (CI-1, CI-2, CI-3)
│   │   ├── azure-pipelines.yml
│   │   ├── azure-pipelines-prod.yml
│   │   └── azure-pipelines-release.yml
│   └── cd/                        #    release pipelines (CD-1, CD-2, CD-3)
│       ├── mlops-sample-uat.yml
│       ├── mlops-sample-prod.yml
│       └── mlops-sample-prod-release.yml
│
├── tests/                         # ✅ Automated tests (unit + pipeline)
│   ├── test_data_processing.py
│   └── test_feature_engineering.py
│
├── notebooks/                     # 🔬 Exploration only (the original .ipynb lives here)
│   └── predictive_maintenance.ipynb
│
├── requirements.txt               # 📋 Python dependencies (pinned versions)
├── sonar-project.properties       # 🔍 SonarQube config (code quality)
├── .gitignore                     # 🚫 What NOT to commit (secrets, data, venv)
└── README.md                      # 📖 Project overview for humans
```

---

## 🧭 Why each top-level folder exists

| Folder | Why it's separate |
|--------|-------------------|
| `conf/` | **Config separate from code.** Change a setting (model depth, data path) without touching code. `conf/local/` holds secrets and is **never** committed. |
| `data/` | Kedro's **numbered layers** make the data flow obvious: raw → intermediate → primary → models → reporting. You always know where a dataset sits in the journey. |
| `src/.../pipelines/` | **One folder per ML stage.** Each has `nodes.py` (the logic) and `pipeline.py` (the wiring). Test and reuse each stage independently. |
| `api/` | The **serving** code is a different job from **training** code. Keeping them apart means the deployable API is small and clean. |
| `docker/` | Packaging instructions live with neither the app nor the infra — their own clear home. |
| `k8s/` | **Infrastructure as code** for how AKS runs things. Versioned like everything else. |
| `azure-pipelines/` | CI/CD definitions, split into `ci/` (build) and `cd/` (release), matching the templates in your architecture screenshots. |
| `tests/` | Automated tests gate every change. Pros never ship untested pipeline code. |
| `notebooks/` | Exploration is fine — but it's quarantined here, never imported by production code. |

---

## 📛 Naming conventions (industrial standard)

Real teams agree on naming so resources are predictable. We'll use:

**Code:** lower_snake_case for Python files/functions; kebab-case for repo and pipeline names.

**Azure resources** — pattern `<project>-<service>-<env>`:
| Resource | Example (UAT) | Example (PROD) |
|----------|---------------|----------------|
| Resource Group | `rg-predmaint-uat` | `rg-predmaint-prod` |
| Storage (ADLS) | `stpredmaintuat` (no dashes allowed) | `stpredmaintprod` |
| Azure ML workspace | `mlw-predmaint-uat` | `mlw-predmaint-prod` |
| Container Registry | `acrpredmaintuat` (ACR002 = dev) | `acrpredmaintprod` (ACR001 = release) |
| AKS cluster | `aks-predmaint-uat` | `aks-predmaint-prod` |

> 💡 **Why naming conventions?** When you have 50 resources across 3 environments, consistent names let
> anyone instantly know *what* a resource is and *which environment* it belongs to. It also makes
> automation (scripts/pipelines) reliable because names are predictable.

> 📝 In file `04` you'll **write down the exact names you choose** and reuse them everywhere.

---

## 🔐 What never goes in Git (critical)

Your `.gitignore` will exclude:
- `conf/local/` and any `credentials.yml` → **secrets**
- `data/` contents → big files live in ADLS, not git
- `.venv/`, `__pycache__/` → environment junk
- `*.pkl` large models → those go to the Model Registry/ADLS

> ⚠️ **Why?** Committing a storage key or password to git is the #1 cause of cloud breaches. Secrets go
> in Azure Key Vault / pipeline variables, never in source control. We enforce this from day one.

---

## ✅ Checkpoint

- [ ] Can you explain why training code (`src/`) and serving code (`api/`) are kept separate?
- [ ] What do the **numbered `data/` layers** represent?
- [ ] Why does `conf/local/` never get committed to git?
- [ ] What's our Azure resource naming pattern?

Next → **`04_azure_setup.md`** — time to create the actual Azure resources. 🛠️
