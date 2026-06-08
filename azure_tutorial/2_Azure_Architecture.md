# Module 2 — Azure Architecture (Where your stuff lives & how it's organized)

> Goal: Understand the *physical map* of Azure (regions, zones) and the *filing system* that keeps
> your resources organized (the hierarchy). Two separate ideas — keep them apart in your head.

---

## Topic 2.1 — Datacenters, Availability Zones, Regions, Region Pairs

**First principle:** Your cloud "computer" is real hardware sitting in a building somewhere. Azure
organizes those buildings in layers so your app can survive disasters. Each layer = "what happens if
*this* breaks?"

Work from smallest to biggest:

1. **Datacenter** = one physical building full of servers, power, cooling, network. The actual bricks.
2. **Availability Zone (AZ)** = one or more datacenters within a region that have *independent* power,
   cooling, and network. Zones in a region are physically separated (miles apart). If one zone loses
   power, the others keep running.
3. **Region** = a geographic area (e.g. "East US", "Central India") containing a group of datacenters/zones.
   When you create a resource, you pick a region — that's *where it physically lives.*
4. **Region Pair** = every Azure region is paired with another region in the same geography, ≥300 miles
   away. If a whole region goes down (natural disaster), the pair acts as backup.

### Walk-through: 5 questions

**Q1. Why does Azure separate datacenters into "zones" within a region?**
For *fault tolerance against local disasters*. If a single building loses power or floods, an app
spread across multiple zones keeps running from the other buildings. Zones are deliberately placed
far enough apart that one disaster can't take them all out.

**Q2. Why are zones "miles apart" but still in the same region?**
Two competing needs: **far enough** that a fire/flood/power-cut won't hit both, but **close enough**
that data can travel between them fast (low latency). Miles apart is the sweet spot.

**Q3. What does picking a "region" actually decide for me?**
Where your data and servers physically sit. You'd pick a region **close to your users** (lower
latency = faster app) and one that meets **data-residency laws** (some data must legally stay in a
country). Example: serving Indian users → "Central India" region.

**Q4. If a whole region is wiped out by an earthquake, what saves me?**
The **region pair**. Azure pairs each region with another ≥300 miles away in the same geography, so a
single disaster can't hit both. Critical data can be replicated to the pair for region-level disaster recovery.

**Q5. Gotcha: does every region support availability zones?**
**No.** Not all regions have availability zones. Some smaller/newer regions don't. If your app needs
zone-level resilience, you must pick a region that *supports* zones. (This exact "not all regions
support AZs" fact shows up on exams.)

> **Remember the nesting:** Datacenter ⊂ Availability Zone ⊂ Region ⊂ Region Pair.
> Zones protect against *building* failure; region pairs protect against *region* failure.
> Region pairs are ≥300 miles apart; not all regions have zones.

---

## Topic 2.2 — The Resource Hierarchy (Azure's filing system)

**First principle:** If a company runs thousands of cloud resources, it needs an *organizing system*
— for billing, permissions, and tidiness. Azure uses a 4-level tree. Permissions and policies set at
the top **flow downward** to everything beneath (this "inheritance" is the key idea).

Top to bottom:

1. **Management Group** *(optional)* = a folder for grouping **subscriptions**. Set a rule here once,
   and all subscriptions inside inherit it. Good for big companies with many subscriptions.
2. **Subscription** = a billing & access boundary. Resources in a subscription roll up to *one bill*.
   A company might use one subscription for "Sales", another for "IT". *(At least one is mandatory —
   you get a default one when you create your Azure account.)*
3. **Resource Group** = a folder for related **resources**. Logical grouping — e.g. all the pieces of
   your "Dev" app in one group. Delete the group → delete everything in it. A resource lives in
   **exactly one** resource group.
4. **Resource** = the actual thing: a VM, a database, a storage account, etc.

```
Management Group   (folder of subscriptions)   ← optional
   └── Subscription   (billing + access boundary)   ← at least 1 required
          └── Resource Group   (folder of related resources)
                   └── Resource   (VM, DB, storage, ...)
```

### Walk-through: 5 questions

**Q1. Why not just dump every resource in one big pile?**
Because you need to manage *billing, access, and lifecycle* in sensible chunks. Grouping lets you say
"bill the Sales team separately," "give the Dev team access to only Dev resources," and "delete the
whole test environment in one click." Organization = control.

**Q2. What's the difference between a Subscription and a Resource Group?**
A **Subscription** is mainly a **billing + access boundary** (everything inside lands on one invoice,
and is a unit for big-picture access control). A **Resource Group** is a **logical folder** for related
resources *inside* a subscription — for day-to-day organizing and bulk actions (like deleting a whole
environment at once).

**Q3. What does "inheritance / top-down" mean here, and why does it matter?**
Permissions and policies set at a *higher* level automatically apply to everything *below*. Set
"deny X" on a subscription → every resource group and resource under it also gets "deny X." This
means you can govern thousands of resources from one place. Exam phrasing: *"applied at the top,
inherited by all below."*

**Q4. Can one resource be in two resource groups at once?**
**No.** A resource belongs to exactly **one** resource group at a time. (You can *move* it to another,
but it's never in two simultaneously.)

**Q5. What's the point of Management Groups if Subscriptions already exist?**
Scale. A large enterprise might have *dozens* of subscriptions. Without management groups you'd set
the same policy on each subscription one by one. With a management group, you set it **once** at the
top and all subscriptions inherit it. They're **optional** — small setups don't need them — but
subscriptions are **mandatory** (you always have at least one).

> **Remember:** Management Group → Subscription → Resource Group → Resource.
> Rules set high up flow down (inheritance). One resource = one resource group.
> Subscription is mandatory; Management Group is optional.

---

## Topic 2.3 — The Azure Portal (your control panel)

**First principle:** You need *some way* to create and manage all this without writing code. The
**Azure Portal** (portal.azure.com) is the web dashboard — point-and-click. There are other ways too,
and the exam likes knowing the options exist.

Ways to manage Azure:
- **Azure Portal** = the web GUI. Click buttons to create/manage resources. Beginner-friendly.
- **Azure CLI** = type commands (`az ...`) in a terminal. Good for automation/scripting.
- **Azure PowerShell** = like CLI but using PowerShell commands. Same idea, different language.
- **Cloud Shell** = a terminal *built into the portal* (in your browser) that already has CLI and
  PowerShell installed — nothing to set up locally.

### Walk-through: 5 questions

**Q1. Why offer a GUI *and* command-line tools?**
Different jobs. The portal is great for **learning and one-off tasks** (you can *see* everything).
CLI/PowerShell are great for **repeatable automation** — script it once, run it 100 times identically.
Humans like the portal; automation likes the command line.

**Q2. What's Cloud Shell and why is it convenient?**
It's a command-line terminal *inside the browser*, opened from the portal. It comes with the `az` CLI
and PowerShell pre-installed, so you don't install anything on your laptop. It needs a small storage
account to keep your files between sessions.

**Q3. Bash or PowerShell in Cloud Shell — do I have to choose?**
No — you can switch between them with a dropdown. Bash for Linux-style commands, PowerShell for
Windows-style. Same Azure underneath.

**Q4. When creating a resource, does the portal give only one way to start?**
No — there are several entry points (the "+ Create a resource" button, the search bar, the left menu).
The exam won't quiz button locations, but know the portal is the *visual* way to do anything.

**Q5. Big picture: is the portal doing anything the CLI can't?**
Mostly they do the same things — they're different *front doors* to the same Azure. Portal = visual,
CLI/PowerShell = typed commands for automation. Pick based on whether a human or a script is driving.

> **Remember:** Portal = web GUI (visual). CLI / PowerShell = command-line (automation).
> Cloud Shell = browser-based terminal with both pre-installed.

---

## ✅ Module 2 recap

1. Physical map: Datacenter → Availability Zone → Region → Region Pair.
2. Zones protect against building failure; region pairs (≥300 mi) protect against region failure;
   not all regions have zones.
3. Organizing tree: Management Group → Subscription → Resource Group → Resource.
4. Higher-level rules flow down (inheritance). One resource = one resource group.
5. Subscription = billing/access boundary (mandatory). Management Group = optional folder of subscriptions.
6. Manage Azure via Portal (GUI), CLI/PowerShell (commands), or Cloud Shell (in-browser terminal).

Next → `3_Compute_Services.md` (the actual machines that run your code).
