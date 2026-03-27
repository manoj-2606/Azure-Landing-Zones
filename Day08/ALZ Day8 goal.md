# ALZ Day 8 -- Hub Peering, Private DNS, and Full Network Flow

> **The single goal of Day 8:**  
> Connect your Day 7 landing zone spoke to the hub --  
> add VNet peering with all three settings, Private DNS Zone linkage,  
> and Defender for Cloud configuration --  
> then trace the complete network flow from a VM request  
> to its destination end to end.
>
> **Hard rule for Day 8:**  
> Write every block yourself first.  
> Reference the file only to check after you have written it.  
> If you reference before writing, you are practising transcription not understanding.
>
> **Time Required:** 2.5 hours  
> **Split:** 20 min review Day 7 code | 80 min writing new blocks | 40 min tracing flow

---

## What You Are Adding to Your Day 7 Code

Your Day 7 code created an isolated spoke. It has a VNet, subnets, NSGs, UDRs, and RBAC.  
But it is disconnected. No path to the hub. No DNS. No security posture visibility.

Day 8 fixes that.

```
Day 7 Landing Zone (isolated):
[Spoke VNet] -- no connection to anything

Day 8 Landing Zone (connected):
[Hub VNet] <--peering--> [Spoke VNet]
[Private DNS Zone] <--linked--> [Spoke VNet]
[Defender for Cloud] -- monitoring spoke subscription
```

---

## Section 1 -- VNet Peering (The Three Settings Revisited)

You learned these on Day 3. Today you write them in code.

### The Three Settings -- Mandatory Review Before Writing

Before writing a single line, answer these from memory:

1. **Allow Gateway Transit** -- enabled on which side? What does it do?
2. **Use Remote Gateways** -- enabled on which side? What does it do?
3. **Allow Forwarded Traffic** -- enabled on which side? What does it do?

If you cannot answer all three without looking -- re-read Day 3 Concept 6 before continuing.

### What Peering Looks Like in Code

VNet peering requires two peering resources -- one from hub to spoke, one from spoke to hub.  
Both directions must exist. One direction alone does not work.

**Spoke to Hub peering (add this to your main.tf):**

```hcl
# Data source to reference the existing hub VNet
# In a real ALZ this hub VNet already exists -- deployed by the platform team
data "azurerm_virtual_network" "hub" {
  name                = var.hub_vnet_name
  resource_group_name = var.hub_resource_group_name
}

# Spoke to Hub peering
resource "azurerm_virtual_network_peering" "spoke_to_hub" {
  name                      = "peer-spoke-${var.landing_zone_name}-to-hub"
  resource_group_name       = azurerm_resource_group.spoke.name
  virtual_network_name      = module.spoke_vnet.name
  remote_virtual_network_id = data.azurerm_virtual_network.hub.id

  # Spoke side settings
  allow_forwarded_traffic   = true   # Allow on-prem traffic forwarded through hub
  use_remote_gateways       = false  # Set to true when hub has a real gateway deployed
  allow_virtual_network_access = true
}
```

**Why `use_remote_gateways = false` for now:**  
In your personal subscription you likely do not have a VPN Gateway or ExpressRoute  
deployed in a hub VNet. Setting this to true without a gateway causes an error.  
In a real ALZ with a hub gateway this is set to true.  
Note this in a comment in your code so you remember to change it for production.

**Hub to Spoke peering:**  
In a real ALZ the platform team owns the hub.  
The hub-to-spoke peering is managed in the platform Terraform code, not the landing zone code.  
For Day 8 on your personal subscription, if you have a hub VNet you can add it.  
If not, the spoke-to-hub resource is sufficient to understand the concept.

---

## Section 2 -- New Variables to Add

Add these to your variables.tf:

```hcl
variable "hub_vnet_name" {
  description = "Name of the hub Virtual Network in the Connectivity subscription"
  type        = string
  # Example: "vnet-hub-connectivity-prod"
}

variable "hub_resource_group_name" {
  description = "Resource group containing the hub Virtual Network"
  type        = string
  # Example: "rg-connectivity-prod"
}

variable "private_dns_zone_name" {
  description = "Name of the Private DNS Zone to link to this spoke"
  type        = string
  default     = "privatelink.blob.core.windows.net"
  # This is the DNS zone for Azure Blob Storage private endpoints
}
```

Add these values to your terraform.tfvars:

```hcl
hub_vnet_name           = "vnet-hub-prod"
hub_resource_group_name = "rg-hub-prod"
# Use placeholder values if you do not have a real hub VNet
# The plan will still show what would be created

private_dns_zone_name = "privatelink.blob.core.windows.net"
```

---

## Section 3 -- Private DNS Zone and VNet Link

### Why This Is Needed (Refresher from Day 3)

Without a Private DNS Zone link:  
VM queries `mystorageaccount.blob.core.windows.net`  
Gets back the PUBLIC IP  
Traffic goes over the internet  
Private endpoint is bypassed  

With a Private DNS Zone link:  
VM queries `mystorageaccount.blob.core.windows.net`  
Resolver checks Private DNS Zone  
Gets back the PRIVATE IP of the private endpoint  
Traffic stays inside Azure  

### The Code (Add to main.tf)

```hcl
# Create the Private DNS Zone
# In a real ALZ this lives in the hub -- linked to all spokes
# For Day 8 we create it in the spoke for learning purposes
resource "azurerm_private_dns_zone" "blob" {
  name                = var.private_dns_zone_name
  resource_group_name = azurerm_resource_group.spoke.name
  tags                = var.tags
}

# Link the Private DNS Zone to the spoke VNet
# This is what makes the spoke VMs use this zone for DNS resolution
resource "azurerm_private_dns_zone_virtual_network_link" "spoke" {
  name                  = "link-${var.landing_zone_name}-to-dns-zone"
  resource_group_name   = azurerm_resource_group.spoke.name
  private_dns_zone_name = azurerm_private_dns_zone.blob.name
  virtual_network_id    = module.spoke_vnet.resource_id

  # registration_enabled = false means VMs do not auto-register their names
  # in this zone -- this zone is for private endpoints only
  registration_enabled  = false

  tags = var.tags
}
```

**Why `registration_enabled = false`:**  
There are two uses for DNS Zone VNet links:  
Auto-registration -- VMs automatically register their hostnames in the zone (used for internal VM DNS).  
Resolution only -- VMs can resolve records in the zone but do not register themselves.  
Private endpoint DNS zones should always be resolution-only.  
You do not want VMs registering in `privatelink.blob.core.windows.net`.

---

## Section 4 -- Defender for Cloud Configuration

### What You Are Adding

In a real ALZ, Defender for Cloud is enabled at the Management Group level via policy.  
For Day 8 on your personal subscription, you configure it at the subscription level  
to understand what it does and how it is expressed in Terraform.

```hcl
# Enable Defender for Cloud contact details
# This is where security alerts are sent
resource "azurerm_security_center_contact" "main" {
  name  = "security-contact"
  email = "security@yourdomain.com"
  phone = "+1-555-000-0000"

  alert_notifications = true
  alerts_to_admins    = true
}

# Enable Defender for Servers plan on the subscription
resource "azurerm_security_center_subscription_pricing" "servers" {
  tier          = "Free"
  resource_type = "VirtualMachines"
  # Use "Free" to avoid charges on your personal subscription
  # In production ALZ this is "Standard" for full protection
}
```

**What this does in the portal:**  
After applying this, open Defender for Cloud in the portal.  
You will see your subscription listed with a Secure Score.  
Security recommendations will appear for your resources.  
This is the visibility the security team gets across all landing zones in ALZ.

---

## Section 5 -- Updated outputs.tf

Add these outputs for the new resources:

```hcl
output "private_dns_zone_id" {
  description = "Resource ID of the Private DNS Zone -- used when creating private endpoints"
  value       = azurerm_private_dns_zone.blob.id
}

output "private_dns_zone_name" {
  description = "Name of the Private DNS Zone"
  value       = azurerm_private_dns_zone.blob.name
}

output "vnet_peering_id" {
  description = "Resource ID of the spoke-to-hub peering"
  value       = azurerm_virtual_network_peering.spoke_to_hub.id
}
```

---

## Section 6 -- The Full Network Flow (Trace This After Plan Succeeds)

After running terraform plan, trace this complete flow for a VM in your landing zone:

### Flow 1: VM to Internet

```
VM in app subnet sends request to api.github.com
        |
        v
NSG on app subnet evaluates: is this allowed outbound?
Default outbound rules allow -- passes
        |
        v
UDR on app subnet matches 0.0.0.0/0
Next hop: Azure Firewall private IP in hub
        |
        v
Traffic crosses VNet peering to hub
(allow_forwarded_traffic = true enables this)
        |
        v
Azure Firewall evaluates application rules
If api.github.com is allowed -- forwards to internet
If blocked -- drops and logs
        |
        v
Response returns through Firewall back to VM
```

### Flow 2: VM to Private Storage Account

```
VM queries DNS: mystorageaccount.blob.core.windows.net
        |
        v
DNS query goes to Azure DNS (168.63.129.16)
In production ALZ: goes to DNS Private Resolver in hub
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
Public internet never involved
```

### Flow 3: VM to On-Premises (When Hub Has Gateway)

```
VM sends traffic to on-premises IP (e.g. 192.168.1.10)
        |
        v
UDR matches 0.0.0.0/0 -- routes to Azure Firewall
        |
        v
Firewall evaluates -- allows on-premises bound traffic
        |
        v
Traffic crosses peering to hub
(use_remote_gateways = true enables hub gateway use)
        |
        v
Hub VPN Gateway or ExpressRoute forwards to on-premises
        |
        v
On-premises server receives request
```

---

## Running Terraform Plan on Day 8 Code

```bash
cd alz-spoke-day7
terraform init    # Re-run to download any new providers needed
terraform validate
terraform plan
```

**Expected new resources in the plan:**

| New Resource | What It Does |
|---|---|
| `azurerm_virtual_network_peering.spoke_to_hub` | Connects spoke to hub |
| `data.azurerm_virtual_network.hub` | Reads existing hub VNet (no + sign -- data source) |
| `azurerm_private_dns_zone.blob` | Creates private DNS zone for storage |
| `azurerm_private_dns_zone_virtual_network_link.spoke` | Links DNS zone to spoke VNet |
| `azurerm_security_center_contact.main` | Defender alert contact |
| `azurerm_security_center_subscription_pricing.servers` | Defender for Servers plan |

Total from Day 7: 12 resources  
New on Day 8: 5-6 resources  
**New total: approximately 17-18 resources to add**

---

## Self-Test Questions and Correct Answers

---

### Question 1: Why does VNet peering require two separate peering resources?

**Correct Answer:**  
VNet peering in Azure is not automatically bidirectional.  
Creating a peering from Spoke to Hub allows the spoke to send traffic to the hub.  
But the hub cannot send traffic back to the spoke without a peering in the other direction.  
Both directions must be explicitly configured.  
In ALZ the spoke-to-hub peering lives in the landing zone Terraform code.  
The hub-to-spoke peering lives in the platform team's connectivity Terraform code.  
Two separate resources. Two separate scopes. Both required.

---

### Question 2: What happens if `allow_forwarded_traffic = false` on the spoke-to-hub peering?

**Correct Answer:**  
Traffic that originates outside the spoke VNet and is forwarded through the hub  
gets dropped at the peering boundary.  
The most critical scenario: on-premises traffic.  
A server on-premises sends a request that travels through the ExpressRoute Gateway,  
through the hub, and should reach the spoke VM.  
With `allow_forwarded_traffic = false`, the traffic is dropped when it tries  
to cross the spoke peering because it did not originate in the hub VNet.  
On-premises to spoke connectivity breaks silently -- the peering shows as Connected  
but traffic never arrives.  
This is the same silent failure pattern as the DINE policy Managed Identity issue.

---

### Question 3: Why is `registration_enabled = false` on the Private DNS Zone VNet link?

**Correct Answer:**  
A VNet link with registration enabled automatically registers the hostnames  
of VMs in that VNet into the DNS zone.  
Private endpoint DNS zones like `privatelink.blob.core.windows.net`  
are for resolving private endpoint records only.  
You do not want VM hostnames like `vm-app-001` registering in  
`privatelink.blob.core.windows.net` -- that is the wrong zone for VM records.  
Registration should only be enabled on zones specifically created for VM auto-registration  
such as a custom internal domain like `internal.company.com`.  
Private endpoint zones are always resolution-only.

---

### Question 4: A VM in your spoke tries to reach a storage account. DNS returns the public IP instead of the private IP. What is wrong?

**Correct Answer:**  
Three possible causes in order of likelihood:  
First -- the Private DNS Zone VNet link is missing or not linked to the spoke VNet.  
The spoke VNet DNS cannot see the private DNS zone records.  
Fix: ensure `azurerm_private_dns_zone_virtual_network_link` exists and points to the spoke VNet.  
Second -- the A record for the storage account does not exist in the Private DNS Zone.  
The zone exists but the private endpoint was not created or its DNS record was not registered.  
Fix: verify the private endpoint exists and its DNS group is configured correctly.  
Third -- in production ALZ, the DNS Private Resolver in the hub is not forwarding  
queries to the Private DNS Zone.  
Fix: check the resolver outbound endpoint forwarding rules.  
The symptom -- public IP returned -- always means the private DNS zone  
is not being consulted for that query.

---

### Question 5: What is the difference between a data source and a resource in Terraform?

**Correct Answer:**  
A resource block creates, updates, or destroys infrastructure in Azure.  
It appears with a `+` sign in terraform plan.  
Terraform owns its lifecycle -- if you delete it from code, Terraform destroys it in Azure.  
A data source reads existing infrastructure that Terraform did not create  
and does not manage.  
It appears without a `+` sign in terraform plan -- it is a read operation only.  
Deleting a data source from code does nothing to the Azure resource.  
In ALZ, data sources are used to reference existing platform resources  
the landing zone code needs but does not own --  
the hub VNet, the central Log Analytics Workspace, the platform resource groups.  
You read their IDs to reference them. You never manage them from landing zone code.

---

## Day 8 Completion Checklist

- [ ] Day 7 main.tf reviewed -- every line has a mental answer before starting
- [ ] Three peering settings recalled from memory before writing peering block
- [ ] New variables added to variables.tf with descriptions
- [ ] Peering block written independently then checked against file
- [ ] Private DNS Zone block written independently then checked
- [ ] DNS Zone VNet link block written independently then checked
- [ ] Defender for Cloud block written independently then checked
- [ ] New outputs added
- [ ] terraform init re-run successfully
- [ ] terraform validate passes
- [ ] terraform plan shows 17-18 resources
- [ ] Every new resource in the plan explained -- what it does, what ALZ concept it implements
- [ ] Three network flows traced end to end from Section 6
- [ ] All 5 self-test questions answered without notes
- [ ] Score yourself -- 75 and above means proceed to Day 9

---

## Day 9 Preview -- Full Code Review and Deployment Prep

Day 9 is the last day before real deployment.

- Review the complete Terraform codebase written across Days 7 and 8
- Fix any gaps, add missing tags, tighten NSG rules
- Write a README for your landing zone module
- Understand what terraform apply will actually do to your subscription
- Prepare for Day 10 -- the first real deployment

Day 9 has no new concepts.  
It is entirely about making what you have built production-quality  
and being able to explain every single line before you run apply.

---

*Day 8 of 21 | Azure Landing Zone Study Track -- Terraform Track | March 2026*