# 02 — MLOps Concepts (and *why* each tool exists)

> **Goal:** Understand what "MLOps" means and why our stack has so many tools. After this you'll see
> the whole architecture as one logical story, not a scary pile of Azure services.

---

## 🤔 What is MLOps, really?

**MLOps = DevOps for Machine Learning.** It's the practice of taking ML models from "works in a
notebook" to "runs reliably in production and keeps working."

A regular software app has code → build → test → deploy. ML adds two messy extras:
- **Data** (which changes over time)
- **Models** (which are *trained*, not just written, and can go stale)

So MLOps has to manage **code + data + models** together. That's why we need more tools than a normal app.

> 🧠 **The core problem MLOps solves:** A model that was accurate last year may be wrong today because
> the real world changed ("drift"). MLOps makes the whole loop — train, deploy, monitor, retrain —
> **automatic and trustworthy.**

---

## 🔄 The MLOps lifecycle (the loop everything serves)

```
   ┌──────────────────────────────────────────────────────────────────┐
   │                                                                  │
   ▼                                                                  │
 DATA ──► PROCESS & ──► TRAIN ──► EVALUATE ──► REGISTER ──► DEPLOY ──► MONITOR
        FEATURE-ENG     MODEL      (good?)     MODEL       (serve)    (drift?)
                                                                       │
                                              retrain when stale ◄─────┘
```

Every tool in our stack owns one part of this loop. Let's go tool by tool and answer **"why this?"**

---

## 🧰 The stack — why each tool

### 1. Azure Data Lake Storage (ADLS) — *the data home*
**What:** A massive, cheap cloud storage built for analytics data (files, Parquet, folders).
**Why:** Your data can't live on a laptop. It needs **one central place** that's scalable, secure, and
reachable by every pipeline and the Azure ML service. ADLS is that single source of truth.
> Analogy: the **warehouse** where all raw materials and finished goods are stored.

### 2. Kedro — *the pipeline framework*
**What:** An open-source Python framework that structures ML code into **pipelines** made of **nodes**
(functions), with a **data catalog** that handles reading/writing data.
**Why:** A notebook runs top-to-bottom, by hand, and is impossible to test or reuse. Kedro forces your
code into clean, reusable, testable steps with clear inputs/outputs. The *same pipeline* runs on your
laptop or in the cloud, unchanged.
> Analogy: the **assembly line** — each station (node) does one job and passes the product along.
> *This is the "industrial standard" structure your project requested.*

### 3. Azure Machine Learning (Azure ML) — *training + tracking + registry*
**What:** Azure's managed ML platform: rentable compute to train on, **experiment tracking** (records
every run's metrics/params), and a **Model Registry** (versioned store of trained models).
**Why:** You need to (a) train on bigger machines than your laptop, (b) **remember** which run produced
which accuracy, and (c) keep **versioned models** so you can promote/rollback. The registry is the
"approved models shelf."
> Analogy: the **R&D lab** with a logbook and a shelf of labeled, versioned products.

### 4. FastAPI — *the model's front door*
**What:** A Python web framework to wrap the model in a REST **API** (send sensor data → get an RUL
prediction back over HTTP).
**Why:** A `.pkl` model file is useless to other apps. They need to *call* it. FastAPI turns the model
into a service anyone can request predictions from. It's fast and auto-generates docs.
> Analogy: the **service counter** where customers hand in a request and get a result.

### 5. Docker — *the universal package*
**What:** Packages your API + its exact Python version + all libraries into one **container image**
that runs identically anywhere.
**Why:** "It works on my machine" is the classic disaster. Docker guarantees the model runs the same on
your laptop, in test, and in production — same dependencies, same behavior.
> Analogy: a **shipping container** — standardized box that fits any truck, ship, or crane.

### 6. Azure Container Registry (ACR) — *the image store*
**What:** A private, secure cloud store for Docker images (like a warehouse of shipping containers).
**Why:** Once you build an image, you need somewhere safe and versioned to keep it so Kubernetes can
pull and run it. We also **sign** images here to prove they're trusted (see CI/CD files).
> Analogy: the **bonded warehouse** that holds sealed, labeled containers ready to ship.

### 7. Azure Kubernetes Service (AKS) — *the runtime*
**What:** Managed **Kubernetes** — it runs your containers, restarts them if they crash, and scales
them up/down with demand (**pods**, **deployments**, **services**).
**Why:** One container on one machine = single point of failure. Kubernetes keeps the API **always on**,
**self-healing**, and **auto-scaling** under load. Essential for production reliability.
> Analogy: the **factory floor manager** who keeps machines running and adds more when orders spike.

### 8. Azure API Management (APIM) — *the secure gateway*
**What:** A managed **API gateway** in front of AKS — handles authentication, rate limits, a stable URL,
and monitoring for callers.
**Why:** You don't expose Kubernetes directly to the world. APIM is the **guarded front door**: it
secures the API, gives a clean public URL, and controls who can call it and how often.
> Analogy: the **reception desk + security guard** at the building entrance.

### 9. Azure DevOps — *the automation brain*
**What:** Source control (**Repos**) + **Pipelines** (CI/CD) that automatically build, test, scan, and
deploy on every code change.
**Why:** Doing all the above by hand every time is slow and error-prone. CI/CD makes it **automatic,
repeatable, and safe** — push code, and the system builds, tests, and promotes it through environments.
> Analogy: the **conveyor + robots** that move a product through every station without humans.

### 10. Evidently / SonarQube / Fortify / Mend — *the guards*
- **Evidently** → watches for **data/model drift** in production (is the model still accurate?). Triggers
  retraining. *(Closes the MLOps loop.)*
- **SonarQube** → checks **code quality** (bugs, smells) before it ships.
- **Fortify / Mend** → **security scanning** (vulnerabilities in your code and dependencies).
**Why:** In a real company, you can't ship code that's buggy, insecure, or models that have silently gone
stale. These tools are the **quality gates**.
> Analogy: the **inspectors** at the end of the line who reject defective products.

---

## 🌍 Why three environments (UAT → Pre-PROD → PROD)?

You'll see this everywhere in the CI/CD files. The idea:

| Environment | Purpose | Who uses it |
|-------------|---------|-------------|
| **UAT** (User Acceptance Testing) | First place new builds land; test changes safely | Developers / testers |
| **Pre-PROD** | A near-copy of production for final validation | QA / release team |
| **PROD** (Production) | The real, live system serving real users | End users |

> **Why not deploy straight to PROD?** Because a bug in PROD is a disaster. Changes get proven in UAT,
> validated in Pre-PROD, and only then promoted to PROD — each step a safety net. This is what the
> branching strategy (`feature → develop → main`) and the signed ACR001/ACR002 image flow enforce.

---

## 🔁 Putting it together — one sentence per stage

> Raw data lands in **ADLS** → **Kedro** pipelines clean/feature/train it → **Azure ML** tracks runs and
> registers the best model → we wrap it in **FastAPI**, package with **Docker**, store in **ACR** →
> deploy to **AKS**, expose through **APIM** → **Azure DevOps** automates all of it across UAT/Pre-PROD/PROD
> → **Evidently/SonarQube/Fortify/Mend** keep it healthy and safe.

That single sentence *is* the project. Everything else is detail.

---

## ✅ Checkpoint

- [ ] Can you say what problem MLOps solves that plain DevOps doesn't? (answer: it manages **data + models**, not just code)
- [ ] Can you name the tool for each loop stage: store data, build pipelines, train/register, serve, package, store image, run, expose, automate, monitor?
- [ ] Why do we use three environments instead of deploying straight to production?

Next → **`03_project_structure.md`** to see the exact folders we'll create.
