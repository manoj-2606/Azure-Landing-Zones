# ALZ Day 2 -- Azure Policy, Governance Engine, Hands-On and Self-Test

> **The single goal of Day 2:**  
> Understand Azure Policy so well that you can explain what happens  
> to a resource the moment it is deployed into a Landing Zone subscription --  
> without looking at any notes.
>
> **Time Required:** 2 hours  
> **Split:** 45 min reading | 45 min hands-on portal | 30 min AVM on GitHub

---

## Why Day 2 Is About Policy Before Anything Else

Yesterday you learned the skeleton of ALZ -- the hierarchy, the subscriptions, the flow.

But you left out the engine.

The reason ALZ works -- the reason a developer cannot deploy an insecure resource, the reason monitoring agents appear automatically, the reason tags are enforced -- is entirely because of **Azure Policy.**

Without Policy, ALZ is just a folder structure with subscriptions in it. Policy is what gives the hierarchy its power.

You cannot do any meaningful hands-on with ALZ until this is locked in.

---

## Section 1 -- The Four Policy Effects (Know These Cold)

Every Azure Policy has an **effect** -- what it does when a resource is evaluated against the rule.

---

### Effect 1: Deny
**What it does:** Blocks the resource from being created or updated if it violates the rule.  
**When the block happens:** At deployment time. The resource never exists.  
**Real example:** A policy that denies creation of any storage account with public blob access enabled.

```
Developer tries to deploy storage account with public access ON
                    |
                    v
          Policy evaluates the resource
                    |
                    v
              Rule is violated
                    |
                    v
         Deployment BLOCKED. Error returned.
         Resource never created.
```

**Client scenario:** A bank needs to ensure no storage account ever has public access. Deny policy enforces this with zero human intervention.

---

### Effect 2: Audit
**What it does:** Allows the resource to be created but marks it as non-compliant in the policy compliance dashboard.  
**When it fires:** After deployment. Resource exists but is flagged.  
**Real example:** A policy that audits any VM that does not have a specific tag.

```
Developer deploys VM without required tag
                    |
                    v
          Policy evaluates the resource
                    |
                    v
              Rule is violated
                    |
                    v
         VM is created successfully
         BUT marked as Non-Compliant
         Visible on live compliance dashboard (not just at audit time)
```

**Important:** Audit compliance is monitored continuously and in real time. It is not a once-a-year event. The dashboard is always live.

**Client scenario:** A company wants visibility into which resources are missing cost-centre tags before enforcing a hard deny. Audit gives them the live report without blocking work.

---

### Effect 3: DINE (DeployIfNotExists)
**What it does:** After a resource is deployed, if a related **separate** resource or configuration is missing, it automatically deploys that missing resource.  
**When it fires:** After deployment of the triggering resource.  
**Real example:** When a VM is created, if the Log Analytics monitoring agent is not installed, the policy deploys it automatically.

```
Developer deploys a VM
                    |
                    v
          Policy evaluates the VM
                    |
                    v
    Checks: Is monitoring agent installed?
                    |
                    v
              NO -- agent missing
                    |
                    v
    Policy triggers a remediation deployment
    using its Managed Identity
                    |
                    v
    Monitoring agent is installed automatically
    Developer did not have to do anything
```

**Why DINE needs a Managed Identity:** DINE has to perform a real Azure deployment action on your behalf to create the missing resource. It needs an identity with the right RBAC permissions to do that. The Managed Identity is that identity. Without it, the policy has no way to deploy anything.

**Key distinction to memorise:**  
DINE deploys a **separate, missing dependent resource.**

---

### Effect 4: Modify
**What it does:** Adds, replaces, or removes a **property on the resource itself** during creation or update.  
**Real example:** Automatically adds a required tag to every resource group that is created, even if the developer forgot.

```
Developer creates Resource Group without environment tag
                    |
                    v
          Policy evaluates the Resource Group
                    |
                    v
    Tag missing -- Modify effect fires
                    |
                    v
    Policy adds tag: environment = production
    directly onto the resource itself
```

**Key distinction to memorise:**  
Modify changes **a property on the resource that already exists.**

---

### DINE vs Modify -- The One Sentence That Separates Them

| Effect | What It Does | Operates On |
|---|---|---|
| DINE | Deploys a missing dependent resource | A separate new resource |
| Modify | Changes a property on the resource | The resource itself |

**Say this out loud and memorise it:**  
"DINE deploys a missing dependent resource. Modify changes a property on the resource that already exists."

---

## Section 2 -- Policy Initiatives (How ALZ Bundles Policies)

A single policy governs one rule. A **Policy Initiative** (also called a Policy Set) is a bundle of multiple policies assigned together as one unit.

ALZ uses built-in Policy Initiatives to apply dozens of policies in one assignment.

```
ALZ Policy Initiative: "Enforce recommended guardrails for Landing Zones"
│
├── Deny: Storage accounts with public access
├── Deny: VMs without disk encryption
├── Audit: Resources without required tags
├── DINE: Deploy Log Analytics agent to VMs
├── DINE: Deploy diagnostic settings to Key Vault
├── DINE: Deploy Microsoft Defender for VMs
├── Modify: Add environment tag to resource groups
└── ... (dozens more)

One Initiative Assignment at Management Group level
= All of the above rules enforced on every subscription below
```

**The ALZ pattern:** Always assign initiatives at the **Management Group level**, not at the subscription level. Subscription-level assignment is possible but is not the ALZ approach. The whole point is to assign once at the top and let every child subscription inherit automatically.

---

## Section 3 -- Policy Scope and Inheritance

Policies cascade strictly top-down:

```
Management Group   <-- Assign here (ALZ default)
    └── Subscription
            └── Resource Group
                    └── Individual Resource
```

A policy assigned at Management Group level applies to:
- Every subscription under that Management Group
- Every Resource Group inside those subscriptions
- Every resource inside those Resource Groups

You assign once. Everything below inherits. No manual work per subscription.

---

## Section 4 -- AVM vs Manual IaC

| Manual IaC | AVM |
|---|---|
| Written by one engineer, their style | Standardised across all engineers |
| Security settings depend on who wrote it | Security hardening built into every module |
| Diagnostic settings often forgotten | Diagnostic settings parameter built in by default |
| Tags applied inconsistently | Tags parameter built into every module |
| Breaks when Azure APIs change | Microsoft maintains and updates modules |
| No formal review process | Reviewed and tested by Microsoft against Well-Architected Framework |
| Different result every deployment | Identical result every deployment |

**The real superiority of AVM in one sentence:**  
A junior engineer using an AVM module produces the same secure, compliant, monitored result as a senior engineer -- because the best practices are baked into the module, not dependent on the person writing the code.

---

## Hands-On Tasks -- Do These Today

### Task 1: Explore Azure Policy in the Portal (20 minutes)

1. Go to [portal.azure.com](https://portal.azure.com)
2. Search for **Policy** in the top search bar
3. Click **Definitions** in the left menu
4. Search for: `Require a tag on resources`
5. Open the definition and read:
   - What is the Effect?
   - What parameter does it take?
   - Read the JSON rule -- do not memorise, just read the structure
6. Go back and search for: `Deploy Log Analytics agent`
7. Open it and identify:
   - Is this Deny, Audit, DINE, or Modify?
   - What does the `then` block say?
   - Does it mention a `roleDefinitionIds` field? (This is the RBAC the Managed Identity needs)

---

### Task 2: Look at a Policy Initiative (15 minutes)

1. In Azure Policy, click **Definitions** and search for **Azure Security Benchmark**
2. Open it
3. Scroll through the policies bundled inside it
4. Count how many are Audit vs Deny vs DINE
5. This is structurally similar to what ALZ uses for its own policy initiatives

---

### Task 3: Open Real ALZ Policy Definitions on GitHub (10 minutes)

1. Go to the ALZ Enterprise Scale repository:  
   [https://github.com/Azure/Enterprise-Scale](https://github.com/Azure/Enterprise-Scale)
2. Navigate to:  
   `eslzArm/managementGroupTemplates/policyDefinitions`
3. Open any one JSON file
4. Find the `policyRule` section
5. Identify the `effect` field
6. Read the `if` condition -- what resource type is it targeting?

---

### Task 4: Open Your First AVM Module on GitHub (10 minutes)

1. Go to the AVM Bicep Registry:  
   [https://github.com/Azure/bicep-registry-modules](https://github.com/Azure/bicep-registry-modules)
2. Navigate to: `avm/res/network/virtual-network`
3. Open the `README.md`
4. Look at:
   - What parameters does it take?
   - Find the `diagnosticSettings` parameter -- it is built in, you do not add monitoring separately
   - Find the `tags` parameter -- tagging is not an afterthought
   - Find the `lock` parameter -- resource locking is built in
5. Do not write code yet. Read and observe what the module provides by default.

---

## GitHub Reference Pages

| Resource | URL | What to Use It For |
|---|---|---|
| ALZ Enterprise Scale Repository | [https://github.com/Azure/Enterprise-Scale](https://github.com/Azure/Enterprise-Scale) | Real ALZ policy definitions, ARM templates, management group structure |
| ALZ Bicep Repository | [https://github.com/Azure/ALZ-Bicep](https://github.com/Azure/ALZ-Bicep) | ALZ deployment using Bicep modules |
| ALZ Terraform Repository | [https://github.com/Azure/terraform-azurerm-caf-enterprise-scale](https://github.com/Azure/terraform-azurerm-caf-enterprise-scale) | ALZ deployment using Terraform |
| AVM Bicep Registry Modules | [https://github.com/Azure/bicep-registry-modules](https://github.com/Azure/bicep-registry-modules) | All AVM resource modules in Bicep |
| AVM Official Site | [https://azure.github.io/Azure-Verified-Modules](https://azure.github.io/Azure-Verified-Modules) | AVM module catalogue, guidelines, and documentation |
| ALZ Documentation (CAF) | [https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone) | Official Microsoft conceptual documentation |
| Azure Policy Built-in Definitions | [https://learn.microsoft.com/en-us/azure/governance/policy/samples/built-in-policies](https://learn.microsoft.com/en-us/azure/governance/policy/samples/built-in-policies) | Full list of built-in Azure Policy definitions |
| AMBA-ALZ Alerts | [https://azure.github.io/azure-monitor-baseline-alerts/alz](https://azure.github.io/azure-monitor-baseline-alerts/alz) | Azure Monitor baseline alert definitions for ALZ |

---

## Self-Test Questions and Correct Answers

Answer each question without looking at notes first. Then verify against the correct answer below.

---

### Question 1: What is the difference between Deny and Audit policy effects?

**Your answer space:** Write or say your answer before reading below.

**Correct Answer:**  
Deny blocks a resource from being created or updated at deployment time if it violates the policy rule. The resource never exists in Azure. Audit allows the resource to be created but marks it as non-compliant on the policy compliance dashboard. The compliance dashboard is live and continuous -- it is not a once-a-year event. Audit is used when a company wants visibility before enforcing a hard block.

---

### Question 2: A VM is deployed without a monitoring agent. Which policy effect handles this automatically and how?

**Your answer space:** Write or say your answer before reading below.

**Correct Answer:**  
DINE (DeployIfNotExists). After the VM is deployed, the DINE policy evaluates it and checks whether the monitoring agent exists as a dependent resource. If it does not exist, the policy triggers a remediation deployment using its Managed Identity to install the agent automatically. The developer does not need to do anything.

---

### Question 3: What is a Policy Initiative and why does ALZ assign it at Management Group level instead of subscription level?

**Your answer space:** Write or say your answer before reading below.

**Correct Answer:**  
A Policy Initiative is a bundle of multiple policies grouped and assigned together as a single unit. ALZ always assigns initiatives at the Management Group level -- not the subscription level -- because any assignment at the Management Group cascades automatically down to every subscription, resource group, and resource beneath it. Assigning at the subscription level would mean repeating the assignment for every new subscription that gets created. The Management Group assignment means every new landing zone subscription inherits all policies automatically the moment it is placed under the correct Management Group.

---

### Question 4: If a policy is assigned at Management Group level, which resources does it apply to?

**Your answer space:** Write or say your answer before reading below.

**Correct Answer:**  
It applies to every subscription under that Management Group, every resource group inside those subscriptions, and every resource inside those resource groups. The assignment cascades strictly top-down through the entire hierarchy. You assign once at the top and every child resource in the tree inherits it.

---

### Question 5: What is the difference between DINE and Modify?

**Your answer space:** Write or say your answer before reading below.

**Correct Answer:**  
DINE deploys a separate, missing dependent resource. Example: a VM is created and DINE detects the monitoring agent is missing, so it deploys the agent as a new resource.  
Modify changes a property directly on the resource itself. Example: a resource group is created without a required tag, so Modify adds the tag directly onto that resource group.  
The key distinction: DINE creates something new and separate. Modify changes something on the existing resource.

---

### Question 6: Why does a DINE policy require a Managed Identity?

**Your answer space:** Write or say your answer before reading below.

**Correct Answer:**  
DINE has to perform a real Azure deployment action -- it needs to create or configure a resource inside Azure on behalf of the policy. To do that, it needs an authenticated identity with the correct RBAC permissions assigned to it. The Managed Identity is that identity. Without it, the policy cannot execute the deployment -- it has no credentials or permissions to act.

---

### Question 7: What did you notice about the AVM Virtual Network module that makes it better than a manually written deployment?

**Your answer space:** Write or say your answer before reading below.

**Correct Answer:**  
The AVM Virtual Network module has the following built in by default -- none of which require the engineer to add manually:  
- `diagnosticSettings` parameter for monitoring and log forwarding  
- `tags` parameter for consistent tagging  
- `lock` parameter for resource locking  
- `roleAssignments` parameter for RBAC  
- Security defaults aligned to the Well-Architected Framework  

In a manually written deployment, each of these is dependent on the engineer remembering to include them. AVM makes them impossible to forget because they are part of the module interface. A junior engineer using AVM produces the same secure, monitored, tagged result as a senior engineer.

---

## Day 2 Completion Checklist

- [ ] All four policy effects understood and explainable without notes
- [ ] DINE vs Modify distinction memorised as a single sentence
- [ ] Policy Initiative concept and ALZ assignment pattern understood
- [ ] Cascade and inheritance logic understood
- [ ] Azure Policy portal explored -- at least 2 definitions read (Tasks 1 and 2)
- [ ] One ALZ policy definition read on GitHub (Task 3)
- [ ] One AVM module README opened and reviewed (Task 4)
- [ ] All 7 self-test questions answered without notes before checking answers
- [ ] Score yourself honestly -- 80 and above means proceed to Day 3

---

## Day 3 Preview -- What Is Coming

Day 3 is networking. It is the hardest conceptual day in the first week.

- Hub-Spoke topology in detail
- What a UDR (User Defined Route) does and why it matters
- How DNS resolution works inside a Landing Zone
- NSGs vs Azure Firewall -- when each is used and why both exist
- Hands-on: AVM Azure Firewall and Virtual Network modules side by side

Networking is where most people go shallow and get exposed in client conversations. Day 3 will not let you do that.

---

*Day 2 of 21 | Azure Landing Zone Study Track | March 2026*