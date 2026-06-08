# Module 5 — Storage & Databases (Where your data lives)

> Goal: Understand how Azure stores files and data, how it keeps copies safe (redundancy), how it
> saves you money on rarely-used data (tiers), and the main database options. The exam loves
> redundancy (LRS/ZRS/GRS) and tiers (hot/cool/archive).

---

## Topic 5.1 — Storage Accounts & Blob Storage

**First principle:** Apps produce *files and data* — photos, documents, logs, backups, videos. You
need a place to dump them cheaply and grab them over the internet. Azure's answer is the **Storage
Account**, and the most common thing inside it is **Blob Storage**.

- **Storage Account** = the top-level container for your Azure storage. It holds different storage
  types (blobs, files, queues, tables).
- **Blob Storage** = storage for **unstructured data** — "blobs" = Binary Large OBjects = any file:
  images, videos, documents, backups, logs. (This is Azure's equivalent of AWS S3.)

"Unstructured" just means "files that don't fit neatly in a database table" — a photo, a PDF, a video.

### Walk-through: 5 questions

**Q1. What does "unstructured data" mean and why does blob storage exist for it?**
Structured data fits in rows/columns (a database). **Unstructured** = files that don't — images,
videos, PDFs, logs. You can't sensibly put a 2 GB video in a database row, so blob storage gives you a
cheap, internet-accessible bucket to store huge numbers of files of any type.

**Q2. What's a "blob," really?**
Just a file. "BLOB" = Binary Large Object. A photo is a blob, a backup file is a blob, a log file is a
blob. Blob storage = a giant filing cabinet for these.

**Q3. Why a "storage account" wrapper instead of just storing files directly?**
The storage account is the management/billing/security boundary — it sets things like which region the
data lives in, how it's replicated, and access keys. Inside it you can use blobs, file shares, queues,
and tables. One account, multiple storage tools.

**Q4. When would I reach for blob storage in real life?**
Storing user uploads (profile pictures), website images/videos, backups, big data files, logs.
Anytime you have *files* to keep and serve, not relational records.

**Q5. Is blob storage like a database?**
No. A database stores **structured, queryable records** (customers, orders). Blob storage stores
**files**. Different jobs. You'd often use both: database for the order record, blob storage for the
invoice PDF attached to it.

> **Remember:** Storage Account = the container for Azure storage. Blob Storage = files/unstructured
> data (images, videos, backups, logs). Azure's version of AWS S3.

---

## Topic 5.2 — Redundancy: LRS, ZRS, GRS (keeping copies safe)

**First principle:** Disks fail, buildings flood, regions get hit by disasters. To not lose data, Azure
keeps **multiple copies**. *How far apart* those copies are = how much disaster you can survive = the
**redundancy** option. More spread = safer = (usually) pricier.

The three you must know:

| Option | Copies kept | Survives | Think |
|--------|-------------|----------|-------|
| **LRS** (Locally Redundant Storage) | 3 copies in **one datacenter** | a disk/server failure | cheapest, least safe |
| **ZRS** (Zone-Redundant Storage) | copies across **3 availability zones** in a region | a whole datacenter/zone failure | middle |
| **GRS** (Geo-Redundant Storage) | copies in your region **+ a paired region** far away | a whole **region** disaster | safest, pricier |

### Walk-through: 5 questions

**Q1. Why keep multiple copies at all — isn't one enough?**
Because hardware fails. If your only copy sits on a disk that dies, your data is gone. Redundancy keeps
several copies so a failure of one doesn't lose anything. The question is just *how far apart* the
copies are.

**Q2. What's the difference between LRS, ZRS, and GRS in one breath each?**
- **LRS** = 3 copies in **one building**. Survives a disk failure, but not the building burning down.
- **ZRS** = copies spread across **3 zones** (separate buildings) in the region. Survives one building
  going down.
- **GRS** = copies in your region **plus a far-away paired region**. Survives the *entire region* being
  destroyed.

**Q3. If GRS is safest, why not always use it?**
**Cost.** More copies, spread further, costs more. You match redundancy to how critical the data is.
Throwaway temp files → LRS is fine. Irreplaceable business data → GRS for region-level disaster protection.

**Q4. A flood destroys an entire region. Which option saves my data?**
**GRS** — it keeps a copy in a *paired region* hundreds of miles away. LRS and ZRS both keep all copies
*within one region*, so a region-wide disaster would lose them.

**Q5. Memory hook for the three?**
**L**RS = **L**ocal (one building). **Z**RS = **Z**ones (multiple buildings, one region). **G**RS =
**G**eography (two regions). Safety and price increase L → Z → G.

> **Remember:** LRS = 1 datacenter (cheapest). ZRS = 3 zones in a region. GRS = 2 regions (survives a
> regional disaster, safest). Pick based on how bad a disaster you must survive vs cost.

---

## Topic 5.3 — Access Tiers: Hot, Cool, Archive (saving money on cold data)

**First principle:** Not all data is accessed equally. Today's files are touched constantly; last
year's tax records almost never. Paying premium storage prices for rarely-touched data is wasteful.
**Access tiers** let you pay less for data you access less.

| Tier | For data you access... | Storage cost | Access cost/speed |
|------|------------------------|--------------|-------------------|
| **Hot** | frequently (active files) | highest | cheapest & instant to read |
| **Cool** | infrequently (≥30 days idle) | lower | costs more to read; instant |
| **Archive** | rarely (≥180 days; backups, compliance) | lowest | cheapest to store, but slow to retrieve (hours) + retrieval fee |

The trade-off flips: as **storage** gets cheaper (Hot→Cool→Archive), **accessing** the data gets more
expensive/slower.

### Walk-through: 5 questions

**Q1. Why have tiers instead of one price for all storage?**
Because data usage varies wildly. Charging the same for a file opened 1,000×/day and one opened
once/year is wasteful. Tiers let you pay *premium for active data* and *pennies for dormant data*,
optimizing cost.

**Q2. What's the catch with the cheaper tiers?**
The trade-off reverses: cheaper to *store* = more expensive/slower to *access*. **Archive** is dirt
cheap to hold data but you can't read it instantly — retrieval takes **hours** and costs extra. So
cheap tiers are only smart for data you *rarely* touch.

**Q3. Where would I put: live website images, 6-month-old logs, 7-year-old tax records?**
- Live website images → **Hot** (accessed constantly, need instant access).
- 6-month-old logs → **Cool** (rarely accessed but might need quickly).
- 7-year-old tax records kept for compliance → **Archive** (almost never accessed, cheapest to hoard).

**Q4. Why is Archive so slow to read?**
Because the data is essentially put into deep "cold storage" to make holding it ultra-cheap. Pulling it
back out (called *rehydration*) takes hours. That's the deal you accept for the lowest storage price.
Never use Archive for data you might need *now*.

**Q5. One-line rule for choosing a tier?**
**Access often → Hot. Access rarely → Cool. Almost never (but must keep) → Archive.** Storage gets
cheaper down the list; retrieval gets pricier/slower.

> **Remember:** Hot = frequent access (priciest storage, instant). Cool = infrequent (cheaper storage).
> Archive = rare access, backups/compliance (cheapest storage, but slow + costly to retrieve).

---

## Topic 5.4 — Databases: Azure SQL vs Cosmos DB

**First principle:** For *structured*, queryable data (customers, orders, transactions) you want a
**database**, not blob storage. Azure offers managed (PaaS) databases so you don't run database servers
yourself. The two headline names for AZ-900:

- **Azure SQL Database** = a managed **relational** database (tables with rows/columns, SQL queries).
  Microsoft handles patching, backups, scaling. Great for traditional business apps. (Relational =
  data organized in related tables, like Excel sheets that link to each other.)
- **Azure Cosmos DB** = a managed **NoSQL**, globally distributed database. Built for **massive scale**
  and **low latency worldwide**, with data automatically replicated across regions. Great for global
  apps needing fast access everywhere. (NoSQL = flexible, non-table data formats like JSON documents.)

### Walk-through: 5 questions

**Q1. Why use a managed database instead of installing one on a VM?**
Because on a VM *you* must patch, back up, secure, and scale the database — lots of work. A **managed
(PaaS)** database lets Microsoft do all that; you just use it. Less maintenance, more focus on your app.
(This is the IaaS-vs-PaaS idea again, applied to databases.)

**Q2. Relational (SQL) vs NoSQL (Cosmos) — what's the difference in plain terms?**
**Relational/SQL** = data in neat linked tables (rows & columns), great when data has a fixed,
structured shape (orders, invoices). **NoSQL** = flexible formats (like JSON documents) that don't need
a fixed table shape, great for varied or rapidly changing data and huge scale. Structured & traditional
→ SQL. Flexible & massive-scale → NoSQL.

**Q3. What's special about Cosmos DB's "globally distributed" feature?**
It can automatically keep copies of your data in **multiple regions around the world**, so users
everywhere get **fast, low-latency** access to nearby data. Ideal for global apps (a worldwide game,
a global e-commerce site) that need quick responses on every continent.

**Q4. Scenario: a traditional business app with customers and orders in structured tables. Which DB?**
**Azure SQL Database** — relational, structured, SQL queries, fully managed.

**Q5. Scenario: a global app needing single-digit-millisecond response times for users on every
continent, with flexible data. Which DB?**
**Azure Cosmos DB** — NoSQL, globally distributed, low latency worldwide.

> **Remember:** Azure SQL = managed **relational** (tables, SQL) for structured business data. Cosmos DB
> = managed **NoSQL**, globally distributed, ultra-low latency for global, flexible, massive-scale apps.

---

## ✅ Module 5 recap

1. Storage Account holds Azure storage; Blob Storage = files/unstructured data (images, videos, backups).
2. Redundancy (how many copies, how far apart): **LRS** = 1 datacenter, **ZRS** = 3 zones in a region,
   **GRS** = 2 regions (survives a regional disaster). Safety & cost rise L→Z→G.
3. Access tiers (match price to usage): **Hot** = frequent, **Cool** = infrequent, **Archive** = rare
   (cheapest to store, slow + costly to retrieve).
4. Databases: **Azure SQL** = managed relational (structured); **Cosmos DB** = managed NoSQL, globally
   distributed, low latency.

Next → `6_Identity_Governance_Management.md` (who's allowed in, and how you stay in control).
