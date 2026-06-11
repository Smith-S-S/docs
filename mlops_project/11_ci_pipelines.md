# 11 — CI Pipelines (build, test, scan, push image)

> **Goal:** Write the **Continuous Integration (CI)** pipelines in Azure DevOps. CI runs automatically on
> every push/PR: it **builds** the code, runs **tests**, runs **quality/security scans**, builds the
> **Docker image**, and pushes it to **ACR**. We create CI-1, CI-2, CI-3 from your screenshots.

> ✅ **Prerequisite:** Repo + branches from file `10`; image build understood from file `08`.

---

## 🤔 What is CI and why? (the "why" first)

**Continuous Integration** = every time code changes, an automated pipeline immediately verifies it's
good *before* it can be merged or deployed. It catches broken code, failing tests, and security holes in
minutes — not after they reach production.

> 🧠 Without CI, "it works on my machine" reaches production and breaks. With CI, **the robot checks
> every change the same way, every time.** Humans forget steps; pipelines never do.

**Our CI does 5 jobs, in order:**
1. **Lint + unit tests** — is the code correct?
2. **SonarQube scan** — is the code quality acceptable? (file `13`)
3. **Security scan (Fortify/Mend)** — any vulnerabilities? (file `13`)
4. **Build Docker image** — package the app.
5. **Push image to ACR** — store it (ACR002 for dev, ACR001 for release).

> **Why fail fast?** We run cheap checks (tests, lint) before expensive ones (Docker build). If tests
> fail, we stop immediately and don't waste time building an image. Order matters.

---

## Step 1 — Connect Azure DevOps to Azure (service connection)

The pipeline needs permission to push to ACR. In Azure DevOps → **Project Settings → Service connections
→ New service connection → Azure Resource Manager** → pick your subscription → name it
`azure-mlops-connection`.

> **Why a service connection?** It's how the pipeline **authenticates to Azure** without you pasting
> credentials into YAML. Azure DevOps stores it securely and the pipeline references it by name. Secure
> by default — the same principle as `DefaultAzureCredential` in file `07`.

Also store reusable values in a **Variable Group** (Pipelines → Library → + Variable group, name it
`mlops-common`):
```
acrDev:     acrpredmaintssuat01        # ACR002 (dev/build)
acrRelease: acrpredmaintprod01         # ACR001 (signed release)
imageName:  predmaint-api
```

> **Why a variable group?** So registry names and the image name live in *one* place, shared by all
> pipelines. Change once, everywhere updates. No copy-paste drift.

---

## Step 2 — CI-1: the main build pipeline (`azure-pipelines.yml`)

This runs on **feature** and **develop** branches → builds and pushes to **ACR002 (dev)**.
Create `azure-pipelines/ci/azure-pipelines.yml`:

```yaml
# CI-1 : iras-udp-apps-ml-mlops
# Runs on feature/* and develop. Build + test + scan + push to ACR002 (dev).
trigger:
  branches:
    include: [ develop, feature/* ]

pool:
  vmImage: ubuntu-latest

variables:
  - group: mlops-common               # acrDev, acrRelease, imageName
  - name: tag
    value: '$(Build.BuildId)'         # unique image tag per build (traceable!)

stages:
# ---------- 1. TEST ----------
- stage: Test
  displayName: Lint & Unit Tests
  jobs:
    - job: test
      steps:
        - task: UsePythonVersion@0
          inputs: { versionSpec: '3.11' }
        - script: |
            python -m pip install --upgrade pip
            pip install -r requirements.txt
            pip install pytest flake8
          displayName: Install deps
        - script: flake8 src api --max-line-length=120
          displayName: Lint (flake8)
        - script: pytest tests/ --junitxml=test-results.xml
          displayName: Run unit tests
        - task: PublishTestResults@2
          inputs: { testResultsFiles: 'test-results.xml' }
          condition: always()

# ---------- 2. CODE QUALITY (SonarQube) ----------
- stage: Quality
  dependsOn: Test
  jobs:
    - job: sonar
      steps:
        - task: SonarQubePrepare@5
          inputs:
            SonarQube: 'sonarqube-connection'
            scannerMode: 'CLI'
            configFile: 'sonar-project.properties'
        - task: SonarQubeAnalyze@5
        - task: SonarQubePublish@5    # fails the build if quality gate fails

# ---------- 3. SECURITY SCAN (Mend/Fortify) ----------
- stage: Security
  dependsOn: Test
  jobs:
    - job: scan
      steps:
        - script: echo "Run Mend/Fortify scan here (see file 13)"
          displayName: Dependency & SAST scan

# ---------- 4 & 5. BUILD + PUSH IMAGE to ACR002 ----------
- stage: BuildPush
  dependsOn: [ Quality, Security ]
  jobs:
    - job: docker
      steps:
        - task: AzureCLI@2
          displayName: Build & push image to ACR002 (dev)
          inputs:
            azureSubscription: 'azure-mlops-connection'
            scriptType: pscore
            scriptLocation: inlineScript
            inlineScript: |
              az acr build `
                --registry $(acrDev) `
                --image $(imageName):$(tag) `
                --image $(imageName):latest `
                -f docker/Dockerfile .
```

> **Read it top-down:** trigger on feature/develop → test → quality → security → build & push. Each
> `stage` only runs if the one it `dependsOn` passed. **`az acr build` builds the image in the cloud**
> (no Docker agent needed) and pushes straight to ACR002.
> **Why tag with `$(Build.BuildId)`?** Every image is traceable to the exact pipeline run and commit —
> crucial for debugging "which build is in production?"

Create the pipeline in DevOps: **Pipelines → New pipeline → Azure Repos Git → your repo → Existing YAML
file →** select this file. Name it **`iras-udp-apps-ml-mlops`** (CI-1).

---

## Step 3 — Write at least one real test (so CI has something to run)

`tests/test_data_processing.py`:
```python
import pandas as pd
from predictive_maintenance.pipelines.data_processing.nodes import add_train_rul, COLUMN_NAMES

def test_rul_is_zero_at_failure():
    # engine 1 runs for 3 cycles; RUL at the last cycle must be 0
    df = pd.DataFrame({
        "UnitNumber": [1, 1, 1],
        "Cycle": [1, 2, 3],
        **{c: [0, 0, 0] for c in COLUMN_NAMES if c not in ("UnitNumber", "Cycle")},
    })
    out = add_train_rul(df)
    assert out.loc[out.Cycle == 3, "target_RUL"].iloc[0] == 0
    assert out.loc[out.Cycle == 1, "target_RUL"].iloc[0] == 2
```

> **Why test the RUL logic specifically?** It's the heart of the project (file `01`). If RUL is computed
> wrong, the whole model is wrong. A tiny test like this catches that instantly on every change. **Tests
> are your safety net** — they let you refactor confidently.

---

## Step 4 — CI-2: PROD build for local PROD testing (`azure-pipelines-release.yml` variant)

When experimenting in PROD (per your doc step 4), CI-2 builds with the **PROD ACR002** base and pushes
there. It's the same YAML as CI-1 but triggered differently and pointed at PROD's ACR002. Create
`azure-pipelines/ci/azure-pipelines-prod.yml`:

```yaml
# CI-2 : iras-udp-apps-ml-mlops-prod
# Build with PROD ACR002 base image, push to PROD ACR002 for local PROD testing.
trigger:
  branches: { include: [ develop ] }
  paths:
    include: [ 'src/**', 'api/**', 'docker/**' ]
variables:
  - group: mlops-common-prod          # a PROD-scoped variable group
  - name: tag
    value: '$(Build.BuildId)'
pool: { vmImage: ubuntu-latest }
stages:
  - stage: BuildPushProd
    jobs:
      - job: docker
        steps:
          - task: AzureCLI@2
            inputs:
              azureSubscription: 'azure-mlops-connection-prod'
              scriptType: pscore
              scriptLocation: inlineScript
              inlineScript: |
                az acr build --registry $(acrProdDev) `
                  --image $(imageName):$(tag) -f docker/Dockerfile .
```

> **Why a separate PROD-build pipeline?** Sometimes you must test a build using *production's* base
> images/config before release. CI-2 produces that PROD-flavored image into PROD ACR002 for safe local
> testing — without touching the real released registry (ACR001).

---

## Step 5 — CI-3: the release build with signing (`azure-pipelines-release.yml`)

This runs on **main** (after develop→main PR). It builds, pushes to **UAT ACR001**, then **signs and
verifies** the image. Create `azure-pipelines/ci/azure-pipelines-release.yml`:

```yaml
# CI-3 : iras-udp-apps-ml-mlops-release
# On main: build, push to UAT ACR001, SIGN and VERIFY the image.
trigger:
  branches: { include: [ main ] }
variables:
  - group: mlops-common
  - name: tag
    value: '$(Build.BuildId)'
pool: { vmImage: ubuntu-latest }
stages:
  - stage: ReleaseBuild
    jobs:
      - job: build_sign
        steps:
          - task: AzureCLI@2
            displayName: Build & push to UAT ACR001
            inputs:
              azureSubscription: 'azure-mlops-connection'
              scriptType: pscore
              scriptLocation: inlineScript
              inlineScript: |
                az acr build --registry $(acrRelease) `
                  --image $(imageName):$(tag) -f docker/Dockerfile .
          - task: AzureCLI@2
            displayName: Sign image (content trust / notation)
            inputs:
              azureSubscription: 'azure-mlops-connection'
              scriptType: pscore
              scriptLocation: inlineScript
              inlineScript: |
                # Sign the image so its authenticity can be verified before PROD deploy
                notation sign $(acrRelease).azurecr.io/$(imageName):$(tag)
          - task: AzureCLI@2
            displayName: Verify signature
            inputs:
              azureSubscription: 'azure-mlops-connection'
              scriptType: pscore
              scriptLocation: inlineScript
              inlineScript: |
                notation verify $(acrRelease).azurecr.io/$(imageName):$(tag)
```

> **Why sign the image?** Signing creates a cryptographic seal proving "this exact image was built by our
> trusted release pipeline and hasn't been altered." Before deploying to PROD, we **verify** that seal.
> This stops a tampered or rogue image from ever reaching production — a core supply-chain security
> control. This is the ACR001 "validation & signing" box in your diagram.

---

## How the three CI pipelines line up with your screenshots

| Your doc | This file | Trigger | Pushes to | Special |
|----------|-----------|---------|-----------|---------|
| CI-1 `iras-udp-apps-ml-mlops` | `azure-pipelines.yml` | feature/develop | UAT **ACR002** | test+scan+build |
| CI-2 `...-prod` | `azure-pipelines-prod.yml` | develop | PROD **ACR002** | PROD-flavored build |
| CI-3 `...-release` | `azure-pipelines-release.yml` | main | UAT **ACR001** | **sign + verify** |

---

## ✅ Checkpoint

- [ ] CI-1 pipeline exists and runs on a push to `develop` (build goes green, image lands in ACR002).
- [ ] At least one unit test runs in CI.
- [ ] You understand the stage order (test → quality → security → build → push) and *why fail-fast*.
- [ ] You understand why CI-3 **signs** the image before it can go to PROD.

Next → **`12_cd_release_pipelines.md`** — the release pipelines that deploy UAT → Pre-PROD → PROD. 🚀
