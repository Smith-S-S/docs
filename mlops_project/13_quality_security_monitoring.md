# 13 — Quality, Security & Monitoring (the guards)

> **Goal:** Add the four "inspector" tools from the stack: **SonarQube** (code quality), **Fortify /
> Mend** (security), and **Evidently** (model/data drift monitoring). These keep your system trustworthy
> *after* the pipelines are running.

> ✅ **Prerequisite:** CI/CD pipelines exist (files `11`–`12`); a deployed API (file `09`).

---

## 🤔 Why bother? (the "why" first)

Pipelines deploy code fast — but fast is dangerous if the code is **buggy**, **insecure**, or the model
has silently gone **stale**. These tools are the quality gates that a real company *requires* before and
after shipping:

| Tool | Guards against | When it runs |
|------|----------------|--------------|
| **SonarQube** | Bad code (bugs, smells, low test coverage) | CI, on every PR |
| **Fortify / Mend** | Security holes (vulnerable code & dependencies) | CI, on every PR |
| **Evidently** | Model drift (real world changed, model now wrong) | In production, continuously |

> 🧠 The first two protect the **code before it ships**. The last protects the **model after it ships**.
> Together they close the loop: nothing bad gets in, and nothing quietly rots once live.

---

## Part 1 — SonarQube (code quality gate)

**What:** Scans your code for bugs, "code smells" (messy patterns), duplication, and test coverage, then
gives a **pass/fail quality gate.**

**Why:** Messy code is where bugs hide and where future-you wastes hours. SonarQube enforces a minimum
standard *automatically* — no reliance on a reviewer noticing. If the gate fails, the PR can't merge.

**Setup:**
1. Add a SonarQube **service connection** in Azure DevOps (Project Settings → Service connections →
   SonarQube), name it `sonarqube-connection` (already referenced in CI-1, file `11`).
2. Add `sonar-project.properties` at the repo root:
```properties
sonar.projectKey=predictive-maintenance-mlops
sonar.projectName=Predictive Maintenance MLOps
sonar.sources=src,api
sonar.tests=tests
sonar.python.version=3.11
sonar.python.coverage.reportPaths=coverage.xml
sonar.qualitygate.wait=true
```
> **Why `sonar.qualitygate.wait=true`?** It makes the pipeline **block and fail** if the quality gate
> isn't met — turning quality from a suggestion into a hard requirement. That's the whole value.

The CI-1 `Quality` stage (file `11`) already runs `SonarQubePrepare/Analyze/Publish`. To feed it coverage:
```yaml
        - script: pytest tests/ --cov=src --cov-report=xml
          displayName: Tests with coverage
```

---

## Part 2 — Fortify / Mend (security scanning)

Two complementary kinds of security scanning, both run in CI:

| Type | Tool | Finds |
|------|------|-------|
| **SAST** (Static App Security Testing) | **Fortify** | Vulnerabilities in *your own code* (e.g. injection, unsafe handling) |
| **SCA** (Software Composition Analysis) | **Mend** (formerly WhiteSource) | Vulnerabilities in *third-party libraries* you depend on |

**Why both?** Most apps are *mostly* other people's code (your `requirements.txt` libraries). A bug can
be in **your code** (Fortify's job) *or* in a **library you imported** (Mend's job). You need to scan
both surfaces.

**Why this matters for ML especially:** ML projects pull in huge dependency trees (numpy, sklearn,
tensorflow, …). One vulnerable transitive dependency can compromise the whole service. SCA catches the
known-vulnerable versions and tells you to upgrade.

**Where they fit:** the CI-1 `Security` stage (file `11`). Replace the placeholder with the real tasks
(each vendor provides an Azure DevOps extension), conceptually:
```yaml
- stage: Security
  jobs:
    - job: scan
      steps:
        - task: FortifyScan@x         # SAST: scan our source code
          inputs: { applicationName: 'predictive-maintenance-mlops' }
        - task: MendScan@x            # SCA: scan dependencies in requirements.txt
          inputs: { failOnError: true }
```
> **Why fail the build on findings?** A "warning we ignore" gets ignored forever. Failing the build on
> high-severity issues forces them to be fixed before code ships. Security as a gate, not a hope.

> 🔐 **Tie-back:** This is the same trust theme as **image signing** in file `11`/`12`. Sign the image so
> it can't be tampered with; scan the code/deps so it isn't vulnerable in the first place. Defense in depth.

---

## Part 3 — Evidently (model & data drift monitoring)

This is the part that makes it **ML**Ops, not just DevOps.

**The problem:** Your model learned patterns from *past* engine data. Over months, real engines, sensors,
or operating conditions change. The model's inputs no longer match what it trained on → predictions
silently get worse. Nobody notices until something fails. This is **drift** (the "domain mismatch" warning
from file `01`, now happening in production).

**What Evidently does:** compares **live production data** against the **training data** (and live
predictions vs. expectations) and reports:
- **Data drift** — are incoming sensor distributions different from training?
- **Target/prediction drift** — are the predictions shifting in a way that suggests trouble?
- **Data quality** — missing values, out-of-range sensors, etc.

**Why this closes the MLOps loop:** when drift crosses a threshold, you **trigger retraining**
(re-run the Kedro training pipeline on fresh data → register a new model → CI/CD redeploys). The system
heals itself instead of rotting.

**Simple setup — a monitoring script** (`monitoring/drift_check.py`):
```python
import pandas as pd
from evidently.report import Report
from evidently.metric_preset import DataDriftPreset

# reference = the data the model was trained on (from ADLS)
reference = pd.read_parquet("abfs://predictive-maintenance/processed/X_train.parquet",
                            storage_options={"account_name": "stpredmaintssuat01",
                                             "account_key": "<key>"})
# current = recent data captured from live /predict requests
current = pd.read_parquet("abfs://predictive-maintenance/monitoring/recent_requests.parquet",
                          storage_options={...})

report = Report(metrics=[DataDriftPreset()])
report.run(reference_data=reference, current_data=current)
report.save_html("drift_report.html")

# Decide: if too many features drifted, signal a retrain
result = report.as_dict()
n_drifted = result["metrics"][0]["result"]["number_of_drifted_columns"]
if n_drifted > 5:
    print("::DRIFT DETECTED:: trigger retraining pipeline")
    # e.g. call the Azure ML training job or set a pipeline variable
```

**How to run it:** schedule it (e.g. an Azure DevOps **scheduled pipeline** nightly, or an Azure Function
on a timer — remember serverless from AZ-900!). It reads recent live data, compares, and alerts/retrains.

> **Why log live requests in the first place?** To monitor drift you must **capture production inputs**.
> Add a line in the FastAPI `/predict` endpoint (file `08`) to append each request to a monitoring store
> in ADLS. No captured data = nothing to compare = blind to drift.

```python
# add inside api/main.py predict() — capture inputs for monitoring
import json, datetime
with open("/tmp/requests.log", "a") as f:
    f.write(json.dumps({"ts": datetime.datetime.utcnow().isoformat(), **reading.dict()}) + "\n")
# a sidecar/job periodically ships these logs to ADLS monitoring/ folder
```

---

## The full quality/security/monitoring picture

```
        ┌──────────── CI (every PR) ────────────┐      ┌──── PRODUCTION (continuous) ────┐
PR  ──►  flake8 → pytest → SonarQube → Fortify → Mend  │   Evidently drift check          │
          (lint)  (tests) (quality)   (SAST)   (SCA)   │   → if drift: retrain + redeploy │
                          │ all must pass to merge      │            ▲                     │
                          ▼                              └────────────┘ (closes the loop)  │
                  build + SIGN image (files 11/12)
```

> **One sentence:** *Code is linted, tested, quality-gated, and security-scanned before it can merge; the
> image is signed before it can ship; and the live model is watched for drift so it can be retrained
> before it goes stale.* That's a trustworthy MLOps system.

---

## ✅ Checkpoint

- [ ] You can explain the difference between **SonarQube** (quality), **Fortify** (your code's security),
      and **Mend** (dependencies' security).
- [ ] You understand **drift** and why Evidently is what makes this "MLOps."
- [ ] You see how drift detection **triggers retraining** to close the loop.
- [ ] You know you must **capture live requests** to monitor drift at all.

Next → **`14_end_to_end_runbook.md`** — run the whole thing, troubleshoot, and prep your handover. 🏁
