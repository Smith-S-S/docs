# 01 — Understand the ML Project (the notebook explained)

> **Goal of this file:** Before we touch Azure, you must understand *what the model does and how*.
> You can't deploy what you don't understand. We'll walk through `predictive_maintenance.ipynb`
> in plain English. **No Azure here — just the data science.**

---

## 🎯 The business problem (why this project exists)

Imagine you run an airline. Jet engines are insanely expensive, and if one fails mid-flight, that's a
catastrophe. So you do **maintenance**. But maintenance has two bad extremes:

- **Too early** = you replace healthy parts → wasted money + downtime.
- **Too late** = the engine fails → danger + huge cost.

**Predictive maintenance** finds the sweet spot: use sensor data to predict **how much life an engine
has left**, so you service it *just before* it would fail. That "life left" number is called the
**Remaining Useful Life (RUL)**.

> 🧠 **Why ML?** An engine has 21 sensors producing numbers every cycle. A human can't eyeball
> thousands of sensor streams and predict failure. A model can learn the patterns that precede failure.

---

## 📦 The data (NASA turbofan engine dataset)

The notebook downloads a famous NASA dataset (`FD001`). It simulates engines running until they break.

**What one row looks like:** each row = one engine, at one moment in time ("cycle"), with its sensor readings.

| Column group | Example columns | Meaning |
|--------------|-----------------|---------|
| Identity | `UnitNumber` | Which engine (1–100) |
| Time | `Cycle` | Which time-step (1, 2, 3, … until failure) |
| Settings | `Op_Setting_1..3` | Operating conditions (e.g. altitude, throttle) |
| Sensors | `Sensor_1..21` | Temperature, pressure, vibration, etc. |

- **Training data** (`train_FD001.txt`): each engine runs **until it fails**. So the last cycle of each
  engine = the moment of failure.
- **Test data** (`test_FD001.txt`): engines are stopped *before* failure; a separate file
  (`RUL_FD001.txt`) tells us the true remaining life so we can score our predictions.

> 💡 **Why two files?** Same logic as any ML project: you **train** on data where you know the answer,
> then **test** on unseen data to check the model didn't just memorize.

---

## 🔧 Step-by-step: what the notebook code does

I'll map each part of the notebook to a job. This *same logic* gets rebuilt as Kedro pipelines later,
so understanding it now makes file `06` easy.

### 1. Download & load the data
Downloads the zip from NASA, extracts the `.txt` files, and reads them into pandas DataFrames.
The raw files have **no column names** and two junk empty columns at the end.

### 2. Clean the data
```python
train_df.drop(train_df.columns[[26, 27]], axis=1, inplace=True)  # drop 2 empty (NaN) columns
train_df.columns = column_names                                  # add proper names
```
> **Why drop columns 26 & 27?** They're all `NaN` (empty) — leftovers from the file format. Garbage in
> = garbage out, so we remove them.

### 3. Create the target: Remaining Useful Life (RUL)
This is the clever part. In training data, an engine's **last cycle = failure**. So for any earlier
cycle, RUL = (last cycle) − (current cycle).
```python
max_cycle = train_df.groupby('UnitNumber')['Cycle'].max()   # last cycle per engine
train_df["target_RUL"] = max_cycle - train_df["Cycle"]       # how many cycles are left
```
Example: if engine #1 dies at cycle 192, then at cycle 50 it has `192 - 50 = 142` cycles left.

> **Why compute it ourselves?** The dataset doesn't hand us the answer column — we *derive* it from the
> structure of the data ("engine ran until it broke"). This is **feature/label engineering.**

For the **test set**, engines didn't run to failure, so the true RUL comes from the separate
`RUL_FD001.txt` file and is added on.

### 4. Pick features & avoid traps
```python
basic_features = train_df.columns.difference(["UnitNumber","target_RUL"])
```
We use the sensors + settings as inputs (`X`), and `target_RUL` as the answer (`y`). We exclude
`UnitNumber` (the engine's ID number tells us nothing about its health).

> ⚠️ **Two traps the notebook warns about:**
> - **Target leakage** — accidentally feeding the model info it won't have in real life (cheating).
> - **Domain mismatch** — if training data looks different from real-world data, the model fails in
>   production. *This is exactly why MLOps monitors "drift" later — see file `13`.*

### 5. Feature scaling (preprocessing)
```python
scaler = preprocessing.StandardScaler()
X = scaler.fit_transform(X_unscaled)   # rescale all sensors to comparable ranges
```
> **Why scale?** Sensor 1 might read in thousands, Sensor 5 in decimals. Many ML algorithms get
> confused when numbers are on wildly different scales. `StandardScaler` puts them all on a common
> footing (mean 0, similar spread). **Important:** we `fit` the scaler on training data, then only
> `transform` the test data — never re-fit on test data (that would leak info).

### 6. Train a baseline model (Random Forest)
```python
simple_rf = ensemble.RandomForestRegressor(n_estimators=200, max_depth=15)
simple_rf.fit(X, y)
```
A **Random Forest** is a model made of many decision trees that vote together. It's a great *baseline*:
robust, needs little tuning, and tells us which sensors matter most (**feature importance**).

> **Why a baseline first?** You always want a simple "good enough" model to compare against. If your
> fancy model can't beat the baseline, the fancy model isn't worth it.

### 7. Measure how good it is (metrics)
```python
mean_squared_error(y, y_pred)   # MSE  - average squared error (punishes big misses)
mean_absolute_error(y, y_pred)  # MAE  - average error in "cycles" (easy to read)
r2_score(y, y_pred)             # R²   - 0..1, how much variance the model explains
```
> **Why three metrics?** Each tells a different story. **MAE** is the most intuitive: "on average we're
> off by N cycles." **R²** tells you overall fit (closer to 1 = better). We watch these to know if a
> *new* model is better than the current one — central to deciding what to deploy.

### 8. Improve it (hyper-parameter tuning)
```python
GridSearchCV(estimator=rf, param_grid={...}, cv=GroupKFold(5), scoring='neg_mean_squared_error')
```
This tries many settings (different tree depths, leaf sizes) and **cross-validates** to find the best
combo. Note `GroupKFold` groups by `UnitNumber` so the *same engine* is never in both train and
validation — preventing leakage again.

> **Why tune?** The baseline uses default settings. Tuning squeezes out extra accuracy by searching for
> the best configuration automatically, instead of guessing.

---

## 🧩 How this maps to the MLOps pipeline (the big payoff)

Here's the key insight: **every notebook step becomes a reusable pipeline node later.**

| Notebook step | Becomes (Kedro node, file `06`) | Output stored in ADLS |
|---------------|----------------------------------|------------------------|
| Download/load + clean | `data_processing` pipeline | cleaned Parquet |
| Compute RUL, scale | `feature_engineering` pipeline | feature table + fitted scaler |
| Train Random Forest | `model_training` pipeline | trained model (`.pkl`) |
| GridSearch tuning | `model_tuning` pipeline | best model + metrics |
| Metrics + importance plot | logged as **artifacts/graphs** | metrics.json, feature_importance.png |

> 🧠 **The whole idea of MLOps:** take this one-time notebook and make each step **automatic, repeatable,
> versioned, and deployable.** That's what the rest of the tutorial does.

---

## ✅ Checkpoint — make sure you can answer these

Before moving on, can you explain (in your own words):
- [ ] What is **RUL** and why is it the thing we predict?
- [ ] Why do we **drop columns 26/27** and **scale** the features?
- [ ] What does the **Random Forest** do, and why start with it as a baseline?
- [ ] What do **MSE / MAE / R²** tell us?
- [ ] What are **target leakage** and **domain mismatch**, and why will MLOps monitoring care?

If yes → open **`02_mlops_concepts.md`** to learn *why* we wrap all this in MLOps tooling.
