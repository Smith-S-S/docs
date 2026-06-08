# Module 3 — Compute Services (The things that run your code)

> Goal: Understand the menu of "ways to run your application" in Azure and — most importantly —
> *when to pick which one.* The exam constantly asks "which compute service fits this scenario?"

**The big mental model:** all these services answer the same question — *"where does my code run?"* —
but trade **control** for **convenience**. More control = more work (VMs). More convenience = less
control (Functions). Here's the spectrum:

```
MORE CONTROL  ◄─────────────────────────────────────►  MORE CONVENIENCE
 Virtual        Containers      App Service     Azure
 Machines       (ACI)           (Web Apps)      Functions
 (IaaS)         (PaaS-ish)      (PaaS)          (Serverless)
```

---

## Topic 3.1 — Virtual Machines (VMs)

**First principle:** A VM is a *software-simulated computer* running on Azure's physical hardware. You
get a whole machine — OS and all — that you fully control, without owning the hardware. This is **IaaS**.

How it works: physical server → **hypervisor** (software that splits one physical machine into many
virtual ones) → multiple **guest VMs** for different customers, each isolated. You request a VM, pick
its size/OS, and it's yours.

### Walk-through: 5 questions

**Q1. What exactly am I renting with a VM?**
A complete, isolated computer: CPU, RAM, disk, an operating system (Windows or Linux) you choose, and
full admin access. You can install *anything*. It behaves like a physical PC, but it's running as
software on Azure's shared hardware.

**Q2. What's a "hypervisor" and why do I care?**
It's the software that lets one physical server run *many* separate VMs at once, each walled off from
the others. It's *why* cloud is cheap: Microsoft buys one big machine and rents slices to many
customers safely. You don't manage the hypervisor — Microsoft does — but knowing the word explains
how virtualization works.

**Q3. VMs are IaaS — so what's my responsibility?**
Everything from the OS up: patching the OS, installing/updating software, configuring security,
backups. Microsoft only handles the physical hardware below. Maximum control, maximum chores.

**Q4. When is a VM the *right* choice?**
When you need **full control** or are doing a **"lift and shift"** — moving an existing on-premises app
to the cloud *unchanged*. The app expects a normal OS/server, so you give it one. Also good for
legacy software or special OS requirements.

**Q5. Downside of VMs vs the fancier options?**
You manage everything (time + effort), and a VM bills you while it's running even if it's idle.
The newer services (App Service, Functions) remove that maintenance burden — but also remove control.

> **Remember:** VM = a full computer you rent and fully control (IaaS). Best for full control and
> lift-and-shift. You patch and maintain it.

---

## Topic 3.2 — VM Scale Sets & Availability Sets (keeping VMs reliable)

**First principle:** One VM = one point of failure. If it dies, your app dies. To make VM-based apps
**highly available** and **elastic**, Azure adds two tools.

- **VM Scale Set (VMSS)** = a group of identical VMs that **auto-grows and shrinks** with demand.
  Define a template; the scale set keeps the right number of VMs running. CPU too high? It adds VMs
  (**scale out**). Load drops? It removes them (**scale in**). This is *elasticity* for VMs.
  (AWS calls this an "Auto Scaling Group"; Azure calls it a VM Scale Set.)
- **Availability Set** = spreads your VMs across different physical hardware *within one datacenter* so
  a single hardware failure or maintenance reboot doesn't take all of them down. It uses:
  - **Fault domains** = different physical server racks (separate power/network). Protects against a
    *rack* failing.
  - **Update domains** = groups that reboot at *different times* during maintenance. Protects against
    *all VMs rebooting at once* during updates.

### Walk-through: 5 questions

**Q1. What problem does a VM Scale Set solve that a single VM can't?**
Two: (1) **single point of failure** — if your one VM crashes, the app is down. A scale set keeps
multiple VMs and replaces dead ones. (2) **changing load** — it auto-adds/removes VMs so you're never
over- or under-provisioned. It delivers HA *and* elasticity for VMs.

**Q2. Walk me through auto-scaling in a scale set.**
You set rules like "if average CPU > 75% for 10 minutes, add a VM" (**scale out**) and "if CPU < 25%,
remove a VM" (**scale in**), within a min/max count. The 10-minute window prevents overreacting to a
brief spike. So the fleet breathes with demand, and you pay only for what's running.

**Q3. What's the difference between an Availability Set and a Scale Set?**
**Availability Set** = reliability for a *fixed* set of VMs (spreads them across hardware so failures/
maintenance don't kill them all). **Scale Set** = a *variable* set of VMs that auto-scales with load.
One is about *surviving failures*; the other adds *auto-scaling* on top.

**Q4. Fault domain vs update domain — what's the difference?**
**Fault domain** = separate physical racks (own power/network) → protects against *hardware* failure.
**Update domain** = groups rebooted at different times during Azure maintenance → protects against
*all your VMs going down together during planned updates*. Availability sets spread VMs across both.

**Q5. Scenario: an app needs to handle unpredictable traffic spikes automatically. Which tool?**
**VM Scale Set** — it auto-scales out and in. (An Availability Set alone wouldn't *scale*; it only
spreads a fixed number of VMs for reliability.)

> **Remember:** Scale Set = auto-scaling group of VMs (elasticity + HA). Availability Set = spreads a
> fixed set of VMs across fault domains (racks) and update domains (reboot groups) for reliability.

---

## Topic 3.3 — Azure Virtual Desktop (bonus, light touch)

**First principle:** Sometimes you don't want to run an *app* in the cloud — you want a whole **desktop**
in the cloud that people log into from anywhere. That's Azure Virtual Desktop (AVD).

It's a *managed* virtual desktop: employees access a Windows desktop and apps from any device (laptop,
tablet, phone), from anywhere. Microsoft manages the underlying maintenance. Supports many users on
the same host (multi-session Windows 10/11), secured with multi-factor auth and role-based access.

### Walk-through: 5 questions

**Q1. How is Azure Virtual Desktop different from a normal VM?**
A VM is a *server/computer* you manage and usually one person/app uses. AVD is a *managed desktop
experience* designed for **many users** to log into Windows desktops remotely. AVD is managed by
Microsoft; a VM's OS is managed by you.

**Q2. Why would a company want desktops in the cloud?**
Remote work and security. Staff (or contractors) get a consistent, secure company desktop from any
device, anywhere — and the data stays in Azure, not on personal laptops. Easy to provision and revoke.

**Q3. Can multiple people share one AVD host?**
Yes — multi-session Windows 10/11 lets several users use the same host at once, which saves cost. A
normal VM is typically one user at a time.

**Q4. Who handles patching/maintenance for AVD?**
Microsoft — it's a *managed* service. With a plain VM, that's your job.

**Q5. How is it billed vs a VM?**
AVD is generally **per user per month** (flexible for changing headcount); VMs are **pay-as-you-go**
for the running machine.

> **Remember:** AVD = managed cloud desktops for many remote users (Microsoft-managed, often per-user
> billing). VM = a server/computer you manage yourself.

---

## Topic 3.4 — Containers & Azure Container Instances (ACI)

**First principle:** VMs are heavy — each one carries a *full operating system*. If you just want to
run an app reliably anywhere, that's wasteful. **Containers** package only your app + its dependencies
(libraries, config) and **share the host's OS**, making them lightweight, fast, and portable.

VM vs Container under the hood:
- VM: hardware → host OS → **hypervisor** → each VM has its *own full OS* → app. (Heavy.)
- Container: hardware → host OS → **container engine** → containers share the *one* host OS → app. (Light.)

**Azure Container Instances (ACI)** = the easiest way to run a container in Azure. You give it a
container image; Azure runs it — no servers to manage. (Images come from a registry like Azure
Container Registry or Docker Hub.)

### Walk-through: 5 questions

**Q1. What's actually inside a "container"?**
Your application plus everything it needs to run — code, libraries, binaries, config — bundled into
one package (an "image"). Because the bundle is self-contained, it runs the *same* on your laptop, a
test server, or Azure. "Works on my machine" stops being an excuse.

**Q2. Why is a container lighter than a VM?**
A VM hauls around a *full operating system* (gigabytes, slow to boot). Containers **share the host's
OS** via a container engine, so each one carries only the app. Result: containers start in seconds and
you can pack far more of them onto one machine.

**Q3. What's a "container engine" — is it like a hypervisor?**
Same idea, different layer. A hypervisor runs multiple *VMs* (each with its own OS) on one machine. A
container engine runs multiple *containers* (sharing one OS) on one machine. Engine : containers ::
hypervisor : VMs.

**Q4. What does Azure Container Instances give me?**
A way to run a container *without managing any servers*. You point ACI at your container image and it
runs it for you (PaaS — Microsoft handles the infrastructure, patching, etc.). Great for quick jobs,
simple apps, and tasks that don't need a full orchestration system.

**Q5. When pick a container over a VM?**
When you want **lightweight, portable, fast-starting** app packaging and you don't need to control a
whole OS. Containers are ideal for microservices and consistent deployments. Pick a VM when you need
the full OS/control; pick a container when you just need the app to run reliably anywhere.

> **Remember:** Container = lightweight bundle of app + dependencies sharing the host OS (portable,
> fast). ACI = run a container in Azure with no servers to manage. Hypervisor→VMs, container engine→containers.

---

## Topic 3.5 — Azure Functions (Serverless)

**First principle:** Sometimes you don't need a machine running 24/7 — you just need a small piece of
code to run *when something happens*, then stop. Why pay for an idle server? **Azure Functions** runs
your code only when *triggered*, and you pay only for those runs. This is **serverless**.

Two key words:
- **Event-driven** = the code runs in response to an *event/trigger*: an HTTP request, a timer (like a
  cron schedule), a file uploaded to storage, a message in a queue, etc.
- **Serverless** = *you* don't manage any server. (There IS a server — but it's invisible to you;
  Azure spins it up only when needed.) You write a function; Azure handles the rest.

### Walk-through: 5 questions

**Q1. "Serverless" — so there's literally no server?**
There *is* a server, but you never see or manage it. Azure provisions it on demand, runs your code,
then tears it down. From your point of view there's "no server to babysit" — hence the name. You only
care about your code.

**Q2. What does "event-driven" mean with a concrete example?**
Your code sits idle until an *event* wakes it. Example: a user uploads a photo to Blob Storage → that
upload is an event → it triggers your function → the function makes a thumbnail and emails the user.
No upload, no run. Triggers can also be HTTP calls, timers, or queue messages.

**Q3. Why is this cheaper than a VM for some tasks?**
A VM bills you 24/7 even when idle. A Function bills you **only for the executions** — number of runs ×
how long they take. If your task happens 50 times a day for half a second each, you pay almost nothing.
No idle cost.

**Q4. So should I run *everything* as a Function?**
No. Functions shine for **short, event-triggered tasks** that don't need to run continuously. For an
app that must always be on and serving traffic (a busy website), App Service or VMs fit better.
Functions are for "do this quick thing when X happens."

**Q5. Scenario: "Resize an image automatically whenever one is uploaded to storage." Best service?**
**Azure Functions** — it's event-driven (the upload is the trigger), short-lived, and you pay only per
execution. Perfect serverless use case.

> **Remember:** Azure Functions = serverless, event-driven code. Runs only when triggered, bills only
> per execution. Best for short tasks reacting to events — not always-on apps.

---

## Topic 3.6 — Azure App Service (Web Apps)

**First principle:** Most people just want to **host a website or API** without dealing with servers,
OS patching, or scaling plumbing. **Azure App Service** is the PaaS that does exactly that: you bring
your code, it runs and hosts your web app. Microsoft manages everything underneath.

App Service hosts four flavors:
- **Web Apps** = host websites/web apps (.NET, Java, Node.js, Python, PHP, Ruby).
- **API Apps** = host REST APIs.
- **WebJobs** = run background tasks/scripts (like scheduled jobs).
- **Mobile Apps** = backends for iOS/Android apps (data, auth, push notifications).

It supports **auto-scaling**, **high availability**, Windows *and* Linux, and **deployment slots**
(staging environments you can swap into production with zero downtime).

### Walk-through: 5 questions

**Q1. App Service vs a VM for hosting a website — why bother with App Service?**
With a VM you must install the web server, patch the OS, configure scaling, secure it — all yourself.
With App Service you just **upload your code** and it runs, with patching/scaling/HA handled by Azure
(it's PaaS). Far less work for the common case of "host my web app."

**Q2. App Service vs Azure Functions — both are PaaS-ish, what's the difference?**
App Service hosts **always-on** apps (a website that must respond any time someone visits). Functions
run **short, event-triggered** bursts and idle the rest of the time. Continuous web app → App Service.
Occasional event-driven task → Functions.

**Q3. What's a "deployment slot" and why is it clever?**
A slot is a parallel copy of your app (e.g. a "staging" slot). You deploy the new version to staging,
test it, then **swap** staging ↔ production instantly. Users hit the new version with **no downtime**,
and if something's wrong you can swap back. It's a safe release switch. *(Available on Standard/Premium
plans, not the free/basic tiers.)*

**Q4. Do I have to choose Windows or Linux, and which languages work?**
App Service runs on **both** Windows and Linux, and supports common languages (.NET, Java, Node.js,
Python, PHP, Ruby). You focus on the app; Azure handles the OS underneath.

**Q5. Scenario: "Host a Python web app, auto-scale it, no server management." Best service?**
**Azure App Service** — PaaS web hosting with auto-scaling and zero OS management. Exactly its job.

> **Remember:** App Service = PaaS for hosting web apps / APIs / background jobs / mobile backends.
> Always-on apps, auto-scaling, no OS management. Deployment slots = zero-downtime releases (swap
> staging ↔ production).

---

## 🧭 Compute decision cheat-sheet (memorize this table)

| If you need... | Use | Why |
|----------------|-----|-----|
| Full control of the OS / lift-and-shift a legacy app | **Virtual Machine** | IaaS, you control everything |
| Auto-scaling group of identical VMs | **VM Scale Set** | Elasticity for VMs |
| Reliability for a fixed set of VMs | **Availability Set** | Spreads across fault/update domains |
| Cloud desktops for remote workers | **Azure Virtual Desktop** | Managed multi-user desktops |
| Lightweight, portable app packaging | **Container / ACI** | Shares host OS, fast & portable |
| Run code only when an event happens | **Azure Functions** | Serverless, pay-per-execution |
| Host an always-on website / API, no server mgmt | **App Service** | PaaS web hosting, auto-scale |

---

## ✅ Module 3 recap

1. Compute = "where your code runs," trading control for convenience: VM → Container → App Service → Function.
2. VM = full computer, full control (IaaS), you patch it. Best for lift-and-shift.
3. VM Scale Set = auto-scaling VMs (elasticity). Availability Set = spread VMs across fault/update domains.
4. Container/ACI = lightweight app bundle sharing host OS; run with no server management.
5. Azure Functions = serverless, event-driven, pay-per-run. Best for short event-triggered tasks.
6. App Service = PaaS hosting for always-on web apps/APIs; auto-scaling; deployment slots = zero-downtime releases.

Next → `4_Networking.md` (how your resources talk to each other and the outside world).
