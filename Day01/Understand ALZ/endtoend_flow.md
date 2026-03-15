# Azure Landing Zone - End to End Workflow (Layman's Guide)

> Read this like a story, not a textbook.  
> Every section builds on the one before it.  
> By the end you will understand how ALZ works and where AVM fits in.

---

## The One Analogy That Explains Everything

Think of Azure Landing Zone like building a **city before residents move in.**

- The **government** (Platform Team) builds the roads, power grid, water supply, and laws before anyone lives there.
- The **residents** (Application Teams) move into pre-built plots that already have electricity, water, and safety codes enforced.
- The **residents cannot break the safety codes** no matter what they do -- the rules are baked into the city, not trusted to each resident individually.
- **AVM** is the construction company that builds each house using approved blueprints, consistently, every single time.

That is Azure Landing Zone in one paragraph.

---

## The Big Picture -- 5 Phases of ALZ

```
Phase 1          Phase 2          Phase 3          Phase 4          Phase 5
---------        ---------        ---------        ---------        ---------
Set Up           Set Up           Set Up           Hand Over        Deploy
Billing &        Governance       Platform         Landing Zone     Workloads
Identity         (Policies)       Network          to App Team      (Migration)
```

Everything flows left to right. You cannot skip a phase.

---

## Phase 1 -- Set Up the Foundation (Billing + Identity)

### What happens here:
Before Azure can do anything, two things must exist:

**1. A billing account (Enterprise Agreement or Microsoft Customer Agreement)**
- This is the contract between your company and Microsoft
- It creates the Enrollment Account, Department, and Subscription structure
- Think of it as the legal ownership layer -- who pays for what

**2. Identity (Microsoft Entra ID + Active Directory)**
- Entra ID = Your company's user directory in the cloud (who your employees are)
- If your company already has on-premises Active Directory (Windows AD), it gets synced to Entra ID
- This sync is called Hybrid Identity
- Privileged Identity Management (PIM) is set up here -- this controls who gets admin access and only when they need it (not permanently)

### Why this comes first:
Every resource in Azure is owned by a subscription. Every subscription is owned by a billing account. Every person who touches Azure is authenticated by Entra ID. Nothing works without these two things existing first.

---

## Phase 2 -- Build the Governance Layer (Management Groups + Policies)

### What is a Management Group?
A folder above subscriptions. You apply rules at the folder level and every subscription inside automatically inherits those rules.

### The Hierarchy (Read Top to Bottom):

```
Tenant Root Group  <-- The top of everything, auto-created by Azure
│
└── Contoso  <-- Your company's top-level management group
    │
    ├── Platform  <-- Subscriptions for the platform team's tools
    │   ├── Security
    │   ├── Management
    │   ├── Identity
    │   └── Connectivity
    │
    ├── Landing Zones  <-- Where application workloads live
    │   ├── Corp  (connected to company network, stricter policies)
    │   └── Online  (internet-facing apps, different policy set)
    │
    ├── Decommissioned  <-- Dead subscriptions go here before deletion
    │
    └── Sandbox  <-- Playground for developers, relaxed rules, isolated
```

### What are Azure Policies?
Policies are automated rules applied to the entire management group hierarchy.

There are 4 types you need to know:

| Policy Effect | What It Does | Real Example |
|---|---|---|
| **Deny** | Blocks a resource from being created if it breaks a rule | Blocks creation of a VM without a tag |
| **Audit** | Allows it but flags it as non-compliant | Logs that a storage account has public access |
| **DINE** (Deploy-If-Not-Exists) | Automatically fixes a missing config | Auto-installs a monitoring agent on every VM |
| **Modify** | Automatically adds/changes a property | Auto-adds a required tag to every resource |

### Why this matters in plain English:
Instead of trusting every developer to follow security rules manually, the rules enforce themselves. A developer cannot deploy an insecure resource even if they try -- the policy either blocks it or auto-remediates it.

---

## Phase 3 -- Build the Platform Subscriptions (The Infrastructure Behind the Scenes)

These are 4 subscriptions the platform team owns. Application teams never touch these. They exist to support everything else.

---

### Subscription 1: Management Subscription
**Plain English: The control room.**

- Central Log Analytics Workspace collects logs from every landing zone
- Azure Monitor dashboards show the health of the entire platform
- Cost Management tracks spending across all subscriptions
- AMBA-ALZ sets up alerts so the platform team is notified before something breaks

---

### Subscription 2: Security Subscription
**Plain English: The security camera system.**

- Microsoft Sentinel acts as the SIEM (Security Information and Event Management) -- it collects and analyses security events across the entire estate
- Defender for Cloud monitors every resource for vulnerabilities
- Separate Log Analytics Workspace specifically for security logs
- Azure Update Manager keeps VMs patched

---

### Subscription 3: Identity Subscription
**Plain English: The company's ID office.**

- Domain Controllers (DC1, DC2, DC3) hosted here for on-premises AD DS
- Recovery Services Vaults for backup of identity infrastructure
- Multiple virtual networks across regions for redundancy
- This is where hybrid identity is anchored in Azure

---

### Subscription 4: Connectivity Subscription
**Plain English: The highway system.**

This is the most important platform subscription to understand for migration work.

```
On-Premises Data Center
        |
        | (ExpressRoute = Private Dedicated Line)
        | (VPN Gateway = Encrypted Tunnel Over Internet)
        |
Connectivity Subscription  <-- The HUB
        |
        |-- Peering --> Landing Zone A1 (Spoke)
        |-- Peering --> Landing Zone A2 (Spoke)
        |-- Peering --> Identity Subscription (Spoke)
```

What lives here:
- **Azure Firewall** -- inspects all traffic between spokes and the internet
- **Azure DNS + Private DNS Resolver** -- name resolution for private resources
- **ExpressRoute Circuit** -- dedicated private connection from on-prem to Azure
- **VPN Gateway** -- backup or alternative connectivity via internet
- **DDoS Protection** -- absorbs volumetric attacks before they hit your resources
- **Hub Virtual Networks** per region

**Hub-Spoke in one sentence:** All traffic flows through the hub. No spoke talks directly to another spoke. The hub is the traffic cop.

---

## Phase 4 -- Stamp Out a Landing Zone (Subscription Vending)

### What is Subscription Vending?
It is the automated process of creating a new, fully configured landing zone subscription on demand, like a vending machine dispensing a can -- the output is always the same shape, always compliant, always connected.

### The End-to-End Flow of Creating One Landing Zone:

```
Step 1: App team raises a request (ticket, form, or pipeline trigger)
          |
Step 2: Platform DevOps pipeline runs
          |
Step 3: Pipeline creates a new Subscription
          |
Step 4: Subscription is placed under the correct Management Group
        (Corp or Online depending on workload type)
          |
Step 5: Management Group policies automatically apply to the subscription
          |
Step 6: Networking is configured
        - Spoke Virtual Network created
        - Peered to Hub (Connectivity Subscription)
        - DNS settings pointed to central resolver
        - UDRs configured to route traffic through Azure Firewall
          |
Step 7: RBAC assigned
        - App team gets Contributor on their subscription
        - Platform team retains Owner at Management Group level
          |
Step 8: Monitoring connected
        - Diagnostic settings configured
        - Logs flowing to central Log Analytics Workspace
          |
Step 9: Landing Zone is handed to the App team -- ready to use
```

The app team receives a subscription that is already:
- Connected to the network
- Monitored
- Policy-compliant
- Secured
- Billed correctly

They did not build any of that. The platform team built it once, put it in code, and the pipeline stamps it out identically every time.

---

## Phase 5 -- Deploy Workloads (Migration)

Once a landing zone exists, workloads land into it. In a migration context this means moving on-premises applications and infrastructure into the landing zone subscriptions.

### The Migration Path:

```
On-Premises                                   Azure Landing Zone
-----------                                   ------------------
Physical/Virtual Servers                      Landing Zone A2 Subscription
        |                                             |
        | Step 1: Discover & Assess                   |
        | (Azure Migrate scans on-prem)               |
        |                                             |
        | Step 2: Replicate                           |
        | (Azure Migrate replicates VM data           |
        |  to Azure in the background)                |
        |                                             |
        | Step 3: Test Migration                      |
        | (Run migrated VM without cutting over)      |
        |                                             |
        | Step 4: Cutover                             |
        | (DNS/IP switched, on-prem VM turned off) -->|
        |                                             |
        |                                    VM now runs in Azure
        |                                    inside the Landing Zone
        |                                    already compliant,
        |                                    already monitored,
        |                                    already connected to on-prem
        |                                    via ExpressRoute
```

### What the Landing Zone provides during migration:
- Network path back to on-prem (ExpressRoute or VPN) is already there
- Firewall rules are already in place
- Monitoring agents auto-deploy via DINE policies
- Tags are auto-applied via Modify policies
- The migrated VM is compliant from the moment it lands

---

## How AVM Fits Into All of This

### What is AVM (Azure Verified Modules)?

AVM is a library of pre-built, Microsoft-tested Infrastructure as Code modules.

Think of it this way:
- The ALZ is the **architectural blueprint** for the city
- AVM is the **pre-fabricated building components** (walls, plumbing, electrical)
- Bicep or Terraform is the **language** used to assemble those components
- The DevOps pipeline is the **construction crew** that runs the assembly

### Two Languages, Same Purpose:

| Language | What It Is | When to Use |
|---|---|---|
| **Bicep** | Microsoft-native IaC language, simpler syntax | If the team is Azure-native |
| **Terraform** | HashiCorp IaC, works across clouds | If the team manages multi-cloud or already uses Terraform |

Both have full AVM module libraries. Pick one and go deep. Do not try to learn both at once.

---

### AVM Module Types -- Know These Three:

**1. Resource Modules**
One module = One Azure resource
Example: `avm/res/network/virtual-network` deploys one Virtual Network with all best-practice configurations baked in

**2. Pattern Modules**
One module = A combination of resources that work together
Example: `avm/ptn/authorization/resource-role-assignment` deploys a role assignment pattern across multiple scopes

**3. Utility Modules**
Helper modules used inside other modules
Example: generating consistent naming conventions across all resources

---

### How AVM Is Used to Deploy an ALZ (The Full IaC Workflow):

```
GitHub / Azure DevOps Repository
│
├── main.bicep (or main.tf)   <-- The orchestrator file
│   │
│   ├── calls avm/res/management/management-group
│   ├── calls avm/res/authorization/policy-assignment
│   ├── calls avm/res/network/virtual-network (Hub)
│   ├── calls avm/res/network/azure-firewall
│   ├── calls avm/res/network/virtual-network (Spoke)
│   ├── calls avm/res/network/virtual-network-peering
│   └── calls avm/res/log-analytics/workspace
│
└── Pipeline runs main.bicep
    │
    └── Azure deploys all resources in correct order
        with correct configurations
        every single time
        identically
```

### Why Not Just Write Your Own Code?
You can. But:
- AVM modules are tested by Microsoft
- They follow the Azure Well-Architected Framework
- They are maintained -- if Azure changes an API, the module is updated
- They enforce consistent naming, tagging, and diagnostic settings by default
- In a client engagement, using AVM signals maturity and reduces risk

---

## The Full End-to-End Story in One Flow

```
CLIENT COMES IN
      |
      v
1. Understand their billing structure
   --> Set up EA/MCA, billing hierarchy
      |
      v
2. Set up identity
   --> Configure Entra ID
   --> Sync on-prem AD DS if hybrid
   --> Configure PIM for privileged access
      |
      v
3. Design management group hierarchy
   --> Decide how many management groups needed
   --> Map workloads to Corp vs Online vs Sandbox
      |
      v
4. Assign ALZ policies to management groups
   --> Use built-in ALZ policy initiatives
   --> Customise where client compliance requires it
      |
      v
5. Deploy platform subscriptions using AVM
   --> Management, Security, Identity, Connectivity
   --> All deployed via Bicep/Terraform AVM modules
   --> Pipeline in GitHub or Azure DevOps
      |
      v
6. Build the hub network (Connectivity Subscription)
   --> Azure Firewall
   --> VPN Gateway or ExpressRoute
   --> DNS Resolver
   --> Hub Virtual Networks per region
      |
      v
7. Stamp out landing zone subscriptions (Subscription Vending)
   --> Spoke Virtual Networks created
   --> Peered to Hub
   --> RBAC assigned to App teams
   --> Monitoring connected
      |
      v
8. App teams deploy workloads OR migration happens
   --> Azure Migrate for lift-and-shift
   --> Policies auto-remediate on arrival
   --> Everything is monitored, governed, billed correctly
      |
      v
PLATFORM IS RUNNING
```

---

## The 10 Terms a Client Will Test You On

| Term | One Line Answer |
|---|---|
| Landing Zone | Pre-configured, policy-governed subscription ready for a workload |
| Management Group | A governance container that sits above subscriptions |
| Subscription Vending | Automated pipeline that stamps out new landing zones consistently |
| Hub-Spoke | Central network hub connected to multiple spoke networks via peering |
| DINE Policy | A policy that auto-deploys missing resources (like monitoring agents) |
| AVM | Microsoft-verified IaC modules for consistent, tested Azure deployments |
| ExpressRoute | A private dedicated network connection from on-premises to Azure |
| PIM | Just-in-time privileged access management -- no permanent admin rights |
| Azure Migrate | The tool used to discover, assess, replicate, and cut over on-prem workloads |
| Policy Initiative | A bundle of multiple policies assigned together as one unit |

---

## What You Do NOT Need to Know Right Now

Be honest about scope. These are real ALZ topics you can deprioritise until you are on an actual engagement:

- Azure Virtual WAN (advanced alternative to hub-spoke)
- Custom policy authoring in JSON
- ALZ accelerator deployment step-by-step (learn the concept, not the CLI commands yet)
- Multi-region failover design
- Cost Management chargebacks at scale

---

*Version: End-to-End Workflow | Track: Azure Landing Zone Middle Level | March 2026*