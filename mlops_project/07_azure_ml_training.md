# 07 — Train in Azure ML & Register the Model

> **Goal:** Run our training in **Azure Machine Learning** (not just on the laptop), track the run's
> metrics, and **register the trained model** in the Model Registry so it can be versioned, approved,
> and deployed.

> ✅ **Prerequisite:** File `06` (Kedro pipelines work and write to ADLS) and file `04` (the `$MLW`
> workspace exists).

---

## 🤔 Why move training to Azure ML? (the "why" first)

Our Kedro pipeline already trains a model on the laptop. So why involve Azure ML?

1. **Compute power** — real models need more CPU/RAM/GPU than a laptop. Azure ML rents big machines on demand.
2. **Experiment tracking** — every training run is recorded: which params, which metrics, which data. So
   you can answer "why did we pick *this* model?" months later.
3. **Model Registry** — a versioned shelf of trained models (`model:1`, `model:2`, …) with stages like
   "approved for production." This is what CI/CD deploys from.
4. **Reproducibility** — runs happen in a defined environment, not "whatever was on my laptop."

> 🧠 Think of Azure ML as the **lab notebook + product shelf** for models. The registry is the bridge
> between "data science" and "deployment": only registered, approved models get shipped.

---

## Step 1 — Connect to your Azure ML workspace

Install the SDK (in your `.venv`):
```powershell
pip install azure-ai-ml azure-identity mlflow azureml-mlflow
```

Create `aml/config.py` to connect:
```python
# aml/config.py
from azure.ai.ml import MLClient
from azure.identity import DefaultAzureCredential

ml_client = MLClient(
    credential=DefaultAzureCredential(),
    subscription_id="<your-subscription-id>",
    resource_group_name="rg-predmaint-uat",     # your $RG
    workspace_name="mlw-predmaint-uat",          # your $MLW
)
```
> `DefaultAzureCredential` reuses your `az login` session — no passwords in code. **Why?** Secure by
> default: the SDK proves who you are via your existing Azure login, not a hard-coded secret.

---

## Step 2 — Add experiment tracking to training (MLflow)

Azure ML speaks **MLflow**, the standard for logging ML runs. We add a few `mlflow.log_*` lines to our
training node so metrics and the model get recorded. Update `model_training/nodes.py`:

```python
import mlflow

def train_model(X_train, y_train, model_params: dict):
    mlflow.sklearn.autolog()              # auto-logs params, metrics, and the model
    model = RandomForestRegressor(**model_params)
    model.fit(X_train, y_train.values.ravel())
    return model

def evaluate_model(model, X_test, y_test) -> dict:
    pred = model.predict(X_test)
    metrics = {
        "mse": float(mean_squared_error(y_test, pred)),
        "mae": float(mean_absolute_error(y_test, pred)),
        "r2":  float(r2_score(y_test, pred)),
    }
    for name, value in metrics.items():
        mlflow.log_metric(name, value)    # record each metric in Azure ML
    return metrics
```

> **Why MLflow?** It's the universal "flight recorder" for ML. With `autolog()` plus a few `log_metric`
> calls, every run shows up in Azure ML Studio with its params, metrics, and artifacts — comparable
> side by side. No manual spreadsheet of results.

---

## Step 3 — Run training as an Azure ML Job

Instead of `kedro run` on your laptop, submit it as a **job** to Azure ML compute. Create
`aml/train_job.yml`:
```yaml
# aml/train_job.yml — defines a training job for Azure ML
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
command: >
  pip install -r requirements.txt &&
  kedro run --pipeline=__default__
environment:
  image: mcr.microsoft.com/azureml/openmpi4.1.0-ubuntu20.04:latest
compute: azureml:cpu-cluster
experiment_name: predictive-maintenance
display_name: rf-rul-training
```

First create a small compute cluster (auto-scales to 0 when idle = cheap):
```powershell
az ml compute create --name cpu-cluster --type AmlCompute `
  --min-instances 0 --max-instances 1 --size Standard_DS3_v2 `
  --resource-group $RG --workspace-name $MLW
```

Submit the job:
```powershell
az ml job create --file aml/train_job.yml --resource-group $RG --workspace-name $MLW
```

> **Why `min-instances 0`?** The cluster shuts down to **zero machines when not training**, so you pay
> nothing while idle and it spins up only for the job. This is **elasticity** (AZ-900!) saving you money.

You can watch the run in **Azure ML Studio** (ml.azure.com) → your workspace → **Jobs**. You'll see the
metrics (MSE/MAE/R²) logged for the run.

---

## Step 4 — Register the model

Once a run produces a good model, register it so it's versioned and deployable:
```powershell
az ml model create `
  --name predictive-maintenance-rul `
  --version 1 `
  --path azureml://datastores/workspaceblobstore/paths/models/model.pkl `
  --type custom_model `
  --resource-group $RG --workspace-name $MLW
```
Or from Python after a run:
```python
# aml/register_model.py
from azure.ai.ml.entities import Model
from azure.ai.ml.constants import AssetTypes
from config import ml_client

model = Model(
    path="abfs://predictive-maintenance/models/model.pkl",  # or local path
    name="predictive-maintenance-rul",
    type=AssetTypes.CUSTOM_MODEL,
    description="RandomForest RUL model. Trained on FD001.",
)
registered = ml_client.models.create_or_update(model)
print(f"Registered {registered.name} version {registered.version}")
```

> **Why register instead of just using `model.pkl` from ADLS?** Registration gives the model a **name +
> version + lineage** (which run/data produced it) and an **approval stage**. CI/CD deploys "the latest
> *approved* version of `predictive-maintenance-rul`," not a random file. It's the controlled handoff
> from training to production.

---

## Step 5 — (Concept) Register the scaler too

Remember from file `06`: the **scaler is part of the model** — the API must scale incoming data the same
way. Register it alongside, or bundle scaler + model into one artifact:
```python
# Recommended: save a single bundle so they can never drift apart
import joblib
joblib.dump({"model": model, "scaler": scaler}, "model_bundle.pkl")
```
> **Why bundle them?** If the model and scaler are registered separately, someone could deploy v2 of the
> model with v1 of the scaler → wrong predictions, silently. Bundling guarantees they always match. A
> subtle but real production bug we're preventing.

---

## Step 6 — Understand the registry → deployment bridge

Here's the flow you've now set up, and where it goes next:

```
Kedro training  ─►  Azure ML Job  ─►  metrics logged + model registered (v1, v2, …)
                                              │
                                  approve best version
                                              ▼
                            file 08: pull registered model into FastAPI image
```

> The Model Registry is the **single decision point**: "this version is good enough to ship." Everything
> downstream (Docker, ACR, AKS) deploys whatever the registry says is approved.

---

## ✅ Checkpoint

- [ ] A training **job** appears in Azure ML Studio with MSE/MAE/R² logged.
- [ ] A model named `predictive-maintenance-rul` shows in the **Models** registry with a version.
- [ ] You understand why we register (versioning + approval + lineage) vs using a loose `.pkl`.
- [ ] You understand why the **scaler is bundled with the model**.

Next → **`08_fastapi_docker_acr.md`** — wrap the model in an API, containerize it, push to ACR. 🌐🐳
