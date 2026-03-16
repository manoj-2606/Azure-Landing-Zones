# Azure Networking Reference -- VNet, Subnets, and CIDR

> This is a personal reference file.  
> Everything here is written in plain language.  
> Use this before any client conversation involving Azure networking.

---

## The One Analogy That Explains Everything

| Azure Concept | Real World Equivalent |
|---|---|
| Virtual Network (VNet) | The entire apartment building |
| Subnet | One floor of the building |
| VM / Resource | A flat on that floor |
| IP Address | The flat's door number |
| CIDR range | How many flats exist on that floor |
| Routing | The building's reception desk |
| NSG | The doorman checking who can enter a floor |
| UDR | A sign at reception saying "all visitors must go through security first" |
| Azure Firewall | The security checkpoint all visitors must pass through |

---

## Part 1 -- Virtual Network (VNet)

### What It Is

A VNet is a private, isolated network inside Azure that belongs to you.  
Nothing outside can reach inside unless you explicitly open a path.  
Everything inside can see each other by default -- unless you add rules.

### The Structure

```
Virtual Network: 10.0.0.0/16
(The entire building -- 65,534 addresses available)
│
├── Subnet 1 -- Web tier:  10.0.1.0/24  (Floor 1 -- 254 usable addresses)
│   ├── Web VM 1 --> gets IP: 10.0.1.4
│   ├── Web VM 2 --> gets IP: 10.0.1.5
│   └── Web VM 3 --> gets IP: 10.0.1.6
│
├── Subnet 2 -- App tier:  10.0.2.0/24  (Floor 2 -- 254 usable addresses)
│   ├── App VM 1 --> gets IP: 10.0.2.4
│   └── App VM 2 --> gets IP: 10.0.2.5
│
└── Subnet 3 -- DB tier:   10.0.3.0/24  (Floor 3 -- 254 usable addresses)
    ├── DB VM 1  --> gets IP: 10.0.3.4
    └── DB VM 2  --> gets IP: 10.0.3.5
```

### The Traffic Rules

**Same subnet (same floor):**  
VMs talk to each other directly. No routing needed. No rules needed.  
Web VM 1 can reach Web VM 2 automatically.

**Different subnet (different floor):**  
Traffic must go through the router (reception desk).  
Web VM 1 wanting to reach DB VM 1 goes: Web VM 1 → Router → DB VM 1.  
In ALZ, a UDR intercepts that router decision and forces it through the Azure Firewall first.

**Outside the VNet (internet or on-premises):**  
Blocked by default. Only opens if you explicitly configure:
- A Public IP + firewall rule (for internet)
- VNet Peering (for another VNet)
- VPN Gateway or ExpressRoute (for on-premises)

---

## Part 2 -- CIDR Notation

### The One Sentence That Unlocks CIDR

> The number after the slash tells you how many bits are LOCKED.  
> The remaining free bits are yours to use as addresses.

An IP address has 32 bits total.  
/24 locks 24 bits. Leaves 8 free. 2^8 = 256 addresses.  
/16 locks 16 bits. Leaves 16 free. 2^16 = 65,536 addresses.

### The Quick Formula

```
Total addresses  = 2 ^ (32 minus the slash number)
Usable addresses = Total addresses minus 5
                   (Azure reserves 5 per subnet: network, gateway, broadcast, and 2 DNS)
```

### Common CIDR Values -- Memorise These 5

| CIDR | Total Addresses | Usable | Use In ALZ |
|---|---|---|---|
| /16 | 65,536 | 65,531 | VNet address space (the whole building) |
| /24 | 256 | 251 | Standard subnet (one floor, general workloads) |
| /26 | 64 | 59 | Small subnet (Azure Firewall, Gateway subnet) |
| /27 | 32 | 27 | Tiny subnet (DNS Resolver, Bastion) |
| /32 | 1 | 1 | Single IP address (used in firewall rules only) |

### How to Read Any CIDR in 3 Seconds

**Given: 10.0.2.0/24**

```
Step 1 -- Base IP:    10.0.2.0 -- this is where the subnet starts
Step 2 -- Slash:      /24 = 32 - 24 = 8 free bits = 2^8 = 256 addresses
Step 3 -- Subtract:   256 - 5 Azure reservations = 251 usable VMs
Step 4 -- Range:      First address: 10.0.2.0 (network)
                      First usable VM: 10.0.2.4
                      Last usable VM:  10.0.2.254
                      Last address:    10.0.2.255 (broadcast)
```

### The Golden Rule of CIDR in Azure

> Subnets cannot overlap and must fit inside the VNet range.

If your VNet is 10.0.0.0/16:
- Every subnet must start with 10.0.x.x
- No two subnets can share the same address range
- A /24 at 10.0.1.0 and a /26 at 10.0.1.0 will CONFLICT -- the /26 is inside the /24

---

## Part 3 -- How Subnets Are Carved (The Pizza Rule)

A VNet is a pizza. Subnets are the slices.  
The slices cannot overlap. Every slice must stay inside the pizza.  
Unsliced pizza is unused address space -- that is fine, you can always slice it later.

### Simple 3-Tier App Layout

```
VNet: 10.0.0.0/16
├── Web subnet:  10.0.1.0/24  [-----254 addresses-----]
├── App subnet:  10.0.2.0/24  [-----254 addresses-----]
├── DB subnet:   10.0.3.0/24  [-----254 addresses-----]
└── Unallocated: 10.0.4.0 onwards [remaining space unused]
```

### ALZ Hub VNet Layout (Connectivity Subscription)

```
Hub VNet: 10.0.0.0/16
│
├── AzureFirewallSubnet:    10.0.0.0/26   [64 addresses]
│   IMPORTANT: This subnet name is MANDATORY and exact.
│   Azure Firewall will not deploy without this exact name.
│
├── GatewaySubnet:          10.0.1.0/27   [32 addresses]
│   IMPORTANT: This subnet name is MANDATORY and exact.
│   VPN Gateway and ExpressRoute require this exact name.
│
├── AzureBastionSubnet:     10.0.2.0/26   [64 addresses]
│   IMPORTANT: This subnet name is MANDATORY and exact.
│   Azure Bastion requires minimum /26.
│
├── DNS Resolver inbound:   10.0.3.0/28   [16 addresses]
│   Minimum /28 required by Azure DNS Private Resolver.
│
└── Management subnet:      10.0.4.0/24   [254 addresses]
    Jump servers, monitoring agents, platform tooling.
```

### ALZ Landing Zone Spoke VNet Layout

```
Spoke VNet: 10.1.0.0/24
│
├── App tier subnet:        10.1.0.0/26   [64 addresses]
│   Application VMs, containers, App Service
│
├── DB tier subnet:         10.1.0.64/26  [64 addresses]
│   Databases, cache, storage backends
│
├── Private Endpoint subnet:10.1.0.128/27 [32 addresses]
│   Private endpoints for Key Vault, Storage, SQL etc.
│
└── Unallocated:            10.1.0.160/27 [remaining]
    Reserved for future subnets
```

---

## Part 4 -- Routing and UDR in Plain English

### Default Azure Routing (No UDR)

```
VM in Web subnet
      |
      | (wants to reach internet)
      |
Azure default routing --> goes directly to internet
      NO inspection. NOT secure.
```

### ALZ Routing With UDR Applied

```
VM in Web subnet
      |
      | (UDR rule: 0.0.0.0/0 --> Azure Firewall IP)
      |
Azure Firewall in Hub (inspects all traffic)
      |
      | (if allowed by firewall rule)
      |
Destination (internet, another subnet, on-premises)
```

### The UDR Rule That Does This

Every landing zone subnet has a Route Table attached with this one rule:

| Address Prefix | Next Hop Type | Next Hop IP |
|---|---|---|
| 0.0.0.0/0 | Virtual Appliance | Azure Firewall private IP |

0.0.0.0/0 means "match every possible destination."  
So every packet leaving any subnet -- to any destination -- hits the firewall first.

---

## Part 5 -- NSG vs Azure Firewall

They are not alternatives. They work at different layers and in combination.

| | NSG | Azure Firewall |
|---|---|---|
| What it is | Basic layer 4 filter | Full layer 7 firewall |
| Where it lives | Attached to a subnet or NIC | Deployed in the hub VNet |
| Controls | IP address and port rules | URLs, FQDNs, threat intelligence, TLS |
| Cost | Free | Paid (significant) |
| Blocks at | Subnet entry/exit | Hub crossing point |

### How They Work Together

```
VM sends traffic
      |
      v
NSG on subnet (first check -- basic IP/port filter)
      |
      v
UDR routes to Azure Firewall (second check -- deep inspection)
      |
      v
Destination reached (or blocked)
```

NSG is the doorman at the floor.  
Azure Firewall is the security checkpoint at the building exit.  
Both are needed. Neither replaces the other.

---

## Part 6 -- Real World CIDR Assignments in a Full ALZ

This is the complete IP plan for a typical ALZ deployment.  
Study this until you can read it without calculating.

```
10.0.0.0/8  --> Reserved for the entire Azure estate (never assign this directly)

HUB (Connectivity Subscription)
10.0.0.0/16    --> Hub VNet (entire hub address space)
10.0.0.0/26    --> AzureFirewallSubnet (mandatory name, 64 addresses)
10.0.1.0/27    --> GatewaySubnet (mandatory name, 32 addresses)
10.0.2.0/26    --> AzureBastionSubnet (mandatory name, 64 addresses)
10.0.3.0/28    --> DNS Resolver inbound endpoint (16 addresses)
10.0.3.16/28   --> DNS Resolver outbound endpoint (16 addresses)
10.0.4.0/24    --> Management/jump subnet (254 addresses)

LANDING ZONE A1 (Corp workload 1)
10.1.0.0/24    --> Spoke VNet A1
10.1.0.0/26    --> App subnet (64 addresses)
10.1.0.64/26   --> DB subnet (64 addresses)
10.1.0.128/27  --> Private endpoint subnet (32 addresses)

LANDING ZONE A2 (Corp workload 2)
10.2.0.0/24    --> Spoke VNet A2
10.2.0.0/26    --> App subnet (64 addresses)
10.2.0.64/26   --> DB subnet (64 addresses)
10.2.0.128/27  --> Private endpoint subnet (32 addresses)

IDENTITY SUBSCRIPTION
10.3.0.0/24    --> Identity VNet
10.3.0.0/26    --> Domain Controller subnet (DC1, DC2, DC3)
10.3.0.64/26   --> Recovery Services subnet
```

---

## Part 7 -- The 3 Things That Will Catch You Out in a Client Conversation

### Catch 1: Mandatory Subnet Names
Three subnets in Azure have mandatory exact names -- if you name them wrong, the service refuses to deploy:

- `AzureFirewallSubnet` -- must be exactly this, minimum /26
- `GatewaySubnet` -- must be exactly this, minimum /27
- `AzureBastionSubnet` -- must be exactly this, minimum /26

### Catch 2: Overlapping Subnets
You cannot assign 10.0.1.0/24 and 10.0.1.128/25 to the same VNet.  
The /25 range (10.0.1.128 to 10.0.1.255) is already inside the /24 range (10.0.1.0 to 10.0.1.255).  
They overlap. Azure will reject it.

### Catch 3: Azure Reserves 5 Addresses Per Subnet
In every subnet, the first 4 addresses and the last 1 are reserved by Azure:
- x.x.x.0 -- Network address
- x.x.x.1 -- Default gateway
- x.x.x.2 -- Azure DNS mapping
- x.x.x.3 -- Azure DNS mapping
- x.x.x.255 -- Broadcast

VMs start getting IPs from x.x.x.4 onwards.  
A /29 (8 addresses minus 5 reserved) gives you only 3 usable VMs.  
Never size a subnet too small.

---

## Quick Reference Card

```
/16 = 65,534 addresses --> VNet
/24 = 254 addresses    --> Standard subnet
/26 = 62 addresses     --> Small subnet (Firewall, Bastion, Gateway)
/27 = 30 addresses     --> Tiny subnet (DNS Resolver)
/28 = 14 addresses     --> Micro subnet (DNS endpoints only)
/32 = 1 address        --> Single IP (firewall rules)

Formula:   2 ^ (32 - slash) = total    minus 5 = usable
Example:   2 ^ (32 - 24)   = 256       minus 5 = 251 usable
```

---

*Personal Reference | Azure Networking -- VNet, Subnet, CIDR | March 2026*  
*Part of the ALZ 21-Day Study Track*