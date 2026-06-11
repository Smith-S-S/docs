# 08 — FastAPI + Docker + ACR (turn the model into a shippable service)

> **Goal:** Wrap the trained model in a **FastAPI** web service (so apps can request predictions),
> package it with **Docker** into a container image, and push that image to **ACR**.

> ✅ **Prerequisite:** A trained model + scaler exist (files `06`/`07`), and `$ACR` exists (file `04`).

---

## 🤔 Why this step? (the "why" first)

A `model.pkl` file just sits there. To be *useful*, other systems must be able to **ask it for a
prediction**. The standard way is an **API**: send sensor readings over HTTP, get back the predicted RUL.

Then, to run that API **reliably anywhere**, we put it in a **Docker container** (so it carries its exact
Python + libraries), and store that container in **ACR** (so Kubernetes can pull it). This is the
"service counter → shipping container → warehouse" chain from file `02`.

---

## Step 1 — Build the FastAPI inference service

This is a **separate, small app** (the `api/` folder from file `03`) — it only *loads the model and
serves predictions*. It does NOT include training code (keep the deployable image lean).

`api/schema.py` — defines what a request/response looks like:
```python
from pydantic import BaseModel

class EngineReading(BaseModel):
    # the 24 input features (3 op settings + 21 sensors), names match training
    Op_Setting_1: float
    Op_Setting_2: float
    Op_Setting_3: float
    Sensor_1: float
    Sensor_2: float
    # ... Sensor_3 .. Sensor_21 (list all 21)
    Cycle: float

class Prediction(BaseModel):
    remaining_useful_life: float
    unit: str = "cycles"
```
> **Why a schema?** Pydantic validates incoming data automatically. If someone sends a string where a
> number belongs, FastAPI rejects it with a clear error — no garbage reaches the model.

`api/main.py` — the service:
```python
import joblib
import pandas as pd
from fastapi import FastAPI
from schema import EngineReading, Prediction

app = FastAPI(title="Predictive Maintenance API", version="1.0")

# Load the bundled model + scaler ONCE at startup (not per request)
bundle = joblib.load("model_bundle.pkl")
model, scaler = bundle["model"], bundle["scaler"]

@app.get("/health")
def health():
    """Kubernetes pings this to know the app is alive."""
    return {"status": "ok"}

@app.post("/predict", response_model=Prediction)
def predict(reading: EngineReading):
    # turn the request into a single-row DataFrame with the right columns
    df = pd.DataFrame([reading.dict()])
    X = scaler.transform(df[scaler.feature_names_in_])   # SAME scaler as training
    rul = float(model.predict(X)[0])
    return Prediction(remaining_useful_life=round(rul, 1))
```

> **Why load the model once at startup?** Loading a model takes time. Doing it on every request would be
> slow. We load it once when the app boots, then reuse it for every prediction. Standard pattern.
> **Why use the saved scaler?** The model was trained on *scaled* data. Incoming data must be scaled the
> exact same way or predictions are nonsense. This is *why* file `07` bundled scaler + model together.

`api/requirements.txt`:
```
fastapi==0.111.0
uvicorn[standard]==0.30.0
scikit-learn==1.5.0
pandas==2.2.2
joblib==1.4.2
```
> **Why pin exact versions?** So the container builds identically every time. A surprise library upgrade
> can change model behavior. Pinning = reproducibility.

---

## Step 2 — Test the API locally first

Copy your `model_bundle.pkl` into `api/`, then:
```powershell
cd api
pip install -r requirements.txt
uvicorn main:app --reload --port 8000
```
Open **http://localhost:8000/docs** — FastAPI auto-generates an interactive page. Click `/predict`, "Try
it out", paste sample sensor values, and you'll get an RUL back.

> **Why test locally before Docker?** Debugging is 10× easier outside a container. Get it working here
> first; then containerize a *known-good* app. Never debug two new things at once.

---

## Step 3 — Containerize with Docker

`docker/Dockerfile`:
```dockerfile
# Start from a small, official Python image
FROM python:3.11-slim

# Don't run as root (security best practice)
RUN useradd --create-home appuser
WORKDIR /home/appuser/app

# Install deps first (Docker caches this layer if requirements don't change = faster builds)
COPY api/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy the app + the model bundle
COPY api/ .
COPY model_bundle.pkl .

USER appuser
EXPOSE 8000

# Start the API; 0.0.0.0 so it's reachable from outside the container
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

`docker/.dockerignore`:
```
__pycache__/
*.pyc
.venv/
data/
notebooks/
```

> **Why each choice?**
> - `python:3.11-slim` → small image = faster pulls, smaller attack surface.
> - **Install deps before copying code** → Docker layer caching; if only your code changes, it skips
>   re-installing packages (much faster rebuilds).
> - **Non-root user** → if the container is breached, the attacker isn't root. Security 101.
> - `.dockerignore` → keeps junk (and big data) out of the image, keeping it lean.

Build and test it locally:
```powershell
cd "C:\Users\User\Documents\code_base\Azure Export Form Beginner\project_mlops\predictive-maintenance"
docker build -t predmaint-api:local -f docker/Dockerfile .
docker run -p 8000:8000 predmaint-api:local
```
Visit http://localhost:8000/docs again — same API, now running inside a container. 🎉

> **Why does "it runs in a container" matter so much?** Because that exact container will run *identically*
> on AKS. No "works on my machine" surprises. The thing you tested *is* the thing that ships.

---

## Step 4 — Push the image to ACR

Log in to your registry and push:
```powershell
# Log Docker into your ACR
az acr login --name $ACR

# Tag the image with the ACR address (format: <acr>.azurecr.io/<repo>:<tag>)
$IMAGE = "$ACR.azurecr.io/predmaint-api:v1"
docker tag predmaint-api:local $IMAGE

# Push it
docker push $IMAGE
```

Verify it's there:
```powershell
az acr repository show-tags --name $ACR --repository predmaint-api --output table
```

> **Why tags like `:v1`?** Tags version your images so you can deploy a specific build and roll back if
> needed. In CI/CD (files `11`–`12`) the tag is usually the **build number or git commit**, so every
> image traces back to exact code. Never rely on `:latest` in production — it's ambiguous.

> 💡 **Pro shortcut:** `az acr build` can build the image *in the cloud* (no local Docker needed):
> `az acr build --registry $ACR --image predmaint-api:v1 -f docker/Dockerfile .`
> We use local build here so you *see* Docker working; CI will use cloud build later.

---

## ✅ Checkpoint

- [ ] `http://localhost:8000/docs` returns an RUL prediction (local, then in Docker).
- [ ] `docker build` and `docker run` work.
- [ ] `predmaint-api:v1` appears in ACR (`az acr repository show-tags`).
- [ ] You can explain why we load the model once, use the saved scaler, pin versions, run as non-root.

Next → **`09_aks_apim_deployment.md`** — run the container on Kubernetes and expose it via APIM. ☸️
