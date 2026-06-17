# 12 — CD / Release Pipelines (UAT → Pre-PROD → PROD)

> **Goal:** Write the **Continuous Deployment (CD)** pipelines that take the images CI produced and
> **deploy them through the environments** — automatically to UAT, then with approvals to PROD. We build
> CD-1, CD-2, CD-3 from your screenshots, including the **3-step signed release**.

> ✅ **Prerequisite:** CI pipelines from file `11`; AKS deploy understood from file `09`.

---

## 🤔 CI vs CD — what's the difference? (the "why" first)

- **CI** (file `11`) = *build & verify* the code → produces a tested, scanned **image**.
- **CD** (this file) = *deploy* that image to running environments (AKS + APIM), with **approval gates**
  before sensitive environments.

> 🧠 CI answers "is this build good?" CD answers "let's ship this good build, carefully, environment by
> environment." Together they're the full automated path from `git push` to live service.

**The promotion ladder:**
```
CI builds image ──► CD-1 deploy to UAT ──► (validate) ──► CD-2/3 deploy to PROD (after human approval)
```

---

## Step 1 — Define environments with approvals

In Azure DevOps → **Pipelines → Environments → New environment**, create three: `UAT`, `Pre-PROD`,
`PROD`. On **PROD**, add an **Approval check** (Approvals & checks → Approvals → add yourself/gatekeepers).

> **Why approvals on PROD?** This is the **human gate** ("gatekeepers" in your doc step 7). Even with
> perfect automation, a person must consciously click "approve" before real users are affected. UAT
> deploys automatically (fast feedback); PROD requires sign-off (safety). That balance is the whole point
> of multi-environment CD.

---

## Step 2 — CD-1: deploy to UAT (`Mlops-Sample-UAT`)

Triggered after CI-1 succeeds on `develop`. Pulls the dev image from **ACR002** and deploys to **UAT AKS
+ APIM**. Create `azure-pipelines/cd/mlops-sample-uat.yml`:

```yaml
# CD-1 : Mlops-Sample-UAT
# Deploy the latest develop image (ACR002) to UAT AKS + APIM.
trigger: none                          # triggered by CI completion, not by code push
resources:
  pipelines:
    - pipeline: ci
      source: kivas-udp-apps-ml-mlops   # CI-1
      trigger: true                    # run this CD when CI-1 finishes
variables:
  - group: mlops-common
  - name: tag
    value: '$(resources.pipeline.ci.runID)'
pool: { vmImage: ubuntu-latest }
stages:
  - stage: DeployUAT
    jobs:
      - deployment: deploy
        environment: UAT               # ties to the UAT environment
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  displayName: Set image on UAT AKS
                  inputs:
                    azureSubscription: 'azure-mlops-connection'
                    scriptType: pscore
                    scriptLocation: inlineScript
                    inlineScript: |
                      az aks get-credentials -n aks-predmaint-uat -g rg-predmaint-uat --overwrite-existing
                      kubectl set image deployment/predmaint-api `
                        predmaint-api=$(acrDev).azurecr.io/$(imageName):$(tag)
                      kubectl rollout status deployment/predmaint-api
```

> **Why `kubectl set image` + `rollout status`?** This does a **rolling update**: Kubernetes swaps pods
> to the new image one at a time, keeping the API up the whole time, and `rollout status` waits to confirm
> success (and auto-rolls-back if the new pods fail their health checks from file `09`). Zero-downtime
> deploys, automatically.
> **Why `trigger: none` + pipeline resource?** CD shouldn't run on code pushes — it runs when *CI
> finishes*. This chains CI → CD automatically.

---

## Step 3 — CD-2: deploy to PROD for local testing (`Mlops-Sample-PROD`)

Per your doc step 4, this pushes the develop-experiment image to **PROD ACR002** and deploys for local
PROD testing (not the final release). Create `azure-pipelines/cd/mlops-sample-prod.yml`:

```yaml
# CD-2 : Mlops-Sample-PROD
# Deploy PROD-ACR002 image to PROD AKS for local testing in PROD (pre-release).
trigger: none
resources:
  pipelines:
    - pipeline: ciprod
      source: kivas-udp-apps-ml-mlops-prod   # CI-2
      trigger: true
variables:
  - group: mlops-common-prod
pool: { vmImage: ubuntu-latest }
stages:
  - stage: DeployProdLocalTest
    jobs:
      - deployment: deploy
        environment: Pre-PROD                # validate here before true release
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  inputs:
                    azureSubscription: 'azure-mlops-connection-prod'
                    scriptType: pscore
                    scriptLocation: inlineScript
                    inlineScript: |
                      az aks get-credentials -n aks-predmaint-prod -g rg-predmaint-prod --overwrite-existing
                      kubectl set image deployment/predmaint-api `
                        predmaint-api=$(acrProdDev).azurecr.io/$(imageName):$(tag)
                      kubectl rollout status deployment/predmaint-api
```

> **Why deploy to PROD *before* the real release?** Some bugs only appear with production-grade config/
> data. CD-2 lets the team validate in a PROD-like setting (Pre-PROD) using the PROD ACR002 image, before
> committing to the official signed release. It's an extra safety rung on the ladder.

---

## Step 4 — CD-3: the 3-step signed release (`Mlops-Sample-PROD-Release`)

This is the big one — it matches **step 6–7** of your architecture doc. Triggered after CI-3 (main branch,
signed image in UAT ACR001). It's **three steps**:

1. **Import** the validated, **signed** image from **UAT ACR001 → UAT ACR002**, update UAT AKS. (APIM
   doesn't change since the backend URL/openapi is the same.)
2. **(Gatekeeper approval)** Import the signed image from **UAT ACR001 → PROD ACR001**, deploy to **PROD
   AKS**.
3. **Update PROD APIM** for final PROD validation.

Create `azure-pipelines/cd/mlops-sample-prod-release.yml`:

```yaml
# CD-3 : Mlops-Sample-PROD-Release  — 3-step signed release
trigger: none
resources:
  pipelines:
    - pipeline: cirelease
      source: kivas-udp-apps-ml-mlops-release   # CI-3 (signed image in UAT ACR001)
      trigger: true
variables:
  - group: mlops-common
  - name: tag
    value: '$(resources.pipeline.cirelease.runID)'
pool: { vmImage: ubuntu-latest }

stages:
# ----- STEP 1: import signed image UAT ACR001 -> UAT ACR002, update UAT AKS -----
- stage: Step1_UAT_Import
  jobs:
    - deployment: import_uat
      environment: UAT
      strategy:
        runOnce:
          deploy:
            steps:
              - task: AzureCLI@2
                displayName: Verify signature, import to UAT ACR002, update AKS
                inputs:
                  azureSubscription: 'azure-mlops-connection'
                  scriptType: pscore
                  scriptLocation: inlineScript
                  inlineScript: |
                    notation verify $(acrRelease).azurecr.io/$(imageName):$(tag)
                    az acr import --name $(acrDev) `
                      --source $(acrRelease).azurecr.io/$(imageName):$(tag) `
                      --image $(imageName):$(tag)
                    az aks get-credentials -n aks-predmaint-uat -g rg-predmaint-uat --overwrite-existing
                    kubectl set image deployment/predmaint-api `
                      predmaint-api=$(acrDev).azurecr.io/$(imageName):$(tag)
                    kubectl rollout status deployment/predmaint-api

# ----- STEP 2: (APPROVAL) import to PROD ACR001, deploy to PROD AKS -----
- stage: Step2_PROD_Deploy
  dependsOn: Step1_UAT_Import
  jobs:
    - deployment: deploy_prod
      environment: PROD                 # <-- this environment has the approval gate
      strategy:
        runOnce:
          deploy:
            steps:
              - task: AzureCLI@2
                displayName: Import signed image to PROD ACR001 and deploy PROD AKS
                inputs:
                  azureSubscription: 'azure-mlops-connection-prod'
                  scriptType: pscore
                  scriptLocation: inlineScript
                  inlineScript: |
                    az acr import --name $(acrReleaseProd) `
                      --source $(acrRelease).azurecr.io/$(imageName):$(tag) `
                      --image $(imageName):$(tag)
                    az aks get-credentials -n aks-predmaint-prod -g rg-predmaint-prod --overwrite-existing
                    kubectl set image deployment/predmaint-api `
                      predmaint-api=$(acrReleaseProd).azurecr.io/$(imageName):$(tag)
                    kubectl rollout status deployment/predmaint-api

# ----- STEP 3: update PROD APIM for final validation -----
- stage: Step3_PROD_APIM
  dependsOn: Step2_PROD_Deploy
  jobs:
    - deployment: update_apim
      environment: PROD
      strategy:
        runOnce:
          deploy:
            steps:
              - task: AzureCLI@2
                displayName: Update PROD APIM backend / API
                inputs:
                  azureSubscription: 'azure-mlops-connection-prod'
                  scriptType: pscore
                  scriptLocation: inlineScript
                  inlineScript: |
                    az apim api import --resource-group rg-predmaint-prod `
                      --service-name apim-predmaint-prod `
                      --path predmaint --api-id predmaint-api `
                      --specification-format OpenApi `
                      --specification-url http://<prod-aks-service-ip>/openapi.json
```

> **Why import the *same signed* image between registries instead of rebuilding?** Rebuilding could
> produce a *different* image (different dependencies, timestamps) than the one you tested and signed.
> **Importing the exact signed artifact** guarantees that what you validated in UAT is **byte-for-byte**
> what runs in PROD. This is the core promise of a trustworthy release: *test once, ship that exact thing.*
> **Why is step 2 gated by approval and step 1 isn't?** Step 1 only updates UAT (safe). Step 2 touches
> PROD users → requires a gatekeeper to approve. Step 3 flips PROD APIM to finalize.

---

## How CD lines up with your screenshots

| Your doc | This file | From → To | Gate |
|----------|-----------|-----------|------|
| CD-1 `Mlops-Sample-UAT` | `mlops-sample-uat.yml` | ACR002 → UAT AKS+APIM | auto |
| CD-2 `Mlops-Sample-PROD` | `mlops-sample-prod.yml` | PROD ACR002 → PROD AKS (test) | Pre-PROD |
| CD-3 `Mlops-Sample-PROD-Release` | `mlops-sample-prod-release.yml` | ACR001 → ACR002 (UAT) → PROD ACR001 → PROD AKS → APIM | **approval on PROD** |

---

## The complete end-to-end flow (your 7 steps, automated)

1. Feature → develop PR → **CI-1** builds to UAT ACR002 → **CD-1** deploys to UAT.
2. Test inference on UAT APIM URL.
3. Develop experiments → **CI-2** builds to PROD ACR002 → **CD-2** deploys to Pre-PROD for PROD-local testing.
4. UAT complete → PR develop → main → **CI-3** builds, pushes to UAT ACR001, **signs + verifies**.
5. **CD-3 step 1**: import signed image UAT ACR001 → ACR002, update UAT AKS.
6. **CD-3 step 2** (gatekeeper approves): import to PROD ACR001, deploy PROD AKS.
7. **CD-3 step 3**: update PROD APIM → live for users. ✅

---

## ✅ Checkpoint

- [ ] Environments `UAT`, `Pre-PROD`, `PROD` exist; PROD has an approval check.
- [ ] CD-1 deploys to UAT automatically when CI-1 finishes.
- [ ] You can explain the 3 steps of CD-3 and why the signed image is *imported*, not rebuilt.
- [ ] You can narrate the full feature→develop→main→PROD journey.

Next → **`13_quality_security_monitoring.md`** — the guards: code quality, security, and drift monitoring. 🛡️
