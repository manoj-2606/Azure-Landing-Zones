# ALZ Day 6 -- Concepts, Q&A, and Correct Answers

> This file contains every concept covered in Day 6 and the full Q&A session.  
> Use this as a reference before any client conversation involving ALZ deployment pipelines.  
> Day 6 score: 72/100 -- review Q5 (OIDC mechanism) and Q7 (full vending flow) before Day 7.

---

## Core Concepts Covered in Day 6

---

### Concept 1 -- Subscription Vending

**What it is:**  
The automated pipeline process that stamps out a new, fully configured  
landing zone subscription on demand.

**The problem it solves:**  
Without it: new team waits 2 weeks, manual process, different result every time.  
With it: pull request triggers pipeline, landing zone ready in minutes, identical every time.

**The complete flow:**
```
Step 1: App team submits a pull request
        Adds a tfvars file defining their landing zone:
        subscription name, VNet CIDR, team Entra group ID

Step 2: Pipeline triggers on PR
        terraform init -- connects to remote state backend
        terraform plan -- generates full change preview
        Plan posted as PR comment for platform team review

Step 3: Platform team reviews the PLAN OUTPUT
        Not just the code -- the plan showing exactly what will be created
        Catches CIDR errors, wrong group IDs, wrong MG placement
        before anything is deployed

Step 4: PR approved and merged to main

Step 5: terraform apply runs automatically
        Subscription created and placed under correct Management Group
        Spoke VNet created with subnets (AVM module)
        VNet peered to hub (Gateway Transit, Remote Gateways, Forwarded Traffic)
        UDR attached to subnets forcing traffic to Azure Firewall
        NSGs created per subnet
        Contributor assigned to app team Entra group (AVM module)
        Reader assigned to security team
        Log Analytics Workspace created
        Diagnostic settings configured

Step 6: Landing zone handed to app team
        Subscription ID and VNet details delivered
        Environment is compliant, monitored, and network-connected from minute one
```

**The CTO-level answer:**  
Speed + consistency + auditability.  
Minutes not weeks. Identical every time. Code is the documentation. Pipeline history is the audit trail.

---

### Concept 2 -- Terraform Remote State in ALZ

**Why remote state instead of local state:**

| Local State | Remote State (Azure Storage) |
|---|---|
| Lives on the engineer's machine | Lives in Azure Storage Account |
| Deleted accidentally = gone forever | Protected by resource locks and policies |
| One pipeline at a time | Multiple pipelines can read simultaneously |
| No locking | State locking -- prevents concurrent corruption |
| Cannot share outputs between pipelines | Cross-pipeline output sharing |

**The ALZ-specific reason -- cross-pipeline sharing:**
```
Networking pipeline runs
        |
        v
Hub Firewall created
Firewall private IP written to remote state
        |
Landing zone pipeline runs later
        |
        v
Reads firewall IP from remote state
Uses it to configure UDR in the spoke
```

Without remote state this cross-pipeline data sharing is impossible.

**State locking:**  
When one pipeline run modifies a state file, Azure Storage locks it.  
No other pipeline run can modify the same state simultaneously.  
Prevents two concurrent runs from corrupting the same infrastructure.

**State file structure in ALZ:**
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate-platform"
    storage_account_name = "stftstateprod001"
    container_name       = "tfstate"
    key                  = "alz-landing-zone-a1.tfstate"
  }
}
```

Each landing zone has its own state file key.  
Platform infrastructure has its own separate state files.  
Never share one state file across the entire ALZ.

---

### Concept 3 -- AVM Terraform Modules

**How to call an AVM module:**
```hcl
module "spoke_vnet" {
  source  = "Azure/virtual-network/azurerm"
  version = "~> 1.0"

  name                = "vnet-landingzone-a1-prod"
  resource_group_name = azurerm_resource_group.spoke.name
  location            = "eastus"
  address_space       = ["10.1.0.0/24"]

  subnets = {
    app_subnet = {
      name             = "snet-app"
      address_prefixes = ["10.1.0.0/26"]
    }
    db_subnet = {
      name             = "snet-db"
      address_prefixes = ["10.1.0.64/26"]
    }
  }

  diagnostic_settings = {
    to_platform_workspace = {
      workspace_resource_id = data.azurerm_log_analytics_workspace.platform.id
    }
  }

  tags = {
    environment  = "production"
    landing-zone = "a1"
    team         = "app-team-a"
  }
}
```

**AVM vs custom module -- what cannot be forgotten:**

| Custom Module | AVM Module |
|---|---|
| diagnostic_settings -- forgotten if engineer misses it | BUILT IN -- just pass workspace ID |
| tags -- inconsistent between engineers | BUILT IN -- consistent across all resources |
| role_assignments -- forgotten if engineer misses it | BUILT IN parameter |
| resource locking -- rarely remembered | BUILT IN parameter |
| No formal review | Microsoft-reviewed, Well-Architected aligned |

**The human dependency argument:**  
In a custom module every omission depends on the engineer remembering.  
One engineer adds diagnostic settings. The next forgets.  
Six months later 40 resources have inconsistent monitoring.  
AVM makes omissions impossible -- parameters are visible in the module interface.

---

### Concept 4 -- Terraform Plan on PR, Apply on Merge

**Why this pattern exists:**
```
Developer pushes to feature branch
        |
        v
Pull Request opened
        |
        v
terraform plan runs automatically
Generates: what will be created, modified, destroyed
Posted as PR comment
        |
        v
Platform team reviews THE PLAN OUTPUT
Not just the code -- the plan catches:
- Wrong CIDR ranges
- Wrong RBAC group IDs
- Wrong Management Group placement
- Unexpected resource deletions
        |
        v
PR approved and merged to main
        |
        v
terraform apply runs automatically on main only
        |
        v
Changes deployed to production
```

**The critical distinction:**  
The platform team reviews the plan output -- not just the code.  
Code review alone misses configuration errors.  
The plan shows in plain English exactly what Azure will look like after apply.  
That is the safety gate between development and production.

---

### Concept 5 -- OIDC Authentication

**The problem with Service Principal secrets:**

| Service Principal Secret | Reality |
|---|---|
| Stored in pipeline variables | Can be accidentally logged or leaked |
| Expires every 90 days | Manual rotation -- if forgotten pipeline fails |
| Works from anywhere if stolen | No scope restriction |

**How OIDC works -- no secret at any point:**
```
Pipeline run starts
        |
        v
GitHub generates a short-lived TOKEN for this specific run
Cryptographically signed and tied to:
- This exact repository
- This exact branch
- This exact workflow
        |
        v
Azure validates token against federation trust
configured on the Service Principal
        |
        v
Pipeline receives short-lived Azure access token
Valid only for the duration of this run
        |
        v
Terraform deploys using the token
        |
        v
Token expires when run ends
Nothing stored anywhere at any point
```

**The correct comparison:**

| Service Principal Secret | OIDC |
|---|---|
| Secret stored in pipeline variables | No secret stored anywhere |
| Expires every 90 days -- manual rotation | No expiry -- nothing to rotate |
| Can appear in logs accidentally | Nothing to leak |
| Works from anywhere if stolen | Token tied to specific repo and branch only |

**The client-facing sentence:**  
"With OIDC there is no credential to steal, rotate, or accidentally expose.  
Azure trusts the pipeline identity itself, not a password."

**What OIDC is NOT:**  
It does not create a "temporary secret" or "bypass."  
There is no secret at any stage. The token is a cryptographic proof of identity,  
not a password or secret key.

---

### Concept 6 -- Separate State Files Per Landing Zone

**Two reasons -- both must be remembered:**

**Reason 1 -- Blast Radius:**  
If one state file covers the entire ALZ and a terraform apply corrupts it,  
every landing zone is at risk simultaneously.  
Separate state files mean a failure in Landing Zone A1  
cannot affect Landing Zone A2 or the platform infrastructure.  
Damage is contained to the scope of the problem.

**Reason 2 -- Parallel Deployments:**  
When a pipeline run executes, it locks the state file.  
If everything shares one state file, deploying Landing Zone A1  
blocks Landing Zone A2 from deploying simultaneously.  
Separate state files allow multiple landing zones to deploy in parallel  
with no locking conflicts.

**The one sentence answer:**  
"Separate state files per landing zone contain the blast radius of any failure  
and allow multiple landing zones to deploy simultaneously without locking conflicts."

---

### Concept 7 -- The ALZ Terraform Repository Structure

```
terraform-azurerm-caf-enterprise-scale/
│
├── main.tf                    # Root module -- orchestrates everything
├── variables.tf               # All input variables with descriptions
│
├── modules/
│   ├── management_groups/     # Management group hierarchy
│   ├── policy_definitions/    # ALZ custom policy definitions
│   ├── policy_assignments/    # Policy initiative assignments at MG level
│   ├── connectivity/          # Hub networking -- Firewall, Gateway, DNS
│   ├── identity/              # Identity subscription resources
│   └── management/            # Management subscription -- Log Analytics, Monitor
│
└── locals.tf                  # Internal logic and computed values
```

**The minimal configuration to deploy a full ALZ:**
```hcl
module "enterprise_scale" {
  source  = "Azure/caf-enterprise-scale/azurerm"
  version = "~> 5.0"

  default_location             = "eastus"
  root_parent_id               = data.azurerm_client_config.current.tenant_id
  root_id                      = "contoso"
  root_name                    = "Contoso"

  subscription_id_management   = "xxxx-management-id"
  subscription_id_connectivity = "xxxx-connectivity-id"
  subscription_id_identity     = "xxxx-identity-id"
}
```

One module call. The entire ALZ platform layer.  
Every concept from Days 1 through 5 is an output of this.

---

## Day 6 Q&A Session -- Questions and Correct Answers

---

### Q1: Two-week environment setup problem -- how does ALZ fix it?

**Your answer:** Named AVM and Terraform -- missing mechanism and CTO framing.  
**Score: 60%**

**Correct Answer:**  
The mechanism is subscription vending.  
A new team submits a pull request adding their landing zone definition.  
A pipeline triggers automatically, runs Terraform, and creates the subscription,  
configures networking, sets up access controls, and connects monitoring.  
The whole process takes minutes not weeks.  
Because it is code running the same way every time, the result is always identical --  
no variation between environments depending on who built them.  
The code is the documentation. The pipeline history is the audit trail.

---

### Q2: Why remote state in Azure Storage instead of local state?

**Your answer:** Durability correct -- missing cross-pipeline sharing and locking.  
**Score: 75%**

**Correct Answer:**  
Three reasons:  
First -- durability. Local state deleted means gone forever. Remote state in Azure Storage  
is protected by resource locks and policies.  
Second -- cross-pipeline sharing. In ALZ, the networking pipeline writes the firewall IP  
to remote state. The landing zone pipeline reads it to configure the UDR.  
This cross-pipeline output sharing is impossible with local state.  
Third -- state locking. Azure Storage automatically locks the state file during a pipeline run.  
No two concurrent runs can corrupt the same infrastructure simultaneously.

---

### Q3: AVM modules vs custom-written Terraform modules -- why use AVM?

**Your answer:** Correct -- built-in diagnostic settings, tags, RBAC, locking, Microsoft-verified.  
**Score: 85%**

**Correct Answer:**  
AVM modules have diagnostic settings, tags, role assignments, and resource locking  
built in as parameters -- you cannot forget them because they are visible in the interface.  
In a custom module every omission depends on the engineer remembering to include it.  
One engineer adds diagnostic settings. The next forgets.  
Six months later resources have inconsistent monitoring across the estate.  
AVM removes the human dependency entirely.  
Additionally AVM is Microsoft-reviewed, tested, and aligned to the Well-Architected Framework --  
security defaults are on by default, not opt-in.

---

### Q4: Why terraform plan on PR and terraform apply only on merge to main?

**Your answer:** Correct flow -- missing "review plan not just code" distinction.  
**Score: 85%**

**Correct Answer:**  
terraform plan on PR gives the platform team a review artefact before anything is deployed.  
Critically -- the team reviews the PLAN OUTPUT, not just the code.  
Code review alone misses configuration errors like wrong CIDR ranges,  
wrong RBAC group IDs, or unexpected resource deletions.  
The plan shows in plain English exactly what Azure will look like after apply.  
terraform apply runs only after the PR is approved and merged to main --  
never directly from a feature branch.  
This is the safety gate between development and production.

---

### Q5: OIDC authentication vs Service Principal secret in the pipeline

**Your answer:** 90-day rotation problem correct -- OIDC mechanism wrong.  
**Score: 60%**

**Correct Answer:**  
With a Service Principal secret, a credential is stored in pipeline variables.  
It expires every 90 days, requires manual rotation, and can accidentally appear in logs.  
With OIDC there is no secret at any point -- not temporary, not stored.  
GitHub generates a short-lived cryptographic token tied to the specific  
repository, branch, and workflow for that exact run.  
Azure validates the token against a pre-configured federation trust on the Service Principal.  
The pipeline receives a short-lived Azure access token that expires when the run ends.  
Nothing is stored anywhere. Nothing needs rotating. Nothing can be leaked.  
The client-facing sentence: "With OIDC there is no credential to steal,  
rotate, or accidentally expose. Azure trusts the pipeline identity itself, not a password."

---

### Q6: Why separate state files per landing zone instead of one shared state file?

**Your answer:** Right instinct -- blast radius and locking reasons missing.  
**Score: 55%**

**Correct Answer:**  
Two specific reasons:  
First -- blast radius. If one state file covers the entire ALZ and becomes corrupted,  
every landing zone is at risk simultaneously.  
Separate state files mean a failure in one landing zone cannot affect another.  
Second -- parallel deployments. A pipeline run locks the state file it is using.  
One shared state file means deploying Landing Zone A1 blocks Landing Zone A2  
from deploying simultaneously.  
Separate state files allow multiple landing zones to deploy in parallel  
with no locking conflicts.  
One sentence answer: "Separate state files contain blast radius and enable parallel deployments."

---

### Q7: Full subscription vending walkthrough using Terraform pipeline

**Your answer:** Got outcome and step 5 partially -- steps 1 through 4 missing entirely.  
**Score: 65%**

**Correct Answer:**
```
Step 1: App team submits pull request
        Adds tfvars file: subscription name, VNet CIDR, team group ID

Step 2: Pipeline triggers on PR
        terraform init -- connects to remote state backend
        terraform plan -- generates full change preview
        Plan posted as PR comment

Step 3: Platform team reviews PLAN OUTPUT
        Verifies correct CIDR, correct MG placement,
        correct team group ID, no unexpected deletions

Step 4: PR approved and merged to main

Step 5: terraform apply runs automatically
        Subscription created under correct Management Group
        Spoke VNet created with subnets (AVM module)
        VNet peered to hub with all three peering settings configured
        UDR attached forcing traffic to Azure Firewall
        NSGs created per subnet
        Contributor assigned to app team Entra group (AVM module)
        Reader assigned to security team
        Log Analytics Workspace created
        Diagnostic settings configured

Step 6: App team receives subscription ID and VNet details
        Environment compliant, monitored, and connected from minute one
```

Steps 1 through 4 are what make ALZ production-safe and auditable.  
The outcome without the process is not an answer in a client walkthrough.

---

## Day 6 Final Score: 72 / 100

| Question | Score | Key Gap |
|---|---|---|
| Q1 -- Subscription vending value | 60% | Missing mechanism name and CTO-level framing |
| Q2 -- Remote state vs local | 75% | Cross-pipeline sharing and locking missing |
| Q3 -- AVM vs custom modules | 85% | Human dependency argument missing |
| Q4 -- Plan on PR, apply on merge | 85% | "Review plan not just code" distinction missing |
| Q5 -- OIDC authentication | 60% | Mechanism wrong -- no secret at any point |
| Q6 -- Separate state files | 55% | Blast radius and parallel deployment reasons missing |
| Q7 -- Full vending walkthrough | 65% | Steps 1 through 4 entirely missing |

---

## Before Day 7 -- Two Things to Lock In

**1. OIDC in one sentence from memory:**  
GitHub generates a short-lived token tied to the specific repo and branch.  
Azure validates it. Pipeline gets a short-lived access token. Nothing stored anywhere.  
Write that without looking.

**2. Full vending flow from memory -- all 6 steps:**  
PR submitted -- pipeline triggers -- plan reviewed -- PR approved --  
apply runs -- landing zone handed over.  
If you cannot recall all 6 steps in order, re-read Concept 1 in this file.

---

## Score Trend

| Day | Score | Status |
|---|---|---|
| Day 2 | 82% | Strong start |
| Day 3 | 71% | Networking gaps |
| Day 4 | 59% | Identity gaps -- lowest point |
| Day 5 | 79% | Recovery |
| Day 6 | 72% | Steady -- OIDC and state file depth needed |

**Target for Day 7: 80 or above.**  
Day 7 is hands-on Terraform code writing.  
Performance in Q&A and performance writing real code are different skills.  
Both will be tested.

---

## Day 7 Preview -- Writing Real Terraform ALZ Code

- Write a spoke VNet using the AVM Terraform module
- Write a UDR and attach it to a subnet
- Write NSG rules for app and DB subnets
- Write RBAC assignments using the AVM role assignment module
- Run terraform plan locally and read the output

**Prerequisite:** Terraform must be installed locally before Day 7.  
Download: https://developer.hashicorp.com/terraform/install  
Verify installation: `terraform version`

---

*Day 6 of 21 | Azure Landing Zone Study Track -- Terraform Track | March 2026*