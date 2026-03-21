# ALZ Day 5 -- Monitoring, Logging, and AMBA-ALZ

> **The single goal of Day 5:**  
> Understand how ALZ knows when something is wrong --  
> where logs go, what triggers alerts, who gets notified,  
> and how Defender for Cloud fits into the security picture.  
> Be able to explain the monitoring architecture to a client  
> without referring to notes.
>
> **Time Required:** 2 hours  
> **Split:** 50 min concepts | 40 min hands-on portal | 30 min GitHub

---

## Why Monitoring Is a Day 1 Conversation With Every Client

Clients do not ask about management groups on day one of production.  
They ask: "How do I know if something breaks?"  
"Who gets paged at 3am if a VM goes down?"  
"How do I prove to my auditors that we are monitoring everything?"

Monitoring is what makes the platform visible and trustworthy.  
Without it, ALZ is a governance framework nobody can see inside.  
With it, the platform team has eyes on everything across every subscription  
from a single central location.

---

## Section 1 -- Log Analytics Workspace

### What It Is

Log Analytics Workspace is the central store where logs and metrics  
from Azure resources are collected, stored, and queried.

Think of it as the flight recorder for your entire Azure estate.  
Every event, every metric, every security alert writes to it.  
The platform team queries it to investigate incidents,  
build dashboards, and generate compliance reports.

### Why ALZ Has TWO Separate Workspaces

This is the detail most people miss.

```
┌─────────────────────────────────────────────────────┐
│           MANAGEMENT SUBSCRIPTION                   │
│                                                     │
│   Log Analytics Workspace (Platform Logs)           │
│                                                     │
│   Collects:                                         │
│   - Azure Activity Logs (who did what in Azure)     │
│   - Policy compliance events                        │
│   - Azure Firewall logs                             │
│   - VPN/ExpressRoute gateway logs                   │
│   - Microsoft Defender for Cloud alerts             │
│   - Platform VM diagnostics                         │
│                                                     │
│   Owned by: Platform Team                           │
│   Access: Platform Team + Security Team             │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│           LANDING ZONE SUBSCRIPTION                 │
│           (Each landing zone has its own)           │
│                                                     │
│   Log Analytics Workspace (Application Logs)        │
│                                                     │
│   Collects:                                         │
│   - Application performance metrics                 │
│   - VM logs for that specific workload              │
│   - App-level diagnostics                           │
│   - Custom application events                       │
│                                                     │
│   Owned by: App Team                                │
│   Access: App Team + Platform Team (Reader)         │
└─────────────────────────────────────────────────────┘
```

**Why two and not one:**  
Separation of concerns and data sovereignty.  
The app team should not see platform infrastructure logs.  
The platform team should not have to wade through  
every application's debug logs to find a firewall event.  
Each workspace has its own retention policy and access controls.  
Platform workspace typically has longer retention for compliance.

### KQL -- The Query Language

Logs in Log Analytics are queried using KQL (Kusto Query Language).  
You do not need to write KQL today.  
You need to know it exists and what it looks like.

```kql
// Example: Find all failed VM deployments in the last 24 hours
AzureActivity
| where TimeGenerated > ago(24h)
| where ActivityStatus == "Failed"
| where ResourceProvider == "Microsoft.Compute"
| project TimeGenerated, Caller, OperationName, ResourceGroup
| order by TimeGenerated desc
```

In a client conversation you say:  
"We can query the Log Analytics Workspace using KQL to pull  
any event, alert, or activity across the estate within seconds."  
That sentence is enough for a client meeting.

---

## Section 2 -- Azure Monitor

### What It Is

Azure Monitor is the umbrella service that sits on top of Log Analytics.  
It collects, analyses, and acts on monitoring data from Azure resources.

### The Four Components of Azure Monitor You Need to Know

**1. Metrics**  
Real-time numerical data about resource performance.  
Examples: CPU percentage, memory usage, network bytes in/out, disk IOPS.  
Stored for 93 days by default. Cannot be queried with KQL.  
Used for dashboards and performance-based alerts.

**2. Logs**  
Event and diagnostic data written to Log Analytics Workspace.  
Queryable with KQL. Stored for configurable retention periods.  
Used for investigation, compliance, and complex alert conditions.

**3. Alerts**  
Rules that fire when a condition is met and trigger an action.

```
Alert Rule: CPU > 90% for 5 minutes
        |
        v
Alert fires
        |
        v
Action Group triggered
        |
        v
Email sent to operations team
SMS sent to on-call engineer
ITSM ticket created automatically
```

**4. Dashboards**  
Visual displays of metrics and log query results.  
The Management subscription contains central dashboards  
visible to the platform team across the entire ALZ estate.

### Action Groups

An Action Group is a reusable list of notification targets and actions  
that get triggered when an alert fires.

```
Action Group: "Critical Platform Alerts"
├── Email: platform-team@company.com
├── SMS: On-call engineer mobile number
├── Azure Function: Auto-remediation script
├── ITSM connector: Creates ServiceNow ticket
└── Webhook: Posts to Teams channel
```

One Action Group can be reused across many alert rules.  
In ALZ, Action Groups are deployed centrally in the Management subscription  
and referenced by alert rules across all subscriptions.

---

## Section 3 -- Diagnostic Settings

### What They Are

Diagnostic settings tell an Azure resource where to send its logs and metrics.  
Every Azure resource has diagnostic settings that are OFF by default.  
You must explicitly configure them to start collecting data.

### The Three Destinations

```
Azure Resource (VM, Key Vault, Firewall, Storage Account etc.)
        |
        | Diagnostic Settings configured
        |
        v
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│  Log Analytics│  │ Azure Storage │  │  Event Hub    │
│  Workspace    │  │ Account       │  │               │
│               │  │               │  │               │
│ For querying  │  │ For long-term │  │ For streaming  │
│ and alerting  │  │ archival      │  │ to SIEM tools │
└───────────────┘  └───────────────┘  └───────────────┘
```

### How ALZ Handles Diagnostic Settings Automatically

Remember DINE policies from Day 2?  
ALZ uses DINE policies to automatically configure diagnostic settings  
on every resource deployed into a landing zone.

When a VM is created -- DINE policy fires -- diagnostic settings configured automatically.  
When a Key Vault is created -- DINE policy fires -- logs start flowing to workspace.  
The app team does not configure anything. It is done for them.

This is the enforcement mechanism that ensures nothing goes unmonitored  
regardless of whether the app team remembers to configure it.

---

## Section 4 -- AMBA-ALZ (Azure Monitor Baseline Alerts for ALZ)

### What It Is

AMBA-ALZ is a Microsoft-maintained collection of pre-built alert rules  
specifically designed for Azure Landing Zone deployments.

Think of it as a ready-made monitoring kit.  
Instead of the platform team designing alert thresholds from scratch  
for every resource type, AMBA-ALZ provides tested, recommended  
baseline alerts that cover the most important scenarios.

### What AMBA-ALZ Covers

```
AMBA-ALZ Alert Categories:
│
├── Connectivity Alerts
│   ├── ExpressRoute circuit bandwidth utilisation > 90%
│   ├── VPN Gateway connection dropped
│   ├── Azure Firewall SNAT port exhaustion
│   └── DNS resolution failures
│
├── Management Alerts
│   ├── Log Analytics workspace data ingestion anomaly
│   ├── Azure Monitor agent heartbeat failure
│   └── Automation account job failure
│
├── Identity Alerts
│   ├── Domain Controller CPU > 85%
│   ├── AD replication failure
│   └── Domain Controller disk space < 10%
│
├── Landing Zone Alerts
│   ├── VM CPU > 95% for 15 minutes
│   ├── VM available memory < 10%
│   ├── Storage account availability < 99%
│   └── Key Vault availability < 99%
│
└── Service Health Alerts
    ├── Azure service outage in your regions
    ├── Planned maintenance notifications
    └── Security advisory notifications
```

### How AMBA-ALZ Is Deployed

AMBA-ALZ is deployed as a set of ARM templates or Bicep files  
that create all alert rules, action groups, and alert processing rules  
at the Management Group level.

All alerts feed into Action Groups that notify the platform team.  
The entire deployment is done once and covers all current and future landing zones.

---

## Section 5 -- Microsoft Defender for Cloud

### What It Is

Defender for Cloud is Azure's built-in security posture management tool.  
It continuously assesses your Azure resources against security best practices  
and gives you a Secure Score -- a percentage representing how secure your estate is.

### Two Modes

**Foundational CSPM (Free)**  
Basic security recommendations.  
Secure Score calculation.  
Available to every Azure subscription automatically.

**Defender Plans (Paid -- per resource type)**  
Advanced threat protection per resource category.  
Each plan covers a specific resource type:

| Defender Plan | Protects |
|---|---|
| Defender for Servers | VMs -- malware, vulnerability assessment, JIT VM access |
| Defender for Storage | Storage accounts -- malware scanning, sensitive data discovery |
| Defender for SQL | SQL databases -- SQL injection detection, anomalous access |
| Defender for Key Vault | Key Vault -- unusual access patterns, credential theft |
| Defender for Containers | AKS clusters -- runtime threat protection |
| Defender for App Service | Web apps -- web attack detection |

### Secure Score

The Secure Score is a percentage calculated from how many  
security recommendations you have implemented vs how many apply to you.

```
Secure Score Example: 67%

Recommendations implemented: 134
Recommendations not implemented: 66
        |
        v
Each unimplemented recommendation has:
- Severity: High / Medium / Low
- Impact on score if fixed: e.g. +2%
- Remediation steps: exactly what to do to fix it
- Quick fix option: one-click remediation for some recommendations
```

### How Defender for Cloud Fits in ALZ

In ALZ, Defender for Cloud is enabled at the Management Group level  
via Azure Policy (a DINE policy deploys the Defender plans to every subscription).  
Security alerts from Defender flow to:
- The Security subscription's Log Analytics Workspace
- Microsoft Sentinel for SIEM correlation
- Action Groups for immediate notification

The Security Team has Reader access across the entire estate  
so they can see Defender recommendations for every landing zone  
without having resource-level access.

---

## Section 6 -- Microsoft Sentinel

### What It Is

Microsoft Sentinel is Azure's cloud-native SIEM  
(Security Information and Event Management) and SOAR  
(Security Orchestration, Automation, and Response) tool.

It sits in the Security subscription and connects to the  
security Log Analytics Workspace.

### What It Does

```
Data Sources feeding into Sentinel:
│
├── Azure Activity Logs (all subscriptions)
├── Microsoft Defender for Cloud alerts
├── Azure Firewall logs
├── Entra ID sign-in logs (failed logins, risky sign-ins)
├── Microsoft 365 logs (if connected)
└── Third-party security tools (via connectors)
        |
        v
Sentinel analyses all data using:
- Built-in analytics rules (known attack patterns)
- Machine learning anomaly detection
- Threat intelligence feeds
        |
        v
Incidents created for security team to investigate
        |
        v
Playbooks (automated responses) triggered:
- Block IP address automatically
- Disable compromised user account
- Notify security team via Teams
```

### In Plain English for a Client

*"Sentinel is the security operations brain. All security events from every  
subscription flow into it. It correlates events that look innocent individually  
but together indicate an attack. When it finds something, it creates an incident  
and can automatically respond -- for example, blocking a suspicious IP address  
before anyone even wakes up."*

---

## Section 7 -- The Full Monitoring Flow in ALZ

```
Resource deployed in Landing Zone A2
        |
        v
DINE policy auto-configures diagnostic settings
        |
        v
Logs flow to:
├── Landing Zone Log Analytics Workspace (app-level logs)
└── Platform Log Analytics Workspace (platform-level activity)
        |
        v
Azure Monitor evaluates alert rules (from AMBA-ALZ)
        |
        v
Alert condition met (e.g. VM CPU > 95% for 15 minutes)
        |
        v
Action Group triggered
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

## Hands-On Tasks -- Do These Today

### Task 1: Explore Log Analytics Workspace in the Portal (15 minutes)

1. Go to [portal.azure.com](https://portal.azure.com)
2. Search for **Log Analytics workspaces**
3. Open any existing workspace or create a free trial one
4. Click **Logs** in the left menu
5. In the query editor, type this and run it:
   ```kql
   AzureActivity
   | take 10
   ```
6. Look at the columns returned -- note Caller, OperationName, ActivityStatus, ResourceGroup
7. Modify the query to filter failed operations:
   ```kql
   AzureActivity
   | where ActivityStatus == "Failed"
   | take 10
   ```
8. You are not learning KQL today -- you are getting comfortable that this is how logs are queried

---

### Task 2: Explore Azure Monitor Alerts (10 minutes)

1. Search for **Monitor** in the portal
2. Click **Alerts** in the left menu
3. Click **Alert rules**
4. If any exist, open one and read:
   - What condition triggers it?
   - What Action Group does it use?
   - What is the severity level?
5. Click **Action groups** in the left menu
6. Open or create one and observe what notification types are available

---

### Task 3: Explore Diagnostic Settings on a Resource (10 minutes)

1. Navigate to any Azure resource (a VM, Key Vault, or Storage Account)
2. Look for **Diagnostic settings** in the left menu
3. Observe:
   - Is it currently enabled or disabled?
   - What log categories are available?
   - What destination options are shown (Log Analytics, Storage, Event Hub)?
4. Note: in ALZ this would be auto-configured by a DINE policy

---

### Task 4: Explore Defender for Cloud (15 minutes)

1. Search for **Microsoft Defender for Cloud** in the portal
2. Look at the **Secure Score** on the overview page
3. Click **Recommendations** in the left menu
4. Filter by **Severity: High**
5. Open any recommendation and read:
   - What is the security risk?
   - What resources are affected?
   - What are the remediation steps?
   - Is there a Quick Fix option?
6. Click **Security alerts** in the left menu
7. Note how alerts are categorised by severity and tactic

---

### Task 5: Explore AMBA-ALZ on GitHub (10 minutes)

1. Go to: [https://azure.github.io/azure-monitor-baseline-alerts/alz](https://azure.github.io/azure-monitor-baseline-alerts/alz)
2. Browse the alert categories available
3. Click any alert and read:
   - What metric or log condition triggers it?
   - What is the recommended threshold?
   - What severity is it assigned?
4. This gives you a feel for the breadth of what AMBA-ALZ covers out of the box

---

## GitHub Reference Pages for Day 5

| Resource | URL | What to Use It For |
|---|---|---|
| AMBA-ALZ Official Site | [https://azure.github.io/azure-monitor-baseline-alerts/alz](https://azure.github.io/azure-monitor-baseline-alerts/alz) | Browse all baseline alert definitions |
| AMBA-ALZ GitHub Repository | [https://github.com/Azure/azure-monitor-baseline-alerts](https://github.com/Azure/azure-monitor-baseline-alerts) | Deployment templates for AMBA-ALZ |
| AVM Log Analytics Workspace Module | [https://github.com/Azure/bicep-registry-modules/tree/main/avm/res/operational-insights/workspace](https://github.com/Azure/bicep-registry-modules/tree/main/avm/res/operational-insights/workspace) | How Log Analytics is deployed as code in ALZ |
| AVM Defender for Cloud Module | [https://github.com/Azure/bicep-registry-modules/tree/main/avm/res/security/defender-for-cloud](https://github.com/Azure/bicep-registry-modules/tree/main/avm/res/security/defender-for-cloud) | Defender deployment as code |
| Azure Monitor Documentation | [https://learn.microsoft.com/en-us/azure/azure-monitor/overview](https://learn.microsoft.com/en-us/azure/azure-monitor/overview) | Full Azure Monitor conceptual documentation |
| Defender for Cloud Documentation | [https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-cloud-introduction](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-cloud-introduction) | Defender for Cloud overview |
| Microsoft Sentinel Documentation | [https://learn.microsoft.com/en-us/azure/sentinel/overview](https://learn.microsoft.com/en-us/azure/sentinel/overview) | Sentinel overview and architecture |
| ALZ Monitoring Design (CAF) | [https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/management-monitor](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/management-monitor) | Official ALZ monitoring design guidance |

---

## Self-Test Questions and Correct Answers

Answer each question without notes first. Then check below.

---

### Question 1: Why does ALZ have two separate Log Analytics Workspaces instead of one central one?

**Correct Answer:**  
One workspace lives in the Management subscription and collects platform-level logs --  
Azure Activity Logs, policy events, firewall logs, gateway logs, Defender alerts.  
The platform team owns and accesses this.  
A second workspace lives in each Landing Zone subscription and collects  
application-level logs -- workload performance, app diagnostics, VM logs.  
The app team owns this and the platform team has Reader access.  
Separation exists for two reasons: data sovereignty (app teams should not see  
platform infrastructure logs and vice versa) and access control  
(each workspace has its own RBAC, retention policy, and cost attribution).

---

### Question 2: What are diagnostic settings and why does ALZ configure them automatically instead of relying on app teams to do it?

**Correct Answer:**  
Diagnostic settings tell an Azure resource where to send its logs and metrics.  
They are off by default on every Azure resource.  
ALZ uses DINE policies to configure diagnostic settings automatically  
the moment a resource is deployed into a landing zone.  
This ensures nothing goes unmonitored regardless of whether the app team  
remembers to configure it manually.  
The same problem as ungoverned resource deployment applies here --  
if you rely on humans to configure monitoring consistently, some will forget,  
some will configure it differently, and audit reports will have gaps.  
DINE policies eliminate the human dependency entirely.

---

### Question 3: What is an Action Group and how does it relate to alert rules in Azure Monitor?

**Correct Answer:**  
An Action Group is a reusable collection of notification targets and automated actions  
that gets triggered when an alert rule fires.  
An alert rule defines the condition -- for example CPU above 95% for 15 minutes.  
The Action Group defines what happens when that condition is met --  
send an email to the operations team, send an SMS to the on-call engineer,  
create a ServiceNow ticket, trigger an Azure Function for auto-remediation.  
One Action Group can be attached to many alert rules.  
In ALZ, Action Groups are deployed centrally in the Management subscription  
and referenced by alert rules across all subscriptions and landing zones.

---

### Question 4: What is AMBA-ALZ and what problem does it solve for the platform team?

**Correct Answer:**  
AMBA-ALZ is Azure Monitor Baseline Alerts for Landing Zones --  
a Microsoft-maintained collection of pre-built, tested alert rules  
specifically designed for ALZ deployments.  
The problem it solves is that designing alert thresholds from scratch  
for every resource type across a full ALZ estate is time-consuming  
and requires deep expertise to get right.  
Without AMBA-ALZ the platform team would have to decide: what CPU threshold  
triggers a VM alert? What is the right threshold for ExpressRoute bandwidth?  
When should a DNS failure alert fire?  
AMBA-ALZ provides Microsoft-recommended answers to all of those questions  
in a ready-to-deploy package that covers connectivity, management,  
identity, landing zone resources, and service health out of the box.

---

### Question 5: What is the difference between Azure Monitor and Microsoft Sentinel?

**Correct Answer:**  
Azure Monitor collects and analyses operational data -- performance metrics,  
resource logs, activity logs -- and fires alerts based on defined conditions.  
It answers: is my infrastructure healthy and performing correctly?  
Microsoft Sentinel is a SIEM that collects security events from across the estate,  
correlates them using analytics rules and machine learning,  
and creates security incidents for investigation.  
It answers: is my estate under attack or behaving suspiciously?  
Azure Monitor is for operational monitoring. Sentinel is for security monitoring.  
They are complementary -- Monitor feeds operational logs,  
Sentinel correlates security events. Both use Log Analytics as the underlying store.

---

### Question 6: A client asks: "How do we know if one of our Azure services goes down in our region before our users call to complain?" What is your answer?

**Correct Answer:**  
AMBA-ALZ includes Service Health alerts that are deployed as part of the baseline.  
Service Health monitors Azure platform health in your specific regions and  
notifies you of three types of events:  
Service issues -- active outages affecting Azure services in your region.  
Planned maintenance -- upcoming maintenance windows that may affect your resources.  
Security advisories -- security events that may require action on your part.  
These alerts are configured at the Management Group level and fire to the  
central Action Group in the Management subscription --  
meaning the platform team is notified before users are impacted,  
giving time to activate business continuity procedures or communicate proactively.

---

### Question 7: What is Defender for Cloud's Secure Score and what does it tell a client's CISO?

**Correct Answer:**  
The Secure Score is a percentage that represents how many of the applicable  
security recommendations for your Azure estate have been implemented.  
A score of 67% means 33% of recommended security controls are not yet in place.  
For a CISO it tells three things:  
First, the current security posture of the entire Azure estate in a single number  
that can be tracked over time.  
Second, a prioritised list of what to fix next -- each unimplemented recommendation  
shows its severity, the resources affected, and the exact remediation steps.  
Third, a measurable improvement target -- implementing specific recommendations  
shows exactly how much the score will improve, giving the security team  
a quantifiable roadmap rather than vague security guidance.  
In ALZ, Defender for Cloud is enabled at the Management Group level  
so the Secure Score covers every landing zone subscription automatically.

---

## Day 5 Completion Checklist

- [ ] Two Log Analytics Workspaces -- purpose of each understood
- [ ] Diagnostic settings -- what they are and why DINE policies configure them
- [ ] Azure Monitor four components known: Metrics, Logs, Alerts, Dashboards
- [ ] Action Groups -- what they are and how they relate to alert rules
- [ ] AMBA-ALZ -- what it is and what categories it covers
- [ ] Defender for Cloud -- Secure Score understood and explainable to a CISO
- [ ] Microsoft Sentinel -- distinction from Azure Monitor is clear
- [ ] Full monitoring flow from resource deployment to security incident understood
- [ ] All 5 portal tasks completed
- [ ] AMBA-ALZ GitHub site browsed
- [ ] All 7 self-test questions answered without notes before checking
- [ ] Score yourself -- 80 and above means proceed to Day 6

---

## Day 6 Preview -- Subscription Vending and DevOps Pipeline

- What subscription vending is and how the pipeline works end to end
- Azure DevOps vs GitHub Actions for ALZ deployment
- How AVM modules are called from a pipeline
- The ALZ Bicep accelerator -- what it is and how to read it
- Hands-on: Open the ALZ-Bicep repository and trace a full deployment

Day 6 is where the IaC and DevOps pieces come together.  
Everything from Days 1 to 5 gets deployed by the pipeline on Day 6.

---

*Day 5 of 21 | Azure Landing Zone Study Track | March 2026*