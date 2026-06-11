# 05 — Get the Data into ADLS (the data lake)

> **Goal:** Download the NASA engine data and upload it to Azure Data Lake Storage, organized into
> proper folders. This becomes the **single source of truth** all our pipelines read from.

> ✅ **Prerequisite:** You finished file `04` (the `$ADLS`, `$RG`, `$KEY` variables and the
> `predictive-maintenance` container exist). If you closed PowerShell, re-paste the names block from `04`.

---

## 🤔 Why put data in ADLS at all? (the "why" first)

Right now the notebook downloads data to a `data/` folder on your laptop every time it runs. That's bad
for a real system because:

- **Not shared** — Azure ML compute and your teammates can't see your laptop's files.
- **Not reproducible** — "the data" changes if the download changes; nothing is pinned.
- **Not scalable** — real datasets are gigabytes; laptops run out of space.

ADLS fixes all three: one **central, versioned, scalable** home that every pipeline, the Azure ML
service, and CI/CD can all read. This is the "warehouse" from file `02`.

---

## 🗂️ The folder layout we'll create in ADLS

We mirror the **data journey** (raw → processed → models → reports) so anyone can see where data is in
its lifecycle. Inside the `predictive-maintenance` container:

```
predictive-maintenance/           (the container from file 04)
├── raw/                          # untouched original files (never edited)
│   ├── train_FD001.txt
│   ├── test_FD001.txt
│   └── RUL_FD001.txt
├── processed/                    # cleaned + feature data (Kedro writes here, file 06)
├── models/                       # trained model files (file 07)
└── reporting/                    # metrics + plots (feature importance, etc.)
```

> **Why keep `raw/` untouched forever?** It's your safety net. If a pipeline has a bug, you can always
> re-run from the pristine raw data. **Never overwrite raw data** — golden rule of data engineering.
> **Why Parquet later?** The raw files are `.txt` (slow, large). After processing, Kedro will save as
> **Parquet** — a compressed, columnar format that's far faster and smaller to read. (File `06`.)

---

## Step 1 — Download the data locally

We reuse the notebook's download logic. Save this as `download_data.py` in your project folder and run
it (with your `.venv` active):

```python
# download_data.py — fetches the NASA turbofan dataset into a local ./raw_data folder
import io, os, zipfile, urllib.request

os.makedirs("raw_data", exist_ok=True)
url = "https://phm-datasets.s3.amazonaws.com/NASA/6.+Turbofan+Engine+Degradation+Simulation+Data+Set.zip"

print("Downloading… (this is a NASA dataset, ~a few MB)")
outer = zipfile.ZipFile(io.BytesIO(urllib.request.urlopen(url).read()))
inner_name = [n for n in outer.namelist() if n.endswith(".zip")][0]
inner = zipfile.ZipFile(io.BytesIO(outer.read(inner_name)))
inner.extractall("raw_data")

# we only need these three files for FD001
keep = ["train_FD001.txt", "test_FD001.txt", "RUL_FD001.txt"]
print("Extracted files we care about:", [f for f in os.listdir("raw_data") if f in keep])
```

Run it:
```powershell
python download_data.py
```
You should now have `raw_data\train_FD001.txt`, `test_FD001.txt`, and `RUL_FD001.txt`.

> **Why a script instead of doing it in the notebook?** Scripts are repeatable and can run unattended
> (e.g. in a pipeline). It's the same "automate, don't click" principle from file `04`.

---

## Step 2 — Upload the raw files to ADLS

Using the Azure CLI (your `$ADLS` and `$KEY` variables from file `04`):

```powershell
$FILES = "train_FD001.txt","test_FD001.txt","RUL_FD001.txt"
foreach ($f in $FILES) {
  az storage fs file upload `
    --file-system "predictive-maintenance" `
    --source "raw_data\$f" `
    --path "raw/$f" `
    --account-name $ADLS `
    --account-key $KEY `
    --overwrite
  Write-Host "Uploaded raw/$f"
}
```

> **What just happened?** Each file went into the `raw/` folder inside the `predictive-maintenance`
> container. The `--path "raw/$f"` creates the folder structure automatically.

---

## Step 3 — Verify the upload

```powershell
az storage fs file list `
  --file-system "predictive-maintenance" `
  --account-name $ADLS `
  --account-key $KEY `
  --path "raw" `
  --output table
```
You should see your three files listed under `raw/`.

You can also **see it visually**: Azure portal → your storage account → **Storage browser** →
**Containers** → `predictive-maintenance` → `raw`. (Great for beginners to *see* the lake.)

---

## Step 4 — Understand how Kedro will connect to this (preview)

In file `06`, Kedro reads/writes ADLS through its **Data Catalog**. A taste of what that looks like
(don't create it yet — file `06` does):

```yaml
# conf/base/catalog.yml  (preview)
raw_train:
  type: pandas.CSVDataset
  filepath: abfs://predictive-maintenance/raw/train_FD001.txt
  credentials: adls_creds
  load_args:
    sep: " "
    header: null
```

```yaml
# conf/local/credentials.yml  (preview — NEVER committed to git)
adls_creds:
  account_name: stpredmaintssuat01
  account_key: <your-key>
```

> **Why this matters now:** Notice the path starts with `abfs://` — that's the "Azure Blob File System"
> protocol Kedro uses to talk to ADLS. The **credentials live in `conf/local/`** (git-ignored), keeping
> your storage key out of source control — exactly the security rule from file `03`.

---

## 🔐 A note on credentials (important habit)

Right now we're using the storage **account key** for simplicity. In a real company you'd use:
- **Azure Key Vault** to store the key, or
- **Managed Identity** (the app proves who it is without any key at all).

> **Why mention it?** So you know the simple way here is a *learning shortcut*. We'll point to the secure
> way again in the CI/CD files, where pipeline secrets replace hard-coded keys. **Never** paste your real
> key into a file that goes to git.

---

## ✅ Checkpoint

- [ ] `raw/train_FD001.txt`, `test_FD001.txt`, `RUL_FD001.txt` are visible in ADLS (CLI list or Storage browser).
- [ ] You understand why `raw/` is never edited.
- [ ] You understand that ADLS = the shared source of truth all pipelines read from.
- [ ] You know credentials go in `conf/local/` (git-ignored), not in code.

Next → **`06_kedro_pipelines.md`** — turn the notebook into real, reusable pipelines. 🔧
