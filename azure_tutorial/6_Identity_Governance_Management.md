# Module 6 — Identity, Governance & Management

> Goal: Understand *who is allowed in* (identity), *what they can do* (access control), how a company
> *enforces rules and stays compliant* (governance), and how it *watches and controls cost* (management).
> This area is ~30% of the exam — high value. Don't skip it.

**The big picture, in four questions a company must answer:**
1. *Who are you?* → **Identity** (Entra ID / Azure AD)
2. *What are you allowed to do?* → **RBAC** (role-based access control)
3. *How do we force everyone to follow the rules?* → **Azure Policy, Locks, Blueprints**
4. *How do we watch health and control spend?* → **Azure Monitor, Cost Management, Advisor**

---

## Topic 6.1 — Microsoft Entra ID (Azure AD): Identity

**First principle:** Before anyone uses your cloud, you must answer "who are you?" You need a central
list of users and a way to verify them. **Microsoft Entra ID** (formerly **Azure Active Directory /
Azure AD**) is Azure's identity service — it manages users, groups, and sign-in.

Two words people confuse (the exam loves this):
- **Authentication (AuthN)** = proving *who you are* (logging in with password, fingerprint, etc.).
  "Are you really you?"
- **Authorization (AuthZ)** = deciding *what you're allowed to do* once you're in. "What can you access?"

Extra security features:
- **MFA (Multi-Factor Authentication)** = require *two+* proofs (password **+** a phone code). Even if
  your password leaks, the attacker still can't get in without your phone.
- **SSO (Single Sign-On)** = log in once, access many apps without re-entering passwords.

### Walk-through: 5 questions

**Q1. What problem does Entra ID / Azure AD solve?**
It's the **central directory of who's allowed in**. Instead of every app having its own user list,
Entra ID holds all users/groups and verifies identities for everything. One place to add, remove, and
manage people.

**Q2. Authentication vs authorization — explain like I'm five.**
**Authentication** = showing your ID at the door to prove you're you. **Authorization** = the wristband
that says which rooms you can enter once inside. First we check *who you are*, then *what you can do*.
AuthN before AuthZ.

**Q3. Why is MFA such a big deal?**
Passwords get stolen, guessed, or leaked constantly. MFA adds a **second proof** (a code on your
phone), so a stolen password alone is useless to an attacker. It's the single most effective way to
stop account takeovers — that's why exams and real security teams push it hard.

**Q4. What does SSO save me?**
Re-typing passwords. With **Single Sign-On**, you log in once and then reach many connected apps
without logging in again. Fewer passwords to manage = more convenience *and* better security (people
stop reusing weak passwords everywhere).

**Q5. Is Entra ID the same as the Windows "Active Directory" my office uses?**
Related but not identical. Traditional Active Directory manages on-premises Windows networks; **Entra
ID (Azure AD)** is the **cloud** identity service for Azure and cloud apps. They can be connected, but
for AZ-900 just know Entra ID = cloud identity & access management.

> **Remember:** Entra ID (Azure AD) = cloud identity service (manages users/sign-in).
> **Authentication** = who you are; **Authorization** = what you can do. **MFA** = two proofs (stops
> stolen-password attacks). **SSO** = log in once, reach many apps.

---

## Topic 6.2 — RBAC (Role-Based Access Control): Authorization done right

**First principle:** Not everyone should be able to do everything. You give people *only the access
their job needs* — the **principle of least privilege**. **RBAC** does this by assigning **roles** to
users/groups, where each role is a bundle of permissions.

Instead of "give Priya these 50 individual permissions," you say "Priya is a *Reader*" or "Sanjay is an
*Owner* of this resource group." The role carries the permissions.

### Walk-through: 5 questions

**Q1. What does "least privilege" mean and why does it matter?**
Give each person the **minimum access needed to do their job — nothing more**. Why? If an account is
compromised or someone makes a mistake, the damage is limited to what that account could touch. Less
access = less risk.

**Q2. How does RBAC make access management easier?**
By grouping permissions into **roles**. You assign a *role* (Reader, Contributor, Owner) instead of
hand-picking dozens of permissions per person. Roles are reusable and consistent — onboard a new dev?
Assign the "Contributor" role and done.

**Q3. Give me the three common built-in roles.**
- **Reader** = can *view* resources but not change them.
- **Contributor** = can *create and manage* resources but **can't** grant access to others.
- **Owner** = full control, *including* granting access to others.
Notice the escalating power: view → manage → manage + control access.

**Q4. How does RBAC connect to the resource hierarchy from Module 2?**
You can assign roles at any level — management group, subscription, resource group, or single resource
— and the access **inherits downward**. Assign "Reader" at a resource group → the user can read every
resource in it. This is why the hierarchy matters: it's also the scope for permissions.

**Q5. Scenario: a contractor needs to *view* dashboards but must not change anything. Which role?**
**Reader** — view-only, no modifications. (Giving them Contributor or Owner would violate least privilege.)

> **Remember:** RBAC = assign **roles** (bundles of permissions) for **least-privilege** access.
> Reader (view) < Contributor (manage) < Owner (manage + grant access). Roles inherit down the hierarchy.

---

## Topic 6.3 — Governance: Azure Policy, Resource Locks, Blueprints

**First principle:** In a big company, you can't trust everyone to remember every rule ("only deploy in
India regions," "always add a cost-center tag," "never delete production"). You need to **automatically
enforce** rules. That's **governance**.

Three tools, three jobs:

- **Azure Policy** = enforces **rules about *how resources are configured*.** It can *audit* or *block*
  resources that break the rules. Example: "deny creating any VM outside the Central India region," or
  "require every resource to have a department tag." Policy = guardrails on configuration.
- **Resource Locks** = prevent **accidental deletion or changes** to important resources.
  - **CanNotDelete** lock = you can read/modify but **not delete**.
  - **ReadOnly** lock = you can read but **not modify or delete**.
- **Azure Blueprints** = a **package/template** of resources, policies, and role assignments you can
  deploy repeatedly to create consistent, compliant environments quickly. Like a "starter kit" for new
  subscriptions that already follows company standards.

(Bonus governance helper: **Tags** = labels like `env=prod` or `team=finance` you attach to resources
for organizing, filtering, and *cost reporting*.)

### Walk-through: 5 questions

**Q1. What's the difference between RBAC and Azure Policy? They both sound like "control."**
**RBAC controls *who can do things* (people & permissions).** **Azure Policy controls *what the things
can be* (resource configuration & rules).** Example: RBAC says "Priya can create VMs." Policy says "but
any VM created must be in India and must have a tag." One governs *people*, the other governs *resources*.

**Q2. Give a real Azure Policy example.**
"Deny any storage account that isn't using encryption," or "only allow VM sizes from this approved
list," or "require a `cost-center` tag on every resource." Policy can **block** non-compliant creation
or **flag** existing violations. It's automated rule enforcement.

**Q3. What's a Resource Lock for, and the two types?**
To stop **accidental damage** to critical resources. **CanNotDelete** = can change but not delete.
**ReadOnly** = can't change *or* delete (view only). You'd lock your production database so a tired
admin can't fat-finger a delete. Note: locks apply to *everyone*, even Owners — they override RBAC for
that protective purpose.

**Q4. What problem do Blueprints solve?**
Consistency at scale. When you create many new environments/subscriptions, you want each to start with
the *same* approved resources, policies, and role assignments. A **Blueprint** packages all that so you
deploy a compliant environment in one shot — no manual setup, no forgotten rules.

**Q5. Scenario: "Make sure nobody can delete the production storage account, even by accident." Which tool?**
A **Resource Lock (CanNotDelete)** on that storage account. (Policy enforces *configuration*; a lock
prevents *deletion* — this is a deletion-protection job, so it's a lock.)

> **Remember:**
> - **RBAC** = who can do what (people). **Azure Policy** = rules on how resources are configured.
> - **Resource Lock** = prevent accidental delete/change (CanNotDelete / ReadOnly), overrides everyone.
> - **Blueprint** = repeatable package of resources + policies + roles for consistent environments.
> - **Tags** = labels for organizing & cost reporting.

---

## Topic 6.4 — Monitoring & Cost Management

**First principle:** Once your cloud is running, you must **watch it** (is it healthy? where are
problems?) and **control spending** (where's the money going? am I wasting it?). Azure gives tools for
both.

**Monitoring:**
- **Azure Monitor** = the central service that **collects metrics and logs** from all your resources so
  you can see performance, set alerts, and troubleshoot. (Under it: *Log Analytics* for querying logs,
  *Application Insights* for app performance.)
- **Azure Service Health** = tells you about **Azure's own** outages/maintenance affecting your services
  ("is the problem me or Microsoft?").
- **Azure Advisor** = a free assistant that gives **recommendations** to improve cost, security,
  reliability, and performance ("you have an idle VM — resize or delete it to save money").

**Cost control:**
- **Microsoft Cost Management** = track, analyze, and **control spending**; set **budgets** and alerts;
  see cost breakdowns (this is where Tags pay off for reporting).
- *(Recall from Module 1: Pricing Calculator = estimate future spend; TCO Calculator = estimate savings
  vs on-prem.)*

**Security (bonus):**
- **Microsoft Defender for Cloud** = monitors your security posture, flags vulnerabilities, and gives a
  secure score with recommendations.

### Walk-through: 5 questions

**Q1. What does Azure Monitor actually do for me?**
It's the **central eyes and ears** of your Azure setup — it gathers performance **metrics** and **logs**
from every resource, lets you build dashboards, and fires **alerts** when something's wrong (CPU spikes,
errors rise). Without it you'd be flying blind.

**Q2. My app is down — how do I tell if it's *my* fault or *Azure's*?**
Check **Azure Service Health**. It reports Azure-side outages and planned maintenance affecting your
region/services. If Service Health shows an incident, it's Microsoft's side; if not, look at your own
resources (via Azure Monitor).

**Q3. What's Azure Advisor and why is it free money?**
Advisor scans your setup and gives **personalized recommendations** across cost, security, reliability,
and performance — e.g. "delete this unused disk," "enable MFA," "resize this oversized VM." Following
them often **cuts your bill** and tightens security with zero effort. It's like a free consultant.

**Q4. How do I stop my Azure bill from surprising me?**
Use **Microsoft Cost Management**: analyze where money goes, and set a **budget** with alerts so you get
warned at, say, 80% of your monthly limit. Combine with **Tags** to see cost *per team/project*. Proactive
control instead of bill-shock.

**Q5. Quick match: which tool for (a) "Azure is having an outage," (b) "recommend savings,"
(c) "set a spending budget," (d) "collect performance logs & alerts"?**
(a) **Service Health**, (b) **Azure Advisor**, (c) **Cost Management**, (d) **Azure Monitor**.

> **Remember:**
> - **Azure Monitor** = collect metrics/logs, alerts (watch *your* resources' health).
> - **Service Health** = status of *Azure itself* (outages/maintenance).
> - **Azure Advisor** = free recommendations (cost, security, reliability, performance).
> - **Cost Management** = track spend + set budgets/alerts.
> - **Defender for Cloud** = security posture & recommendations.

---

## ✅ Module 6 recap

1. **Entra ID (Azure AD)** = cloud identity. **AuthN** = who you are; **AuthZ** = what you can do.
   **MFA** = two proofs; **SSO** = log in once.
2. **RBAC** = role-based, least-privilege access. Reader < Contributor < Owner. Inherits down the hierarchy.
3. **Azure Policy** = enforce configuration rules. **Locks** = prevent accidental delete/change.
   **Blueprints** = repeatable compliant environments. **Tags** = labels for org & cost.
4. **Azure Monitor** = metrics/logs/alerts. **Service Health** = Azure's status. **Advisor** =
   recommendations. **Cost Management** = budgets & spend tracking. **Defender for Cloud** = security.

Next → `7_Exam_Cram_Cheatsheet.md` (your night-before-the-exam recap).
