# 09 — Deploy to AKS & Expose via APIM

> **Goal:** Run the container on **Azure Kubernetes Service (AKS)** so the API is always-on and
> auto-scaling, then put **API Management (APIM)** in front as a secure, managed front door.

> ✅ **Prerequisite:** Image `predmaint-api:v1` is in ACR (file `08`), AKS exists and `kubectl get nodes`
> works (file `04`).

---

## 🤔 Why Kubernetes + APIM? (the "why" first)

We *could* just run the Docker container on one VM. But:
- If that VM crashes → API down. **Kubernetes auto-restarts** crashed containers.
- If traffic spikes → one container can't cope. **Kubernetes auto-scales** to more copies (pods).
- We don't want the raw cluster exposed to the internet. **APIM** adds security, a stable URL, auth, and
  rate-limiting in front.

> 🧠 AKS = the **factory floor that keeps machines running**; APIM = the **reception desk + security
> guard** at the door. (Same analogies as file `02`.)

---

## Kubernetes vocabulary (just enough)

| Term | Plain meaning |
|------|---------------|
| **Pod** | The smallest unit — one running copy of your container |
| **Deployment** | A recipe that says "keep N pods of this image running" (and replaces dead ones) |
| **Service** | A stable network address that load-balances across the pods |
| **HPA** (Horizontal Pod Autoscaler) | Auto-adds/removes pods based on CPU load |

> **Why pods *and* deployments?** A pod alone is fragile (if it dies, it's gone). A **Deployment** watches
> the pods and guarantees the desired number always exist — self-healing. You almost never create pods
> directly; you create Deployments.

---

## Step 1 — Deployment manifest

`k8s/deployment.yml` (replace `<ACR>` with your registry name):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: predmaint-api
  labels: { app: predmaint-api }
spec:
  replicas: 2                       # start with 2 pods for high availability
  selector:
    matchLabels: { app: predmaint-api }
  template:
    metadata:
      labels: { app: predmaint-api }
    spec:
      containers:
        - name: predmaint-api
          image: <ACR>.azurecr.io/predmaint-api:v1
          ports:
            - containerPort: 8000
          resources:               # how much CPU/RAM each pod may use
            requests: { cpu: "250m", memory: "256Mi" }
            limits:   { cpu: "500m", memory: "512Mi" }
          readinessProbe:          # K8s waits for this before sending traffic
            httpGet: { path: /health, port: 8000 }
            initialDelaySeconds: 5
          livenessProbe:           # K8s restarts the pod if this fails
            httpGet: { path: /health, port: 8000 }
            initialDelaySeconds: 10
```

> **Why `replicas: 2`?** Two pods = if one dies, the other still serves → **high availability** (AZ-900!).
> **Why the probes?** `readinessProbe` stops traffic going to a pod until the model has loaded.
> `livenessProbe` auto-restarts a pod that has hung. This is the "self-healing" we keep promising — and
> it's *why* we added `/health` to the API in file `08`.
> **Why resource limits?** So one misbehaving pod can't hog the whole node. Predictable, fair sharing.

---

## Step 2 — Service manifest

`k8s/service.yml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: predmaint-api-svc
spec:
  type: LoadBalancer            # gives the service a reachable IP
  selector: { app: predmaint-api }
  ports:
    - port: 80                  # outside port
      targetPort: 8000          # the container's port
```

> **Why a Service?** Pods come and go (and each has a changing IP). A Service is a **stable front** that
> always routes to whichever pods are currently alive, spreading traffic across them. Apps talk to the
> Service, never to pods directly.

---

## Step 3 — Autoscaler manifest

`k8s/hpa.yml`:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: predmaint-api-hpa
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: predmaint-api }
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
```

> **Why an HPA?** This is **elasticity** for your API: when average CPU passes 70%, Kubernetes adds pods
> (up to 5); when load drops, it removes them (down to 2). You handle spikes without paying for idle
> capacity. Exactly the scale-out/scale-in idea from AZ-900, applied to containers.

---

## Step 4 — Deploy to AKS

```powershell
# make sure kubectl points at your cluster (from file 04)
az aks get-credentials --name $AKS --resource-group $RG

kubectl apply -f k8s/deployment.yml
kubectl apply -f k8s/service.yml
kubectl apply -f k8s/hpa.yml

# watch the pods come up
kubectl get pods -w          # press Ctrl+C when both show "Running"
```

Get the public IP of the service:
```powershell
kubectl get service predmaint-api-svc      # wait for EXTERNAL-IP to appear (not <pending>)
```
Test it:
```powershell
curl http://<EXTERNAL-IP>/health
# then open http://<EXTERNAL-IP>/docs in a browser and try /predict
```

> 🎉 Your model is now a **live, self-healing, auto-scaling service in the cloud.** If you delete a pod
> (`kubectl delete pod <name>`), watch Kubernetes instantly create a replacement.

> **Troubleshooting:** if pods show `ImagePullBackOff`, AKS can't pull from ACR — re-run the attach:
> `az aks update --name $AKS --resource-group $RG --attach-acr $ACR`.

---

## Step 5 — Put APIM in front (the secure gateway)

The LoadBalancer IP works but is **raw and unprotected**. APIM adds a managed, secured front door.

Create an APIM instance (note: this can take ~30–45 min to provision — start it and read ahead):
```powershell
az apim create --name "apim-predmaint-uat" --resource-group $RG `
  --publisher-name "PredMaint Team" --publisher-email "you@example.com" `
  --sku-name Developer --no-wait
```

Then in the **Azure portal → API Management → your instance → APIs → Add API → HTTP**:
1. Set the **backend URL** to `http://<EXTERNAL-IP>` (your AKS service).
2. Give it a path like `/predmaint`.
3. Add operations: `POST /predict` and `GET /health`.
4. Under **Settings**, require a **subscription key** (so only authorized callers can use it).

Now callers use a clean URL like `https://apim-predmaint-uat.azure-api.net/predmaint/predict` with a key.

> **Why APIM instead of exposing AKS directly?**
> - **Security** — requires API keys / auth; hides your cluster.
> - **Stable URL** — callers get one friendly address even if the backend changes.
> - **Rate limiting** — stop one caller from overwhelming the model.
> - **Monitoring** — see who calls, how often, how fast.
> This matches your architecture diagram's **AKS → APIM** box. The `Developer` SKU is the cheapest for
> learning (don't use it for real production traffic).

---

## Step 6 — How this maps to the multi-environment architecture

What you just built by hand is the **UAT** version. In the real flow (files `10`–`12`), CI/CD does these
exact `kubectl apply` and image steps automatically for **each environment** (UAT → Pre-PROD → PROD),
each with its own AKS + APIM + ACR. You now understand the *pieces* the pipelines will automate.

---

## 💸 Cost reminder

AKS + APIM cost money while running. When done for the day:
```powershell
az aks stop --name $AKS --resource-group $RG          # pause the cluster
# APIM Developer SKU has no stop; delete it if not needed: az apim delete --name apim-predmaint-uat --resource-group $RG
```

---

## ✅ Checkpoint

- [ ] `kubectl get pods` shows 2 Running pods; deleting one auto-creates a replacement.
- [ ] The service's EXTERNAL-IP serves `/health` and `/docs`.
- [ ] You can explain Pod vs Deployment vs Service vs HPA in one line each.
- [ ] You understand why APIM sits in front of AKS.

Next → **`10_devops_repos_branching.md`** — set up the repo and branching strategy that drives CI/CD. 🌿
