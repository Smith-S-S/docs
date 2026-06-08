# Module 7 — Exam Cram Cheat-Sheet (Night Before the Exam)

> One-page recall of everything. If you can explain each line in your own words, you're ready.
> The AZ-900 mostly asks *"what is this FOR / who manages it / when do I use it?"* — not *how to configure it*.

---

## 1. Cloud concepts

- **Cloud = renting computing over the internet.** Converts **CapEx** (big upfront purchase) → **OpEx**
  (pay-as-you-go).
- **IaaS / PaaS / SaaS** = differ by *how much you manage*:
  - IaaS (VM) → **you manage the OS** and up.
  - PaaS (App Service, Functions) → you manage **only your app**.
  - SaaS (M365, Gmail) → you manage **nothing**.
  - **OS patching:** IaaS = customer; PaaS/SaaS = Microsoft.
  - **"Lift and shift"** → IaaS.
- **Shared responsibility:** Physical = **always Microsoft**. Data + identities + devices = **always you**.
  OS is the swing layer (yours in IaaS).
- **Cloud types:** Public = shared, cheap, no CapEx (bus). Private = dedicated, high control, high CapEx
  (own car). Hybrid = both.
- **Benefits vocabulary:**
  - **High Availability** = minimal downtime. **Fault Tolerance** = survives failures with no interruption.
  - **Scalability** = can grow/shrink. **Elasticity** = grows/shrinks **automatically**.
  - **Scale UP** = bigger machine (vertical). **Scale OUT** = more machines (horizontal, preferred).
- **Cost tools:** Pricing Calculator = estimate Azure spend. TCO Calculator = estimate savings vs on-prem.

## 2. Architecture

- **Physical map (small→big):** Datacenter → **Availability Zone** → **Region** → **Region Pair**.
  - Zones = separate buildings in a region (survive a building failure). **Not all regions have zones.**
  - Region pairs = ≥**300 miles** apart (survive a regional disaster).
- **Org hierarchy (top→down):** **Management Group → Subscription → Resource Group → Resource.**
  - Rules/permissions set high up **inherit downward.**
  - Subscription = billing/access boundary (**mandatory**). Management Group = optional.
  - One resource lives in exactly **one** resource group.
- **Manage Azure via:** Portal (GUI), CLI/PowerShell (commands), Cloud Shell (in-browser terminal).

## 3. Compute — "where my code runs" (control → convenience)

| Service | What | When |
|---|---|---|
| **Virtual Machine** | full computer, IaaS, you patch | full control, lift-and-shift |
| **VM Scale Set** | auto-scaling group of identical VMs | elasticity for VMs |
| **Availability Set** | spread VMs across **fault domains** (racks) + **update domains** (reboot groups) | reliability for fixed VMs |
| **Azure Virtual Desktop** | managed cloud desktops, many users | remote work |
| **Container / ACI** | lightweight app bundle, shares host OS | portable, fast, no servers |
| **Azure Functions** | serverless, **event-driven**, pay-per-run | short tasks reacting to events |
| **App Service** | PaaS web/API hosting, auto-scale | always-on websites/APIs |

- Hypervisor → runs VMs. Container engine → runs containers.
- **Deployment slots** (App Service) = swap staging ↔ production = **zero-downtime** releases.

## 4. Networking

- **VNet** = your private network (resources inside talk by default). **Subnet** = a slice of it.
  (VNet = superset, subnet = subset. No IP math needed.)
- **NSG** = virtual **firewall**: allow/deny by direction, IP, **port** (80/443 web, 22 SSH, 3389 RDP).
  Open only what's needed.
- **VNet Peering** = connect VNets privately over Microsoft's backbone (not the internet).
- **VPN Gateway** = encrypted tunnel over the **public internet** (cheaper).
  - Site-to-Site = whole network. Point-to-Site = single device.
- **ExpressRoute** = **dedicated private** line, bypasses the internet (fastest, most reliable, pricier).

## 5. Storage & databases

- **Storage Account** holds storage; **Blob Storage** = files/unstructured data (images, video, backups).
- **Redundancy (copies, how far apart):**
  - **LRS** = 3 copies, 1 datacenter (cheapest).
  - **ZRS** = across 3 zones in a region.
  - **GRS** = across 2 regions (survives a **regional** disaster, safest).
- **Access tiers (match cost to usage):**
  - **Hot** = frequent access (priciest storage, instant).
  - **Cool** = infrequent.
  - **Archive** = rare; cheapest storage but **slow + costly to retrieve** (hours).
- **Databases:** Azure **SQL** = managed **relational** (structured tables). **Cosmos DB** = managed
  **NoSQL**, globally distributed, ultra-low latency.

## 6. Identity, governance & management

- **Entra ID (Azure AD)** = cloud identity service.
  - **Authentication** = who you are. **Authorization** = what you can do.
  - **MFA** = two+ proofs (stops stolen passwords). **SSO** = log in once, many apps.
- **RBAC** = role-based, **least-privilege** access: **Reader** (view) < **Contributor** (manage) <
  **Owner** (manage + grant access). Inherits down the hierarchy.
- **Governance:**
  - **Azure Policy** = enforce **configuration rules** (e.g. only allow certain regions/sizes, require tags).
  - **Resource Lock** = prevent accidental change/delete — **CanNotDelete** / **ReadOnly** (overrides everyone).
  - **Blueprint** = repeatable package of resources + policies + roles = consistent compliant environments.
  - **Tags** = labels for organizing & **cost reporting**.
  - *RBAC governs **people**; Policy governs **resources**.*
- **Management/monitoring:**
  - **Azure Monitor** = collect metrics/logs, alerts (watch *your* resources).
  - **Service Health** = status of **Azure itself** (outages/maintenance).
  - **Azure Advisor** = free recommendations (cost, security, reliability, performance).
  - **Cost Management** = track spend + set **budgets**.
  - **Microsoft Defender for Cloud** = security posture & recommendations.

---

## 🎯 Top 12 "trap" distinctions the exam loves

1. **IaaS vs PaaS vs SaaS** → who manages the OS? (IaaS = you; PaaS/SaaS = Microsoft)
2. **CapEx vs OpEx** → buying upfront vs pay-as-you-go.
3. **Scalability vs Elasticity** → can scale vs scales *automatically*.
4. **Scale up vs out** → bigger machine vs more machines.
5. **High availability vs Fault tolerance** → minimal downtime vs zero interruption on failure.
6. **Zone vs Region pair** → building-level vs region-level disaster protection.
7. **Availability Set vs Scale Set** → reliability for fixed VMs vs auto-scaling.
8. **VM vs Container vs Function vs App Service** → control vs convenience; always-on vs event-driven.
9. **VPN vs ExpressRoute** → encrypted internet vs dedicated private line.
10. **LRS vs ZRS vs GRS** → 1 datacenter vs zones vs regions.
11. **Hot vs Cool vs Archive** → access frequency vs storage cost vs retrieval speed.
12. **RBAC vs Policy vs Lock** → who can act / how resources are configured / prevent delete.

---

## ✅ Final readiness check

Can you, *without looking*, answer these in one sentence each?
- [ ] What does "convert CapEx to OpEx" mean?
- [ ] In IaaS, who patches the OS?
- [ ] What's the difference between elasticity and scalability?
- [ ] What protects against a *region-wide* disaster (zones or region pairs)?
- [ ] Order the hierarchy: resource, subscription, management group, resource group.
- [ ] When do you use Azure Functions vs App Service?
- [ ] VPN vs ExpressRoute — which avoids the public internet?
- [ ] LRS vs GRS — which survives a region disaster?
- [ ] Hot vs Archive — which is cheapest to store but slow to read?
- [ ] RBAC vs Azure Policy — which governs people, which governs resource config?

If you got 8+/10, you're ready. Good luck — you've got this. 🍀
