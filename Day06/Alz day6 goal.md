# ALZ Day 6 -- Subscription Vending, Terraform AVM, and the Deployment Pipeline

> **The single goal of Day 6:**  
> Understand how the ALZ is actually built and deployed using Terraform and AVM --  
> how the pipeline takes code and turns it into a governed, networked,  
> monitored Azure environment -- and be able to trace that flow  
> from a client requirement all the way to a deployed landing zone.
>
> **Note:** From Day 6 onwards this track uses Terraform + AVM instead of Bicep.  
> Terraform is your existing strength. Build on it.
>
> **Time Required:** 2 hours  
> **Split:** 45 min concepts | 45 min hands-on Terraform + GitHub | 30 min pipeline reading

---

## Where You Are in the 21-Day Track

```
Days 1-5:   WHAT ALZ is and WHY it exists (concepts and governance)
Day 6:      HOW ALZ is built (Terraform, AVM, pipeline) <-- YOU ARE HERE
Days 7-9:   Deeper IaC -- writing and reading real Terraform ALZ code
Day 10+:    Hands-on deployment -- actually building a landing zone
Days 15-21: Migration, brownfield, and client scenario practice
```

Everything from Days 1 to 5 is the output of what you learn today.  
The pipeline builds the management groups, policies, networking,  
identity, and monitoring you have been studying all week.

---

## Section 1 -- Subscription Vending Revisited (Now With Code)

### What It Is (Refresher)

Subscription vending is the automated pipeline process that  
stamps out a new, fully configured landing zone subscription on demand.

### The Problem It Solves at Scale

Without subscription vending:
- A new team needs an environment
- Someone manually creates a subscription
- Manually places it under the right management group
- Manually sets up networking, RBAC, monitoring
- Takes days to weeks
- Result is different every time depending on who did it

With subscription vending:
- New team submits a request (a pull request or a form)
- Pipeline runs automatically
- Subscription created, placed, configured, connected in minutes
- Result is identical every time
- Fully auditable -- the code IS the documentation

### The Subscription Vending Flow With Terraform

```
Step 1: App team submits a pull request
        (adds a new landing zone definition to the Terraform config)
                |
                v
Step 2: Pipeline triggers automatically (GitHub Actions or Azure DevOps)
                |
                v
Step 3: terraform plan runs
        Shows exactly what will be created -- no surprises
        Platform team reviews the plan output
                |
                v
Step 4: Pull request approved and merged
                |
                v
Step 5: terraform apply runs
        ├── Creates new Subscription
        ├── Places it under correct Management Group
        ├── Spoke VNet created and peered to Hub
        ├── UDRs and NSGs configured
        ├── RBAC assigned (Contributor to app team)
        ├── Diagnostic settings configured
        └── Monitoring connected to central workspace
                |
                v
Step 6: Landing zone handed to app team -- ready immediately
```

---

## Section 2 -- Terraform Basics You Need for ALZ Context

You already know Terraform. This section maps your existing knowledge  
to how it is used specifically in an ALZ deployment.

### The Four Files in Every Terraform ALZ Module

```
alz-landing-zone/
├── main.tf          # Calls AVM modules -- the actual deployment logic
├── variables.tf     # Input parameters (subscription name, VNet CIDR, team name)
├── outputs.tf       # Values exported for other modules to consume
└── terraform.tfvars # Actual values for this specific landing zone
```

### Remote State in ALZ -- Why It Matters

In ALZ, multiple pipelines may run simultaneously.  
The networking pipeline, the policy pipeline, and the landing zone pipeline  
all need to read each other's outputs.

Remote state stored in an Azure Storage Account allows this:

```
Networking pipeline runs
        |
        v
Hub VNet created -- firewall private IP written to remote state
        |
Landing zone pipeline runs later
        |
        v
Reads hub firewall IP from remote state
Uses it to configure the UDR in the spoke
```

Without remote state, pipelines cannot share outputs safely.  
In ALZ Terraform, the state file lives in a dedicated Storage Account  
in the Management subscription.

### Terraform State File in Azure

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
The platform infrastructure (hub network, management groups, policies)  
has its own separate state file.  
Never share one state file across the entire ALZ -- blast radius is too large.

---

## Section 3 -- AVM Terraform Modules

### What AVM Looks Like in Terraform

AVM modules for Terraform are published to the Terraform Registry.  
Every module follows a consistent naming convention:

```
azure/RESOURCE-TYPE/azurerm
```

Examples:
- `azure/virtual-network/azurerm` -- deploys a VNet
- `azure/firewall/azurerm` -- deploys Azure Firewall
- `azure/role-assignment/azurerm` -- deploys an RBAC assignment
- `azure/management-group/azurerm` -- deploys a Management Group

### How You Call an AVM Module in Terraform

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
    pe_subnet = {
      name             = "snet-pe"
      address_prefixes = ["10.1.0.128/27"]
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

### What AVM Gives You That Manual Terraform Does Not

```
Manual Terraform module (written from scratch):
├── Deploys VNet -- basic configuration
├── diagnostic_settings -- FORGOTTEN
├── tags -- inconsistent, depends on who wrote it
├── role_assignments -- FORGOTTEN
└── No Microsoft review

AVM Terraform module:
├── Deploys VNet -- best-practice configuration
├── diagnostic_settings -- BUILT IN, just pass the workspace ID
├── tags -- BUILT IN, consistent across all resources
├── role_assignments -- BUILT IN parameter
├── lock -- BUILT IN, resource locking available
└── Microsoft-reviewed and Well-Architected Framework aligned
```

The difference is not what gets deployed -- it is what cannot be forgotten.

---

## Section 4 -- The ALZ Terraform Accelerator

### What It Is

The ALZ Terraform accelerator is Microsoft's official opinionated  
Terraform implementation of the full Enterprise Scale ALZ.

It deploys the entire platform layer:
- Management Group hierarchy
- ALZ policy initiatives and definitions
- Platform subscriptions (Management, Security, Identity, Connectivity)
- Hub networking (Firewall, Gateway, DNS Resolver, Hub VNet)
- RBAC assignments
- Monitoring baseline

It is not a starting point you modify from scratch.  
It is a production-ready deployment you configure through input variables.

### The Repository Structure

```
terraform-azurerm-caf-enterprise-scale/
│
├── main.tf                    # Root module -- orchestrates everything
├── variables.tf               # All input variables with descriptions
│
├── modules/
│   ├── management_groups/     # Management group hierarchy deployment
│   ├── policy_definitions/    # ALZ custom policy definitions
│   ├── policy_assignments/    # Policy initiative assignments at MG level
│   ├── connectivity/          # Hub networking -- Firewall, Gateway, DNS
│   ├── identity/              # Identity subscription resources
│   └── management/            # Management subscription -- Log Analytics, Monitor
│
└── locals.tf                  # Internal logic and computed values
```

### The Minimal Configuration to Deploy a Full ALZ

```hcl
module "enterprise_scale" {
  source  = "Azure/caf-enterprise-scale/azurerm"
  version = "~> 5.0"

  # Required
  default_location = "eastus"
  root_parent_id   = data.azurerm_client_config.current.tenant_id

  # Management Group naming
  root_id   = "contoso"
  root_name = "Contoso"

  # Deploy platform subscriptions
  subscription_id_management   = "xxxx-management-subscription-id"
  subscription_id_connectivity = "xxxx-connectivity-subscription-id"
  subscription_id_identity     = "xxxx-identity-subscription-id"

  # Deploy hub networking
  configure_connectivity_resources = {
    settings = {
      hub_networks = [{
        config = {
          address_space                = ["10.0.0.0/16"]
          location                     = "eastus"
          enable_hub_network_mesh_peering = false
        }
      }]
      azure_firewall = {
        enabled = true
      }
      vpn_gateway = {
        enabled = true
      }
      dns = {
        enabled = true
      }
    }
  }
}
```

That single module call -- with the right variable values --  
deploys the full ALZ platform layer.  
Everything you studied in Days 1 through 5 comes from this.

---

## Section 5 -- The Pipeline Architecture

### GitHub Actions vs Azure DevOps

Both are valid. ALZ supports both. The concepts are identical.  
The syntax differs. Choose based on what the client already uses.

| | GitHub Actions | Azure DevOps |
|---|---|---|
| Config file location | `.github/workflows/` | `azure-pipelines.yml` |
| Trigger | `on: push` or `on: pull_request` | `trigger:` or `pr:` |
| Authentication | OIDC with Managed Identity or Service Principal | Service Connection |
| Best for | Greenfield, GitHub-native teams | Enterprises already on Azure DevOps |

### The ALZ Pipeline Flow (GitHub Actions)

```yaml
name: ALZ Terraform Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  terraform-plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform Init
        run: terraform init
        # Initialises with remote state in Azure Storage

      - name: Terraform Plan
        run: terraform plan -out=tfplan
        # Generates plan -- platform team reviews this output

  terraform-apply:
    needs: terraform-plan
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Terraform Apply
        run: terraform apply tfplan
        # Only runs on merge to main -- not on pull requests
```

### The Branch Strategy That Protects Production

```
feature/new-landing-zone-a3  <-- Developer makes changes here
        |
        | Pull Request opened
        v
main branch (protected)
        |
        | terraform plan runs automatically on PR
        | Platform team reviews plan output
        | Plan shows exactly what will change
        v
        | PR approved and merged
        v
terraform apply runs automatically
        |
        v
Landing zone deployed to production
```

**Why terraform plan on PR is critical:**  
The plan output is the review artefact.  
The platform team does not review code -- they review the plan.  
The plan shows in plain English: "will create VNet 10.1.0.0/24, will create NSG,  
will assign Contributor to App Team A group."  
No surprises in production.

---

## Section 6 -- OIDC Authentication (No Secrets in Pipelines)

### The Old Way (Avoid This)

Store a Service Principal client secret in the pipeline variables.  
Secret expires. Secret gets rotated. Secret gets accidentally logged.  
Every rotation requires updating pipeline secrets manually.

### The Modern ALZ Way -- OIDC

OIDC (OpenID Connect) allows GitHub Actions or Azure DevOps  
to authenticate to Azure using a federated identity -- no stored secret.

```
Pipeline runs
        |
        v
GitHub generates a short-lived token for this specific run
        |
        v
Azure validates the token against the trusted federation config
        |
        v
Pipeline gets a short-lived Azure access token
        |
        v
Terraform uses it to deploy -- token expires after the run
        |
        v
No secret was stored anywhere
        No secret to rotate
        No secret to leak
```

**In ALZ context:**  
The pipeline Service Principal has a federated credential configured  
that trusts tokens from your specific GitHub repository and branch.  
A token from a different repository cannot impersonate your pipeline.

---

## Section 7 -- Landing Zone as Code: The Complete Picture

This is what a single landing zone looks like fully expressed in Terraform:

```
landing-zones/
└── a1-production/
    ├── main.tf
    │   ├── module "subscription"        # Creates and places the subscription
    │   ├── module "resource_groups"     # Creates standard RGs
    │   ├── module "spoke_vnet"          # Spoke VNet + subnets (AVM)
    │   ├── module "vnet_peering"        # Peers spoke to hub (AVM)
    │   ├── module "route_table"         # UDR forcing traffic to Firewall (AVM)
    │   ├── module "nsg_app"             # NSG for app subnet (AVM)
    │   ├── module "nsg_db"              # NSG for DB subnet (AVM)
    │   ├── module "rbac_app_team"       # Contributor for App Team A (AVM)
    │   ├── module "rbac_security_team"  # Reader for Security Team (AVM)
    │   └── module "log_analytics"       # App-level workspace (AVM)
    │
    ├── variables.tf
    │   ├── var.subscription_name
    │   ├── var.vnet_address_space
    │   ├── var.app_team_group_id
    │   └── var.hub_firewall_ip
    │
    ├── outputs.tf
    │   ├── output "subscription_id"
    │   ├── output "spoke_vnet_id"
    │   └── output "log_analytics_workspace_id"
    │
    └── terraform.tfvars
        ├── subscription_name  = "sub-landingzone-a1-prod"
        ├── vnet_address_space = ["10.1.0.0/24"]
        ├── app_team_group_id  = "xxxx-entra-group-id"
        └── hub_firewall_ip    = "10.0.0.4"
```

Every component you studied in Days 1 through 5 has a corresponding  
Terraform module in this structure.  
The pipeline runs this code and the landing zone exists.

---

## Hands-On Tasks -- Do These Today

### Task 1: Open the ALZ Terraform Repository (15 minutes)

1. Go to: [https://github.com/Azure/terraform-azurerm-caf-enterprise-scale](https://github.com/Azure/terraform-azurerm-caf-enterprise-scale)
2. Open `main.tf` in the root
3. Find the `module "enterprise_scale"` block
4. Read the input variables -- how many are required vs optional?
5. Open `variables.tf` and find these three variables:
   - `default_location`
   - `root_id`
   - `subscription_id_management`
6. Read the description field on each -- note how well-documented they are

---

### Task 2: Open an AVM Terraform Module for Virtual Network (15 minutes)

1. Go to: [https://registry.terraform.io/modules/Azure/virtual-network/azurerm](https://registry.terraform.io/modules/Azure/virtual-network/azurerm)
2. Click the **Inputs** tab
3. Find these specific inputs and read their descriptions:
   - `address_space`
   - `subnets`
   - `diagnostic_settings`
   - `tags`
   - `role_assignments`
4. Click the **Examples** tab
5. Open any example and read the full module call
6. Notice: how many lines of code deploy a fully configured VNet vs what you would write manually?

---

### Task 3: Open the AVM Azure Firewall Terraform Module (10 minutes)

1. Go to: [https://registry.terraform.io/modules/Azure/firewall/azurerm](https://registry.terraform.io/modules/Azure/firewall/azurerm)
2. Find these inputs:
   - `virtual_network_name` -- which VNet does it deploy into?
   - `network_rule_collections` -- where do IP/port rules live?
   - `application_rule_collections` -- where do URL/FQDN rules live?
   - `diagnostic_settings` -- note it is built in
3. This is the module that deploys the hub firewall in the Connectivity subscription

---

### Task 4: Read the ALZ Terraform Accelerator Wiki (10 minutes)

1. Go to: [https://github.com/Azure/ALZ-Terraform-Accelerator](https://github.com/Azure/ALZ-Terraform-Accelerator)
2. Read the README overview
3. Look at the prerequisites section -- what does the accelerator need before it can run?
4. Note the phases: Bootstrap, Platform, Landing Zones
5. This is the opinionated end-to-end deployment tool built on top of the CAF module

---

## GitHub and Registry Reference Pages for Day 6

| Resource | URL | What to Use It For |
|---|---|---|
| ALZ Terraform CAF Module | [https://github.com/Azure/terraform-azurerm-caf-enterprise-scale](https://github.com/Azure/terraform-azurerm-caf-enterprise-scale) | Main ALZ Terraform platform deployment module |
| ALZ Terraform Accelerator | [https://github.com/Azure/ALZ-Terraform-Accelerator](https://github.com/Azure/ALZ-Terraform-Accelerator) | End-to-end opinionated ALZ Terraform deployment tool |
| AVM VNet Terraform Module | [https://registry.terraform.io/modules/Azure/virtual-network/azurerm](https://registry.terraform.io/modules/Azure/virtual-network/azurerm) | AVM Virtual Network module for Terraform |
| AVM Firewall Terraform Module | [https://registry.terraform.io/modules/Azure/firewall/azurerm](https://registry.terraform.io/modules/Azure/firewall/azurerm) | AVM Azure Firewall module for Terraform |
| AVM Role Assignment Terraform Module | [https://registry.terraform.io/modules/Azure/authorization/azurerm](https://registry.terraform.io/modules/Azure/authorization/azurerm) | AVM RBAC role assignment module for Terraform |
| Terraform AzureRM Provider Docs | [https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs) | Full AzureRM provider resource documentation |
| ALZ Terraform CAF Documentation | [https://azure.github.io/Azure-Landing-Zones/](https://azure.github.io/Azure-Landing-Zones/) | Official ALZ documentation including Terraform path |
| OIDC Authentication for GitHub Actions | [https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect) | Setting up secretless pipeline authentication |

---

## Self-Test Questions and Correct Answers

---

### Question 1: What is subscription vending and what problem does it solve at enterprise scale?

**Correct Answer:**  
Subscription vending is the automated pipeline process that creates a new,  
fully configured landing zone subscription on demand.  
The problem it solves is consistency and speed at scale.  
Without it, a new team waits days to weeks for a manually configured environment  
that looks different every time depending on who built it.  
With subscription vending, a pull request triggers a pipeline that creates the subscription,  
places it under the correct management group, configures networking, RBAC, and monitoring,  
and delivers a ready landing zone in minutes -- identical every time, fully auditable.  
The code is the documentation. The pipeline history is the audit trail.

---

### Question 2: Why does ALZ Terraform use remote state stored in Azure Storage instead of local state?

**Correct Answer:**  
Multiple pipelines run in ALZ -- one for platform infrastructure,  
one for networking, one per landing zone.  
These pipelines need to share outputs with each other.  
The networking pipeline creates the hub firewall and writes its private IP to remote state.  
The landing zone pipeline reads that IP from remote state to configure the UDR.  
Without remote state, pipelines cannot safely share outputs.  
Remote state also prevents two pipeline runs from modifying the same infrastructure  
simultaneously via state locking -- Azure Storage provides this automatically.  
Each landing zone has its own state file key to limit blast radius.

---

### Question 3: What does terraform plan do in the ALZ pipeline and why is it reviewed before apply?

**Correct Answer:**  
Terraform plan generates a detailed preview of every change that will be made  
to the Azure environment if the code is applied.  
In the ALZ pipeline, plan runs automatically when a pull request is opened.  
The platform team reviews the plan output -- not the code itself.  
The plan shows in plain English exactly what will be created, modified, or destroyed.  
This prevents surprises in production. A misconfigured CIDR range or wrong RBAC assignment  
shows in the plan before anything is deployed.  
Apply only runs after the pull request is approved and merged to the main branch.  
The plan is the safety gate between code review and production deployment.

---

### Question 4: What is the difference between calling an AVM Terraform module and writing a raw azurerm resource block?

**Correct Answer:**  
A raw azurerm resource block deploys exactly what you write -- nothing more.  
If you forget diagnostic settings, they are not configured.  
If you forget tags, they are not applied.  
If you forget role assignments, RBAC is not set.  
Every omission is a gap that depends on the engineer remembering.  
An AVM module wraps the azurerm resource with Microsoft-validated defaults.  
Diagnostic settings, tags, role assignments, and resource locking  
are built-in parameters -- you pass the workspace ID and they are configured.  
You cannot forget them because the module interface makes them visible.  
AVM also follows the Well-Architected Framework standards --  
security defaults are on by default, not opt-in.

---

### Question 5: What is OIDC authentication in the context of an ALZ pipeline and why is it better than storing a Service Principal secret?

**Correct Answer:**  
OIDC is a federated authentication mechanism that allows the pipeline  
to authenticate to Azure using a short-lived token generated per pipeline run.  
No client secret is stored in the pipeline variables.  
The old approach stores a Service Principal secret that can expire unexpectedly,  
requires manual rotation, and can accidentally appear in pipeline logs.  
With OIDC, GitHub generates a token for that specific run,  
Azure validates it against the configured federation trust,  
and grants a short-lived access token that expires when the run ends.  
Nothing is stored. Nothing needs rotating. Nothing can be leaked from a secrets store.  
This is the modern ALZ recommended approach for pipeline authentication.

---

### Question 6: A new application team submits a request for a landing zone. Walk through every step from request to handover using the Terraform pipeline.

**Correct Answer:**
```
Step 1: App team submits a pull request
        Adds a new tfvars file defining their landing zone:
        subscription name, VNet CIDR, team group ID

Step 2: Pipeline triggers on pull request
        terraform init -- connects to remote state backend
        terraform plan -- generates full change preview
        Plan posted as PR comment for platform team review

Step 3: Platform team reviews the plan
        Verifies: correct CIDR, correct MG placement,
        correct team group ID, correct resource names

Step 4: PR approved and merged to main

Step 5: terraform apply runs automatically
        Subscription created and placed under Landing Zones MG
        Spoke VNet created with correct subnets (AVM module)
        VNet peered to hub with correct peering settings
        UDR attached to subnets forcing traffic to Firewall
        NSGs created for each subnet
        Contributor assigned to app team Entra group
        Reader assigned to security team
        Log Analytics Workspace created
        Diagnostic settings configured

Step 6: Landing zone handed to app team
        They receive subscription ID and VNet details
        Environment is compliant, monitored, and connected
        from the first minute
```

---

### Question 7: Why does each landing zone have its own Terraform state file instead of one shared state file for the entire ALZ?

**Correct Answer:**  
Blast radius. If one state file covers the entire ALZ and becomes corrupted  
or if a terraform apply goes wrong, every landing zone is at risk.  
Separate state files per landing zone mean a problem in one landing zone  
cannot affect another landing zone's state.  
The platform infrastructure (management groups, policies, hub networking)  
also has its own separate state files for the same reason --  
a failed landing zone deployment cannot corrupt the platform state.  
State locking prevents two pipeline runs from modifying the same state simultaneously --  
this only works correctly when state files are scoped appropriately.  
One state file for everything at enterprise scale is an operational risk  
that no production ALZ deployment should accept.

---

## Day 6 Completion Checklist

- [ ] Subscription vending flow understood end to end
- [ ] Terraform remote state purpose and structure understood
- [ ] AVM Terraform module call syntax read and understood
- [ ] ALZ Terraform CAF module repository opened and explored
- [ ] AVM VNet and Firewall modules opened on Terraform Registry
- [ ] Pipeline flow -- plan on PR, apply on merge -- understood
- [ ] OIDC authentication concept understood
- [ ] All 4 hands-on tasks completed
- [ ] All 7 self-test questions answered without notes before checking
- [ ] Score yourself -- 80 and above means proceed to Day 7

---

## Day 7 Preview -- Writing Real Terraform ALZ Code

Day 7 is where you write actual Terraform code for an ALZ landing zone.  
Not reading someone else's code -- writing your own from scratch  
using AVM modules, understanding every parameter, and being able  
to explain every line to a client or a code reviewer.

Specifically on Day 7:
- Write a spoke VNet using the AVM Terraform module
- Write a UDR and attach it to a subnet
- Write an NSG with correct rules
- Write RBAC assignments using the AVM role assignment module
- Run terraform plan locally and read the output

Day 7 requires Terraform installed locally.  
If you do not have it installed yet -- install it tonight.  
Download from: https://developer.hashicorp.com/terraform/install

---

*Day 6 of 21 | Azure Landing Zone Study Track -- Terraform Track | March 2026*