# 06 — Build the Kedro Pipelines (notebook → production code)

> **Goal:** Convert the notebook's logic into **four clean Kedro pipelines** (data processing, feature
> engineering, model training, model tuning), reading raw data from ADLS and writing processed data,
> the model, metrics, and plots back to ADLS.

> This is the longest file — take it slowly. Each pipeline is just the notebook code from file `01`,
> reorganized into reusable functions ("nodes").

---

## 🤔 Why Kedro? (the "why" first)

A notebook is one long script. Kedro splits your work into:
- **Nodes** = plain Python functions, each doing one job (e.g. "clean data").
- **Pipelines** = nodes wired together (output of one feeds the next).
- **Data Catalog** = a YAML file that says *where* each dataset lives (local, ADLS, etc.) — so your code
  never hard-codes file paths.

**Benefits this gives us:**
- **Testable** — test one node without running everything.
- **Reusable** — the training node runs the same locally or in Azure ML.
- **Swappable storage** — point the catalog at ADLS without changing code.
- **Visualizable** — `kedro viz` draws your whole pipeline as a diagram.

> 🧠 The big idea: **logic stays in Python, configuration (paths, params) stays in YAML.** Change where
> data lives or a model setting without editing code.

---

## Step 1 — Install Kedro and create the project

With your `.venv` active (file `04`):

```powershell
pip install "kedro~=0.19.0" kedro-datasets[pandas] scikit-learn adlfs pyarrow matplotlib
kedro new --name=predictive_maintenance --tools=none --example=n
```
This scaffolds the standard structure (the `conf/`, `src/`, `data/` folders from file `03`).

```powershell
cd predictive-maintenance
pip install -r requirements.txt
```

> **Why `adlfs` and `pyarrow`?** `adlfs` lets Kedro read/write **ADLS** (the `abfs://` paths). `pyarrow`
> enables fast **Parquet** files. These are the bridges between your code and the cloud data lake.

---

## Step 2 — Tell Kedro where the data lives (the Data Catalog)

Edit `conf/base/catalog.yml`. This declares every dataset and where it lives — **raw from ADLS,
processed/model/reports back to ADLS.**

```yaml
# conf/base/catalog.yml
# ---------- RAW (read from ADLS) ----------
raw_train:
  type: pandas.CSVDataset
  filepath: abfs://predictive-maintenance/raw/train_FD001.txt
  credentials: adls_creds
  load_args: {sep: " ", header: null}

raw_test:
  type: pandas.CSVDataset
  filepath: abfs://predictive-maintenance/raw/test_FD001.txt
  credentials: adls_creds
  load_args: {sep: " ", header: null}

raw_rul:
  type: pandas.CSVDataset
  filepath: abfs://predictive-maintenance/raw/RUL_FD001.txt
  credentials: adls_creds
  load_args: {sep: " ", header: null}

# ---------- PROCESSED (write to ADLS as Parquet) ----------
train_clean:
  type: pandas.ParquetDataset
  filepath: abfs://predictive-maintenance/processed/train_clean.parquet
  credentials: adls_creds

test_clean:
  type: pandas.ParquetDataset
  filepath: abfs://predictive-maintenance/processed/test_clean.parquet
  credentials: adls_creds

X_train:
  type: pandas.ParquetDataset
  filepath: abfs://predictive-maintenance/processed/X_train.parquet
  credentials: adls_creds
X_test:
  type: pandas.ParquetDataset
  filepath: abfs://predictive-maintenance/processed/X_test.parquet
  credentials: adls_creds
y_train:
  type: pandas.ParquetDataset
  filepath: abfs://predictive-maintenance/processed/y_train.parquet
  credentials: adls_creds
y_test:
  type: pandas.ParquetDataset
  filepath: abfs://predictive-maintenance/processed/y_test.parquet
  credentials: adls_creds

# ---------- ARTIFACTS (model, scaler, metrics, plot) ----------
scaler:
  type: pickle.PickleDataset
  filepath: abfs://predictive-maintenance/models/scaler.pkl
  credentials: adls_creds

model:
  type: pickle.PickleDataset
  filepath: abfs://predictive-maintenance/models/model.pkl
  credentials: adls_creds

metrics:
  type: json.JSONDataset
  filepath: abfs://predictive-maintenance/reporting/metrics.json
  credentials: adls_creds

feature_importance_plot:
  type: matplotlib.MatplotlibWriter
  filepath: abfs://predictive-maintenance/reporting/feature_importance.png
  credentials: adls_creds
```

Now the credentials (git-ignored). Edit `conf/local/credentials.yml`:
```yaml
# conf/local/credentials.yml  — NEVER commit this file
adls_creds:
  account_name: stpredmaintssuat01      # your $ADLS name from file 04
  account_key: "<paste-your-storage-key-here>"
```
> Get your key: `az storage account keys list --account-name $ADLS --resource-group $RG --query "[0].value" -o tsv`

> **Why is this powerful?** Your Python code (next steps) will just say "give me `raw_train`" and "save
> this as `model`" — Kedro handles *all* the ADLS reading/writing and Parquet conversion. Code stays
> clean; storage details stay in YAML.

---

## Step 3 — Set tunable parameters

Edit `conf/base/parameters.yml`. These are the knobs from the notebook, now configurable:
```yaml
# conf/base/parameters.yml
drop_columns: [26, 27]            # the empty NaN columns to remove
model:
  n_estimators: 200
  max_depth: 15
tuning:
  param_grid:
    min_samples_leaf: [2, 10, 25, 50, 100]
    max_depth: [7, 8, 9, 10, 11, 12]
  cv_folds: 5
```
> **Why externalize these?** So you can tune the model by editing YAML — no code change, no risk of
> breaking logic. CI/CD can even override them per environment.

---

## Step 4 — Pipeline 1: Data Processing

Create `src/predictive_maintenance/pipelines/data_processing/nodes.py`. This is the notebook's
cleaning + RUL logic, as functions:

```python
# nodes.py — data processing (pure functions, no file paths!)
import pandas as pd

COLUMN_NAMES = (
    ["UnitNumber", "Cycle"]
    + [f"Op_Setting_{i}" for i in range(1, 4)]
    + [f"Sensor_{i}" for i in range(1, 22)]
)

def clean_columns(df: pd.DataFrame, drop_columns: list[int]) -> pd.DataFrame:
    """Drop the empty NaN columns and add proper headers."""
    df = df.drop(df.columns[drop_columns], axis=1)
    df.columns = COLUMN_NAMES
    return df

def add_train_rul(df: pd.DataFrame) -> pd.DataFrame:
    """RUL = (last cycle of the engine) - (current cycle)."""
    max_cycle = df.groupby("UnitNumber")["Cycle"].max().reset_index()
    max_cycle.columns = ["UnitNumber", "MaxOfCycle"]
    df = df.merge(max_cycle, on="UnitNumber", how="inner")
    df["target_RUL"] = df["MaxOfCycle"] - df["Cycle"]
    return df.drop("MaxOfCycle", axis=1)

def add_test_rul(test_df: pd.DataFrame, rul_df: pd.DataFrame) -> pd.DataFrame:
    """Test engines didn't run to failure; true RUL comes from the RUL file."""
    rul_df = rul_df.iloc[:, [0]]
    rul_df.columns = ["RUL_after_last"]
    rul_df["UnitNumber"] = rul_df.index + 1
    max_cycle = test_df.groupby("UnitNumber")["Cycle"].max().reset_index()
    max_cycle.columns = ["UnitNumber", "MaxOfCycle"]
    max_cycle["MaxOfCycle"] += rul_df["RUL_after_last"]
    test_df = test_df.merge(max_cycle, on="UnitNumber", how="inner")
    test_df["target_RUL"] = test_df["MaxOfCycle"] - test_df["Cycle"]
    return test_df.drop("MaxOfCycle", axis=1)
```

Create `src/predictive_maintenance/pipelines/data_processing/pipeline.py` (the wiring):
```python
from kedro.pipeline import Pipeline, node, pipeline
from .nodes import clean_columns, add_train_rul, add_test_rul

def create_pipeline(**kwargs) -> Pipeline:
    return pipeline([
        node(clean_columns, ["raw_train", "params:drop_columns"], "train_named",
             name="clean_train"),
        node(clean_columns, ["raw_test", "params:drop_columns"], "test_named",
             name="clean_test"),
        node(add_train_rul, "train_named", "train_clean", name="rul_train"),
        node(add_test_rul, ["test_named", "raw_rul"], "test_clean", name="rul_test"),
    ])
```

> **Read the `node(...)` calls like sentences:** "run `clean_columns` using inputs `raw_train` and the
> `drop_columns` param, and save the result as `train_named`." Kedro figures out the order from the
> inputs/outputs. This is the assembly line from file `02`.

---

## Step 5 — Pipeline 2: Feature Engineering

`pipelines/feature_engineering/nodes.py`:
```python
import pandas as pd
from sklearn.preprocessing import StandardScaler

def split_features(df: pd.DataFrame):
    """Choose inputs (X) and target (y); drop ID + target from X."""
    features = df.columns.difference(["UnitNumber", "target_RUL"])
    X = df[features].astype("float64")
    y = df[["target_RUL"]]
    return X, y

def fit_scaler(X_train: pd.DataFrame):
    """Fit the StandardScaler on TRAIN only (avoid leakage)."""
    scaler = StandardScaler().fit(X_train)
    return scaler

def apply_scaler(scaler, X: pd.DataFrame) -> pd.DataFrame:
    return pd.DataFrame(scaler.transform(X), columns=X.columns, index=X.index)
```

`pipelines/feature_engineering/pipeline.py`:
```python
from kedro.pipeline import Pipeline, node, pipeline
from .nodes import split_features, fit_scaler, apply_scaler

def create_pipeline(**kwargs) -> Pipeline:
    return pipeline([
        node(split_features, "train_clean", ["X_train_raw", "y_train"], name="split_train"),
        node(split_features, "test_clean", ["X_test_raw", "y_test"], name="split_test"),
        node(fit_scaler, "X_train_raw", "scaler", name="fit_scaler"),
        node(apply_scaler, ["scaler", "X_train_raw"], "X_train", name="scale_train"),
        node(apply_scaler, ["scaler", "X_test_raw"], "X_test", name="scale_test"),
    ])
```
*(Add `X_train_raw`, `X_test_raw` to the catalog as `MemoryDataset` or Parquet — MemoryDataset keeps them
only in RAM between nodes. Simplest: add them as Parquet entries like `X_train`.)*

> **Why fit the scaler on train only?** Exactly the leakage rule from file `01`. The fitted `scaler` is
> **saved to ADLS** because the API needs the *same* scaler at prediction time (file `08`). This is a
> classic MLOps detail beginners miss — the scaler is part of the deployed model.

---

## Step 6 — Pipeline 3: Model Training

`pipelines/model_training/nodes.py`:
```python
import pandas as pd
import matplotlib.pyplot as plt
import numpy as np
from sklearn.ensemble import RandomForestRegressor
from sklearn.metrics import mean_squared_error, mean_absolute_error, r2_score

def train_model(X_train, y_train, model_params: dict) -> RandomForestRegressor:
    model = RandomForestRegressor(**model_params)
    model.fit(X_train, y_train.values.ravel())
    return model

def evaluate_model(model, X_test, y_test) -> dict:
    pred = model.predict(X_test)
    return {
        "mse": float(mean_squared_error(y_test, pred)),
        "mae": float(mean_absolute_error(y_test, pred)),
        "r2":  float(r2_score(y_test, pred)),
    }

def plot_feature_importance(model, X_train):
    importances = model.feature_importances_
    idx = np.argsort(importances)[::-1]
    fig, ax = plt.subplots(figsize=(11, 9))
    ax.set_title("Feature ranking")
    ax.bar(range(X_train.shape[1]), importances[idx], align="center")
    ax.set_xticks(range(X_train.shape[1]))
    ax.set_xticklabels(X_train.columns[idx], rotation="vertical")
    ax.set_ylabel("importance")
    return fig
```

`pipelines/model_training/pipeline.py`:
```python
from kedro.pipeline import Pipeline, node, pipeline
from .nodes import train_model, evaluate_model, plot_feature_importance

def create_pipeline(**kwargs) -> Pipeline:
    return pipeline([
        node(train_model, ["X_train", "y_train", "params:model"], "model", name="train"),
        node(evaluate_model, ["model", "X_test", "y_test"], "metrics", name="evaluate"),
        node(plot_feature_importance, ["model", "X_train"], "feature_importance_plot",
             name="plot_importance"),
    ])
```

> **Why output `metrics` and `feature_importance_plot` to ADLS?** These are your **artifacts** — the
> evidence of how good this model is. Storing them means every training run leaves a record you can
> compare. Azure ML (file `07`) will also track these so you can pick the best model to deploy.

---

## Step 7 — Pipeline 4: Model Tuning (optional but shown)

`pipelines/model_tuning/nodes.py`:
```python
from sklearn.ensemble import RandomForestRegressor
from sklearn.model_selection import GroupKFold, GridSearchCV

def tune_model(X_train, y_train, train_clean, tuning: dict):
    rf = RandomForestRegressor(n_estimators=100)
    cv = GroupKFold(tuning["cv_folds"])
    search = GridSearchCV(rf, tuning["param_grid"],
                          cv=cv, scoring="neg_mean_squared_error", n_jobs=-1, verbose=1)
    search.fit(X_train, y_train.values.ravel(), groups=train_clean["UnitNumber"])
    return search.best_estimator_
```
> **Why pass `train_clean`?** For the `groups=UnitNumber` trick — so the same engine never sits in both
> train and validation folds (the leakage guard from file `01`). The tuned model can replace `model`.

---

## Step 8 — Register pipelines and run

Edit `src/predictive_maintenance/pipeline_registry.py`:
```python
from .pipelines import data_processing, feature_engineering, model_training, model_tuning

def register_pipelines():
    dp = data_processing.create_pipeline()
    fe = feature_engineering.create_pipeline()
    mt = model_training.create_pipeline()
    tune = model_tuning.create_pipeline()
    return {
        "__default__": dp + fe + mt,        # the full train flow
        "data_processing": dp,
        "feature_engineering": fe,
        "model_training": mt,
        "model_tuning": tune,
    }
```

Run the whole thing:
```powershell
kedro run                       # runs __default__: process -> features -> train
kedro run --pipeline=model_tuning   # run tuning separately when you want
```

Visualize it (great for understanding + screenshots for your handover):
```powershell
pip install kedro-viz
kedro viz run
```

> **What success looks like:** after `kedro run`, check ADLS — you'll see `processed/*.parquet`,
> `models/model.pkl`, `models/scaler.pkl`, `reporting/metrics.json`, and
> `reporting/feature_importance.png`. **The notebook is now a real, repeatable pipeline writing to the
> cloud.** 🎉

---

## ✅ Checkpoint

- [ ] `kedro run` completes without errors.
- [ ] ADLS now has `processed/`, `models/`, and `reporting/` files.
- [ ] You can explain: nodes = functions, pipelines = wiring, catalog = where data lives.
- [ ] You understand why the **scaler is saved** (the API needs it) and why we fit it on train only.

Next → **`07_azure_ml_training.md`** — run this in Azure ML and register the model. 🧪
