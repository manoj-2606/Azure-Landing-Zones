# Azure Landing Zone (ALZ) - Day 1 Study Guide

> **Goal:** Build a solid conceptual foundation before touching any code or portal.  
> **Time Required:** 1.5 to 2 hours focused, no interruptions.  
> **Rule:** Do not move to Day 2 until you can explain every section below out loud without reading it.

---

## What Is an Azure Landing Zone (Corrected Mental Model)

An Azure Landing Zone is a **pre-configured, policy-governed, scalable Azure environment** built using:
- Management Groups
- Subscriptions
- Networking baselines
- Identity and access controls
- Security and compliance guardrails

It is **not** just a template. It is an architectural framework that ensures any workload deployed into it is secure, compliant, and production-ready from day one -- whether you are building new (greenfield) or migrating existing workloads (brownfield).

> The Landing Zone is built FOR the application team -- so they cannot misconfigure their way into a security or compliance failure. The platform team builds the zone. The app team lands workloads into it.

---

## The Two Deployment Approaches

| Approach | When to Use | Scale |
|---|---|---|
| **Start Small and Expand** | New to Azure, small team, limited governance needs | Grows over time |
| **Enterprise-Scale (ALZ)** | Large org, multiple teams, strict compliance needs | Production-ready from day one |

Both still use **Subscriptions as the unit of scale** -- not Resource Groups. This is a critical mindset shift.

---

## Core Concepts - Day 1 Focus

### 1. The Governance Hierarchy (Study This First)

```
Tenant Root Group
└── Contoso (Top-Level Management Group)
    ├── Platform
    │   ├── Security Subscription
    │   ├── Management Subscription
    │   ├── Identity Subscription
    │   └── Connectivity Subscription
    ├── Landing Zones
    │   ├── Corp
    │   └── Online
    ├── Decommissioned
    └── Sandbox
```

**Why this matters:** Azure Policy and RBAC assigned at a Management Group level cascade DOWN automatically to every subscription and resource beneath it. Governance is top-down, not bottom-up.

---

### 2. Key Terms You Must Understand Today

| Term | Plain English Meaning |
|---|---|
| **Management Group** | A container above subscriptions used to apply governance at scale |
| **Subscription** | The unit of workload isolation and billing boundary |
| **Resource Group** | A logical container for resources WITHIN a subscription |
| **Azure Policy** | Rules enforced automatically on resources (deny, audit, deploy-if-not-exists) |
| **RBAC** | Who can do what at which scope |
| **DINE Policy** | Deploy-If-Not-Exists -- automatically deploys a resource if it is missing (e.g., installs monitoring agent) |
| **Subscription Vending** | Automated process of stamping out a new landing zone subscription on demand |
| **Greenfield** | Starting fresh with no existing Azure infrastructure |
| **Brownfield** | Migrating or integrating into existing Azure infrastructure |

---

### 3. The Four Platform Subscriptions - Know What Each Does

**Security Subscription**
- Microsoft Sentinel for SIEM
- Log Analytics Workspace for security logs
- Defender for Cloud
- Azure Update Manager

**Management Subscription**
- Log Analytics Workspace for platform logs
- Azure Monitor (dashboards, alerting, queries)
- AMBA-ALZ alerts and action groups
- Cost Management visibility

**Identity Subscription**
- Microsoft Entra ID Domain Services
- Domain Controllers (DC1, DC2, DC3)
- Recovery Services Vaults
- Handles hybrid identity (on-prem AD DS synced to Entra ID)

**Connectivity Subscription**
- The hub in hub-spoke topology
- Azure Firewall + Firewall Policies
- VPN Gateway and/or ExpressRoute for on-premises connectivity
- Azure DNS + Private DNS Resolver
- DDoS Protection
- Hub Virtual Networks (one per region)

> All application landing zones peer their spoke networks back to this Connectivity subscription.

---

### 4. Billing Hierarchy (Where Everything Anchors)

```
Enterprise Agreement / Microsoft Customer Agreement
└── Enrollment / Billing Account
    └── Department / Billing Profile
        └── Account / Invoice Section
            └── Subscriptions
```

In real client engagements, this hierarchy is often messy. Cleaning it up is one of the first migration tasks.

---

### 5. Identity Fundamentals for ALZ

- **Microsoft Entra ID** = Cloud identity (users, groups, service principals)
- **Active Directory Domain Services (AD DS)** = On-premises identity
- **Hybrid Identity** = AD DS synced to Entra ID via Entra Connect
- **Privileged Identity Management (PIM)** = Just-in-time elevated access with approval workflows
- **Service Principals** = Non-human identities used by apps, pipelines, and automation

---

## Day 1 Task List

- [ ] Read the ALZ conceptual overview on [learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/)
- [ ] Open the ALZ reference architecture diagram and label every box in your own words
- [ ] Draw the Management Group hierarchy on paper from memory
- [ ] Watch: "Azure Landing Zones explained" (official Microsoft YouTube, under 20 minutes)
- [ ] Write down 5 questions you still cannot answer -- these are your Day 2 targets

---

## Self-Test Before You Sleep (Answer Without Looking)

1. What is the difference between a Management Group and a Subscription?
2. Why is a Resource Group NOT the right level to apply governance in ALZ?
3. What are the four Platform subscriptions and what does each do?
4. What does a DINE policy do?
5. What is the difference between Entra ID and Active Directory Domain Services?
6. What is subscription vending?
7. Why does the Connectivity subscription exist separately from Landing Zone subscriptions?

If you cannot answer all 7 without notes, you are not done with Day 1.

---

## Reference Links

| Resource | URL |
|---|---|
| ALZ Overview (CAF) | https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/ |
| ALZ Reference Architecture | https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/landing-zone-journey |
| Azure Verified Modules | https://azure.github.io/Azure-Verified-Modules/ |
| ALZ Bicep on GitHub | https://github.com/Azure/ALZ-Bicep |
| ALZ Terraform on GitHub | https://github.com/Azure/terraform-azurerm-caf-enterprise-scale |
| AMBA-ALZ Alerts | https://azure.github.io/azure-monitor-baseline-alerts/alz/ |

---

## Coming Up: Day 2 Preview

- Azure Policy deep dive (Deny, Audit, DINE, Modify effects)
- Policy initiatives vs individual policies
- How ALZ bundles policies into initiative assignments at management group level
- Hands-on: Browse the built-in ALZ policy definitions on GitHub

---

*Version: Day 1 | Last Updated: March 2026 | Study Track: Azure Landing Zone - Middle Level*