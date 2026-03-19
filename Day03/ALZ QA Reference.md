# ALZ Day 3 -- Concepts, Q&A, and Correct Answers

> This file contains every concept covered in Day 3 and the full Q&A session.  
> Use this as a reference before any client conversation involving Azure networking.  
> Day 3 score: 71/100 -- review Q6 and Q7 before moving to Day 4.

---

## Core Concepts Covered in Day 3

---

### Concept 1 -- Hub-Spoke Topology

**What it is:**  
A network design where all traffic between workloads and the outside world flows  
through one central hub network rather than directly between spokes.

**The structure:**
```
On-Premises
      |
      | (ExpressRoute or VPN Gateway)
      |
Connectivity Subscription (HUB)
      |-- Azure Firewall
      |-- VPN / ExpressRoute Gateway
      |-- Azure DNS Private Resolver
      |-- DDoS Protection
      |-- Hub Virtual Network
      |
      |-- Peering --> Landing Zone A1 (Spoke)
      |-- Peering --> Landing Zone A2 (Spoke)
      |-- Peering --> Identity Subscription (Spoke)
```

**Why ALZ uses it:**  
- Centralises firewall, DNS, and gateway in one place instead of duplicating in every landing zone
- No spoke can talk directly to another spoke -- all traffic is inspected
- On-premises connectivity is solved once at the hub -- all spokes inherit it

---

### Concept 2 -- VNet and Subnet Structure

**The apartment building analogy:**

| Azure Concept | Real World |
|---|---|
| VNet | The entire apartment building |
| Subnet | One floor of the building |
| VM | A flat on that floor |
| IP Address | The flat's door number |
| CIDR | How many flats exist on that floor |

**Traffic rules:**

| Scenario | What happens |
|---|---|
| Same subnet | Direct -- no routing needed |
| Different subnet, same VNet | Routed -- goes through UDR and Azure Firewall |
| Different VNet | Routed -- goes through peering and Hub Firewall |

**Critical point most people get wrong:**  
Same VNet does NOT mean direct traffic.  
Web and DB are in the same VNet but different subnets.  
Different subnets = must go through routing = must go through Firewall in ALZ.

---

### Concept 3 -- CIDR

**The one sentence that unlocks CIDR:**  
The number after the slash tells you how many bits are LOCKED.  
The remaining free bits are yours to use as addresses.

**The formula:**
```
Total addresses  = 2 ^ (32 minus slash number)
Usable addresses = Total addresses minus 5 (Azure reserves 5 per subnet)

Example: /24
2 ^ (32 - 24) = 2^8 = 256 total
256 - 5 = 251 usable
```

**The 5 CIDR values to memorise:**

| CIDR | Usable Addresses | Used For |
|---|---|---|
| /16 | 65,531 | VNet address space |
| /24 | 251 | Standard subnet (web, app, DB tiers) |
| /26 | 59 | Small subnet (Firewall, Bastion, Gateway) |
| /27 | 27 | Tiny subnet (DNS Resolver) |
| /32 | 1 | Single IP in firewall rules only |

**Mandatory subnet names in Azure (exact spelling required):**
- `AzureFirewallSubnet` -- minimum /26 -- Azure Firewall will not deploy without this exact name
- `GatewaySubnet` -- minimum /27 -- VPN and ExpressRoute require this exact name
- `AzureBastionSubnet` -- minimum /26 -- Azure Bastion requires this exact name

---

### Concept 4 -- UDR (User Defined Route)

**What it is:**  
A custom routing rule attached to a subnet that overrides Azure's default routing.

**The default route in every ALZ subnet:**

| Address Prefix | Next Hop Type | Next Hop Address |
|---|---|---|
| 0.0.0.0/0 | Virtual Appliance | Azure Firewall private IP |

0.0.0.0/0 matches every possible destination.  
Every packet leaving any subnet -- to anywhere -- hits the Firewall first.

**Where UDRs live:**  
Attached to subnets, not individual VMs.  
Every VM in that subnet automatically follows the route.

---

### Concept 5 -- NSG vs Azure Firewall

**They are not alternatives. They work at different layers in combination.**

| | NSG | Azure Firewall |
|---|---|---|
| Layer | Layer 4 (IP and port) | Layer 7 (URL, FQDN, threat intelligence) |
| Where | Attached to subnet or NIC | Deployed in hub VNet |
| Cost | Free | Paid |
| Can block by URL | No | Yes |
| Can use threat intelligence | No | Yes |

**How they work together:**
```
VM sends traffic
      |
      v
NSG on subnet (first check -- IP and port filter)
      |
      v
UDR routes to Azure Firewall (second check -- deep inspection)
      |
      v
Destination
```

**Why you need both:**  
NSG gives subnet-level isolation. Firewall gives deep inspection.  
NSG without Firewall = no URL filtering, no threat intelligence.  
Firewall without NSG = no subnet isolation, compromised VM reaches all same-subnet VMs freely.

---

### Concept 6 -- VNet Peering Settings for On-Premises Connectivity

**The three settings that must all be configured correctly:**

**1. Allow Gateway Transit**
- Enabled on: Hub VNet
- What it does: Allows the hub to share its VPN or ExpressRoute gateway with peered spokes
- If missing: Spokes have no path to on-premises even if peering is connected

**2. Use Remote Gateways**
- Enabled on: Spoke VNet
- What it does: Tells the spoke to use the hub's gateway for on-premises traffic
- If missing: Spoke ignores the hub gateway and on-premises traffic fails

**3. Allow Forwarded Traffic**
- Enabled on: Both hub and spoke
- What it does: Allows traffic that originated outside the VNet to pass through the peering
- If missing: On-premises traffic forwarded through the hub gets dropped at the peering boundary

**The critical trap:**  
If any one of these three is misconfigured, on-premises connectivity breaks silently.  
The peering still shows as Connected in the portal. No error is thrown. Traffic just drops.  
This is one of the most common real-world ALZ misconfiguration issues.

---

### Concept 7 -- ExpressRoute vs VPN Gateway

| | ExpressRoute | VPN Gateway |
|---|---|---|
| Connection type | Private dedicated circuit via provider | Encrypted tunnel over public internet |
| Bandwidth | Up to 100 Gbps | Up to 10 Gbps |
| Latency | Predictable, low | Variable |
| Cost | High | Lower |
| Setup time | Weeks | Hours |
| Use case | Production migration, large enterprise | Dev/test, smaller workloads, backup path |

Both terminate at the **GatewaySubnet inside the Connectivity subscription hub.**  
Spokes reach on-premises through the hub -- never directly.

---

### Concept 8 -- Private Endpoints and Private DNS Zones

**The problem they solve:**  
A storage account by default has a public IP.  
Even with a Private Endpoint created, Azure's public DNS still returns the public IP.  
Without a Private DNS Zone, traffic would still go over the internet.

**How it works together:**

```
Step 1: Private Endpoint created
        Storage account gets a private IP inside the Landing Zone VNet
        e.g. 10.1.0.132

Step 2: Private DNS Zone created
        Zone: privatelink.blob.core.windows.net
        Record: mystorageaccount --> 10.1.0.132
        Zone is linked to the Hub VNet

Step 3: Landing Zone VNet DNS points to Azure DNS Private Resolver in hub
        Resolver knows about all Private DNS Zones
        Returns private IP when queried

Step 4: VM queries DNS for mystorageaccount.blob.core.windows.net
        Gets back 10.1.0.132 (private IP) instead of public IP
        Traffic never leaves the Azure network
```

**Private DNS Zone names for common services:**

| Service | Private DNS Zone |
|---|---|
| Blob Storage | privatelink.blob.core.windows.net |
| Key Vault | privatelink.vaultcore.azure.net |
| SQL Database | privatelink.database.windows.net |
| Azure Container Registry | privatelink.azurecr.io |

---

## Day 3 Q&A Session -- Questions and Correct Answers

---

### Q1: A client has 3 application tiers -- web, app, and database. How do you structure the Azure network?

**Correct Answer:**  
Create one VNet with address space 10.0.0.0/16 -- that gives approximately 65,000 addresses as a pool.  
Inside the VNet, carve three subnets -- one per tier:
- Web tier: 10.0.1.0/24 -- 251 usable addresses
- App tier: 10.0.2.0/24 -- 251 usable addresses
- DB tier:  10.0.3.0/24 -- 251 usable addresses

Each tier is isolated on its own subnet so cross-tier traffic goes through the Firewall.  
The database subnet is the most locked down -- only the app tier subnet is permitted to reach it via NSG rules.

**Common mistake made:** Assigning /8 to subnets. /8 is 16 million addresses -- never used for subnets in ALZ.

---

### Q2: If a web server sends data to a database server in the same VNet, does it go directly?

**Correct Answer:**  
No. They are in the same VNet but in different subnets.  
Different subnets require routing -- they do not communicate directly.  
In ALZ, the UDR on the web subnet forces the traffic to the Azure Firewall in the hub.  
The Firewall evaluates its rules -- if allowed, forwards the traffic to the DB subnet.  
The DB VM receives it.

Same VNet does not mean same subnet. Same subnet means direct. Different subnets always route.

---

### Q3: What is the difference between NSG and Azure Firewall and why do you need both?

**Correct Answer:**  
NSG is a free layer 4 filter attached to a subnet. It allows or blocks traffic based on IP address and port only. It cannot inspect URLs, domain names, or application content.

Azure Firewall is a paid, managed layer 7 firewall deployed centrally in the hub. It inspects URLs, FQDNs, uses threat intelligence feeds, and understands application protocols.

You need both because:
- NSG provides subnet-level isolation -- a compromised VM cannot freely reach all other same-subnet VMs
- Azure Firewall provides deep inspection -- blocks malicious domains, filters URLs, catches threat intelligence hits
- NSG without Firewall = no application-layer visibility
- Firewall without NSG = no subnet-level segmentation

They operate at different points on the same traffic path. Neither replaces the other.

---

### Q4: How do on-premises servers communicate with Azure resources after migration?

**Correct Answer:**  
Via ExpressRoute or VPN Gateway -- both terminate at the GatewaySubnet in the Connectivity subscription hub.

ExpressRoute = private dedicated circuit provisioned through a network provider. Never touches the public internet. Predictable low latency. Up to 100 Gbps. Takes weeks to provision. Used for production enterprise migrations.

VPN Gateway = encrypted tunnel over the public internet. Variable latency. Up to 10 Gbps. Set up in hours. Used for smaller workloads, dev/test, or as a failover path.

Spokes reach on-premises through the hub -- not directly. The peering settings Allow Gateway Transit and Use Remote Gateways must be configured correctly for this path to work.

---

### Q5: A VM in Landing Zone A2 needs to reach a storage account privately without traffic going over the internet. Walk through what happens step by step.

**Correct Answer:**
```
Step 1: VM asks DNS for the storage account IP
        Query goes to Azure DNS Private Resolver inbound endpoint in the hub
        (because the Landing Zone VNet DNS setting points there)

Step 2: Resolver checks the Private DNS Zone
        Zone: privatelink.blob.core.windows.net
        Returns the PRIVATE IP of the Private Endpoint (e.g. 10.1.0.132)
        NOT the public IP

Step 3: VM sends traffic to 10.1.0.132
        UDR on the subnet forces it through Azure Firewall
        Firewall evaluates rules and allows it

Step 4: Traffic reaches the storage account via its Private Endpoint
        Public internet was never involved at any point
```

**Common mistakes made:**
- Saying the resolver returns the Private Endpoint itself -- it returns the private IP address
- Saying traffic goes over the internet -- the entire point of this architecture is it does not
- Confusing which step the Private Endpoint vs DNS Resolver plays

---

### Q6: What are the three VNet peering settings that make on-premises connectivity work through the hub and what happens if one is misconfigured?

**Correct Answer:**

**Allow Gateway Transit** -- on the hub VNet  
Shares the hub's VPN/ExpressRoute gateway with all peered spokes.  
Without it: spokes have no path to on-premises.

**Use Remote Gateways** -- on the spoke VNet  
Tells the spoke to route on-premises traffic through the hub's gateway.  
Without it: spoke ignores the hub's gateway entirely.

**Allow Forwarded Traffic** -- on both sides  
Allows traffic originating outside the VNet to pass through the peering.  
Without it: on-premises traffic forwarded through the hub drops at the peering boundary.

**If any one is misconfigured:**  
On-premises connectivity breaks silently. The peering still shows Connected in the portal.  
No error message. Traffic just drops. This is the most common real-world ALZ networking mistake.

**Score on this question: 0/100 -- answered with VNet tiers instead of peering settings.**  
Memorise these three before any client engagement.

---

### Q7: A developer says -- why not put everything in one big subnet? It would be simpler. How do you respond?

**Correct Answer:**

Four specific reasons -- not "security purposes":

**Blast radius:**  
If one VM is compromised in a flat single subnet, it can reach every other VM directly.  
Separate subnets mean a compromised web VM cannot freely attack the database -- traffic must cross a subnet boundary and hit the Firewall.

**No granular NSG control:**  
NSGs attach to subnets. One subnet means one NSG governs everything.  
You cannot enforce "only app tier talks to database" because they share the same subnet and same rules.

**No cross-tier traffic inspection:**  
UDRs force cross-subnet traffic through the Firewall.  
One subnet means no subnet boundaries, no UDR triggers, web to database traffic bypasses the Firewall entirely.

**Compliance failure:**  
Security frameworks including ISO 27001 and PCI-DSS require network segmentation.  
A single flat subnet fails that requirement immediately and blocks the client from certification.

---

## Day 3 Final Score: 71 / 100

| Question | Score | Key Gap |
|---|---|---|
| Q1 -- VNet structure | 90% | CIDR sizes now correct |
| Q2 -- Cross-subnet traffic | 70% | Initially said direct -- corrected |
| Q3 -- NSG vs Firewall | 75% | Right direction, missing specific depth |
| Q4 -- On-premises connectivity | 85% | VPN backbone wording imprecise |
| Q5 -- Private Endpoint flow | 75% | Resolver returns private IP not PE |
| Q6 -- Peering settings | 0% | Completely wrong answer -- must review |
| Q7 -- Single subnet argument | 60% | Right instinct, missing 3 of 4 reasons |

**Before Day 4: Write the three peering settings from memory without looking.**  
If you cannot -- re-read Concept 6 in this file before proceeding.

---

## Day 4 Preview -- Identity and RBAC

- Microsoft Entra ID roles vs Azure RBAC -- they are not the same thing
- Where RBAC is assigned in ALZ and why the platform team keeps control at the top
- Managed Identities -- System-assigned vs User-assigned
- Privileged Identity Management -- just-in-time access in practice
- Service Principals vs Managed Identities for pipeline authentication

---

*Day 3 of 21 | Azure Landing Zone Study Track | March 2026*