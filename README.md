# Azure-Cloud-Security-Engineering-Portfolio
Hands-on Azure Cloud Security engineering portfolio demonstrating secure architecture, network security, workload protection, Zero Trust, governance, Infrastructure as Code, DevSecOps, Microsoft Sentinel, detection engineering and incident response.
# Module 8 - Azure Monitoring & Microsoft Sentinel

## 1. Overview

This module implements centralized security monitoring and SIEM capabilities for the Azure Cloud Security Engineering environment.

The objective was to collect security and operational telemetry from Azure resources, centralize the data in Log Analytics, validate ingestion using KQL, and enable Microsoft Sentinel for security monitoring and investigation.

The implementation demonstrates an end-to-end monitoring architecture:

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

The workspace acts as the central telemetry repository for the lab environment.

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

Logs were configured to be sent to:

`law-cloudsec-core-uks-01`

The diagnostic configuration was validated through both the Azure Portal and its underlying JSON configuration.

Initial queries returned no `AzureActivity` records while ingestion was still pending. Subsequent Microsoft Sentinel validation confirmed successful ingestion, with 13 AzureActivity records received.

This demonstrated the importance of accounting for telemetry ingestion latency when validating Azure monitoring configurations.

---

## 5. Resource Diagnostic Settings

Resource-level diagnostic logging was configured for security-sensitive Azure services.

### Azure Key Vault

Resource:

`kv-cloudsec-app-uks-01`

Diagnostic setting:

`ds-keyvault-to-law`

Key Vault audit telemetry and metrics were configured for collection and sent to the Log Analytics workspace.

### Azure Storage

Resource:

`stcloudsecappuks01`

Blob service diagnostic setting:

`ds-storage-blob-to-law`

Storage data-plane logging was configured for:

- StorageRead
- StorageWrite
- StorageDelete
- Transaction metrics

This provides visibility into operations performed against Blob Storage.

---

## 6. Key Vault Telemetry Validation

A legitimate Key Vault operation was generated from:

`vm-cloudsec-app-uks-01`

The VM used its system-assigned Managed Identity to authenticate to Azure Key Vault through the existing private-access architecture.

The secret access operation returned HTTP 200.

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

This confirmed:

**Managed Identity → Private Endpoint → Key Vault → Diagnostic Settings → Log Analytics**

![Key Vault audit log validation](Evidence/M8-T4-KeyVault-Audit-Log-Ingestion-Validated.png)

---

## 7. Storage Telemetry Validation

A legitimate Blob Storage read operation was generated from the application VM using its system-assigned Managed Identity.

Authentication was performed using Microsoft Entra ID rather than Storage Account Shared Key authentication.

The operation generated Storage data-plane telemetry which was successfully queried from Log Analytics.

This validated:

**Managed Identity → Private Endpoint → Blob Storage → Diagnostic Settings → Log Analytics**

![Storage Blob log validation](Evidence/M8-T4-Storage-Blob-Log-Ingestion-Validated.png)

---

## 8. Microsoft Sentinel Enablement

Microsoft Sentinel was enabled on the existing Log Analytics workspace:

`law-cloudsec-core-uks-01`

A separate workspace was not created.

This extended the monitoring architecture from centralized log collection into SIEM capabilities for security monitoring, investigation, detection, and response.

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

## 9. Sentinel Security Telemetry Validation

### Azure Key Vault Connector

The Azure Key Vault connector was validated in Microsoft Sentinel.

The connector showed:

- Status: Connected
- Key Vault telemetry received
- AzureDiagnostics records available
- Recent Key Vault data ingestion

This confirmed that Key Vault security telemetry was available to Microsoft Sentinel.

![Sentinel Key Vault connector](Evidence/M8-T6-Sentinel-KeyVault-Connector-Telemetry.png)

### Azure Activity Connector

The Azure Activity connector was also validated.

The connector showed:

- Status: Connected
- AzureActivity telemetry received
- 13 AzureActivity records available during validation
- Recent Activity Log ingestion

This confirmed successful subscription-level Activity Log ingestion into the Sentinel-enabled workspace.

![Sentinel Azure Activity telemetry](Evidence/M8-T6-Sentinel-AzureActivity-Telemetry-Validated.png)

---

## 10. Security Architecture

The completed monitoring architecture is:

```text
                         Azure Subscription
                                |
              +-----------------+-----------------+
              |                                   |
              v                                   v
       Azure Activity Log                   Azure Resources
                                            /            \
                                           v              v
                                      Key Vault        Storage
                                           \              /
                                            \            /
                                             v          v
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
                     Monitoring              Detection             Investigation
```

The architecture centralizes security telemetry while maintaining the private-access and Managed Identity controls implemented in previous modules.

---

## 11. Security Engineering Outcomes

This module demonstrated the ability to:

- Design centralized Azure monitoring architecture.
- Deploy and configure Log Analytics.
- Export subscription-level Azure Activity Logs.
- Configure resource-level diagnostic settings.
- Collect Key Vault security audit telemetry.
- Collect Storage Blob data-plane telemetry.
- Generate controlled test activity for validation.
- Query security telemetry using KQL.
- Troubleshoot telemetry ingestion delays.
- Enable Microsoft Sentinel on an existing workspace.
- Validate Sentinel data connectors.
- Confirm end-to-end security telemetry ingestion.

---

## 12. Key Engineering Lessons

### Configuration does not equal validation

A diagnostic setting being present does not prove telemetry is being collected.

The implementation therefore validated actual operations through KQL and Sentinel rather than relying solely on configuration screenshots.

### Telemetry can have ingestion latency

Azure Activity Log configuration was correct while initial `AzureActivity` queries returned no records.

Later Sentinel validation confirmed successful ingestion.

Operational monitoring should therefore account for expected ingestion delays before treating missing telemetry as a configuration failure.

### Identity-based access supports stronger security

Key Vault and Storage validation used Managed Identity and Microsoft Entra authorization rather than embedded credentials or Storage Shared Keys.

This maintains the least-privilege and Zero-Trust architecture implemented earlier in the project.

### Monitoring completes the security control lifecycle

Previous modules focused on preventing unauthorized access.

This module added visibility into what actually happens within the environment.

The security lifecycle therefore progresses from:

**Prevent → Monitor → Detect → Investigate → Respond**

---

## 13. Evidence

| Evidence | Description |
|---|---|
| `M8-T4-KeyVault-Audit-Log-Ingestion-Validated.png` | KQL validation of successful Key Vault audit telemetry ingestion. |
| `M8-T4-Storage-Blob-Log-Ingestion-Validated.png` | Validation of Azure Storage Blob data-plane telemetry ingestion. |
| `M8-T6-Sentinel-KeyVault-Connector-Telemetry.png` | Microsoft Sentinel Azure Key Vault connector connected and receiving telemetry. |
| `M8-T6-Sentinel-AzureActivity-Telemetry-Validated.png` | Microsoft Sentinel Azure Activity connector connected with 13 AzureActivity records received during validation. |

---

## 14. Module Status

**Status: COMPLETE**

Azure monitoring and Microsoft Sentinel capabilities have been implemented and validated.

The environment now provides centralized telemetry collection through Log Analytics and SIEM capabilities through Microsoft Sentinel, establishing the monitoring foundation required for detection engineering, threat hunting, and incident response.
