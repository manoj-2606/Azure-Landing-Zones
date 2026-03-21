# ALZ Day 4 -- Concepts, Q&A, and Correct Answers

> This file contains every concept covered in Day 4 and the full Q&A session.  
> Use this as a reference before any client conversation involving Azure identity and access.  
> Day 4 score: 59/100 -- review Q4 and Q7 before moving to Day 5.  
> Re-read Section 4 and Section 5 of the Day 4 Goal file before proceeding.

---

## Core Concepts Covered in Day 4

---

### Concept 1 -- Two Identity Systems in Azure

**The most important distinction in Day 4.**  
These two systems look similar. They are completely separate.

```
┌──────────────────────────────────────────────────┐
│             MICROSOFT ENTRA ID ROLES             │
│                                                  │
│  Controls access to the Entra ID directory       │
│  Examples:                                       │
│  - Who can create users                          │
│  - Who can manage groups                         │
│  - Who can register applications                 │
│  - Who can configure Conditional Access          │
│                                                  │
│  Key roles: Global Administrator,                │
│  User Administrator, Application Administrator  │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│               AZURE RBAC ROLES                   │
│                                                  │
│  Controls access to Azure resources              │
│  Examples:                                       │
│  - Who can create a VM                           │
│  - Who can read a storage account                │
│  - Who can deploy to a subscription              │
│  - Who can assign roles to others                │
│                                                  │
│  Key roles: Owner, Contributor, Reader,          │
│  User Access Administrator                       │
└──────────────────────────────────────────────────┘
```

**The one sentence that must be permanent in your head:**  
Entra ID roles control the identity directory.  
Azure RBAC roles control Azure resources.  
A Global Administrator in Entra ID has ZERO automatic access to Azure resources.

---

### Concept 2 -- Azure RBAC

**Every RBAC assignment has three parts:**

```
WHO (Principal) + WHAT ROLE (Permissions) + WHERE (Scope)

Example:
App Team A Group  +  Contributor  +  Landing Zone A1 Subscription
```

**The four built-in roles to memorise:**

| Role | Can Do | Cannot Do | Used For |
|---|---|---|---|
| Owner | Everything including assigning roles | Nothing -- full control | Platform team via PIM only |
| Contributor | Create and manage all resources | Assign roles to others | App teams on their subscription |
| Reader | View everything | Change anything | Auditors, security team visibility |
| User Access Administrator | Assign roles only | Create or manage resources | Specific delegation only |

**The RBAC scope cascade:**
```
Management Group  <-- Assign here, cascades to everything below
      └── Subscription
              └── Resource Group
                      └── Individual Resource
```

**Why Contributor and not Owner for app teams:**  
The only practical difference is that Owner can assign roles to others.  
If an app team had Owner, they could grant any external person access  
to their subscription without the platform team knowing.  
The governance model breaks silently.  
Contributor lets them build everything they need.  
Role assignment authority stays with the platform team.

---

### Concept 3 -- Privileged Identity Management (PIM)

**The problem PIM solves:**  
Without PIM, privileged roles are permanent.  
Permanent privileged access is a major attack vector.  
If compromised, an attacker has unrestricted access immediately and indefinitely.

**How PIM works:**
```
Person has NO privileged role by default (Eligible state)
        |
        v
Requests activation with justification
        |
        v
Approval required from designated approver
        |
        v
Role activates for a limited time window (e.g. 4 hours)
        |
        v
Every action during this window is logged
        |
        v
Role expires automatically (Expired state)
        |
        v
Back to Eligible -- must request again if needed
```

**The three PIM states:**

| State | Plain English Meaning |
|---|---|
| **Eligible** | On the approved list to request the role. Does not have it yet. Cannot do anything privileged. |
| **Active** | Request was approved. Role is live right now. Timer is running. |
| **Expired** | Time window ended. Role automatically removed. Back to Eligible. Must request again. |

**Critical point:**  
Expired does not mean automatic re-activation.  
The person must submit a new request with a new justification through approval again.  
The cycle never repeats automatically -- that would defeat the purpose of PIM.

**PIM does NOT work on a schedule.**  
You cannot set "platform team can activate between 9am and 5pm."  
Every activation is on-demand with individual approval and justification.

---

### Concept 4 -- Managed Identities

**The problem they solve:**  
Applications need to access Azure resources.  
The old way: create credentials (username/password or secret), store them, rotate them manually, hope nobody leaks them.  
Managed Identity eliminates credentials entirely. Azure manages authentication automatically.

**Two types:**

**System-Assigned Managed Identity**
```
- Created alongside one specific Azure resource
- Lives and dies with that resource
- One-to-one: one identity per resource
- When the resource is deleted, the identity is deleted
- Use when: one resource has one unique access need
- Example: A single VM needs to read from one Key Vault
```

**User-Assigned Managed Identity**
```
- Created as a standalone Azure resource
- Can be attached to multiple Azure resources simultaneously
- Survives if one of the attached resources is deleted
- Use when: multiple resources share the same access needs
- Use when: resource is frequently recreated (identity persists)
- Example: 5 VMs in the same app tier all need the same Key Vault access
```

**When to use each:**

| Scenario | Use |
|---|---|
| Single VM needs Key Vault access | System-Assigned |
| Multiple VMs share the same access | User-Assigned |
| DINE policy performing remediation | System-Assigned (auto-created on policy assignment) |
| Ephemeral resource frequently recreated | User-Assigned |
| Self-hosted Azure DevOps agent (Azure VM) | System-Assigned on the VM |

---

### Concept 5 -- Service Principal vs Managed Identity

**Both are non-human identities. They are not the same.**

| | Service Principal | Managed Identity |
|---|---|---|
| Credentials | Client ID + Secret or Certificate | No credentials -- Azure manages it |
| Secret rotation | Manual -- your responsibility | Automatic -- Azure's responsibility |
| Secret leak risk | Yes -- secrets can be exposed | No -- nothing to leak |
| Works outside Azure | Yes | No -- Azure resources only |
| Used for | Third-party apps, external pipelines | Azure VMs, Functions, Azure resources |

**The rule of thumb:**  
If the consuming system IS an Azure resource -- use Managed Identity.  
If the consuming system is OUTSIDE Azure -- use Service Principal.

**Azure DevOps pipeline specifically:**  
An Azure DevOps pipeline running on a Microsoft-hosted agent is NOT an Azure resource.  
It runs outside Azure on DevOps infrastructure.  
It must use a **Service Principal** to authenticate to Azure.

Exception: If the pipeline runs on a **self-hosted agent that is an Azure VM**,  
that VM can have a Managed Identity and the pipeline inherits it.

---

### Concept 6 -- RBAC Structure Across the Full ALZ Estate

```
Tenant Root Group
│   Owner: Platform Team (via PIM -- never permanent)
│
└── Contoso Management Group
    │   Owner: Platform Team (via PIM)
    │   Policy Contributor: Platform DevOps Service Principal
    │
    ├── Platform Management Group
    │   │   Owner: Platform Team (via PIM)
    │   │
    │   ├── Connectivity Subscription
    │   │       Network Contributor: Network Team
    │   │       Reader: Security Team
    │   │
    │   ├── Management Subscription
    │   │       Monitoring Contributor: Operations Team
    │   │
    │   └── Security Subscription
    │           Security Admin: Security Team
    │
    └── Landing Zones Management Group
            Reader: Platform Team (visibility without write)
            │
            ├── Landing Zone A1 Subscription
            │       Contributor: App Team A (their subscription only)
            │       Reader: Security Team
            │
            └── Landing Zone A2 Subscription
                    Contributor: App Team B (their subscription only)
                    Reader: Security Team
```

**For every row in this diagram ask yourself:**  
Why THIS role and not Owner?  
Why at THIS scope and not lower?  
If you can answer both questions for every row, you understand ALZ identity.

---

### Concept 7 -- DINE Policy Silent Failure (Critical Real-World Scenario)

**The scenario:**  
DINE policy assigned. Compliance dashboard shows non-compliant resources.  
Remediation is not happening. No obvious error message.

**The two most common causes:**

**Cause 1 -- Managed Identity Has No RBAC Role**
```
DINE policy assignment created
        |
        v
Azure auto-creates System-Assigned Managed Identity for the policy
        |
        v
Nobody assigns an RBAC role to that identity
        |
        v
Identity tries to deploy remediation resource
        |
        v
Azure checks permissions -- finds nothing
        |
        v
Deployment fails silently
VM stays non-compliant forever
No obvious error thrown
```

Fix: Find the Managed Identity on the policy assignment.  
Assign it Contributor or the specific role it needs (e.g. Log Analytics Contributor).  
Then manually trigger a remediation task from the Policy compliance blade.

**Cause 2 -- No Remediation Task for Pre-Existing Resources**  
DINE fires automatically for NEW resources created after policy assignment.  
Resources that existed BEFORE the policy was assigned are not automatically remediated.  
You must manually create a remediation task for those existing resources.

**The fix for both causes:**
1. Assign correct RBAC role to the policy assignment's Managed Identity
2. Go to Policy -- Remediation -- Create remediation task for existing non-compliant resources
3. New resources going forward will be auto-remediated

---

### Concept 8 -- Principle of Least Privilege

This phrase must be in your vocabulary for client conversations.

**Definition:**  
Give an identity the minimum permissions it needs to do its job and nothing more.

**Examples of violating it:**
- Assigning Contributor to a VM that only needs to read Key Vault secrets
- Assigning Owner to a pipeline that only needs to deploy to one subscription
- Assigning Reader at the Tenant Root Group when only one subscription needs visibility

**Examples of following it:**
- Assigning Key Vault Secrets User to a VM's Managed Identity scoped to one Key Vault
- Assigning Contributor to a pipeline Service Principal scoped to one subscription only
- Assigning Network Contributor to the network team scoped to the Connectivity subscription only

In ALZ every role assignment should be the smallest role at the smallest scope  
that still allows the work to be done.

---

## Day 4 Q&A Session -- Questions and Correct Answers

---

### Q1: A Global Administrator in Entra ID -- do they automatically have full access to Azure subscriptions and resources?

**Your answer:** Initially wrong -- corrected after prompt.  
**Score: 60%**

**Correct Answer:**  
No. Global Administrator is an Entra ID role that controls the identity directory --  
creating users, managing groups, configuring Conditional Access, registering applications.  
It has zero automatic permissions on Azure resources or subscriptions.  
To manage Azure resources, a separate Azure RBAC role must be explicitly assigned  
at the appropriate scope -- Owner, Contributor, or a specific role.  
A Global Admin CAN elevate themselves using the "Access management for Azure resources"  
toggle in Entra ID -- but this only grants User Access Administrator in Azure RBAC,  
not Owner, and it is a deliberate action not automatic.

**The trap:** These are two completely separate systems. Never use them interchangeably.

---

### Q2: The app team wants Owner on their landing zone subscription. What do you give them and why?

**Your answer:** Contributor only, platform team gets Owner via PIM.  
**Score: 90%**

**Correct Answer:**  
App teams receive Contributor on their own subscription only -- not Owner.  
The reason is specific: the only practical difference between Owner and Contributor  
is that Owner can assign roles to others. Contributor cannot.  
If an app team had Owner, they could grant any external person access  
to their subscription without the platform team's knowledge,  
bypassing the governance model silently.  
Contributor lets them build and manage everything they need  
while keeping role assignment authority with the platform team above.  
The platform team holds Owner at the Management Group level via PIM -- never permanently.

---

### Q3: How do we prevent the platform team from making unauthorised changes at 2am without anyone knowing?

**Your answer:** PIM scheduling and Log Analytics -- PIM scheduling was wrong.  
**Score: 70%**

**Correct Answer:**  
PIM does not work on a schedule. It works on demand with approval.  
No platform team member holds a privileged role permanently.  
They are only Eligible for it via PIM.  
When elevated access is needed, they must:
- Submit a request with a written justification
- Have it approved by a designated approver
- The role activates for a limited time window only (e.g. 4 hours maximum)
- Every action during that window is logged

All PIM activation logs flow to the central Log Analytics Workspace  
in the Management subscription.  
The compliance team can pull a full audit report at any time:  
who activated, when, for how long, what justification they provided.  
This satisfies compliance frameworks that require just-in-time access and audit trails.

---

### Q4: An Azure DevOps pipeline needs to deploy resources into a landing zone. What identity should it use?

**Your answer:** System-Assigned Managed Identity -- wrong.  
**Score: 40%**

**Correct Answer:**  
A Service Principal -- not a Managed Identity.

An Azure DevOps pipeline running on a Microsoft-hosted agent is not an Azure resource.  
It runs outside Azure on DevOps infrastructure.  
Managed Identities can only be attached to Azure resources -- not external pipelines.

The pipeline uses a Service Principal registered in Entra ID,  
assigned Contributor on the target subscription,  
authenticating via a client secret or certificate stored in DevOps pipeline variables.

```
Azure DevOps pipeline (outside Azure)
        |
        | authenticates using Service Principal credentials
        |
        v
Azure -- validates Service Principal identity
        |
        v
Pipeline deploys resources using Contributor permissions
```

**Exception:** If the pipeline runs on a self-hosted agent that IS an Azure VM,  
that VM can have a System-Assigned or User-Assigned Managed Identity  
and the pipeline inherits the VM's identity for authentication.  
No credentials needed in that scenario.

**What System-Assigned MI is actually for:**  
An Azure resource (VM, Function App) that needs to access other Azure resources.  
It is not for pipelines running outside Azure.

---

### Q5: A VM needs to read secrets from Azure Key Vault. What identity and what role?

**Your answer:** System-Assigned MI correct. Security Reader role -- wrong.  
**Score: 65%**

**Correct Answer:**  
Identity: **System-Assigned Managed Identity** -- correct.  
One VM with one specific access need is exactly the one-to-one scenario  
for System-Assigned. Azure manages the credentials automatically.  
No secrets to store or rotate.

Role: **Key Vault Secrets User** -- not Security Reader.  
Security Reader is an Entra ID role for Defender and security posture visibility.  
It has nothing to do with Key Vault secrets.

**Key Vault roles:**

| Need | Correct Role |
|---|---|
| Read secrets only | Key Vault Secrets User |
| Read and write secrets | Key Vault Secrets Officer |
| Full Key Vault control | Key Vault Administrator |

The role should be scoped to that specific Key Vault -- not subscription level,  
not resource group level -- just the Key Vault itself.  
This follows the principle of least privilege.

---

### Q6: What are the three PIM states and what does each mean?

**Your answer:** Correct after explanation.  
**Score: 85%**

**Correct Answer:**

**Eligible** -- The person is on the approved list to request the role.  
They do not have it yet. They cannot perform any privileged actions.  
Think of it as being approved to apply for a visitor badge but not having it yet.

**Active** -- The request was approved. The role is live right now.  
The person has the permissions and a timer is running.  
Think of it as holding an active visitor badge with an expiry time printed on it.

**Expired** -- The time window ended. The role was automatically removed.  
The person is back to Eligible. They must submit a new request with  
a new justification through the approval process again.  
The cycle does not repeat automatically.

---

### Q7: A DINE policy shows non-compliant VMs but the monitoring agent is still not being installed. What is the most likely cause and fix?

**Your answer:** Did not know.  
**Score: 0%**

**Correct Answer:**  
This is one of the most common silent failures in real ALZ deployments.  
Two likely causes:

**Cause 1 -- Managed Identity Has No RBAC Role:**  
When a DINE policy assignment is created, Azure auto-creates a System-Assigned  
Managed Identity for that policy assignment.  
Creating the identity does not give it any permissions automatically.  
If no RBAC role was assigned to it, it attempts the remediation deployment,  
Azure rejects it silently, and the VM stays non-compliant with no obvious error.

Fix:
1. Go to the policy assignment
2. Find the System-Assigned Managed Identity created for it
3. Assign it the correct RBAC role -- Contributor or Log Analytics Contributor --  
   at the scope where it needs to deploy
4. Trigger a manual remediation task from the Policy compliance blade

**Cause 2 -- No Remediation Task for Pre-Existing Resources:**  
DINE fires automatically only for NEW resources created after the policy was assigned.  
Resources that existed before the policy assignment are not automatically remediated.  
You must manually create a remediation task for those existing resources.

Fix: Policy -- Remediation -- Create remediation task -- select the non-compliant resources.

**Memorise this:** When a DINE policy evaluates correctly but does not remediate --  
first check the Managed Identity permissions,  
second check whether a remediation task exists for pre-existing resources.

---

## Day 4 Final Score: 59 / 100

| Question | Score | Key Gap |
|---|---|---|
| Q1 -- Global Admin vs Azure RBAC | 60% | Initially wrong -- two systems are separate |
| Q2 -- Owner vs Contributor for app teams | 90% | Missing role assignment exclusion reason |
| Q3 -- PIM and compliance | 70% | PIM scheduling does not exist -- it is on-demand |
| Q4 -- Pipeline identity | 40% | Service Principal not Managed Identity for external pipelines |
| Q5 -- VM to Key Vault | 65% | Key Vault Secrets User not Security Reader |
| Q6 -- PIM states | 85% | All three correct after explanation |
| Q7 -- DINE silent failure | 0% | Did not know -- must memorise both causes |

---

## Before Day 5 -- Two Things You Must Do

**1. Answer Q4 from memory:**  
An Azure DevOps pipeline needs to deploy resources to Azure.  
What identity does it use and why not a Managed Identity?  
Write the answer without looking at this file.

**2. Answer Q7 from memory:**  
A DINE policy is not remediating. What are the two causes and the fix for each?  
Write both causes and both fixes without looking at this file.

If you cannot answer both -- re-read Concept 4, Concept 5, and Concept 7 in this file  
before opening the Day 5 goal file.

---

## Day 5 Preview -- Monitoring, Logging, and AMBA-ALZ

- Log Analytics Workspace -- what it is and why ALZ has two separate ones
- Azure Monitor -- alerts, dashboards, and action groups
- AMBA-ALZ -- Azure Monitor Baseline Alerts for Landing Zones
- Defender for Cloud -- security posture and recommendations
- Diagnostic settings -- what gets logged, where it goes, and why it matters
- Hands-on: Log Analytics and Defender for Cloud in the portal

---

*Day 4 of 21 | Azure Landing Zone Study Track | March 2026*