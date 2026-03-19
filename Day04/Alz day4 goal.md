# ALZ Day 4 -- Identity, RBAC, and Access Management

> **The single goal of Day 4:**  
> Understand who can do what at which level in Azure --  
> and be able to explain how ALZ controls access so that  
> the platform team governs everything and app teams  
> only touch what they own.
>
> **Time Required:** 2 hours  
> **Split:** 50 min concepts | 40 min hands-on portal | 30 min AVM + GitHub

---

## The Trap That Catches Everyone on Day 4

Most people use "Azure AD roles" and "Azure RBAC roles" interchangeably.  
They are two completely different systems that do completely different things.  
Confusing them in a client conversation will immediately signal that  
you do not understand identity in Azure.

This file will make the distinction permanent before anything else.

---

## Section 1 -- Two Identity Systems, One Azure

Azure has two separate access control systems running in parallel.  
They look similar. They are not.

```
┌─────────────────────────────────────────────────────────────┐
│                     MICROSOFT ENTRA ID                       │
│         (formerly Azure Active Directory)                    │
│                                                              │
│  Controls WHO CAN DO WHAT inside Entra ID itself            │
│  Examples:                                                   │
│  - Who can create users                                      │
│  - Who can manage groups                                     │
│  - Who can register applications                             │
│  - Who can configure Conditional Access policies             │
│                                                              │
│  Roles live here: Global Administrator, User Administrator,  │
│  Application Administrator, Privileged Role Administrator    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      AZURE RBAC                              │
│         (Role Based Access Control)                          │
│                                                              │
│  Controls WHO CAN DO WHAT on AZURE RESOURCES                │
│  Examples:                                                   │
│  - Who can create a VM                                       │
│  - Who can read a storage account                            │
│  - Who can deploy to a subscription                          │
│  - Who can assign policies                                   │
│                                                              │
│  Roles live here: Owner, Contributor, Reader,                │
│  and hundreds of built-in specific roles                     │
└─────────────────────────────────────────────────────────────┘
```

### The One Sentence Distinction

**Entra ID roles** control access to the identity directory itself.  
**Azure RBAC roles** control access to Azure resources.

A Global Administrator in Entra ID does not automatically have  
Owner access to an Azure subscription. These are separate systems.

---

## Section 2 -- Azure RBAC in Detail

### What RBAC Is

Role Based Access Control is the system that answers:  
*"Who is allowed to perform which actions on which Azure resources?"*

Every RBAC assignment has three parts:

```
WHO          +    WHAT ROLE     +    WHERE (SCOPE)
(Principal)       (Permissions)      (Management Group / Subscription / RG / Resource)

Example:
Dev Team Group  +  Contributor  +  Landing Zone A2 Subscription
```

### The Four Built-In Roles You Must Know

| Role | What It Can Do | Used For |
|---|---|---|
| **Owner** | Everything including assigning roles to others | Platform team at Management Group level |
| **Contributor** | Create and manage all resources, cannot assign roles | App teams on their own subscription |
| **Reader** | View everything, change nothing | Auditors, read-only access |
| **User Access Administrator** | Only assign roles, cannot create resources | Specific delegation scenarios |

### The RBAC Scope Hierarchy

RBAC assignments cascade downward exactly like policies do.

```
Management Group  <-- Assign here, applies to everything below
      └── Subscription
              └── Resource Group
                      └── Individual Resource
```

**ALZ pattern:**  
Platform team holds Owner at the Management Group level.  
App teams get Contributor at their own subscription level only.  
This means app teams can build anything inside their subscription  
but cannot touch other subscriptions or change governance above them.

### Why Role Assignment Is Separate From Resource Creation

This is the key security design in ALZ.

In a flat Azure environment, whoever creates a subscription often  
makes themselves Owner with no oversight.  
In ALZ, the platform team creates the subscription via the vending pipeline  
and assigns roles deliberately -- the app team never has Owner.

Owner can assign roles to others. If an app team member had Owner,  
they could grant anyone access to anything in their subscription,  
bypassing the governance model entirely.  
Contributor cannot assign roles. That boundary is intentional and critical.

---

## Section 3 -- Privileged Identity Management (PIM)

### The Problem PIM Solves

Without PIM, privileged roles are permanent.  
A person assigned Owner on a subscription has Owner 24 hours a day,  
7 days a week, forever -- even when they are on holiday,  
even at 3am when no work is being done, even after they leave the company.

Permanent privileged access is one of the most common attack vectors.  
If an attacker compromises an account with permanent Owner access,  
they have unrestricted access to everything under that scope immediately.

### What PIM Does

PIM implements Just-In-Time (JIT) access.  
Nobody has a privileged role permanently.  
When they need elevated access, they request it.  
The request goes through an approval workflow.  
If approved, the role is active for a limited time window (e.g. 4 hours).  
When the window expires, the role is automatically removed.

```
Without PIM:
Engineer --> Permanent Owner --> Always has full access
                                 Even when not working
                                 Even if account is compromised

With PIM:
Engineer --> Has no privileged role by default
         --> Requests Owner access for 4 hours
         --> Approval required from manager or security team
         --> Role activates for 4 hours
         --> All actions during that window are logged
         --> Role expires automatically
         --> Back to no privileged access
```

### PIM in ALZ Context

The platform team members do not have permanent Owner on Management Groups.  
They activate their privileged role via PIM only when they need to  
make a governance change, deploy a new landing zone, or modify policies.  
Every activation is logged -- who activated, when, for how long, what they did.

This audit trail is what compliance frameworks require.

### The Three PIM Role States

| State | Meaning |
|---|---|
| **Eligible** | Person can request the role but does not have it yet |
| **Active** | Role is currently active -- person has the permissions now |
| **Expired** | Time window ended -- role automatically removed |

---

## Section 4 -- Managed Identities

### The Problem They Solve

Applications and pipelines often need to access Azure resources.  
For example: a pipeline needs to deploy resources to a subscription,  
or an application needs to read from a Key Vault.

The old way: create a username and password (a Service Principal with a secret),  
store that secret somewhere, rotate it manually, hope nobody leaks it.  
This is insecure and operationally painful.

Managed Identity solves this by giving an Azure resource its own identity  
that Azure manages automatically. No passwords. No secrets to rotate.  
No credentials that can be leaked.

### Two Types of Managed Identity

**System-Assigned Managed Identity**

```
Created alongside a specific resource
Lives and dies with that resource
One-to-one relationship: one identity per resource
Example: A VM gets its own identity automatically
         When the VM is deleted, the identity is deleted too
```

**User-Assigned Managed Identity**

```
Created as a standalone resource
Can be attached to multiple resources
Survives if one of the resources is deleted
Example: One identity shared across 5 VMs in the same app tier
         Deleting one VM does not affect the identity
         The other 4 VMs keep their access
```

### When to Use Each

| Scenario | Use |
|---|---|
| Single VM needs to access Key Vault | System-Assigned |
| Multiple VMs share the same access needs | User-Assigned |
| DINE policy needs to deploy remediation resources | System-Assigned on the Policy Assignment |
| A DevOps pipeline needs to deploy Azure resources | User-Assigned (shared across pipeline agents) |
| Resource is frequently recreated (ephemeral) | User-Assigned (identity persists) |

### Managed Identity in ALZ Context

Remember from Day 2 -- DINE policies need a Managed Identity to perform  
their remediation deployments. That identity is System-Assigned  
and created automatically when you assign the policy.  
It needs the correct RBAC role (usually Contributor or a specific role)  
on the scope where it needs to deploy.

---

## Section 5 -- Service Principals vs Managed Identities

Both are non-human identities used by applications and automation.  
They are not the same.

| | Service Principal | Managed Identity |
|---|---|---|
| What it is | An application identity in Entra ID | An Azure-managed identity for Azure resources |
| Credentials | Client ID + Secret or Certificate | No credentials -- Azure handles it |
| Secret rotation | Manual -- your responsibility | Automatic -- Azure's responsibility |
| Where it works | Anywhere (including outside Azure) | Only on Azure resources |
| Used for | Third-party apps, cross-cloud pipelines | Azure VMs, Functions, pipelines within Azure |
| Risk | Secret can be leaked or expire unexpectedly | No secret to leak |

### The Rule of Thumb

If the thing needing access is an Azure resource -- use Managed Identity.  
If the thing needing access is outside Azure or is a third-party tool -- use Service Principal.

In ALZ, pipelines running in Azure DevOps or GitHub Actions use  
a Service Principal OR a Managed Identity depending on the setup.  
Modern ALZ deployments prefer Managed Identity wherever possible.

---

## Section 6 -- RBAC in ALZ: The Full Picture

This is how access is structured across the entire ALZ estate.

```
Tenant Root Group
│   Owner: Platform Team (via PIM -- not permanent)
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
        │   Reader: Platform Team (visibility without write access)
        │
        ├── Landing Zone A1 Subscription
        │       Contributor: App Team A (their subscription only)
        │       Reader: Security Team
        │
        └── Landing Zone A2 Subscription
                Contributor: App Team B (their subscription only)
                Reader: Security Team
```

**What this means in plain English:**  
- App Team A has Contributor on A1 only -- cannot touch A2 or any platform subscription
- App Team B has Contributor on A2 only -- cannot touch A1 or any platform subscription
- Security Team has Reader everywhere -- visibility without the ability to change anything
- Platform Team has Owner everywhere but only via PIM -- never permanently active
- The DevOps pipeline uses a Service Principal with exactly the permissions it needs to deploy

---

## Section 7 -- The Three Concepts That Will Catch You in a Client Conversation

### Catch 1: Entra ID Global Admin Does Not Equal Azure Owner

A Global Administrator in Entra ID can manage users, groups, and applications.  
They cannot automatically manage Azure subscriptions or resources.  
These are separate systems.  
A Global Admin CAN elevate themselves to User Access Administrator in Azure RBAC  
but it is not automatic and requires a deliberate action.

### Catch 2: Owner vs Contributor -- The Role Assignment Difference

The only practical difference between Owner and Contributor is that  
Owner can assign roles to others. Contributor cannot.  
This single difference is why app teams get Contributor, not Owner.  
If an app team had Owner, they could grant anyone access to their subscription,  
bypassing the entire governance model.

### Catch 3: PIM Eligible Does Not Mean Active

Being eligible for a role in PIM means you CAN request it.  
It does not mean you have it.  
Many people see "Eligible" in PIM and assume they have the access.  
They do not -- until they activate the request and it is approved.

---

## Hands-On Tasks -- Do These Today

### Task 1: Explore RBAC in the Azure Portal (15 minutes)

1. Go to [portal.azure.com](https://portal.azure.com)
2. Navigate to any Subscription or Resource Group
3. Click **Access control (IAM)** in the left menu
4. Click **Role assignments** tab
5. Observe:
   - What roles are assigned?
   - What is the scope column showing?
   - What principal types exist (user, group, service principal, managed identity)?
6. Click **Roles** tab and search for **Contributor**
7. Open it and read what permissions it includes and what it explicitly excludes
8. Note the exclusion: `Microsoft.Authorization/*/Write` -- this is why Contributor cannot assign roles

---

### Task 2: Look at Built-In ALZ Role Definitions on GitHub (10 minutes)

1. Go to: [https://github.com/Azure/Enterprise-Scale](https://github.com/Azure/Enterprise-Scale)
2. Navigate to: `eslzArm/managementGroupTemplates/roleDefinitions`
3. Open any custom role definition JSON file
4. Identify:
   - `assignableScopes` -- where can this role be assigned?
   - `actions` -- what can this role do?
   - `notActions` -- what is explicitly excluded?
5. Notice how granular these custom roles are compared to built-in Owner/Contributor

---

### Task 3: Explore PIM in the Portal (10 minutes)

1. Search for **Privileged Identity Management** in the portal search bar
2. Click **My roles** in the left menu
3. Look at the difference between **Eligible assignments** and **Active assignments**
4. Click **Azure resources** tab
5. Even if you have no PIM roles, read the interface to understand:
   - What an activation request looks like
   - What the time-bound access window means
   - What justification fields exist

---

### Task 4: Explore Managed Identities in the Portal (10 minutes)

1. Search for **Managed Identities** in the portal
2. If any exist, open one and look at:
   - Is it System-Assigned or User-Assigned?
   - Click **Azure role assignments** -- what RBAC roles does it have?
   - What scope are those roles assigned at?
3. If none exist, navigate to any VM and look for **Identity** in the left menu
   - Observe the System-Assigned toggle (on/off)
   - Observe the User-Assigned tab

---

### Task 5: Open the AVM Role Assignment Module on GitHub (10 minutes)

1. Go to: [https://github.com/Azure/bicep-registry-modules](https://github.com/Azure/bicep-registry-modules)
2. Navigate to: `avm/res/authorization/role-assignment`
3. Open the `README.md`
4. Look at:
   - `principalId` -- the identity being assigned the role
   - `roleDefinitionIdOrName` -- can use built-in role name or custom role ID
   - `principalType` -- User, Group, ServicePrincipal, or ManagedIdentity
   - `scope` -- where the assignment is applied
5. This is how ALZ deploys RBAC assignments as code instead of manual portal clicks

---

## GitHub Reference Pages for Day 4

| Resource | URL | What to Use It For |
|---|---|---|
| AVM Role Assignment Module | [https://github.com/Azure/bicep-registry-modules/tree/main/avm/res/authorization/role-assignment](https://github.com/Azure/bicep-registry-modules/tree/main/avm/res/authorization/role-assignment) | RBAC deployment as code |
| AVM Role Definition Module | [https://github.com/Azure/bicep-registry-modules/tree/main/avm/res/authorization/role-definition](https://github.com/Azure/bicep-registry-modules/tree/main/avm/res/authorization/role-definition) | Custom role definitions as code |
| ALZ Role Definitions on GitHub | [https://github.com/Azure/Enterprise-Scale/tree/main/eslzArm/managementGroupTemplates/roleDefinitions](https://github.com/Azure/Enterprise-Scale/tree/main/eslzArm/managementGroupTemplates/roleDefinitions) | Real ALZ custom role JSON definitions |
| Azure Built-In Roles Reference | [https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles) | Full list of all built-in Azure RBAC roles |
| PIM Documentation | [https://learn.microsoft.com/en-us/azure/active-directory/privileged-identity-management/pim-configure](https://learn.microsoft.com/en-us/azure/active-directory/privileged-identity-management/pim-configure) | PIM setup and configuration |
| Managed Identity Documentation | [https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview) | Managed Identity overview and setup |
| ALZ Identity Design (CAF) | [https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/identity-access](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/identity-access) | Official ALZ identity and access design guidance |

---

## Self-Test Questions and Correct Answers

Answer each question in your own words first. Then check below.

---

### Question 1: What is the difference between a Microsoft Entra ID role and an Azure RBAC role?

**Correct Answer:**  
Entra ID roles control access to the identity directory itself -- who can create users,  
manage groups, register applications, configure Conditional Access.  
Azure RBAC roles control access to Azure resources -- who can create VMs,  
read storage accounts, deploy to subscriptions.  
A Global Administrator in Entra ID does not automatically have Owner in Azure RBAC.  
They are completely separate systems. Confusing them is one of the most common  
identity mistakes in Azure engagements.

---

### Question 2: Why do app teams get Contributor on their subscription instead of Owner?

**Correct Answer:**  
The only practical difference between Owner and Contributor is that Owner can  
assign roles to others. Contributor cannot.  
If an app team had Owner on their subscription, they could grant anyone --  
including external users -- access to their subscription without the platform team knowing.  
This would bypass the entire governance model.  
Contributor lets them build and manage everything they need  
while keeping role assignment authority with the platform team above.

---

### Question 3: What is PIM and why does ALZ use it instead of permanent role assignments?

**Correct Answer:**  
PIM is Privileged Identity Management. It implements just-in-time access --  
nobody holds a privileged role permanently.  
When elevated access is needed, the person requests it,  
it goes through an approval workflow, the role activates for a limited time window,  
and expires automatically.  
ALZ uses PIM because permanent privileged access is a major security risk --  
if a privileged account is compromised, an attacker has unrestricted access  
immediately and indefinitely. With PIM, the window of exposure is limited to  
the activation period, every activation is logged, and access is automatically removed.  
This also satisfies compliance frameworks that require just-in-time access.

---

### Question 4: What is the difference between a System-Assigned and User-Assigned Managed Identity?

**Correct Answer:**  
System-Assigned is created with and tied to one specific resource.  
It lives and dies with that resource. One-to-one relationship.  
User-Assigned is created as a standalone resource and can be attached  
to multiple Azure resources. It survives if one of the attached resources is deleted.  
Use System-Assigned when one resource needs its own identity.  
Use User-Assigned when multiple resources share the same access needs  
or when the resource is frequently recreated and you need the identity to persist.

---

### Question 5: What is the difference between a Service Principal and a Managed Identity?

**Correct Answer:**  
Both are non-human identities for applications and automation.  
A Service Principal requires credentials -- a client secret or certificate --  
that you create, store, rotate, and are responsible for securing.  
A Managed Identity has no credentials -- Azure manages the authentication automatically.  
No passwords, no secrets, nothing to leak or rotate.  
Service Principals are used when the consuming system is outside Azure  
or is a third-party tool that cannot use Managed Identity.  
Managed Identities are used whenever the consuming system is an Azure resource  
because they are more secure and require zero credential management.

---

### Question 6: Where does the platform team assign RBAC in ALZ and why at that level?

**Correct Answer:**  
The platform team holds Owner at the Management Group level -- assigned via PIM, not permanently.  
This is the highest scope available below the Tenant Root Group.  
Assigning at the Management Group level means it cascades to every subscription below,  
giving the platform team visibility and control across the entire estate  
without having to make individual assignments per subscription.  
App teams get Contributor at their own subscription level only --  
high enough to build what they need, too low to affect anything outside their scope.

---

### Question 7: A DINE policy deploys a monitoring agent to every VM. What identity does it use and what RBAC role does that identity need?

**Correct Answer:**  
A DINE policy uses a System-Assigned Managed Identity that is automatically created  
when the policy assignment is made.  
The identity needs the Contributor role (or a more specific role like  
Virtual Machine Contributor or Log Analytics Contributor) on the scope  
where it needs to perform the remediation deployment.  
Without the correct RBAC role on that identity,  
the policy will evaluate correctly and flag non-compliance  
but the remediation deployment will fail with an authorization error.  
This is a common silent failure in ALZ deployments --  
the policy exists, the compliance dashboard shows non-compliant resources,  
but nothing gets fixed because the Managed Identity has no permissions.

---

## Day 4 Completion Checklist

- [ ] Entra ID roles vs Azure RBAC roles -- distinction is permanent in your head
- [ ] Four built-in RBAC roles known: Owner, Contributor, Reader, User Access Administrator
- [ ] RBAC cascade understood -- assignment at MG level covers everything below
- [ ] PIM understood -- eligible vs active vs expired states
- [ ] System-Assigned vs User-Assigned Managed Identity distinction clear
- [ ] Service Principal vs Managed Identity -- know when to use each
- [ ] All 5 portal tasks completed
- [ ] AVM Role Assignment module opened on GitHub
- [ ] All 7 self-test questions answered without notes before checking
- [ ] Score yourself -- 80 and above means proceed to Day 5

---

## Day 5 Preview -- Monitoring, Logging, and AMBA-ALZ

- Log Analytics Workspace -- what it is and why ALZ has two (platform + app level)
- Azure Monitor -- alerts, dashboards, and action groups
- AMBA-ALZ -- Azure Monitor Baseline Alerts for Landing Zones
- Defender for Cloud -- security posture and recommendations
- Diagnostic settings -- what gets logged and where it goes
- Hands-on: Explore Log Analytics and Defender for Cloud in the portal

---

*Day 4 of 21 | Azure Landing Zone Study Track | March 2026*