# Module 9 — Detection Engineering, Threat Hunting & SOAR

## 1. Module Overview

This module focused on using Microsoft Sentinel and Azure security telemetry to detect, investigate, and hunt for potentially suspicious activity.

The implementation built on the monitoring and telemetry pipeline established in Module 8.

The primary security scenario was monitoring Azure RBAC role-assignment changes because unauthorized changes to Azure permissions can result in privilege escalation and increased access to cloud resources.

The module covered:

- KQL security queries
- Detection engineering
- Microsoft Sentinel analytics rules
- MITRE ATT&CK mapping
- Controlled detection validation
- Threat hunting
- Incident-generation configuration
- SOAR and automation assessment

---

## 2. Security Scenario

Azure Role-Based Access Control (RBAC) determines which identities can access Azure resources and what actions they can perform.

An unexpected role assignment could allow an attacker or compromised administrator to increase an identity's privileges.

The detection scenario therefore focused on monitoring successful Azure RBAC role-assignment changes.

Relevant Azure operation:

`MICROSOFT.AUTHORIZATION/ROLEASSIGNMENTS/WRITE`

---

## 3. KQL Detection Engineering

Azure Activity Log telemetry ingested into the Log Analytics workspace was queried using Kusto Query Language (KQL).

The following query identifies successful Azure RBAC role-assignment changes:

```kusto
AzureActivity
| where OperationNameValue == "MICROSOFT.AUTHORIZATION/ROLEASSIGNMENTS/WRITE"
| where ActivityStatusValue == "Success"
| project TimeGenerated, Caller, ResourceGroup, ResourceId
```

This provides visibility into:

- When the role assignment occurred
- Which identity performed the change
- Which resource group was affected
- Which Azure resource was involved

The query was successfully validated against real Azure Activity Log telemetry generated within the lab environment.

---

## 4. Microsoft Sentinel Analytics Rule

A scheduled Microsoft Sentinel analytics rule was created.

**Rule Name**

`CloudSec - Azure RBAC Role Assignment Change`

**Description**

Detects successful Azure RBAC role assignment changes. Unexpected role assignments may indicate privilege escalation or unauthorized access changes.

**Severity**

Medium

**Rule Type**

Scheduled

**Status**

Enabled

The rule uses the KQL detection query to identify successful RBAC role-assignment changes.

At creation, the rule was configured to:

- Run every 5 minutes
- Look back over the previous 5 minutes
- Trigger when more than 0 matching results are returned
- Generate an alert for each event
- Create incidents from alerts

---

## 5. MITRE ATT&CK Mapping

The analytics rule was mapped to the MITRE ATT&CK tactic:

**Privilege Escalation**

This reflects the security risk associated with unauthorized Azure RBAC changes.

An attacker who obtains sufficient Azure permissions could potentially assign additional roles to themselves or another identity, increasing access within the cloud environment.

No specific MITRE ATT&CK technique was configured during this exercise.

---

## 6. Controlled Detection Validation

A controlled RBAC change was performed to generate realistic security telemetry.

At resource-group scope:

`rg-cloudsec-core-uks-01`

the identity:

`Entra Project2`

was assigned the:

`Reader`

role.

The resulting RBAC operation was successfully recorded in the `AzureActivity` table.

The successful event included:

`MICROSOFT.AUTHORIZATION/ROLEASSIGNMENTS/WRITE`

with:

`ActivityStatusValue = Success`

This validated the following detection path:

```text
Azure RBAC Change
        |
        v
Azure Activity Log
        |
        v
Log Analytics Workspace
        |
        v
AzureActivity Table
        |
        v
KQL Detection Query
        |
        v
Microsoft Sentinel Analytics Rule
```

The underlying telemetry and KQL detection logic were successfully validated.

Alert and incident generation were not successfully validated during the lab session.

---

## 7. Threat Hunting

A threat-hunting query was developed to identify successful Azure RBAC role-assignment changes and summarize the activity by caller and resource group.

```kusto
AzureActivity
| where TimeGenerated > ago(24h)
| where OperationNameValue =~ "MICROSOFT.AUTHORIZATION/ROLEASSIGNMENTS/WRITE"
| where ActivityStatusValue =~ "Success"
| summarize
    RoleChanges=count(),
    FirstSeen=min(TimeGenerated),
    LastSeen=max(TimeGenerated)
    by Caller, ResourceGroup
| order by RoleChanges desc
```

The query successfully returned activity from the lab environment.

The hunt identified:

- 3 successful RBAC role-assignment changes
- The identity responsible for the changes
- The affected resource group
- The first observed change
- The most recent change

This demonstrates how a security analyst can proactively hunt for privilege-management activity rather than relying exclusively on generated alerts.

---

## 8. Incident Validation

The Sentinel analytics rule was configured with incident creation enabled.

However, an associated Microsoft Sentinel incident was not successfully validated during the lab session.

Queries against the `SecurityAlert` table did not return an alert generated by the RBAC analytics rule during the validation period.

Additionally, navigation to the unified Microsoft Sentinel Incidents experience redirected the lab tenant to the Microsoft Defender for Business experience.

Therefore:

- **Analytics rule configuration:** Validated
- **Underlying RBAC telemetry:** Validated
- **KQL detection logic:** Validated
- **SecurityAlert generation:** Not validated
- **Sentinel incident generation:** Not validated

No claim is made that a Microsoft Sentinel incident was successfully generated.

---

## 9. SOAR and Automation Assessment

Microsoft Sentinel automation was assessed as part of this module.

The Azure portal displayed a notification explaining that the Microsoft Sentinel Automation experience had moved to the Microsoft Defender portal.

The lab tenant's Defender portal presented the Microsoft Defender for Business experience rather than the required Sentinel automation configuration experience.

As a result:

- Sentinel Automation Rule — Not implemented
- Logic App playbook — Not implemented
- Automated response execution — Not validated

This environmental limitation was documented rather than changing unrelated Defender for Business configuration solely to force SOAR deployment.

### Intended SOAR Architecture

A production implementation could use the following workflow:

```text
RBAC Role Assignment Change
          |
          v
Microsoft Sentinel Analytics Rule
          |
          v
Security Alert / Incident
          |
          v
Sentinel Automation Rule
          |
          v
Logic App Playbook
          |
          v
SOC Notification / Enrichment / Response
```

Potential automated actions could include:

- Notify the SOC when a privilege change occurs
- Enrich the incident with identity information
- Record the affected Azure resource
- Escalate high-risk administrative changes
- Trigger an approval workflow for investigation
- Create a ticket for SOC investigation

No automated remediation action was implemented in this lab.

---

## 10. Module Architecture

```text
                    Azure Subscription
                           |
                           v
                    Azure RBAC Change
                           |
                           v
                   Azure Activity Log
                           |
                           v
                Log Analytics Workspace
                law-cloudsec-core-uks-01
                           |
                           v
                     AzureActivity
                           |
                +----------+----------+
                |                     |
                v                     v
          KQL Threat Hunt      Sentinel Analytics
                                      Rule
                                       |
                                       v
                              MITRE ATT&CK Mapping
                              Privilege Escalation
                                       |
                                       v
                              Alert / Incident
                               Not Validated
                                       |
                                       v
                              SOAR / Automation
                              Not Implemented
```

---

## 11. Evidence

### Evidence 1 — RBAC Threat Hunting

**Filename**

`M9-Threat-Hunting-RBAC-Role-Changes.png`

**Description**

KQL-based threat hunting against Azure Activity telemetry identified successful Azure RBAC role-assignment changes.

The query summarized:

- Number of successful role changes
- Identity responsible for the changes
- Affected resource group
- First observed activity
- Most recent activity

This demonstrates proactive investigation of privilege-management activity using Azure security telemetry.

---

### Evidence 2 — SOAR Automation Portal Limitation

**Filename**

`M9-SOAR-Automation-Portal-Redirect.png`

**Description**

Microsoft Sentinel Automation in the Azure portal displayed the migration notice directing automation management to the Microsoft Defender portal.

The lab tenant subsequently presented the Microsoft Defender for Business experience, preventing practical Sentinel Automation Rule and Logic App playbook configuration during this exercise.

This evidence documents the environmental limitation rather than representing SOAR as successfully implemented.

---

## 12. Security Engineering Outcomes

This module demonstrated practical experience with:

- Writing KQL queries for security telemetry
- Querying Azure Activity Log data
- Identifying security-relevant Azure control-plane operations
- Developing Microsoft Sentinel detection logic
- Creating a scheduled analytics rule
- Enabling the analytics rule
- Mapping a detection to MITRE ATT&CK
- Generating controlled Azure activity for testing
- Validating detection logic against real telemetry
- Performing proactive threat hunting
- Investigating Azure privilege-management activity
- Understanding the relationship between telemetry, detections, alerts and incidents
- Assessing Microsoft Sentinel SOAR architecture
- Understanding the purpose of Automation Rules and Logic App playbooks
- Documenting environmental limitations without overstating implementation

The practical detection-engineering lifecycle demonstrated was:

**Collect → Query → Detect → Validate → Hunt**

The intended extended SOC lifecycle is:

**Collect → Query → Detect → Validate → Hunt → Investigate → Automate → Respond**

The final incident and automation stages could not be practically validated in this lab environment.

---

## 13. Production Relevance

In a production Azure environment, RBAC changes are high-value security events because changes to authorization can increase an identity's access to sensitive cloud resources.

A mature security monitoring implementation would combine:

- Azure Activity Log collection
- Centralized Log Analytics ingestion
- KQL detection logic
- Microsoft Sentinel analytics rules
- MITRE ATT&CK mapping
- Incident investigation
- Threat hunting
- Automated enrichment
- SOAR playbooks
- SOC escalation procedures

Detection logic should also be tuned to distinguish legitimate administrative activity from unexpected or unauthorized privilege changes.

Automation should be introduced carefully to avoid automatically removing legitimate access without sufficient investigation or approval.

---

## 14. Lessons Learned

Key engineering lessons from this module include:

1. Security telemetry must be successfully collected before detections can operate.

2. A detection rule should be tested against known activity rather than assumed to work because configuration succeeded.

3. KQL can be used both for automated detection logic and proactive threat hunting.

4. Azure RBAC changes provide valuable telemetry for identifying potential privilege escalation.

5. Successful telemetry ingestion does not automatically prove that alert or incident generation is functioning.

6. Environmental, licensing, portal, and tenant limitations should be documented explicitly.

7. Security engineering documentation should distinguish between configured, validated, and unimplemented controls.

---

## 15. Module Status

**COMPLETE — with documented environmental limitations**

Successfully implemented and validated:

- Azure Activity security telemetry
- KQL detection logic
- Microsoft Sentinel scheduled analytics rule
- MITRE ATT&CK Privilege Escalation mapping
- Controlled RBAC detection testing
- KQL-based threat hunting

Configured but not successfully validated:

- Security alert generation
- Microsoft Sentinel incident generation

Assessed but not implemented due to the lab portal limitation:

- Sentinel Automation Rule
- Logic App playbook
- Automated SOAR response

The limitations are explicitly documented and no unvalidated capability is represented as successfully implemented.
