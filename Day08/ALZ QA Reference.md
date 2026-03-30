# ALZ Day 8 -- Concepts, Q&A, and Correct Answers

> Day 8 score: 78/100 -- solid improvement from Day 7's 60.  
> Key gaps: DNS resolver precision, registration_enabled full consequence.  
> Before Day 10: delete main.tf and rewrite from scratch without referencing.

---

## Core Concepts Covered in Day 8

---

### Concept 1 -- VNet Peering Requires Two Resources

Peering is not automatically bidirectional.  
Two resources required -- one per direction.

```
spoke-to-hub peering:
Lives in: Landing zone Terraform code
Owned by: Landing zone pipeline
Settings: allow_forwarded_traffic = true, use_remote_gateways = true (prod)

hub-to-spoke peering:
Lives in: Platform connectivity Terraform code
Owned by: Platform team pipeline
Settings: allow_gateway_transit = true
Created after: spoke VNet ID is available from landing zone outputs
```

**The dependency chain:**  
Landing zone pipeline runs -- outputs spoke_vnet_id  
Platform pipeline reads spoke_vnet_id from remote state  
Creates hub-to-spoke peering using that ID  
Two pipelines. Two state files. One connected network.

---

### Concept 2 -- Deployment Order in ALZ

Data sources execute during terraform plan -- not just apply.  
If the referenced resource does not exist, plan fails immediately.

```
Correct deployment order:
1. Management Groups
2. Platform subscriptions (Management, Security, Identity, Connectivity)
3. Hub network (Firewall, Gateway, DNS Resolver, Hub VNet)
4. Landing zone spokes (data source can now find hub VNet)
5. Hub-to-spoke peering (platform pipeline reads spoke VNet ID)
```

Deploying Step 4 before Step 3 = data source fails = deployment stops.  
This is why the ALZ Terraform Accelerator runs bootstrap phases in strict order.

---

### Concept 3 -- registration_enabled on Private DNS Zone Links

**Always false for private endpoint zones.**

`registration_enabled = true` causes VMs to auto-register their hostnames in the zone.  
Private endpoint zones like `privatelink.blob.core.windows.net` must never have this enabled.

Two consequences if set to true:
1. Azure blocks it -- deployment fails with an error during apply
2. Even if allowed -- VM hostnames pollute the zone meant only for private endpoint A records

**The rule:**  
`privatelink.*` zones = `registration_enabled = false` always.  
Custom internal zones like `internal.company.com` = `registration_enabled = true` for VM auto-registration.

---

### Concept 4 -- Private DNS Zone VNet Link Purpose

`azurerm_private_dns_zone_virtual_network_link` does NOT pass traffic.  
It tells the spoke VNet to consult this Private DNS Zone when resolving names.  
Traffic routing is handled by UDR and Azure Firewall separately.  
The link only affects name resolution -- which IP gets returned for a hostname query.

---

### Concept 5 -- The Complete Private Endpoint DNS Flow

```
VM queries: mystorageaccount.blob.core.windows.net
        |
        v
DNS query goes to Azure DNS Private Resolver
inbound endpoint in the hub
(In personal subscription: Azure DNS directly via zone VNet link)
        |
        v
Resolver checks Private DNS Zone: privatelink.blob.core.windows.net
Record found: mystorageaccount --> 10.1.0.140 (private endpoint IP)
        |
        v
VM sends traffic to 10.1.0.140
UDR matches 0.0.0.0/0 -- routes to Azure Firewall
        |
        v
Firewall evaluates -- allows private endpoint traffic
        |
        v
Traffic reaches storage account via Private Endpoint
Public internet never involved at any point
```

**The critical distinction:**  
Production ALZ uses Azure DNS Private Resolver in the hub as the central DNS point.  
Generic Azure DNS (168.63.129.16) does not know about Private DNS Zones  
unless the resolver is configured and the zone is linked.

---

### Concept 6 -- Defender for Cloud in ALZ Context

In production ALZ: enabled at Management Group level via DINE policy.  
Every new subscription automatically gets Defender configured.  
Platform team never touches individual subscriptions.

In personal subscription: configured directly at subscription level  
using `azurerm_security_center_contact` and `azurerm_security_center_subscription_pricing`.  
This is for learning purposes only -- not the production pattern.

---

## Day 8 Q&A -- Questions and Correct Answers

---

### Q1: Why only spoke-to-hub peering in Day 8 code? Where does hub-to-spoke live in production?

**Score: 95%**

**Correct Answer:**  
Hub-to-spoke peering lives in the platform team's connectivity Terraform code --  
a completely separate state file from the landing zone.  
The platform team pipeline creates it after reading the spoke VNet ID  
from the landing zone pipeline's remote state output.  
In a personal subscription without a real hub, only the spoke-to-hub resource was written.  
In production both exist -- different code, different pipelines, different owners.

---

### Q2: What happens if registration_enabled = true on a privatelink DNS zone?

**Score: 65%**

**Correct Answer:**  
First -- Azure blocks it at apply time with a deployment error.  
Private endpoint DNS zones cannot have auto-registration enabled -- Azure enforces this.  
Second -- even conceptually, VMs would register hostnames like `vm-app-001`  
into `privatelink.blob.core.windows.net` which is only meant for  
private endpoint A records mapping storage account names to private IPs.  
This would break private endpoint DNS resolution entirely.  
The rule: all `privatelink.*` zones always have `registration_enabled = false`.

---

### Q3: What happens when terraform plan runs and the hub VNet does not exist yet?

**Score: 90%**

**Correct Answer:**  
Terraform plan fails immediately with a resource not found error.  
Data sources execute during plan -- not just during apply.  
This enforces deployment sequencing -- hub network must exist before landing zone spokes.  
In the ALZ Terraform Accelerator this is handled by running bootstrap phases in order.  
In a personal subscription without a hub, placeholder values or  
commenting out the data source temporarily is the workaround for Day 10.

---

### Q4: Three new Day 8 resources and their ALZ concepts

**Score: 85%**

**Correct Answer:**

`azurerm_virtual_network_peering` -- Hub-spoke topology connection.  
Connects the spoke to the hub so traffic can flow through the central firewall  
and reach on-premises via the hub gateway.

`azurerm_private_dns_zone_virtual_network_link` -- Private DNS resolution for the spoke.  
Tells the spoke VNet to consult the Private DNS Zone for name resolution.  
Does not pass traffic -- only affects which IP address is returned for a hostname query.

`azurerm_security_center_contact` -- Defender for Cloud alert routing.  
Configures where security alerts are sent.  
In production ALZ this is handled by policy at Management Group level automatically.

---

### Q5: Trace the full Flow 2 -- VM to private storage account

**Score: 80%**

**Correct Answer:**
```
VM queries: mystorageaccount.blob.core.windows.net
        |
DNS query goes to Azure DNS Private Resolver
inbound endpoint in the hub
(not generic Azure DNS -- the resolver knows about Private DNS Zones)
        |
Resolver checks Private DNS Zone: privatelink.blob.core.windows.net
Returns PRIVATE IP of the private endpoint (e.g. 10.1.0.140)
NOT the public IP
        |
VM sends traffic to 10.1.0.140
UDR on subnet matches 0.0.0.0/0 -- routes to Azure Firewall
        |
Firewall evaluates rules -- allows private endpoint traffic
        |
Traffic reaches storage account via Private Endpoint
Public internet never involved
```

Gap: said "Azure DNS" -- correct answer is "Azure DNS Private Resolver in the hub."

---

## Day 8 Final Score: 78 / 100

| Question | Score | Key Gap |
|---|---|---|
| Q1 -- Peering ownership | 95% | Nearly perfect |
| Q2 -- registration_enabled consequence | 65% | Azure blocks it -- deployment fails |
| Q3 -- Data source before resource exists | 90% | Correct with good sequencing context |
| Q4 -- Three new Day 8 resources | 85% | DNS link purpose imprecise |
| Q5 -- Full DNS to storage flow | 80% | DNS Private Resolver vs generic Azure DNS |

---

## Score Trend

| Day | Score | Status |
|---|---|---|
| Day 2 | 82% | Strong start |
| Day 3 | 71% | Networking gaps |
| Day 4 | 59% | Lowest point |
| Day 5 | 79% | Recovery |
| Day 6 | 72% | Steady |
| Day 7 | 60% | Referencing gap exposed |
| Day 8 | 78% | Strong recovery |

---

## Before Day 9 -- Non-Negotiable

Delete main.tf and rewrite it from scratch without referencing any file.  
Both Day 7 and Day 8 blocks.  
If you cannot write it from memory after writing it twice --  
you do not understand it well enough to deploy it on Day 10.  
This is the preparation Finland requires.

---

*Day 8 of 21 | Azure Landing Zone Study Track -- Terraform Track | March 2026*
