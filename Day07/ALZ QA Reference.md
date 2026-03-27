# ALZ Day 7 -- Concepts, Q&A, and Correct Answers

> This file contains every concept covered in Day 7 and the full Q&A session.  
> Day 7 was the first hands-on Terraform code writing day.  
> Key lesson: referencing without understanding produces code you cannot explain.  
> Day 8 target: write more independently, reference less.

---

## Core Concepts Covered in Day 7

---

### Concept 1 -- The Five Files of a Terraform Landing Zone Module

```
alz-spoke-day7/
├── providers.tf     # Provider configuration and authentication
├── variables.tf     # Input parameters with types and descriptions
├── terraform.tfvars # Actual values for this specific landing zone
├── main.tf          # All AVM module calls and resource blocks
└── outputs.tf       # Values exported for other modules to consume
```

**Why this structure:**  
Separation of concerns. Variables define the interface.  
tfvars provides the values. main.tf contains the logic.  
Outputs expose what other modules need.  
The same main.tf deploys any landing zone by swapping the tfvars file.

---

### Concept 2 -- UDR Default Route (0.0.0.0/0)

**What it does:**  
0.0.0.0/0 is the default route -- it matches every possible destination address.  
Every packet leaving the subnet, regardless of destination, is forced through the Azure Firewall.

**Why NOT 10.0.0.0/8:**  
10.0.0.0/8 only matches private IP ranges.  
Internet-bound traffic would bypass the firewall entirely.  
That is a critical security gap -- internet traffic never gets inspected.

**The rule:**  
0.0.0.0/0 ensures nothing escapes firewall inspection.  
Traffic to internet, to other subnets, to on-premises -- all hits the firewall first.

```hcl
route {
  name                   = "default-to-firewall"
  address_prefix         = "0.0.0.0/0"        # Catch ALL traffic
  next_hop_type          = "VirtualAppliance"  # Route to a specific IP
  next_hop_in_ip_address = var.hub_firewall_private_ip
}
```

---

### Concept 3 -- NSG Source Address: VirtualNetwork vs Subnet CIDR

**App subnet NSG uses `"VirtualNetwork"`:**  
Allows traffic from any resource in the VNet or peered VNets.  
Acceptable for app tier -- needs to receive traffic from multiple sources.

**DB subnet NSG uses `var.app_subnet_prefix`:**  
Allows traffic ONLY from the exact app subnet CIDR -- nothing else.

```
"VirtualNetwork" allows:
├── App subnet VMs     --> DB  (correct)
├── Web subnet VMs     --> DB  (WRONG)
├── Management VMs     --> DB  (WRONG)
└── Peered VNet VMs    --> DB  (WRONG)

var.app_subnet_prefix allows:
└── App subnet VMs only --> DB  (correct -- only what is needed)
```

**Why this matters:**  
DB contains the most sensitive data.  
Principle of least privilege at the network rule level.  
Even if another VM is compromised it cannot reach the database.

---

### Concept 4 -- Variable Descriptions

**Why descriptions are mandatory, not optional:**

Three reasons:
1. Documentation that lives with the code -- future engineers read it, not you
2. terraform plan output is human-readable -- reviewers understand what they are approving
3. Client handover -- undocumented variables are a support burden on their team

**The rule:** Every variable without a description is a question someone will ask later. Write the description now, answer the question never.

```hcl
variable "hub_firewall_private_ip" {
  description = "Private IP of the Azure Firewall in the hub -- used for UDR next hop"
  type        = string
  # Without description: what does this value mean? What format? Who knows?
  # With description: self-explanatory, no questions needed
}
```

---

### Concept 5 -- NSG Priority Numbers

**How priorities work:**  
Lower number = evaluated first.  
Priority 100 fires before priority 4096.

**Why deny-all goes at 4096 specifically:**  
4096 is the highest assignable priority in Azure NSGs.  
Placing deny-all here leaves priorities 101 through 4095 free for future allow rules.  
Nearly 4000 slots for rules you have not written yet.

**Why allow rules increment by 100:**  
100, 200, 300 -- not 101, 102, 103.  
Incrementing by 100 leaves room to insert rules between existing ones  
without renumbering everything.

```
Priority 100  --> Allow HTTPS inbound from VNet
Priority 200  --> (future rule slot)
Priority 300  --> (future rule slot)
...
Priority 4096 --> Deny all inbound (always last)
```

---

### Concept 6 -- Data Sources vs Hardcoded Values

**What a data source is:**  
A Terraform block that reads existing information from Azure  
without creating anything.

**`data.azurerm_client_config.current`:**  
Reads the authentication context of the currently logged-in identity.  
Returns: subscription_id, tenant_id, object_id.

**Why use it instead of hardcoding:**

| Hardcoded | Data Source |
|---|---|
| Wrong GUID = silent error until apply | Always correct for current context |
| Breaks when deployed to different subscription | Automatically uses current subscription |
| Must be updated manually per environment | Works everywhere without modification |

**The broader principle:**  
Never hardcode values that Azure can tell you dynamically.  
Subscription IDs, tenant IDs, resource IDs of existing resources --  
all come from data sources or variables, never copy-pasted GUIDs.

```hcl
# Wrong
scope = "/subscriptions/xxxx-1234-xxxx-1234-xxxxxxxxxxxx"

# Right
scope = "/subscriptions/${data.azurerm_client_config.current.subscription_id}"
```

---

### Concept 7 -- Module Version Pinning

**Why `version = "~> 1.0"` and not latest:**

`~> 1.0` means: allow 1.0, 1.1, 1.2 -- block 2.0 and above.  
Minor versions add features without breaking changes -- allowed.  
Major versions may introduce breaking changes -- blocked.

**Without pinning:**  
terraform init downloads latest version every time.  
A breaking change in a module release breaks your production pipeline  
without you changing any code.  
Version pinning makes deployments reproducible.

---

### Concept 8 -- The 12 Resources in terraform plan

Every resource maps to a specific ALZ concept.  
Memorise this table -- it is what you say when a client asks  
"what does your Terraform actually create?"

| Resource | ALZ Concept |
|---|---|
| `azurerm_resource_group.spoke` | Landing zone container |
| `azurerm_virtual_network` (VNet module) | Spoke network -- hub-spoke topology |
| `azurerm_subnet` app tier | Workload isolation |
| `azurerm_subnet` DB tier | Least privilege -- DB isolated from other tiers |
| `azurerm_subnet` PE tier | Private endpoint landing -- PaaS off internet |
| `azurerm_route_table` | UDR -- forces traffic through Azure Firewall |
| `azurerm_subnet_route_table_association` | Attaches UDR to app subnet |
| `azurerm_network_security_group` app | Layer 4 subnet filter -- first inspection point |
| `azurerm_network_security_group` DB | Least privilege -- only app subnet reaches DB |
| `azurerm_subnet_nsg_association` app | Attaches NSG to app subnet |
| `azurerm_subnet_nsg_association` DB | Attaches NSG to DB subnet |
| `azurerm_role_assignment` (RBAC module) | Contributor for app team -- subscription vending access |

---

## Day 7 Q&A Session -- Questions and Correct Answers

---

### Q1: Why does the UDR use 0.0.0.0/0 and what happens if you use 10.0.0.0/8?

**Your answer:** Wrong -- said it was about public internet access.  
**Score: 0%**

**Correct Answer:**  
0.0.0.0/0 is the default route -- it matches every possible destination.  
It forces ALL outbound traffic through the Azure Firewall regardless of destination.  
If changed to 10.0.0.0/8, only traffic to private IP ranges would hit the firewall.  
Internet-bound traffic would bypass the firewall entirely -- a critical security gap.  
The firewall would never see a VM calling an external API or a malicious domain.  
0.0.0.0/0 ensures nothing escapes inspection. That is its entire purpose.

---

### Q2: Why use `var.app_subnet_prefix` as source in the DB NSG instead of `"VirtualNetwork"`?

**Your answer:** Did not know.  
**Score: 0%**

**Correct Answer:**  
`"VirtualNetwork"` allows traffic from any resource in the VNet or peered VNets --  
web tier, management VMs, other landing zones. Too broad for a database.  
`var.app_subnet_prefix` pins the source to exactly the app subnet CIDR only.  
Nothing else in the network can reach the DB subnet regardless of what it tries.  
This is the principle of least privilege applied at the network rule level.  
The DB is the most sensitive resource -- it gets the most specific rule.

---

### Q3: Why write descriptions on variables when Terraform runs fine without them?

**Your answer:** Partially right -- said descriptions align input values.  
**Score: 50%**

**Correct Answer:**  
Descriptions serve three purposes:  
First -- documentation that lives with the code.  
Future engineers read the description to understand what value to pass.  
Second -- terraform plan output is human-readable.  
Reviewers understand what each variable means without opening the code.  
Third -- client handover.  
Undocumented variables are a support burden on the client's team.  
Every variable without a description is a question someone will ask later.  
Write it now, answer the question never.

---

### Q4: Why does another module need the spoke_vnet_id output?

**Your answer:** Correct -- VNet peering needs the ID.  
**Score: 85%**

**Correct Answer:**  
The spoke VNet ID is consumed by:  
VNet peering module -- to peer the spoke to the hub using the resource ID.  
Private DNS Zone link -- to link the DNS zone to this VNet for private name resolution.  
Diagnostic settings -- to reference this VNet when configuring log forwarding.  
Other spoke peering -- if two spokes need controlled connectivity.  
The VNet ID is the primary reference other modules use to interact with this network.

---

### Q5: Why is the deny-all rule at priority 4096 and not 200 or 500?

**Your answer:** Correct on priority ordering -- did not explain why 4096 specifically.  
**Score: 60%**

**Correct Answer:**  
4096 is the highest assignable priority number in Azure NSGs.  
Placing deny-all here leaves priorities 101 through 4095 free for future allow rules.  
Nearly 4000 slots for rules not yet written.  
If deny-all was at 200, only 99 slots would exist between the allow rules and the deny.  
In a real landing zone over time new rules are added --  
monitoring agents, backup traffic, specific application ports.  
Running out of priority space means rewriting the entire NSG to make room.  
4096 for deny-all is a design decision that keeps the NSG maintainable indefinitely.  
Allow rules increment by 100 -- 100, 200, 300 -- for the same reason:  
room to insert between existing rules without renumbering.

---

### Q6: What is `data.azurerm_client_config.current` and why use it instead of hardcoding?

**Your answer:** Directionally right -- "importing from Azure" imprecise.  
**Score: 55%**

**Correct Answer:**  
It is a Terraform data source -- a block that reads existing information from Azure  
without creating anything.  
This specific one reads the authentication context of the currently logged-in identity:  
subscription ID, tenant ID, and object ID.  
Using it instead of hardcoding solves three problems:  
First -- a wrong character in a hardcoded GUID is silent until apply fails.  
Second -- hardcoded IDs break when the same code runs against a different subscription.  
Third -- data source makes the code portable -- works for any subscription  
without modification.  
Never hardcode values Azure can tell you dynamically.

---

### Q7: Name all 12 resources from terraform plan and what ALZ concept each implements.

**Your answer:** Initially vague -- then named all 12 correctly from memory after prompt.  
**Score: 70%**

**Correct Answer:** See Concept 8 table above.  
All 12 resources named and mapped to ALZ concepts.  
The shift from "and all" to precise recall happened in one attempt.  
That is the standard. Maintain it on Day 8.

---

## Day 7 Final Score: 60 / 100

| Question | Score | Key Gap |
|---|---|---|
| Q1 -- UDR 0.0.0.0/0 | 0% | Fundamental misunderstanding of default route |
| Q2 -- DB NSG source prefix | 0% | Did not know -- least privilege at network level |
| Q3 -- Variable descriptions | 50% | Right direction, missing three specific reasons |
| Q4 -- spoke_vnet_id output | 85% | Peering correct, other consumers missing |
| Q5 -- NSG priority 4096 | 60% | Priority order correct, 4096 reasoning missing |
| Q6 -- data source vs hardcode | 55% | Direction right, precise mechanism missing |
| Q7 -- 12 resources from memory | 70% | Named all 12 after one prompt -- correct |

---

## The Honest Assessment

60 is the score. The reason is referencing heavily while writing.  
Code you transcribe without understanding produces zero answers in a Q&A.  
Day 8 has a specific instruction: write more independently.  
Reference the file only to check, not to write.  
The gap between 60 and 80 is understanding what you type.

---

## Score Trend

| Day | Score | Status |
|---|---|---|
| Day 2 | 82% | Strong start |
| Day 3 | 71% | Networking gaps |
| Day 4 | 59% | Identity gaps -- lowest point |
| Day 5 | 79% | Recovery |
| Day 6 | 72% | Steady |
| Day 7 | 60% | Referencing gap exposed |

**Target for Day 8: 75 or above.**  
Write the code yourself. Understand every line before moving to the next.

---

## Before Day 8 -- One Exercise

Open your main.tf right now.  
Read every line.  
For each line ask: why does this exist?  
If you cannot answer -- write the answer in a comment above that line.  
Do not start Day 8 until every line in main.tf has an answer.

---

## Day 8 Preview -- Hub Peering, DNS, and Full Network Flow

- Add VNet peering from spoke to hub with all three settings
- Add Private DNS Zone and link to spoke VNet
- Add basic Defender for Cloud configuration
- Run terraform plan on the extended code
- Trace the full network flow from your code to Azure

Day 8 connects your spoke to the hub.  
Without peering your landing zone is an island.  
Day 8 makes it part of the ALZ.

---

*Day 7 of 21 | Azure Landing Zone Study Track -- Terraform Track | March 2026*