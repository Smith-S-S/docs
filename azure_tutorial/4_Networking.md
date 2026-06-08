# Module 4 — Networking (How your resources talk)

> Goal: Understand how Azure resources get their own private network, how you control who can reach
> them, and the ways to connect Azure to your office/other networks. Don't panic — for AZ-900 you need
> the *purpose* of each piece, not IP math.

**The big picture:** In the cloud you build a *private network* for your resources (a VNet), slice it
into sections (subnets), put a *firewall* around things (NSG), and then optionally connect that network
to other networks (peering, VPN, ExpressRoute).

---

## Topic 4.1 — Virtual Network (VNet) & Subnets

**First principle:** Your cloud resources need a private space to live and talk to each other safely —
just like an office building has internal wiring separate from the public street. A **Virtual Network
(VNet)** is your own isolated private network inside Azure. **Subnets** divide that network into
smaller sections.

- **VNet** = your private network in Azure. Resources inside it can talk to each other by default.
  You give it an address range (a block of IP addresses).
- **Subnet** = a slice of the VNet's address range. You group resources by subnet — e.g. web servers
  in "subnet A," databases in "subnet B" — to organize and secure them separately.

Analogy: VNet = an office building with its own internal phone system. Subnets = floors/departments
in that building. People on the same building network can call each other; you control who can call
in from outside.

### Walk-through: 5 questions

**Q1. Why do I need a "private network" in the cloud at all?**
So your resources can communicate **privately and securely** without exposing everything to the public
internet. A VNet is a walled-off space where, by default, your VMs and databases can reach each other
but outsiders can't — unless you deliberately open a door.

**Q2. Can resources in the same VNet talk to each other automatically?**
Yes. By default, resources within the same VNet can reach each other. The VNet is the trusted internal
network. (You then *restrict* traffic with NSGs — see next topic.)

**Q3. Why split a VNet into subnets instead of one big network?**
**Organization and security.** You group similar resources and apply different rules per subnet.
Example: put public-facing web servers in one subnet (allow internet on ports 80/443) and databases in
another subnet (block internet entirely, allow only the web subnet). Separation = safer.

**Q4. Do I need to learn IP address math for AZ-900?**
**No.** For this exam, just know: the **VNet has a big address range (the superset)** and each **subnet
takes a smaller slice (a subset)** of it. Azure also reserves a few IPs in every subnet for its own use.
That conceptual relationship is all you need.

**Q5. Real example of subnet design?**
"Web subnet" = your front-end VMs, reachable from the internet on web ports. "DB subnet" = your
database, *not* reachable from the internet — only from the web subnet. Two subnets, two security
postures, one VNet.

> **Remember:** VNet = your private isolated network in Azure (resources inside talk by default).
> Subnet = a slice of the VNet for grouping/securing resources. VNet = superset, subnet = subset. No IP
> math needed for AZ-900.

---

## Topic 4.2 — Network Security Group (NSG) — the cloud firewall

**First principle:** A private network is only safe if you control *who can come in and go out*. An
**NSG (Network Security Group)** is a set of **allow/deny rules** for network traffic — basically a
firewall you attach to a subnet or a VM's network card.

NSG rules filter traffic by:
- **Direction** — inbound (coming in) or outbound (going out)
- **Source / destination** — which IP addresses
- **Port** — which "door number" (e.g. 22 = SSH/remote login, 80 = web, 443 = secure web)
- **Action** — Allow or Deny

### Walk-through: 5 questions

**Q1. What is an NSG in one sentence?**
A virtual firewall — a list of rules that decides which network traffic is **allowed** or **denied**
to/from your resources, based on direction, IP, and port.

**Q2. What's a "port" and why do rules mention port numbers?**
A port is like a numbered door into a machine; each service listens on its own door. Web traffic uses
port **80/443**, remote login (SSH) uses port **22**, Windows remote desktop uses **3389**. NSG rules
say things like "allow port 443 from anyone" (let the public reach the website) but "allow port 22 only
from the office IP" (only admins can remote-login). Controlling ports = controlling which services are
reachable.

**Q3. Give me a concrete NSG rule set for a web server.**
- Allow inbound port **80 and 443** from **anyone** → public can view the website.
- Allow inbound port **22** only from **your office IP range** → only you can remote-manage it.
- Deny everything else inbound → no other doors are open.
That's a secure-by-default posture: open only what's needed.

**Q4. Why is "deny everything else" important?**
Because every open door is a risk. The safe pattern is **default-deny**: block everything, then
explicitly allow only the few things you need. Fewer open ports = smaller attack surface for hackers.

**Q5. Where does an NSG attach?**
To a **subnet** (protects all resources in it) and/or to a **VM's network interface** (protects that one
VM). Attaching at the subnet level lets you secure many resources with one rule set.

> **Remember:** NSG = virtual firewall of allow/deny rules (by direction, IP, port). Open only the
> ports you need (e.g. 443 for web, 22 for SSH from your IP), deny the rest. Attach to subnet or VM NIC.

---

## Topic 4.3 — VNet Peering (connecting two VNets)

**First principle:** Sometimes resources in *different* VNets need to talk — maybe different teams,
regions, or projects. **VNet Peering** connects two VNets so their resources communicate **privately**,
as if on one network, *without* going over the public internet.

### Walk-through: 5 questions

**Q1. Why would I have multiple VNets that need connecting?**
Big organizations separate workloads into different VNets (by team, environment, or region) for
isolation. But sometimes those workloads must talk — e.g. an app in VNet-A needs a service in VNet-B.
Peering links them.

**Q2. How is peered traffic different from just using the internet?**
Peered traffic stays on **Microsoft's private backbone network** — it never touches the public
internet. That makes it **faster and more secure** (no exposure to the open web).

**Q3. Can I peer VNets in different regions?**
Yes. Peering works within the same region *or* across regions (called *global* peering). Resources
behave as if on the same network regardless of region.

**Q4. After peering, how do resources address each other?**
By their **private IP addresses**, as if they were in one VNet. No public IPs needed for them to
communicate.

**Q5. One-line summary for the exam?**
VNet Peering = privately connect two (or more) VNets so resources talk over Microsoft's backbone, not
the public internet.

> **Remember:** VNet Peering = connect VNets together for private, fast, secure communication over
> Microsoft's backbone (not the internet). Works same-region or cross-region.

---

## Topic 4.4 — Connecting Azure to your on-premises network: VPN vs ExpressRoute

**First principle:** Companies still have their own offices/datacenters ("on-premises"). They often
need their on-prem network and their Azure network to act like one connected network. There are two
ways, and the exam tests *which to pick*.

### Option A — VPN Gateway (Site-to-Site VPN)
An **encrypted tunnel over the public internet** connecting your on-prem network to Azure.
- Uses a **VPN Gateway** on the Azure side and your VPN device on-prem.
- Traffic travels over the *public internet* but is **encrypted** (safe but shares the public road).
- **Point-to-Site VPN** = a single user/laptop connects securely to Azure (vs Site-to-Site = whole
  office network connects).

### Option B — ExpressRoute
A **dedicated private connection** from your on-prem to Microsoft — it does **not** use the public
internet at all.
- Much **faster, lower latency, higher bandwidth, more reliable** (up to very high speeds).
- More expensive; needs a connectivity provider.
- Best for mission-critical workloads needing guaranteed performance.

### Walk-through: 5 questions

**Q1. What's the core difference between VPN and ExpressRoute?**
The road they use. **VPN** sends encrypted traffic over the **public internet** (shared road, cheaper).
**ExpressRoute** uses a **dedicated private line** that bypasses the internet entirely (private road,
faster and more reliable, pricier). Same goal (connect on-prem to Azure), different quality of road.

**Q2. If VPN is encrypted, why pay more for ExpressRoute?**
Because encryption ≠ performance. VPN traffic still competes with all the internet's congestion, so
speed/latency vary. ExpressRoute gives a **private, predictable, high-bandwidth** line — essential when
you need consistent performance and reliability (think banks, large data transfers).

**Q3. Site-to-Site vs Point-to-Site VPN?**
**Site-to-Site** connects an entire **network** (your whole office) to Azure via a gateway.
**Point-to-Site** connects a **single device** (one laptop/user) to Azure. Many users from home →
point-to-site; whole office → site-to-site.

**Q4. Scenario: "Connect our office network to Azure securely, but cheaply, over the internet." Which?**
**Site-to-Site VPN** — encrypted tunnel over the public internet, cost-effective.

**Q5. Scenario: "We need a high-bandwidth, low-latency, reliable private link to Azure that avoids the
public internet." Which?**
**ExpressRoute** — dedicated private connection, no public internet, high performance.

> **Remember:**
> - **VPN Gateway** = encrypted tunnel over the *public internet* (cheaper). Site-to-Site = whole
>   network; Point-to-Site = single device.
> - **ExpressRoute** = *dedicated private* connection, bypasses the internet (faster, more reliable,
>   pricier).

---

## 🧭 Networking cheat-sheet

| Need | Use |
|------|-----|
| A private network for my resources | **VNet** |
| Divide the network into sections | **Subnets** |
| Firewall — control who reaches what (ports/IPs) | **NSG** |
| Connect two Azure VNets privately | **VNet Peering** |
| Connect office ↔ Azure over encrypted internet (cheap) | **VPN Gateway (Site-to-Site)** |
| Connect one device ↔ Azure securely | **Point-to-Site VPN** |
| Connect office ↔ Azure on a private high-speed line | **ExpressRoute** |

---

## ✅ Module 4 recap

1. VNet = your private isolated network; Subnet = a slice of it for grouping/securing resources.
2. Resources in the same VNet talk by default; you restrict traffic with NSGs.
3. NSG = firewall of allow/deny rules by direction, IP, and port. Open only what's needed.
4. VNet Peering = privately connect VNets over Microsoft's backbone (not the internet).
5. VPN Gateway = encrypted tunnel over the public internet (Site-to-Site = network, Point-to-Site = device).
6. ExpressRoute = dedicated private line, bypasses the internet, fastest & most reliable.

Next → `5_Storage_and_Databases.md` (where your data lives).
