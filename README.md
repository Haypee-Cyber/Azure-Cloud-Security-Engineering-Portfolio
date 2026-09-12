# Module 8 - Azure Monitoring & Microsoft Sentinel

## 1. Overview

This module implements centralized security monitoring and SIEM capabilities for the Azure Cloud Security Engineering environment.

The objective was to collect security and operational telemetry from Azure resources, centralize the data in Log Analytics, validate ingestion using KQL, and enable Microsoft Sentinel for security monitoring and investigation.

The implemented monitoring architecture is:

**Azure Resources → Diagnostic Settings → Log Analytics Workspace → Microsoft Sentinel**

---

## 2. Objectives

The objectives of this module were to:

- Deploy a centralized Log Analytics workspace.
- Configure Azure subscription Activity Log collection.
- Configure resource-level diagnostic settings.
- Collect Key Vault and Storage security telemetry.
- Validate log ingestion using KQL.
- Enable Microsoft Sentinel on the existing Log Analytics workspace.
- Validate Sentinel data connectors and security telemetry.
- Demonstrate an end-to-end Azure monitoring and SIEM architecture.

---

## 3. Log Analytics Workspace

A centralized Log Analytics workspace was deployed for monitoring and security telemetry.

**Workspace:** `law-cloudsec-core-uks-01`  
**Resource Group:** `rg-cloudsec-core-uks-01`  
**Region:** UK South  
**Subscription:** `Azure-Cloud-Security-Lab`

The workspace acts as the central telemetry repository for the Azure security environment.

---

## 4. Azure Activity Log Collection

A subscription-level diagnostic setting was configured:

`ds-activitylog-to-law`

The following Activity Log categories were enabled:

- Administrative
- Security
- ServiceHealth
- Alert
- Recommendation
- Policy
- Autoscale
- ResourceHealth

The logs were configured to be sent to:

`law-cloudsec-core-uks-01`

Initial `AzureActivity` queries returned no records while ingestion was still pending.

Subsequent validation in Microsoft Sentinel confirmed successful ingestion, with **13 AzureActivity records received**.

This demonstrated the importance of accounting for telemetry ingestion latency when validating Azure monitoring configurations.

---

## 5. Resource Diagnostic Settings

Resource-level diagnostic logging was configured for security-sensitive Azure services.

### Azure Key Vault

**Resource:** `kv-cloudsec-app-uks-01`  
**Diagnostic setting:** `ds-keyvault-to-law`

Key Vault audit telemetry and metrics were configured for collection and sent to the Log Analytics workspace.

### Azure Storage

**Resource:** `stcloudsecappuks01`  
**Blob diagnostic setting:** `ds-storage-blob-to-law`

Storage data-plane logging was configured for:

- StorageRead
- StorageWrite
- StorageDelete
- Transaction metrics

This provides visibility into operations performed against Azure Blob Storage.

---

## 6. Key Vault Telemetry Validation

A legitimate Key Vault operation was generated from:

`vm-cloudsec-app-uks-01`

The VM used its **system-assigned Managed Identity** to authenticate to Azure Key Vault through the private-access architecture implemented previously.

The secret access operation returned **HTTP 200 OK**.

KQL was then used to validate ingestion:

```kusto
AzureDiagnostics
| where TimeGenerated > ago(1h)
| where ResourceProvider == "MICROSOFT.KEYVAULT"
| project TimeGenerated, OperationName, ResultType, Resource, CallerIPAddress
| order by TimeGenerated desc
```

The query returned successful `VaultGet` operations for:

`KV-CLOUDSEC-APP-UKS-01`

This validated the following telemetry path:

**Managed Identity → Private Endpoint → Key Vault → Diagnostic Settings → Log Analytics**

### Evidence

![Key Vault audit log ingestion validation](Evidence/M8-T4-KeyVault-Audit-Log-Ingestion-Validated.png)

---

## 7. Storage Telemetry Validation

A legitimate Blob Storage read operation was generated from the application VM using its **system-assigned Managed Identity**.

Authentication was performed using Microsoft Entra ID rather than Storage Account Shared Key authentication.

The operation generated Storage data-plane telemetry which was successfully queried from Log Analytics.

This validated the following telemetry path:

**Managed Identity → Private Endpoint → Blob Storage → Diagnostic Settings → Log Analytics**

### Evidence

![Storage Blob log ingestion validation](Evidence/M8-T4-Storage-Blob-Log-Ingestion-Validated.png.png)

---

## 8. Microsoft Sentinel Enablement

Microsoft Sentinel was enabled on the existing Log Analytics workspace:

`law-cloudsec-core-uks-01`

A separate Log Analytics workspace was not created.

This extended the architecture from centralized telemetry collection into SIEM capabilities for security monitoring, detection, investigation and response.

The resulting architecture is:

```text
Azure Resources
      |
      v
Diagnostic Settings
      |
      v
Log Analytics Workspace
law-cloudsec-core-uks-01
      |
      v
Microsoft Sentinel
      |
      v
Security Monitoring
Detection
Investigation
Response
```

---

## 9. Microsoft Sentinel Security Telemetry Validation

### Azure Key Vault

The Azure Key Vault connector was validated in Microsoft Sentinel.

The connector showed:

- Status: Connected
- Microsoft provider
- Key Vault telemetry received
- `AzureDiagnostics` records available
- Recent Key Vault data ingestion

This confirmed that Key Vault security telemetry was available to Microsoft Sentinel.

### Evidence

![Microsoft Sentinel Key Vault connector telemetry](Evidence/M8-T6-Sentinel-KeyVault-Connector-Telemetry.png)

---

### Azure Activity

The Azure Activity connector was validated in Microsoft Sentinel.

The connector showed:

- Status: Connected
- Microsoft provider
- Recent Activity Log ingestion
- `AzureActivity` telemetry available
- **13 AzureActivity records received during validation**

This confirmed successful subscription-level Activity Log ingestion into the Sentinel-enabled Log Analytics workspace.

### Evidence

![Microsoft Sentinel Azure Activity telemetry validation](Evidence/M8-T6-Sentinel-AzureActivity-Telemetry-Validated.png)

---

## 10. Security Monitoring Architecture

The completed monitoring architecture is:

```text
                         Azure Subscription
                                |
               +----------------+----------------+
               |                                 |
               v                                 v
       Azure Activity Log                 Azure Resources
                                         /              \
                                        v                v
                                   Key Vault          Storage
                                        \                /
                                         \              /
                                          v            v
                                        Diagnostic Settings
                                                |
                                                v
                                    Log Analytics Workspace
                                    law-cloudsec-core-uks-01
                                                |
                                                v
                                       Microsoft Sentinel
                                                |
                        +-----------------------+-----------------------+
                        |                       |                       |
                        v                       v                       v
                   Monitoring               Detection            Investigation
```

This architecture centralizes security telemetry while maintaining the private-access, Managed Identity and Zero-Trust controls implemented in previous modules.

---

## 11. Security Engineering Outcomes

This module demonstrated practical experience in:

- Designing centralized Azure monitoring architecture.
- Deploying and configuring Log Analytics.
- Exporting subscription-level Azure Activity Logs.
- Configuring resource-level diagnostic settings.
- Collecting Key Vault security audit telemetry.
- Collecting Azure Storage Blob data-plane telemetry.
- Generating controlled activity to validate monitoring.
- Querying security telemetry using KQL.
- Troubleshooting telemetry ingestion.
- Understanding Azure telemetry ingestion latency.
- Enabling Microsoft Sentinel on an existing workspace.
- Validating Microsoft Sentinel data connectors.
- Confirming end-to-end security telemetry ingestion.

---

## 12. Key Engineering Lessons

### Configuration Does Not Equal Validation

A diagnostic setting being present does not prove that telemetry is successfully reaching the monitoring platform.

Actual resource operations were therefore generated and the resulting telemetry was queried to confirm end-to-end ingestion.

### Telemetry Can Have Ingestion Latency

The Azure Activity diagnostic configuration was present while initial `AzureActivity` queries returned no records.

Later Microsoft Sentinel validation confirmed successful ingestion.

Monitoring implementations must therefore account for ingestion latency before treating missing telemetry as a configuration failure.

### Identity-Based Access Supports Stronger Security

Key Vault and Storage validation used Managed Identity and Microsoft Entra authorization rather than embedded credentials or Storage Shared Keys.

This maintained the least-privilege and Zero-Trust architecture implemented in earlier modules.

### Monitoring Completes the Security Control Lifecycle

Previous modules concentrated primarily on securing and restricting access to Azure resources.

This module introduced centralized visibility into activity occurring within the environment.

The security lifecycle therefore progresses from:

**Prevent → Monitor → Detect → Investigate → Respond**

---

## 13. Evidence Register

| Evidence | Validation |
|---|---|
| `M8-T4-KeyVault-Audit-Log-Ingestion-Validated.png` | KQL validation of successful Key Vault audit telemetry ingestion. |
| `M8-T4-Storage-Blob-Log-Ingestion-Validated.png.png` | Validation of Azure Storage Blob data-plane telemetry ingestion. |
| `M8-T6-Sentinel-KeyVault-Connector-Telemetry.png` | Microsoft Sentinel Azure Key Vault connector connected and receiving telemetry. |
| `M8-T6-Sentinel-AzureActivity-Telemetry-Validated.png` | Microsoft Sentinel Azure Activity connector connected with 13 AzureActivity records received during validation. |

---

## 14. Module Status

**Status: COMPLETE**

Azure monitoring and Microsoft Sentinel capabilities have been successfully implemented and validated.

The environment now provides centralized telemetry collection through Log Analytics and SIEM capabilities through Microsoft Sentinel.

This establishes the monitoring foundation required for the next stage of the portfolio:

**Module 9 — Detection Engineering, Threat Hunting & SOAR**
