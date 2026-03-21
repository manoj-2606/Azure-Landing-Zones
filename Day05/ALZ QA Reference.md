# ALZ Day 5 -- Concepts, Q&A, and Correct Answers

> This file contains every concept covered in Day 5 and the full Q&A session.  
> Use this as a reference before any client conversation involving Azure monitoring.  
> Day 5 score: 79/100 -- significant improvement from Day 4's 59.  
> Review Q6 before Day 6 -- the DINE silent failure pattern must be locked in.

---

## Core Concepts Covered in Day 5

---

### Concept 1 -- Log Analytics Workspace

**What it is:**  
The central store where logs and metrics from Azure resources are collected,  
stored, and queried. Think of it as the flight recorder for your Azure estate.

**Why ALZ has TWO separate workspaces:**

```
Management Subscription Workspace (Platform Logs)
├── Azure Activity Logs (who did what across all subscriptions)
├── Policy compliance events
├── Azure Firewall logs
├── VPN/ExpressRoute gateway logs
├── Microsoft Defender for Cloud alerts
└── Platform VM diagnostics
Owned by: Platform Team
Access: Platform Team + Security Team

Landing Zone Workspace (Application Logs) -- one per landing zone
├── Application performance metrics
├── VM logs for that specific workload
├── App-level diagnostics
└── Custom application events
Owned by: App Team
Access: App Team + Platform Team (Reader)
```

**Why two and not one:**  
Data sovereignty -- app teams should not see platform infrastructure logs.  
Access control -- each workspace has its own RBAC and retention policy.  
Cost attribution -- each team's logging costs are separately trackable.

**How logs get there:**  
DINE policies automatically configure diagnostic settings on every resource  
deployed into a landing zone. The app team configures nothing manually.  
Every resource is wired to feed into the correct workspace from the moment it is deployed.

**KQL -- what you need to know:**  
Logs are queried using KQL (Kusto Query Language).  
You do not need to write it from memory.  
You need to know it exists and that it is how the platform team  
investigates incidents and generates compliance reports.

---

### Concept 2 -- Azure Monitor

**What it is:**  
The umbrella service that sits on top of Log Analytics.  
Collects, analyses, and acts on monitoring data from Azure resources.

**The four components:**

| Component | What It Does | Used For |
|---|---|---|
| Metrics | Real-time numerical performance data | CPU, memory, network, disk -- dashboards and performance alerts |
| Logs | Event and diagnostic data in Log Analytics | Investigation, compliance, complex alert conditions |
| Alerts | Rules that fire when a condition is met | Notifying teams, triggering automation |
| Dashboards | Visual displays of metrics and log queries | Platform team visibility across the entire estate |

**Action Groups:**  
A reusable collection of notification targets triggered when an alert fires.

```
Action Group: "Critical Platform Alerts"
├── Email: platform-team@company.com
├── SMS: On-call engineer mobile
├── Azure Function: Auto-remediation script
├── ITSM connector: Creates ServiceNow ticket
└── Webhook: Posts to Teams channel
```

One Action Group deployed centrally in the Management subscription  
is referenced by alert rules across ALL subscriptions.  
You do not create a new Action Group per landing zone.

**The alert flow:**
```
Alert Rule: VM CPU > 95% for 15 minutes
        |
        v
Alert fires
        |
        v
Action Group triggered
        |
        v
Email to operations team
SMS to on-call engineer
ITSM ticket created automatically
```

---

### Concept 3 -- Diagnostic Settings

**What they are:**  
Configuration that tells an Azure resource where to send its logs and metrics.  
OFF by default on every Azure resource.  
Must be explicitly configured to start collecting data.

**Three destinations:**

| Destination | Used For |
|---|---|
| Log Analytics Workspace | Querying, alerting, dashboards |
| Azure Storage Account | Long-term archival, compliance retention |
| Event Hub | Streaming to third-party SIEM tools |

**How ALZ handles them automatically:**  
DINE policies deploy diagnostic settings the moment a resource is created.  
VM created -- DINE fires -- diagnostic settings configured -- logs flowing.  
Key Vault created -- DINE fires -- diagnostic settings configured -- logs flowing.  
The app team does nothing. It is enforced, not trusted.

**Critical point -- you CANNOT use Deny for diagnostic settings:**  
Deny policies block resource creation based on resource properties.  
Diagnostic settings are a separate child resource -- not a property of the VM.  
You cannot block a VM deployment because it does not yet have diagnostic settings.  
That is exactly why DINE exists. Deny cannot do this job.

---

### Concept 4 -- AMBA-ALZ

**What it is:**  
Azure Monitor Baseline Alerts for Landing Zones.  
A Microsoft-maintained collection of pre-built, tested alert rules  
specifically designed for ALZ deployments.

**Why use it instead of building from scratch:**

| Building From Scratch | Using AMBA-ALZ |
|---|---|
| Weeks deciding thresholds | Deployed in hours |
| Alerts fire too much or too little until tuned | Microsoft-tested thresholds from real deployments |
| Teams typically cover VM CPU and memory only | Covers connectivity, identity, management, landing zones, service health |
| Requires deep expertise to get right | Expertise already baked in |
| Maintained by your team | Maintained by Microsoft |

**What AMBA-ALZ covers:**
- Connectivity: ExpressRoute bandwidth, VPN connection drops, Firewall SNAT exhaustion, DNS failures
- Management: Log Analytics ingestion anomalies, Monitor agent heartbeat failures
- Identity: Domain Controller CPU, AD replication failures, DC disk space
- Landing Zone: VM CPU, VM memory, Storage availability, Key Vault availability
- Service Health: Azure outages in your regions, planned maintenance, security advisories

**The client objection answer:**  
"How do you know what thresholds to set?"  
AMBA-ALZ is built from Microsoft's real-world ALZ deployment experience.  
The thresholds are not guesses -- they are tested recommendations.  
Same principle as AVM for infrastructure code.

---

### Concept 5 -- Microsoft Defender for Cloud

**Two modes:**

**Foundational CSPM (Free)**  
Basic security recommendations.  
Secure Score calculation.  
Available to every Azure subscription automatically.

**Defender Plans (Paid -- per resource type)**  
Active threat protection per resource category.

| Plan | Protects |
|---|---|
| Defender for Servers | VMs -- malware, vulnerability assessment, JIT VM access |
| Defender for Storage | Storage accounts -- malware scanning, sensitive data discovery |
| Defender for SQL | SQL databases -- SQL injection detection, anomalous access |
| Defender for Key Vault | Key Vault -- unusual access patterns, credential theft |
| Defender for Containers | AKS clusters -- runtime threat protection |

Plans are enabled selectively based on which resource types carry the most risk.  
Not every workload needs every plan. Enable based on risk profile.

**Secure Score:**  
A percentage calculated from how many applicable security recommendations  
have been implemented vs how many apply to the estate.

```
Secure Score: 54%

What it means:
- 46% of recommended security controls are not yet in place
- Each unimplemented recommendation shows:
  - Severity: High / Medium / Low
  - Exact impact on score if fixed: e.g. +2%
  - Step-by-step remediation instructions
  - Quick Fix option for some recommendations

What to do with it:
- Fix High severity items first
- Use the score improvement numbers to prioritise
- Track score over time as a security improvement metric
- Present to CISO as a measurable, prioritised remediation roadmap
```

**In ALZ:**  
Defender for Cloud enabled at Management Group level via Azure Policy.  
DINE policy deploys Defender plans to every new subscription automatically.  
Security alerts flow to Security subscription Log Analytics Workspace and Sentinel.  
Security Team has Reader across the estate -- visibility without write access.

---

### Concept 6 -- Microsoft Sentinel

**What it is:**  
Azure's cloud-native SIEM (Security Information and Event Management)  
and SOAR (Security Orchestration, Automation, and Response) tool.  
Lives in the Security subscription.

**What it does:**
```
Data feeds into Sentinel:
├── Azure Activity Logs
├── Defender for Cloud alerts
├── Azure Firewall logs
├── Entra ID sign-in logs
└── Third-party security tools
        |
        v
Analytics rules + machine learning correlate events
        |
        v
Incidents created for security team
        |
        v
Playbooks auto-respond:
├── Block suspicious IP
├── Disable compromised account
└── Notify security team via Teams
```

**The critical distinction from Azure Monitor:**

| Azure Monitor | Microsoft Sentinel |
|---|---|
| Operational monitoring | Security monitoring |
| Answers: Is infrastructure healthy? | Answers: Is infrastructure under attack? |
| Fires when CPU is high | Fires when 10 failed logins from 3 countries happen in 2 minutes |
| Threshold-based alerts | Pattern and anomaly detection |

**Why you need both:**  
No single metric threshold was breached in the login attack example.  
Azure Monitor would never fire. Sentinel detects the pattern across events.  
They answer different questions. Neither replaces the other.

---

### Concept 7 -- The Full Monitoring Flow in ALZ

```
Resource deployed in Landing Zone A2
        |
        v
DINE policy auto-configures diagnostic settings
        |
        v
Logs flow to:
├── Landing Zone Log Analytics Workspace (app-level logs)
└── Platform Log Analytics Workspace (platform activity)
        |
        v
Azure Monitor evaluates AMBA-ALZ alert rules
        |
        v
Alert condition met (e.g. VM CPU > 95% for 15 minutes)
        |
        v
Action Group triggered (deployed centrally in Management subscription)
├── Email to app team
├── SMS to on-call engineer
└── ITSM ticket created
        |
        v
Defender for Cloud continuously assesses security posture
        |
        v
Security events forwarded to Sentinel in Security subscription
        |
        v
Sentinel correlates events and creates incidents
        |
        v
Security team investigates and responds
```

---

## Day 5 Q&A Session -- Questions and Correct Answers

---

### Q1: How does the platform team monitor dozens of Azure subscriptions from one place?

**Your answer:** Log Analytics and Azure Monitor -- correct direction, missing wiring detail.  
**Score: 70%**

**Correct Answer:**  
A central Log Analytics Workspace is deployed in the Management subscription.  
Every resource across every landing zone subscription is automatically configured  
via DINE policies to send its activity logs and diagnostic logs to that central workspace.  
Azure Monitor sits on top and provides dashboards and alert rules  
that query across the entire estate.  
The platform team never logs into individual subscriptions --  
everything flows up to one central workspace automatically.  
The DINE policy is what makes this work -- not just the existence of the workspace.

---

### Q2: Who gets notified at 3am if a critical VM goes down and how does that notification reach them?

**Your answer:** Correct -- Alert rule fires, Action Group triggered, notifications sent.  
**Score: 90%**

**Correct Answer:**  
Azure Monitor alert rules define the condition -- VM CPU above 95% for 15 minutes.  
When the condition is met, the alert fires and triggers an Action Group.  
The Action Group is a reusable notification list deployed centrally  
in the Management subscription -- one Action Group referenced by all alert rules  
across all subscriptions, not one per landing zone.  
The Action Group sends email to the operations team, SMS to the on-call engineer,  
and can create an ITSM ticket automatically.  
The alert thresholds come from AMBA-ALZ -- Microsoft-tested baseline thresholds  
so the platform team does not guess what values to use.

---

### Q3: How do you prove to auditors that every resource is being monitored?

**Your answer:** DINE policy configures diagnostic settings automatically -- correct.  
**Score: 85%**

**Correct Answer:**  
DINE policies automatically configure diagnostic settings on every resource  
the moment it is deployed into a landing zone.  
The app team configures nothing -- it is enforced by policy, not trusted to humans.  
The proof for auditors is the Azure Policy compliance dashboard --  
a real-time report showing every resource across every subscription  
and whether its diagnostic settings are correctly configured.  
Any resource that bypassed the policy shows as non-compliant immediately.  
That compliance report is the audit evidence -- not just the policy itself.

---

### Q4: What is AMBA-ALZ and why use it instead of building alert rules from scratch?

**Your answer:** Microsoft-maintained pre-built alerts, same principle as AVM -- correct.  
**Score: 85%**

**Correct Answer:**  
AMBA-ALZ is Azure Monitor Baseline Alerts for Landing Zones --  
a Microsoft-maintained collection of pre-built, tested alert rules  
designed specifically for ALZ deployments.  
Building from scratch takes weeks of threshold design, testing, and tuning.  
AMBA-ALZ collapses that to a single deployment.  
Teams building from scratch typically cover VM CPU and memory  
but miss ExpressRoute bandwidth, DNS failures, SNAT port exhaustion,  
AD replication failures, and Log Analytics ingestion anomalies.  
AMBA-ALZ covers all of those because Microsoft built it from  
real-world ALZ deployment experience. Same principle as AVM --  
Microsoft-verified, tested, and maintained so you do not start from zero.

---

### Q5: What is the difference between Azure Monitor and Microsoft Sentinel and does a client need both?

**Your answer:** Correct definitions, missing the concrete "why both" example.  
**Score: 85%**

**Correct Answer:**  
Azure Monitor collects operational data and fires alerts based on defined thresholds.  
It answers: is my infrastructure healthy and performing correctly?  
Microsoft Sentinel is a SIEM that collects security events, correlates them  
using analytics rules and machine learning, and creates security incidents.  
It answers: is my infrastructure under attack or behaving suspiciously?

They cannot replace each other because they answer different questions.  
Concrete example: ten failed logins from three different countries in two minutes  
is a credential attack. No single metric threshold was breached.  
Azure Monitor would never detect it. Sentinel detects the pattern across events.  
Both are needed -- Monitor for operational health, Sentinel for security threats.

---

### Q6: A VM in Landing Zone A2 has no monitoring agent and no diagnostic settings after three weeks. How did this happen and how do you prevent it?

**Your answer:** DINE correct, proposed Deny policy -- wrong. Root cause missed.  
**Score: 60%**

**Correct Answer:**  
This should not happen in a correctly configured ALZ.  
If it did, one of two things occurred:

**Cause 1 -- DINE policy Managed Identity has no RBAC role:**  
The DINE policy assignment auto-creates a System-Assigned Managed Identity.  
If nobody assigned an RBAC role to that identity,  
it attempts the remediation, Azure rejects it silently,  
and the VM stays unconfigured with no obvious error in the portal.

**Cause 2 -- VM existed before the policy was assigned:**  
DINE fires automatically for new resources only.  
Pre-existing resources require a manual remediation task.

**The fix:**  
Assign the correct RBAC role to the policy assignment's Managed Identity.  
Create a manual remediation task for the existing non-compliant VM.  
New VMs going forward will be handled automatically.

**What NOT to do:**  
You cannot use a Deny policy to enforce diagnostic settings.  
Deny blocks resource creation based on resource properties.  
Diagnostic settings are a separate child resource -- not a VM property.  
A VM does not have diagnostic settings at the moment of creation.  
Deny cannot enforce the presence of something that does not exist yet.  
That is exactly the job DINE was designed for.

---

### Q7: A client's CISO sees a Secure Score of 54%. What does it mean and what do they do with it?

**Your answer:** Correct definition, missing actionable framing and Defender plan nuance.  
**Score: 80%**

**Correct Answer:**  
The Secure Score is a percentage representing how many applicable  
security recommendations have been implemented across the estate.  
54% means 46% of recommended security controls are not yet in place.

What to do with it:  
The score is not a vague problem -- it is a prioritised remediation roadmap.  
Each unimplemented recommendation shows its severity (High, Medium, Low),  
exactly how much the score improves if fixed, and step-by-step remediation instructions.  
Fix High severity items first. Use the score impact numbers to prioritise the backlog.  
Track the score over time as a measurable security improvement metric.  
Present to the board as: "We were at 54% in March. We are at 71% in June."  
That is a defensible security posture story.

On Defender Plans:  
The free Foundational CSPM already provides recommendations and the Secure Score.  
Paid Defender Plans add active threat protection per resource type.  
Enable plans selectively based on risk profile --  
not every workload needs every plan.  
Prioritise Defender for Servers and Defender for Key Vault first  
as these cover the highest-risk resources in most ALZ deployments.

---

## Day 5 Final Score: 79 / 100

| Question | Score | Key Gap |
|---|---|---|
| Q1 -- Central monitoring architecture | 70% | DINE as the wiring mechanism was missing |
| Q2 -- Alert notification flow | 90% | AMBA-ALZ and reusable Action Group not mentioned |
| Q3 -- Proving monitoring to auditors | 85% | Compliance dashboard as audit evidence missing |
| Q4 -- AMBA-ALZ value | 85% | Specific missed categories not mentioned |
| Q5 -- Monitor vs Sentinel | 85% | Concrete example of why both needed was missing |
| Q6 -- VM with no monitoring | 60% | Deny logic wrong, root cause pattern missed |
| Q7 -- Secure Score 54% | 80% | Missing actionable framing and plan selectivity |

---

## Before Day 6 -- One Thing You Must Lock In

**Q6 root cause pattern -- write this from memory:**  
A DINE policy is not remediating. What are the two causes?  
1. _______________  
2. _______________  

This same pattern appeared in Day 4 Q7 and Day 5 Q6.  
It will appear again in a real engagement.  
If you cannot write both causes without looking -- re-read Concept 3 in this file.

---

## Day 6 Preview -- Subscription Vending and DevOps Pipeline

- What subscription vending is and how the pipeline works end to end
- Azure DevOps vs GitHub Actions for ALZ deployment
- How AVM modules are called from a pipeline
- The ALZ Bicep accelerator -- what it is and how to read it
- Hands-on: Open the ALZ-Bicep repository and trace a full deployment

Day 6 is where everything from Days 1 to 5 comes together.  
The pipeline is what actually builds the ALZ.  
Every concept you have learned so far is an output of that pipeline.

---

*Day 5 of 21 | Azure Landing Zone Study Track | March 2026*