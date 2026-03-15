# ALZ Day 2 -- Azure Policy, Governance Engine, and First Hands-On

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
         Appears in compliance report
```

**Client scenario:** A company wants visibility into which resources are missing cost-centre tags before enforcing a hard deny. Audit gives them the report without blocking work.

---

### Effect 3: DINE (DeployIfNotExists)
**What it does:** After a resource is deployed, if a related resource or configuration is missing, it automatically deploys it.  
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
                    |
                    v
    Monitoring agent is installed automatically
    Developer did not have to do anything
```

**Why this is powerful:** Application teams cannot forget to install monitoring agents. They cannot skip diagnostic settings. The policy does it for them, silently, every time.

**Important:** DINE policies require a **Managed Identity** to perform the deployment. This identity needs the right RBAC permissions to deploy the remediation resource.

---

### Effect 4: Modify
**What it does:** Adds, replaces, or removes properties on a resource during creation or update.  
**Real example:** Automatically adds a required tag to every resource group that is created, even if the developer forgot to add it.

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
    automatically during creation
```

**Difference from DINE:** Modify changes properties on the resource itself. DINE deploys a separate related resource.

---

## Section 2 -- Policy Initiatives (How ALZ Bundles Policies)

A single policy governs one rule. A **Policy Initiative** (also called a Policy Set) is a bundle of multiple policies assigned together as one unit.

ALZ uses built-in Policy Initiatives to apply dozens of policies in one assignment.

```
ALZ Policy Initiative Example: "Enforce recommended guardrails for Landing Zones"
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

**The cascade in plain English:**
You assign one initiative to the Landing Zones management group. Every subscription under Landing Zones -- A1, A2, P1, every future subscription -- inherits all of those policies automatically. You do not touch individual subscriptions.

---

## Section 3 -- Policy Scope and Inheritance

Policies can be assigned at four scopes:

```
Management Group  <-- Assign here, cascades to everything below
    └── Subscription  <-- Assign here, cascades to RGs and resources below
            └── Resource Group  <-- Assign here, applies to resources inside
                    └── Individual Resource  <-- Can assign here but rarely done
```

**ALZ approach:** Always assign at the highest relevant Management Group level. Let inheritance do the work. Never rely on resource-level or resource-group-level policy assignments for governance -- those are too granular and easy to miss.

---

## Section 4 -- Policy Compliance Dashboard (What You Will See in the Portal)

When you open Azure Policy in the portal, you see:

- **Overall compliance percentage** -- what percentage of resources comply with assigned policies
- **Non-compliant resources** -- list of every resource that violates a policy
- **Remediation tasks** -- for DINE and Modify policies, you can trigger bulk remediation on existing non-compliant resources
- **Policy assignments** -- what policies are assigned at what scope

This dashboard is what you show a client when they ask "how do we know our environment is compliant?"

---

## Hands-On Tasks -- Do These Today

### Task 1: Explore Azure Policy in the Portal (20 minutes)

1. Go to portal.azure.com
2. Search for "Policy" in the top search bar
3. Click on "Definitions" in the left menu
4. In the search box, search for: `Require a tag on resources`
5. Open the policy definition and read:
   - What is the Effect?
   - What parameter does it take?
   - Read the actual JSON rule -- do not memorise it, just read the structure
6. Go back and search for: `Deploy Log Analytics agent`
7. Open it and identify:
   - Is this Deny, Audit, DINE, or Modify?
   - What does the "then" block say?
   - Does it mention a roleDefinitionIds field? (This is the RBAC the Managed Identity needs)

**What you are building:** The ability to read a policy definition and immediately know what it does and when it fires.

---

### Task 2: Look at a Policy Initiative (15 minutes)

1. In Azure Policy, click "Definitions" and filter Category by "Security Center" or search for "Azure Security Benchmark"
2. Open it
3. Scroll through the list of policies inside it
4. Count how many are Audit vs Deny vs DINE
5. Note: this is similar in structure to what ALZ bundles into its own initiatives

---

### Task 3: Open Real ALZ Policy Definitions on GitHub (10 minutes)

1. Go to: `https://github.com/Azure/Enterprise-Scale`
2. Navigate to: `eslzArm/managementGroupTemplates/policyDefinitions`
3. Open any one JSON file
4. Find the `policyRule` section
5. Identify the `effect` field
6. Read the `if` condition -- what resource type is it targeting?

You are not memorising this. You are getting comfortable reading real ALZ policy code so it is not alien when you encounter it on an engagement.

---

### Task 4: Open Your First AVM Module on GitHub (10 minutes)

1. Go to: `https://github.com/Azure/bicep-registry-modules`
2. Navigate to: `avm/res/network/virtual-network`
3. Open the `README.md`
4. Look at:
   - What parameters does it take?
   - What is the `diagnosticSettings` parameter? (Notice it is built in -- you do not have to add monitoring separately)
   - What is the `tags` parameter? (Built in -- tagging is not an afterthought)
5. Do not try to write code with it yet. Just read what it offers.

---

## Day 2 Self-Test (Answer Without Notes)

1. What is the difference between Deny and Audit policy effects?
2. A VM is deployed without a monitoring agent. Which policy effect handles this automatically?
3. What is a Policy Initiative and why does ALZ use them instead of individual policies?
4. If a policy is assigned at a Management Group level, which resources does it apply to?
5. What is the difference between DINE and Modify?
6. Why does a DINE policy need a Managed Identity?
7. What did you notice about the AVM Virtual Network module that makes it better than a manually written deployment?

If you cannot answer all 7 without looking -- you are not done with Day 2.

---

## Day 2 Completion Checklist

- [ ] All four policy effects understood and explainable
- [ ] Policy Initiative concept understood
- [ ] Cascade / inheritance logic understood
- [ ] Azure Policy portal explored -- at least 2 definitions read
- [ ] One ALZ policy definition read on GitHub
- [ ] One AVM module README opened and reviewed
- [ ] All 7 self-test questions answered without notes

---

## Day 3 Preview -- What Is Coming

Day 3 is where networking begins.

- Hub-Spoke topology in detail
- What a UDR (User Defined Route) does and why it matters
- How DNS resolution works in a Landing Zone
- NSGs vs Azure Firewall -- when each is used
- Hands-on: Look at the AVM Azure Firewall and Virtual Network modules side by side

Networking is where most people go shallow. Day 3 will not let you do that.

---

*Day 2 of 21 | Azure Landing Zone Study Track | March 2026*