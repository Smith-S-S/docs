# 10 — Azure DevOps Repos & Branching Strategy

> **Goal:** Put the project in an Azure DevOps Git repo and set up the **feature → develop → main**
> branching strategy with three environments (UAT → Pre-PROD → PROD) and two registries (ACR002, ACR001),
> exactly as in your architecture screenshots.

> ✅ **Prerequisite:** The Azure DevOps project from file `04` exists. Git is installed.

---

## 🤔 Why a branching strategy? (the "why" first)

If everyone edited the live production code directly, every typo would break the real system. A
branching strategy creates **safe lanes**: you develop in isolation, prove your change in test
environments, and only *then* let it reach production. The branch a change is on decides *which
environment* it deploys to.

> 🧠 The rule: **code flows one way — feature → develop → main — and each hop is a safety gate** (review,
> tests, scans, sign-off). Nothing reaches PROD without passing every gate.

---

## The branching model (from your screenshots)

```
feature/xxx  ──PR──►  develop  ──PR──►  main
   │                    │                 │
   │ (local work)       │ deploys to      │ deploys to
   │                    │ UAT + Pre-PROD  │ PROD (after sign-off)
   ▼                    ▼                 ▼
 you code          test/validate     release to users
```

| Branch | Purpose | Triggers |
|--------|---------|----------|
| `feature/xxx` | One developer's change (e.g. `feature/better-scaler`) | CI build + tests only |
| `develop` | Integration branch; the "next release" | CI build → deploy to **UAT** (and Pre-PROD experiments) |
| `main` | Production-ready, released code | CI build → **signed** image → release to **PROD** |

> **Why a separate `develop` and `main`?** `develop` is where changes mingle and get tested together
> (UAT). `main` only ever holds code that's been fully validated and is safe to release. This separation
> is what lets a team keep shipping to test while production stays stable.

---

## The two registries (ACR001 vs ACR002)

Your diagram shows **two** Container Registries — this is a real-world security pattern:

| Registry | Holds | Role |
|----------|-------|------|
| **ACR002** | Dev/develop-branch images | "Build/work" registry — images get built and tested here |
| **ACR001** | Main-branch, **signed & verified** images | "Release" registry — only trusted, signed images |

> **Why two?** Separation of trust. Anyone's feature build can land in ACR002. But only a reviewed,
> merged-to-`main`, **digitally signed** image earns its way into ACR001 and on to production. Signing
> proves the image wasn't tampered with between build and deploy. (We do the signing in file `12`.)

---

## The pipeline names (matching your screenshots)

Keep these names — they're referenced across files `11`–`12`:

**CI (build) pipelines:**
- **CI-1**: `iras-udp-apps-ml-mlops` → `azure-pipelines.yml` (feature/develop builds → UAT ACR002)
- **CI-2**: `iras-udp-apps-ml-mlops-prod` → `azure-pipelines-release.yml` (develop → PROD ACR002 for local PROD testing)
- **CI-3**: `iras-udp-apps-ml-mlops-release` → `azure-pipelines-release.yml` (main → UAT ACR001, sign + verify)

**CD (release) pipelines:**
- **CD-1**: `Mlops-Sample-UAT` (deploy to UAT AKS + APIM)
- **CD-2**: `Mlops-Sample-PROD` (deploy to PROD ACR002 for local PROD testing)
- **CD-3**: `Mlops-Sample-PROD-Release` (3-step signed release to PROD)

> Don't worry about memorizing these yet — files `11` and `12` build each one. This table is your map.

---

## Step 1 — Initialize git and push to Azure DevOps

In your project folder:
```powershell
cd "C:\Users\User\Documents\code_base\Azure Export Form Beginner\project_mlops\predictive-maintenance"
git init
git branch -M main
```

Create `.gitignore` first (so secrets/junk never get committed — file `03`'s rule):
```gitignore
# .gitignore
.venv/
__pycache__/
*.pyc
data/01_raw/*
data/02_intermediate/*
data/06_models/*
*.pkl
conf/local/**           # secrets! never commit
!conf/local/.gitkeep
.env
```

Commit and push:
```powershell
git add .
git commit -m "Initial MLOps project: kedro pipelines, api, docker, k8s"

# Add the Azure DevOps repo as remote (copy the URL from your DevOps project → Repos → Clone)
git remote add origin https://dev.azure.com/<your-org>/predictive-maintenance-mlops/_git/predictive-maintenance-mlops
git push -u origin main
```

> **Why `.gitignore` before the first commit?** Once a secret is committed, it's in git history forever
> (even if you delete it later). Setting `.gitignore` first prevents the mistake entirely.

---

## Step 2 — Create the develop branch

```powershell
git checkout -b develop
git push -u origin develop
```
Now you have `main` and `develop`. Features will branch off `develop`.

---

## Step 3 — Protect the branches (branch policies)

In Azure DevOps → **Repos → Branches** → on both `main` and `develop`, click **⋯ → Branch policies** and turn on:
- **Require a minimum number of reviewers** (e.g. 1) → no self-merging without review.
- **Check for linked work items** (optional) → traceability.
- **Build validation** → the CI pipeline must pass before a PR can merge.

> **Why branch policies?** They *enforce* the safety gates. Without them, the branching strategy is just
> a suggestion. With them, **code physically cannot reach `main` without passing CI and review.** This is
> the backbone of safe MLOps.

---

## Step 4 — The everyday developer workflow

This is how you (and future teammates) will actually work:
```powershell
# 1. start a feature from develop
git checkout develop
git pull
git checkout -b feature/better-scaler

# 2. make changes, commit
git add .
git commit -m "Use RobustScaler for noisy sensors"
git push -u origin feature/better-scaler

# 3. open a Pull Request feature/better-scaler -> develop  (in the DevOps web UI)
#    -> CI-1 runs automatically (build + tests + scans)
#    -> a reviewer approves
#    -> merge. CD-1 then deploys to UAT.
```

Then, when `develop` is proven in UAT, you open a PR **develop → main**, which triggers the
**signed release** flow to PROD (file `12`).

> **Why this exact flow?** It mirrors the 7-step process in your architecture document:
> feature→develop (CI build to UAT ACR002, deploy to UAT), develop experiments to PROD ACR002, then
> develop→main (build, push to ACR001, **sign + verify**), then the 3-step CD-3 release into PROD. You're
> setting up the rails now; the pipelines (next files) ride on them.

---

## ✅ Checkpoint

- [ ] Your code is in Azure DevOps Repos on `main` and `develop`.
- [ ] `.gitignore` excludes secrets and big files (nothing sensitive was pushed).
- [ ] Branch policies require review + build validation on `main` and `develop`.
- [ ] You can explain why there are two registries (ACR001 vs ACR002) and three environments.

Next → **`11_ci_pipelines.md`** — write the CI pipelines that build, test, scan, and push images. 🏗️
