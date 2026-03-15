# My Understanding of Azure Landing Zone (ALZ) and AVM

> This document is my own understanding of ALZ and AVM written in my own words.  
> It is not a textbook. It is how I have internalized the concept so far.  
> Corrections and missing concepts are marked clearly so I know where to grow.

---

## The Problem That Made ALZ Necessary

When a client comes with a requirement -- whether it is moving from on-premises to Azure, or migrating existing Azure workloads into a better structure -- the challenge is not just technical. It is human.

Every developer works in their own unique way.

- Some create tags on resources. Some do not.
- Some create restriction policies. Some do not.
- Some set up monitoring. Some forget.
- Some follow naming conventions. Some make up their own.

Over time this creates an unprofessional, ungoverned, inconsistent Azure environment. When a client asks for a compliance report or a security audit, nobody can produce one cleanly. When a new team needs an environment, it takes weeks and looks different every time.

This inconsistency is the exact problem Azure Landing Zone is designed to eliminate.

---

## What Azure Landing Zone (ALZ) Actually Is

ALZ is a structured, governed, secure framework for setting up Azure infrastructure the right way before any workload arrives.

It is not just a template. It is an opinionated architecture that enforces:
- Consistent governance through policies
- Secure networking through hub-spoke topology
- Clear billing and cost visibility
- Role-based access so the right people have the right permissions
- Monitoring and logging baked in from day one

It works for both:
- **Greenfield** -- building in Azure from scratch with no existing infrastructure
- **Brownfield** -- migrating from on-premises to Azure, or restructuring an existing messy Azure environment (Azure-to-Azure migration)

---

## How the Setup Process Works (My Understanding of the Flow)

### Step 1: Billing Discussion First

Before anything is built, the first conversation with the client is about billing.

- How is the company structured with Microsoft? (Enterprise Agreement or Microsoft Customer Agreement)
- Who owns the billing account?
- How are departments and cost centres organised?

This creates the billing hierarchy -- Enrollment Account, Department, Billing Profile, Subscriptions. Everything in Azure costs money and has to sit inside a billing boundary. This is why it comes first.

---

### Step 2: Identity Is Established

- Microsoft Entra ID is set up as the cloud identity layer (who the users are, what groups they belong to)
- If the client has on-premises Active Directory, it is synced to Entra ID (this is called Hybrid Identity)
- Privileged Identity Management (PIM) is configured so that no one has permanent admin access -- elevated access is requested, approved, time-limited, and logged

---

### Step 3: The Management Group Hierarchy Is Created

Management Groups are folders that sit above subscriptions. The reason they matter is that any policy or RBAC assignment placed on a Management Group automatically cascades down to every subscription and resource beneath it.

This is the governance engine of ALZ. You do not rely on each developer to apply the right policies. The policies apply themselves because of where the subscription sits in the hierarchy.

The standard hierarchy looks like this:

```
Tenant Root Group
└── Company Top-Level Management Group (e.g. Contoso)
    ├── Platform          <-- Platform team's subscriptions
    │   ├── Security
    │   ├── Management
    │   ├── Identity
    │   └── Connectivity
    ├── Landing Zones     <-- Where application workloads live
    │   ├── Corp          (connected to company network, stricter policies)
    │   └── Online        (internet-facing, different policy set)
    ├── Decommissioned    <-- Old or retired subscriptions
    └── Sandbox           <-- Developer playground, relaxed rules, isolated
```

---

### Step 4: The Platform Team Builds the Platform Subscriptions

The platform team has elevated privileges that the application teams do not have. They own and build:

- **Management Subscription** -- Central monitoring, logging, cost dashboards
- **Security Subscription** -- Microsoft Sentinel, Defender for Cloud, security logs
- **Identity Subscription** -- Domain controllers, hybrid identity infrastructure
- **Connectivity Subscription** -- The hub network, Azure Firewall, VPN/ExpressRoute, DNS

These four subscriptions support every landing zone that gets created. Application teams never touch them.

---

### Step 5: The Hub Network Is Built (Connectivity Subscription)

This is a concept I initially missed but it is critical, especially for migration.

The Connectivity subscription contains the central hub network. Every landing zone subscription has its own spoke network that peers back to this hub.

```
On-Premises
     |
     | (ExpressRoute or VPN Gateway)
     |
Connectivity Subscription (HUB)
     |-- Azure Firewall (all traffic inspected here)
     |-- Azure DNS Private Resolver
     |-- VPN / ExpressRoute Gateway
     |
     |-- Peering --> Landing Zone A1 (Spoke)
     |-- Peering --> Landing Zone A2 (Spoke)
     |-- Peering --> Identity Subscription (Spoke)
```

All traffic between spokes and the internet, and between on-premises and Azure, flows through the hub. No spoke talks directly to another spoke. The firewall inspects everything.

This is what makes migration from on-premises to Azure possible -- the network path already exists before the first workload moves.

---

### Step 6: Landing Zones Are Stamped Out via Subscription Vending

When the application team needs an environment, the platform team does not manually build it.

A pipeline runs (in GitHub or Azure DevOps) and automatically creates a new subscription that:
- Is placed directly under the correct Management Group (Corp or Online depending on the workload)
- Inherits all policies from that Management Group automatically
- Has a spoke Virtual Network already created and peered to the hub
- Has RBAC already assigned (app team gets Contributor, platform team retains control above)
- Is already connected to central monitoring and logging

This automated process of creating a ready-to-use landing zone subscription is called **Subscription Vending.**

The output is a subscription the app team can use immediately -- already compliant, already connected, already monitored.

---

### Step 7: Application Teams Deploy Workloads Into the Landing Zone

Once the landing zone is handed over, the application team deploys their workloads into it.

For migration scenarios, Azure Migrate is used to:
1. Discover and assess on-premises workloads
2. Replicate the data to Azure in the background
3. Run a test migration (without cutting over)
4. Cut over when ready -- DNS and IPs switch, on-premises VM is decommissioned

Because the landing zone is already set up with policies:
- Monitoring agents are auto-installed on migrated VMs (DINE policy)
- Tags are auto-applied (Modify policy)
- Non-compliant configurations are flagged or blocked immediately (Deny/Audit policy)

The migrated workload is compliant from the moment it lands.

---

## Where AVM Fits In -- My Understanding

### The Problem AVM Solves

When the platform team builds all of the above -- management groups, policies, networking, monitoring, landing zones -- they write Infrastructure as Code (IaC) to do it. The IaC is what the pipeline runs to deploy everything consistently.

But writing IaC from scratch has the same human inconsistency problem as the developers described at the start. One engineer writes it one way. Another engineer writes it differently. Security settings get missed. Diagnostic settings get forgotten.

This is where AVM comes in.

---

### What AVM Is

**Azure Verified Modules** is a library of pre-built, pre-tested, Microsoft-approved IaC modules.

Instead of writing a Virtual Network deployment from scratch, the platform team calls an AVM module:

```
avm/res/network/virtual-network
```

That module already has:
- All required parameters defined
- Best-practice defaults built in
- Diagnostic settings included
- Tagging support included
- Microsoft's Well-Architected Framework standards baked in

The platform team still writes the IaC code. But instead of building every component from scratch, they assemble the landing zone using verified building blocks.

---

### The Three Types of AVM Modules

| Module Type | What It Deploys | Example |
|---|---|---|
| **Resource Module** | One Azure resource | A single Virtual Network |
| **Pattern Module** | Multiple resources that work together | A Role Assignment pattern across scopes |
| **Utility Module** | Helper logic used inside other modules | Consistent naming conventions |

---

### Why AVM Is Better Than Writing Raw IaC

| Raw IaC | AVM |
|---|---|
| Written by one engineer, their style | Standardised across all engineers |
| Security settings depend on who wrote it | Security settings built into the module |
| Breaks when Azure APIs change | Microsoft maintains and updates the modules |
| Different every deployment | Identical every deployment |
| Reviewed internally | Reviewed and tested by Microsoft |

---

### Who Uses AVM -- Clarification

This is something I initially got slightly wrong.

- **Platform team** uses AVM to build and deploy the landing zone infrastructure (management groups, policies, networking, platform subscriptions)
- **Application teams** consume the landing zone that the platform team built -- they do not build with AVM directly in most cases

The platform team builds the city. The app team moves into the house. AVM is what the construction crew uses to build the houses consistently.

---

## The Full Flow in One View

```
Client Requirement Arrives
         |
         v
1. Billing structure agreed and set up
         |
         v
2. Identity configured (Entra ID + Hybrid if needed + PIM)
         |
         v
3. Management Group hierarchy designed and deployed (via AVM)
         |
         v
4. ALZ Policies assigned to Management Groups (cascade down automatically)
         |
         v
5. Platform subscriptions deployed (Management, Security, Identity, Connectivity)
   --> All deployed using AVM modules via DevOps pipeline
         |
         v
6. Hub network built in Connectivity subscription
   --> Azure Firewall, VPN/ExpressRoute, DNS, Hub VNets
         |
         v
7. Landing Zone subscription stamped out via Subscription Vending pipeline
   --> Spoke VNet created and peered to Hub
   --> RBAC assigned
   --> Monitoring connected
   --> Policies inherited from Management Group
         |
         v
8. Landing Zone handed to Application Team
         |
         v
9. Workloads deployed or migrated into Landing Zone
   --> Azure Migrate for on-prem to Azure
   --> Policies auto-remediate on arrival
   --> Workload is compliant from day one
         |
         v
DELIVERY IS CONSISTENT. GOVERNED. PROFESSIONAL.
```

---

## What I Know I Still Need to Go Deeper On

- Azure Policy effects in detail (Deny, Audit, DINE, Modify) -- I understand what they do but not how to write or customise them
- Hub-Spoke networking in detail -- I understand the concept but need to understand UDRs, NSGs, and DNS resolution flow more precisely
- AVM module structure -- I need to actually open a module on GitHub and read how it is written
- Subscription Vending implementation -- I understand the concept but need to see the pipeline code

---

*This is my personal understanding document -- Version 2 | March 2026*
*Concepts will be updated as my understanding deepens each day*