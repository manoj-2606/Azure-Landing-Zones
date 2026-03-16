# ALZ Day 3 -- Networking, Hub-Spoke Topology, and Connectivity

> **The single goal of Day 3:**  
> Understand how network traffic flows inside an Azure Landing Zone --  
> from on-premises to Azure, between landing zones, and out to the internet --  
> and be able to explain it on a whiteboard without notes.
>
> **Time Required:** 2 hours  
> **Split:** 50 min concepts | 40 min hands-on | 30 min AVM modules

---

## Why Networking Is the Hardest Day in Week 1

Policy tells Azure what rules to enforce.  
Networking determines how traffic physically moves.

In a client migration, the first question after "is it secure?" is always  
"how does my on-premises system talk to Azure after migration?"  
The answer lives entirely in the Connectivity subscription and hub-spoke design.

If you cannot explain traffic flow, you cannot support a migration engagement.  
This is the day most people go shallow. Do not.

---

## Section 1 -- The Core Problem Networking Solves in ALZ

Without a structured network design, an Azure environment looks like this:

```
Landing Zone A  ----direct traffic---->  Landing Zone B
Landing Zone A  ----direct traffic---->  Internet
Landing Zone B  ----direct traffic---->  Internet
On-Premises     ----no path----------->  Azure (broken migration)
```

No inspection. No control. No visibility into what is talking to what.  
If one landing zone is compromised, it can reach every other one directly.

ALZ solves this with a **Hub-Spoke topology.**

---

## Section 2 -- Hub-Spoke Topology Explained

### The Mental Model

Think of an airport hub.  
All flights go through the hub. No flight goes directly city to city.  
The hub is where security screening, customs, and routing decisions happen.

In ALZ:
- The **Hub** = Connectivity Subscription (the central network)
- The **Spokes** = Every Landing Zone Virtual Network
- The **Flights** = Network traffic between workloads, internet, and on-premises

```
                        On-Premises
                             |
                    ExpressRoute / VPN
                             |
              +--------------+--------------+
              |    CONNECTIVITY SUBSCRIPTION |
              |         (THE HUB)           |
              |                             |
              |  [Azure Firewall]           |
              |  [VPN/ExpressRoute Gateway] |
              |  [Azure DNS Private Resolver]|
              |  [DDoS Protection]          |
              |  [Hub Virtual Network]      |
              +---+----------+----------+--+
                  |          |          |
               Peering    Peering    Peering
                  |          |          |
            +-----+--+  +----+---+  +---+----+
            | LZ A1  |  | LZ A2  |  |Identity|
            | (Spoke)|  | (Spoke)|  | (Spoke)|
            +--------+  +--------+  +--------+
```

### The Rules of Hub-Spoke

1. Spokes never peer directly with each other -- all traffic goes through the hub
2. The Azure Firewall in the hub inspects all traffic between spokes
3. All internet-bound traffic from spokes routes through the hub firewall first
4. On-premises connectivity terminates at the hub -- spokes reach on-prem through the hub

### Why This Matters for Migration

When you migrate a server from on-premises to a landing zone spoke, the  
server can still communicate with remaining on-premises systems because  
the ExpressRoute or VPN Gateway in the hub provides that path.  
The app team does not build this -- it is already there when the landing zone is handed to them.

---

## Section 3 -- Virtual Network Peering

Peering is how the hub and spokes connect to each other.

### What Peering Is

A direct, low-latency, private connection between two Azure Virtual Networks.  
Traffic between peered VNets travels over the Microsoft backbone -- never the public internet.

### Two Types of Peering in ALZ

| Type | Direction | Used For |
|---|---|---|
| Hub to Spoke | Connectivity VNet peers to Landing Zone VNet | Hub can reach spoke |
| Spoke to Hub | Landing Zone VNet peers back to Connectivity VNet | Spoke can reach hub |

Both directions must be configured. Peering is not automatic in both directions.

### Key Peering Settings in ALZ

- **Allow Gateway Transit** -- enabled on the hub side so spokes can use the hub's VPN/ExpressRoute gateway
- **Use Remote Gateways** -- enabled on the spoke side so it routes through the hub's gateway to reach on-premises
- **Allow Forwarded Traffic** -- enabled so traffic from on-premises forwarded through the hub reaches the spoke

These three settings are what make the on-prem to Azure migration path work through the hub.

---

## Section 4 -- UDR (User Defined Route)

### What a UDR Is

Azure has default routing built in -- VMs in the same VNet can talk to each other automatically.  
But default routing does not force traffic through the firewall.

A **User Defined Route** is a custom routing rule that overrides Azure's default routing  
and tells traffic where to go next.

### The ALZ UDR Pattern

In every landing zone spoke, a UDR is applied to force all traffic  
through the Azure Firewall in the hub before it goes anywhere.

```
Without UDR:
VM in Spoke A --> directly to internet
                  (no inspection, not secure)

With UDR:
VM in Spoke A --> forced to Azure Firewall in Hub --> inspected --> internet
                  (controlled, logged, secure)
```

### The Default Route UDR

The most important UDR in ALZ is the default route:

```
Address Prefix:  0.0.0.0/0
Next Hop Type:   Virtual Appliance
Next Hop Address: Azure Firewall Private IP
```

This single route entry forces ALL outbound traffic from the spoke  
through the Azure Firewall regardless of destination.

### Where UDRs Are Applied

UDRs are attached to **subnets**, not individual VMs.  
Every subnet in a landing zone spoke has the UDR attached.  
Every VM in that subnet automatically follows the route.

---

## Section 5 -- NSG vs Azure Firewall

This is one of the most common points of confusion. Both control traffic.  
They are not the same thing and they are not alternatives to each other -- they work together.

### NSG (Network Security Group)

| Property | Detail |
|---|---|
| What it is | A basic layer 4 traffic filter (IP address and port rules) |
| Where it lives | Attached to a subnet or a network interface card (NIC) |
| What it controls | Inbound and outbound traffic allow/deny rules |
| What it cannot do | No application-layer inspection, no URL filtering, no threat intelligence |
| Cost | Free |

```
NSG Rule Example:
Allow inbound TCP port 443 from 10.0.0.0/8
Deny all inbound from internet
```

### Azure Firewall

| Property | Detail |
|---|---|
| What it is | A managed, stateful, layer 4 to layer 7 network firewall |
| Where it lives | Deployed in the hub virtual network (Connectivity subscription) |
| What it controls | Network rules, application rules, DNAT rules, TLS inspection |
| What it can do | URL filtering, FQDN-based rules, threat intelligence feed, logging |
| Cost | Paid service (significant cost -- this is why it lives in one central hub) |

```
Azure Firewall Rule Example:
Allow outbound HTTPS to *.microsoft.com from 10.0.0.0/8
Block all outbound to known malicious IPs (threat intelligence)
```

### How They Work Together in ALZ

```
Traffic from VM in Spoke --> NSG on subnet (first filter, layer 4)
                         --> Routes through UDR to Azure Firewall
                         --> Azure Firewall (second filter, layer 7)
                         --> Destination
```

NSG is the first line of defence at the subnet level.  
Azure Firewall is the central inspection point for all traffic crossing the hub.  
They complement each other -- NSG is not a replacement for Firewall and vice versa.

---

## Section 6 -- DNS in Azure Landing Zone

### Why DNS Is Complicated in a Hybrid Setup

On-premises: Servers resolve names using the on-prem DNS server (Active Directory DNS).  
Azure: Private resources use private DNS zones for name resolution.  
After migration: A server in Azure needs to resolve both Azure private names AND on-prem names.

Without a centralised DNS design, name resolution breaks and migrated workloads  
cannot find each other or on-premises systems.

### The ALZ DNS Architecture

```
On-Premises DNS Server (AD DS)
         |
         | (conditional forwarder for Azure private zones)
         |
Azure DNS Private Resolver (in Connectivity Subscription)
         |
         |-- Inbound Endpoint  (on-prem queries come here to resolve Azure names)
         |-- Outbound Endpoint (Azure resources query here to resolve on-prem names)
         |
Azure Private DNS Zones (linked to Hub VNet)
         |
         |-- privatelink.blob.core.windows.net
         |-- privatelink.vaultcore.azure.net
         |-- privatelink.database.windows.net
         |-- (one zone per Azure private endpoint service)
```

### The Key Concept

All landing zone spoke VNets point their DNS settings to the  
**Azure DNS Private Resolver inbound endpoint IP** in the hub.

This means:
- A VM in Landing Zone A2 queries the resolver in the hub
- The resolver checks Azure Private DNS zones for private resource names
- If not found in Azure zones, it forwards to on-premises DNS
- On-premises DNS can also forward Azure-specific queries to the resolver

One central DNS point. All resolution flows through it. No per-landing-zone DNS configuration needed.

---

## Section 7 -- ExpressRoute vs VPN Gateway

Both provide connectivity from on-premises to Azure. They are not the same.

| Property | ExpressRoute | VPN Gateway |
|---|---|---|
| Connection type | Private dedicated circuit via a network provider | Encrypted tunnel over the public internet |
| Bandwidth | Up to 100 Gbps | Up to 10 Gbps |
| Latency | Predictable, low | Variable (depends on internet) |
| Cost | High (circuit + gateway) | Lower (gateway only) |
| Reliability | High (SLA-backed dedicated line) | Lower (internet-dependent) |
| Use case | Large enterprises, production migration | Smaller workloads, backup connectivity, dev/test |
| Setup time | Weeks (provider circuit provisioning) | Hours |

### In a Migration Context

Most large enterprise clients use ExpressRoute for production migration  
because latency is predictable and bandwidth is guaranteed.  
VPN Gateway is used as a failover path or for smaller branch offices.

Both terminate at the **Connectivity subscription hub**. Spokes access  
on-premises through either path via the hub gateway.

---

## Hands-On Tasks -- Do These Today

### Task 1: Explore Virtual Networks in the Azure Portal (15 minutes)

1. Go to [portal.azure.com](https://portal.azure.com)
2. Search for **Virtual Networks** and open it
3. If you have an existing VNet, open it. If not, click Create and explore the configuration options without deploying
4. Find the **DNS Servers** setting -- note where custom DNS is configured
5. Find the **Peerings** blade -- note the fields that appear (Allow Gateway Transit, Use Remote Gateways)
6. Find the **Subnets** blade -- note that each subnet can have an NSG and a Route Table attached

---

### Task 2: Look at a Route Table (10 minutes)

1. Search for **Route Tables** in the portal
2. Click Create and look at the route configuration fields:
   - Address Prefix
   - Next Hop Type (options: Virtual Appliance, Internet, VNet, None)
   - Next Hop IP Address
3. Note that "Virtual Appliance" is what you select when the next hop is Azure Firewall
4. You do not need to deploy -- just read the configuration options

---

### Task 3: Look at an NSG (10 minutes)

1. Search for **Network Security Groups** in the portal
2. Open or create one and look at:
   - Inbound security rules
   - Outbound security rules
   - The default rules that Azure adds automatically (note the priority numbers)
   - The difference between Allow and Deny rules
3. Note that rules are evaluated lowest priority number first

---

### Task 4: Open the AVM Azure Firewall Module on GitHub (15 minutes)

1. Go to: [https://github.com/Azure/bicep-registry-modules](https://github.com/Azure/bicep-registry-modules)
2. Navigate to: `avm/res/network/azure-firewall`
3. Open the `README.md`
4. Look at:
   - What parameters does it require?
   - Find `virtualNetworkResourceId` -- this is how you tell the firewall which VNet (hub) to deploy into
   - Find `applicationRuleCollections` -- this is where URL and FQDN filtering rules live
   - Find `networkRuleCollections` -- this is where IP and port rules live
   - Find `diagnosticSettings` -- note it is built in, all firewall logs captured by default

---

### Task 5: Open the AVM Virtual Network Peering Module on GitHub (10 minutes)

1. In the same repository navigate to: `avm/res/network/virtual-network`
2. Open the `README.md` and find the `peerings` parameter
3. Read what sub-parameters it takes:
   - `remoteVirtualNetworkResourceId` -- the VNet you are peering to
   - `allowGatewayTransit` -- hub side setting
   - `useRemoteGateways` -- spoke side setting
   - `allowForwardedTraffic` -- required for on-prem traffic forwarding
4. This is how ALZ configures peering in code instead of manual portal clicks

---

## GitHub Reference Pages for Day 3

| Resource | URL | What to Use It For |
|---|---|---|
| AVM Azure Firewall Module | [https://github.com/Azure/bicep-registry-modules/tree/main/avm/res/network/azure-firewall](https://github.com/Azure/bicep-registry-modules/tree/main/avm/res/network/azure-firewall) | Understand Firewall parameters and rule structure |
| AVM Virtual Network Module | [https://github.com/Azure/bicep-registry-modules/tree/main/avm/res/network/virtual-network](https://github.com/Azure/bicep-registry-modules/tree/main/avm/res/network/virtual-network) | Understand VNet peering configuration in code |
| AVM Route Table Module | [https://github.com/Azure/bicep-registry-modules/tree/main/avm/res/network/route-table](https://github.com/Azure/bicep-registry-modules/tree/main/avm/res/network/route-table) | Understand UDR configuration in code |
| AVM DNS Private Resolver Module | [https://github.com/Azure/bicep-registry-modules/tree/main/avm/res/network/dns-resolver](https://github.com/Azure/bicep-registry-modules/tree/main/avm/res/network/dns-resolver) | Understand DNS Resolver deployment in code |
| ALZ Connectivity Module (Bicep) | [https://github.com/Azure/ALZ-Bicep/tree/main/infra-as-code/bicep/modules/hubNetworking](https://github.com/Azure/ALZ-Bicep/tree/main/infra-as-code/bicep/modules/hubNetworking) | See how the entire hub is deployed as one module |
| Azure Firewall Documentation | [https://learn.microsoft.com/en-us/azure/firewall/overview](https://learn.microsoft.com/en-us/azure/firewall/overview) | Conceptual documentation on Azure Firewall |
| VNet Peering Documentation | [https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview) | Understand peering settings in detail |
| DNS Private Resolver Documentation | [https://learn.microsoft.com/en-us/azure/dns/dns-private-resolver-overview](https://learn.microsoft.com/en-us/azure/dns/dns-private-resolver-overview) | Understand hybrid DNS resolution architecture |

---

## Self-Test Questions and Correct Answers

Answer each question in your own words first. Then check the correct answer.

---

### Question 1: What is hub-spoke topology and why does ALZ use it?

**Correct Answer:**  
Hub-spoke is a network design where all traffic between workloads and the outside world  
flows through a central hub network rather than directly between spokes.  
ALZ uses it because it centralises security inspection, firewall enforcement, DNS resolution,  
and on-premises connectivity in one place (the Connectivity subscription) instead of  
duplicating these expensive and complex components in every landing zone.  
It also means no landing zone can communicate directly with another --  
all traffic is inspected by the central firewall before reaching its destination.

---

### Question 2: What is a UDR and what specific problem does it solve in ALZ?

**Correct Answer:**  
A User Defined Route is a custom routing rule attached to a subnet that overrides  
Azure's default routing behaviour.  
In ALZ, the problem it solves is that Azure by default lets VMs route traffic directly  
to the internet without inspection. The UDR forces all traffic from landing zone subnets  
through the Azure Firewall in the hub before it reaches any destination.  
The key route is 0.0.0.0/0 with next hop set to the Azure Firewall private IP address --  
this catches all traffic regardless of destination and forces it through the firewall.

---

### Question 3: What is the difference between an NSG and Azure Firewall? Do you need both?

**Correct Answer:**  
An NSG is a free, basic layer 4 filter attached to subnets or NICs. It allows or denies  
traffic based on IP address and port. It cannot inspect application content, filter URLs,  
or use threat intelligence feeds.  
Azure Firewall is a paid, managed, stateful layer 4 to layer 7 firewall deployed centrally  
in the hub. It can filter by URL, FQDN, threat intelligence, and application protocol.  
You need both because they operate at different layers and different points in the traffic path.  
NSG is the first filter at the subnet level. Azure Firewall is the central inspection point  
that all traffic crosses when leaving the spoke. One does not replace the other.

---

### Question 4: What are the three peering settings that make on-premises connectivity work through the hub?

**Correct Answer:**  
Allow Gateway Transit -- enabled on the hub VNet so that spoke VNets can use  
the hub's VPN or ExpressRoute gateway to reach on-premises.  
Use Remote Gateways -- enabled on the spoke VNet so it routes on-premises-bound  
traffic through the hub's gateway rather than trying to reach on-premises directly.  
Allow Forwarded Traffic -- enabled so that traffic originating from on-premises  
and forwarded through the hub actually reaches the spoke VNet.  
All three must be configured correctly or on-premises connectivity breaks.

---

### Question 5: Why do all landing zone VNets point their DNS to the Azure DNS Private Resolver instead of using Azure default DNS?

**Correct Answer:**  
Azure's default DNS (168.63.129.16) can resolve public Azure service names  
but cannot resolve private endpoint names or on-premises hostnames.  
In a hybrid migration environment, workloads in landing zones need to resolve:  
- Private Azure resource names (e.g. a private storage account or Key Vault endpoint)  
- On-premises hostnames (e.g. existing database servers not yet migrated)  
The Azure DNS Private Resolver in the hub handles both by:  
- Resolving private Azure DNS zone records for private endpoints  
- Forwarding on-premises queries to the on-premises DNS server via the inbound/outbound endpoints  
All landing zone VNets point to the resolver so DNS resolution is centralised  
and consistent across the entire estate without per-landing-zone configuration.

---

### Question 6: What is the difference between ExpressRoute and VPN Gateway and when would a migration client use each?

**Correct Answer:**  
ExpressRoute is a private dedicated circuit provisioned through a network provider.  
It offers guaranteed bandwidth up to 100 Gbps, predictable low latency,  
and is not dependent on the public internet. It takes weeks to provision and costs more.  
VPN Gateway is an encrypted tunnel over the public internet.  
It costs less, can be set up in hours, but bandwidth is limited to 10 Gbps  
and latency varies with internet conditions.  
A large enterprise migrating production workloads uses ExpressRoute  
because they need guaranteed performance and reliability.  
VPN Gateway is used for smaller workloads, dev/test environments,  
branch office connectivity, or as a failover path when ExpressRoute is unavailable.

---

### Question 7: If a VM in Landing Zone A2 sends a request to a storage account with a private endpoint, what is the complete traffic and DNS flow?

**Correct Answer:**  
Step 1: VM queries DNS -- the VNet DNS setting points to the Azure DNS Private Resolver  
inbound endpoint in the hub.  
Step 2: The resolver checks the private DNS zone for `privatelink.blob.core.windows.net`  
which is linked to the hub VNet -- it resolves to the private IP of the storage account's  
private endpoint inside Landing Zone A2 or wherever it is deployed.  
Step 3: VM sends traffic to the private IP -- the UDR on the subnet routes it  
to the Azure Firewall first.  
Step 4: Azure Firewall evaluates the traffic against its rule collections --  
if allowed, forwards it to the destination.  
Step 5: Traffic reaches the storage account via its private endpoint --  
never traverses the public internet.  
This is the complete private, inspected, DNS-resolved traffic flow for a private endpoint.

---

## Day 3 Completion Checklist

- [ ] Hub-spoke topology understood and drawable from memory
- [ ] UDR purpose and the 0.0.0.0/0 default route understood
- [ ] NSG vs Azure Firewall distinction clear -- know they work together not instead of each other
- [ ] Three peering settings memorised (Gateway Transit, Remote Gateways, Forwarded Traffic)
- [ ] DNS Private Resolver architecture understood for hybrid scenarios
- [ ] ExpressRoute vs VPN Gateway distinction clear
- [ ] All 5 portal tasks completed
- [ ] AVM Firewall and VNet modules opened and reviewed on GitHub
- [ ] All 7 self-test questions answered without notes before checking
- [ ] Score yourself -- 80 and above means proceed to Day 4

---

## Day 4 Preview -- What Is Coming

Day 4 covers Identity and Access in depth.

- Microsoft Entra ID roles vs Azure RBAC -- they are not the same thing
- Where RBAC is assigned in ALZ and why the platform team keeps control at the top
- Managed Identities -- System-assigned vs User-assigned and when to use each
- Privileged Identity Management (PIM) -- just-in-time access in practice
- Service Principals vs Managed Identities for pipeline authentication
- Hands-on: RBAC assignments in the portal and the AVM role assignment module on GitHub

Identity is what determines who can break your governance model.  
Day 4 closes that gap.

---

*Day 3 of 21 | Azure Landing Zone Study Track | March 2026*