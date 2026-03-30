# ALZ Day 9 -- Full Code Review and Deployment Preparation

> **The single goal of Day 9:**  
> Make your Terraform codebase production-quality and be able to explain  
> every single line before running terraform apply on Day 10.  
> No new concepts today. Only depth, quality, and readiness.
>
> **Hard gate before starting:**  
> Delete main.tf and rewrite it from scratch without referencing any file.  
> Both Day 7 and Day 8 blocks from memory.  
> Do not open Day 9 tasks until this is done.
>
> **Time Required:** 2 hours  
> **Split:** 45 min rewrite from memory | 45 min code review and hardening | 30 min deployment prep

---

## Why Day 9 Has No New Concepts

Days 7 and 8 built the code.  
Day 9 makes it deployable.  
Day 10 runs it against a real subscription.

The gap between "code that plans successfully" and "code ready for production"  
is exactly what Day 9 closes.  
Every architect who has deployed real infrastructure has a story about  
something that planned cleanly and destroyed something unexpected during apply.  
Day 9 prevents that story from being yours on Day 10.

---

## Section 1 -- Rewrite main.tf From Memory (Do This First)

Before any review or hardening -- delete main.tf and write it again.

```bash
cd alz-spoke-day7
rm main.tf
touch main.tf
```

Write all blocks from memory in this order:
1. Resource Group
2. Spoke VNet (AVM module)
3. Route Table (UDR)
4. Route Table Association
5. NSG for app subnet
6. NSG association for app subnet
7. NSG for DB subnet
8. NSG association for DB subnet
9. RBAC assignment (AVM module)
10. Data source for hub VNet
11. VNet peering spoke-to-hub
12. Private DNS Zone
13. Private DNS Zone VNet link
14. Defender for Cloud contact
15. Defender for Cloud pricing
16. Data source for client config

When you finish -- run terraform validate.  
If it passes without looking at any reference file --  
you are ready to deploy on Day 10.  
If it fails -- fix from your own understanding, not from copying the file.

---

## Section 2 -- Code Review Checklist

Go through every file systematically. Fix every gap before Day 10.

### providers.tf Review

- [ ] `required_version` is set -- no version lock = unpredictable behaviour
- [ ] AzureRM provider version is pinned with `~>` constraint
- [ ] `features {}` block exists -- required by AzureRM provider
- [ ] Comment explains that local dev uses Azure CLI, pipeline uses OIDC

### variables.tf Review

- [ ] Every variable has a `description` field -- no exceptions
- [ ] Every variable has a `type` declared -- no untyped variables
- [ ] Sensitive variables (object IDs, IPs) have a comment explaining expected format
- [ ] Default values only where a safe default genuinely exists
- [ ] No hardcoded subscription IDs or tenant IDs anywhere

### terraform.tfvars Review

- [ ] No real secrets or credentials present
- [ ] All placeholder values clearly marked with comments
- [ ] CIDR ranges do not overlap with each other
- [ ] Tags are consistent and complete across all resources

### main.tf Review

- [ ] Every module has a version pin
- [ ] Every resource has `tags = var.tags` -- no untagged resources
- [ ] Resource names follow a consistent naming convention
- [ ] All subnet CIDRs are slices of the VNet address space -- no overlaps
- [ ] UDR uses 0.0.0.0/0 -- not a specific range
- [ ] NSG DB subnet uses `var.app_subnet_prefix` -- not `"VirtualNetwork"`
- [ ] NSG deny-all rule is at priority 4096
- [ ] NSG allow rules start at 100 and increment by 100
- [ ] `registration_enabled = false` on all private DNS zone links
- [ ] Data sources have comments explaining what they reference
- [ ] `use_remote_gateways = false` has a comment: "set to true in production with gateway"
- [ ] RBAC scope uses data source -- not hardcoded subscription ID

### outputs.tf Review

- [ ] Every output has a `description`
- [ ] `spoke_vnet_id` is exported -- platform team needs this for hub peering
- [ ] `app_subnet_id` and `db_subnet_id` are exported -- needed for future private endpoints
- [ ] `private_dns_zone_id` is exported -- needed when creating private endpoints

---

## Section 3 -- Naming Convention Audit

In ALZ every resource name follows a consistent pattern.  
Inconsistent names create confusion during troubleshooting and client audits.

**The recommended naming pattern for this landing zone:**

| Resource Type | Pattern | Example |
|---|---|---|
| Resource Group | `rg-lz-{lz-name}-{env}` | `rg-lz-a1-prod` |
| Virtual Network | `vnet-lz-{lz-name}-{env}` | `vnet-lz-a1-prod` |
| Subnet | `snet-{tier}-{lz-name}` | `snet-app-a1` |
| Route Table | `rt-snet-{tier}-{lz-name}` | `rt-snet-app-a1` |
| NSG | `nsg-snet-{tier}-{lz-name}` | `nsg-snet-app-a1` |
| Private DNS Zone | Use the exact Azure zone name | `privatelink.blob.core.windows.net` |
| DNS Zone Link | `link-{lz-name}-to-{zone-short}` | `link-a1-to-blob` |
| VNet Peering | `peer-spoke-{lz-name}-to-hub` | `peer-spoke-a1-to-hub` |

Check every resource name in your main.tf against this pattern.  
Fix any that do not follow it.

---

## Section 4 -- Tag Audit

Every resource in a production ALZ must be tagged.  
Tags are how the platform team attributes costs, enforces compliance, and filters resources.

**Minimum required tags for every resource:**

```hcl
tags = {
  managed-by    = "terraform"       # How this was deployed
  environment   = "production"      # dev / staging / production
  landing-zone  = "a1"              # Which landing zone
  team          = "app-team-a"      # Which team owns this
  cost-centre   = "CC-1001"         # For billing chargeback
}
```

**Verify in your code:**
- `var.tags` is passed to every resource and module
- No resource is missing `tags = var.tags`
- Your terraform.tfvars has all five tag keys populated

In ALZ a Modify policy auto-adds missing tags at apply time.  
But relying on policy to fix your own code is not acceptable.  
Write the tags correctly in the first place.

---

## Section 5 -- Pre-Deployment Verification

Run these commands in sequence and verify each one before Day 10.

### Step 1: Clean Init

```bash
rm -rf .terraform .terraform.lock.hcl
terraform init
```

A clean init ensures no cached provider versions cause issues.  
Watch for: any warnings about provider version constraints.

### Step 2: Format Check

```bash
terraform fmt -check -recursive
```

If this returns any files -- run `terraform fmt -recursive` to auto-fix.  
Unformatted code in a client engagement signals carelessness.

### Step 3: Validate

```bash
terraform validate
```

Must return: `Success! The configuration is valid.`  
Fix every error before proceeding.

### Step 4: Final Plan

```bash
terraform plan -out=day9.tfplan
```

Read the entire plan output one more time.  
For every resource with a `+` sign ask:
- Do I know why this resource is being created?
- Do I know what ALZ concept it implements?
- Am I comfortable with this being deployed to a real subscription?

If the answer to all three is yes for every resource -- you are ready for Day 10.

### Step 5: Plan Summary Count

```bash
terraform show -json day9.tfplan | python3 -c "
import json, sys
plan = json.load(sys.stdin)
changes = plan.get('resource_changes', [])
adds = [c for c in changes if 'create' in c['change']['actions']]
print(f'Resources to create: {len(adds)}')
for r in adds:
    print(f'  + {r[\"address\"]}')
"
```

This prints every resource that will be created.  
Read the list. Know every item on it.

---

## Section 6 -- What terraform apply Will Actually Do

On Day 10 when you run terraform apply, these things happen to your real subscription:

```
Resources created in your Azure subscription:
├── 1 Resource Group created
├── 1 Virtual Network created with 3 subnets
├── 1 Route Table created and associated to app subnet
├── 2 NSGs created and associated to app and DB subnets
├── 1 RBAC role assignment created
├── 1 VNet peering created (spoke to hub direction)
├── 1 Private DNS Zone created
├── 1 Private DNS Zone VNet link created
├── 1 Defender for Cloud contact configured
└── 1 Defender for Cloud pricing tier set

Cost impact on your subscription:
- Virtual Network: free
- NSGs: free
- Route Table: free
- RBAC assignment: free
- Private DNS Zone: ~$0.50/month for the zone + $0.40 per million queries
- Defender for Cloud (Free tier): no cost
- VNet Peering: charged per GB of data transferred
  (minimal cost for learning -- a few cents at most)
```

Nothing in this deployment will create significant cost.  
The most expensive item would be if you had a real VPN Gateway or Firewall --  
you do not, so cost is negligible.

---

## Section 7 -- Writing a Module README

Every Terraform module in a production ALZ has a README.  
Write one for your landing zone module today.  
It does not need to be long. It needs to answer four questions:

```markdown
# ALZ Landing Zone Spoke Module

## What This Module Deploys
A single Azure Landing Zone spoke including VNet, subnets, NSGs, UDRs,
VNet peering to hub, Private DNS Zone linkage, RBAC, and Defender for Cloud.

## Prerequisites
- Hub VNet must exist (platform team deploys this first)
- Hub firewall private IP must be known
- App team Entra ID group must exist

## Usage
```hcl
module "landing_zone_a1" {
  source = "./alz-spoke-day7"

  landing_zone_name        = "a1"
  vnet_address_space       = ["10.1.0.0/24"]
  app_subnet_prefix        = "10.1.0.0/26"
  db_subnet_prefix         = "10.1.0.64/26"
  pe_subnet_prefix         = "10.1.0.128/27"
  hub_firewall_private_ip  = "10.0.0.4"
  hub_vnet_name            = "vnet-hub-prod"
  hub_resource_group_name  = "rg-connectivity-prod"
  app_team_group_object_id = "xxxx-group-object-id"
}
```

## Resources Created
- azurerm_resource_group
- azurerm_virtual_network (AVM)
- azurerm_subnet x3 (app, db, pe)
- azurerm_route_table + association
- azurerm_network_security_group x2 + associations
- azurerm_role_assignment (AVM)
- azurerm_virtual_network_peering
- azurerm_private_dns_zone
- azurerm_private_dns_zone_virtual_network_link
- azurerm_security_center_contact
- azurerm_security_center_subscription_pricing

## Outputs
- spoke_vnet_id -- used by platform team for hub-to-spoke peering
- app_subnet_id -- used when creating private endpoints
- db_subnet_id -- used when creating private endpoints
- private_dns_zone_id -- used when creating private endpoints
```

Write this README in your alz-spoke-day7 directory.

---

## Day 9 Completion Gate

You are ready for Day 10 when you can answer YES to all of these:

- [ ] main.tf rewritten from scratch without referencing any file
- [ ] terraform validate passes on the rewritten code
- [ ] All checklist items in Section 2 are ticked
- [ ] All resources follow the naming convention in Section 3
- [ ] All resources have correct tags in Section 4
- [ ] terraform fmt -check passes -- no formatting issues
- [ ] terraform plan runs cleanly -- final count matches expectation
- [ ] Every resource in the plan explained from memory
- [ ] Module README written
- [ ] You can explain the entire landing zone to someone in 5 minutes without notes

If you cannot tick every box -- do not proceed to Day 10.  
Running apply against a subscription you do not fully understand is how  
you create infrastructure debt that takes weeks to untangle.

---

## Day 10 Preview -- First Real Deployment

Day 10 is terraform apply against your real Azure subscription.

What will happen:
- All resources deployed to Azure
- Portal verification -- every resource visible and correctly configured
- NSG rules verified -- test that app subnet can reach DB, web cannot
- DNS verification -- confirm private DNS zone is linked and resolving
- Defender for Cloud -- check Secure Score appears for your subscription
- Cost check -- verify no unexpected charges were created

Day 10 is also where things go wrong for the first time.  
Something will not deploy as expected.  
That troubleshooting session is the most valuable learning of the entire track.  
Come prepared -- know your code, know the expected resources,  
know what each error message likely means.

---

*Day 9 of 21 | Azure Landing Zone Study Track -- Terraform Track | March 2026*
