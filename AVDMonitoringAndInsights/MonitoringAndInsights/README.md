# Monitoring and Insights

Read-only configuration checks and ingestion validation for Azure Virtual Desktop, plus a cost-optimized monitoring deployment script, controlled event generator and local diagnostic collector.

## Architecture

AVD Insights combines independent telemetry paths:

~~~text
Host pool + AVD Workspace -> diagnostic settings -> Log Analytics -> WVD* tables
Session host -> events/counters -> AMA + associated DCR -> Log Analytics -> Event/Perf
AMA -> Heartbeat
~~~

AVD Agent health is distinct from AMA health. Guest data and service diagnostics may use different Log Analytics workspaces; supply the appropriate destination and query each one. [Microsoft Learn](https://learn.microsoft.com/en-us/azure/virtual-desktop/insights).

## Scripts

| File | Runs on | Purpose |
| --- | --- | --- |
| [Test-AVDMonitoringPrerequisites.ps1](Test-AVDMonitoringPrerequisites.ps1) | Admin workstation/Cloud Shell | Modules, Azure context and resource read access |
| [Test-AVDHostPoolDiagnosticSettings.ps1](Test-AVDHostPoolDiagnosticSettings.ps1) | Admin workstation/Cloud Shell | Host pool categories, settings, expected destination and per-category coverage |
| [Test-AVDWorkspaceDiagnosticSettings.ps1](Test-AVDWorkspaceDiagnosticSettings.ps1) | Admin workstation/Cloud Shell | AVD Workspace diagnostic coverage |
| [Test-AVDDCRAssociation.ps1](Test-AVDDCRAssociation.ps1) | Admin workstation/Cloud Shell | Every registered VM's AMA extension, identity selection and DCR routes |
| [Set-AVDCostOptimizedMonitoring.ps1](Set-AVDCostOptimizedMonitoring.ps1) | Admin workstation/Cloud Shell | Creates/updates one cost-optimized DCR and associates it with every registered session-host VM |
| [Test-AVDSessionHostMonitoring.ps1](Test-AVDSessionHostMonitoring.ps1) | Each session host, elevated | Services, registration flag, AMA cache/logs, event channels and 20 counters |
| [Test-AVDLogAnalyticsIngestion.ps1](Test-AVDLogAnalyticsIngestion.ps1) | Admin workstation/Cloud Shell | Table activity, expected AMA hosts and generated test events |
| [New-AVDMonitoringTestEvents.ps1](New-AVDMonitoringTestEvents.ps1) | Session host, elevated Windows PowerShell 5.1 | One Application Warning 9001 and Error 9002 |
| [Collect-AVDDiagnosticBundle.ps1](Collect-AVDDiagnosticBundle.ps1) | Session host, elevated Windows PowerShell 5.1 | Bounded local evidence and a manifest in a ZIP |
| [Invoke-AVDSessionHostReport.ps1](Invoke-AVDSessionHostReport.ps1) | Admin workstation/Cloud Shell | Fans the host validation out to every registered session host via Run Command and merges all reports into one summary + CSV |
| [Invoke-AVDMonitoringReportUpload.ps1](Invoke-AVDMonitoringReportUpload.ps1) | Injected into each session host (not run directly) | Runs the validation and PUTs the JSON report to a write-only blob SAS URL |

The scripts are directly in this folder. Shared queries live in [../KQL](../KQL/). The earlier AVDMonitoringAndInsights folder remains unchanged for compatibility.

## Collect a report from every session host at once

`Invoke-AVDSessionHostReport.ps1` runs the per-host validation on all registered hosts
in parallel (PowerShell 7) or sequentially (5.1) using **Run Command**
(`RunPowerShellScript`, runs elevated as SYSTEM). Run Command caps inline output at
4 KB, so each host PUTs its full JSON report to an anonymous, write-only blob SAS URL;
the orchestrator then downloads and merges them. This is the multi-VM alternative to
running `Validate-AVDSessionHostMonitoring-Interactive.ps1` by hand, and is used instead of
Azure Machine Configuration (Guest Configuration), which is for continuous compliance
state rather than on-demand report collection.

Operator requirements: read on the host pool and VMs,
`Microsoft.Compute/virtualMachines/runCommand/action` on each VM (Virtual Machine
Contributor or higher), and blob SAS create + read on the reports container
(Storage Blob Data Contributor).

~~~powershell
$hpId = '/subscriptions/<sub>/resourceGroups/rg-avd/providers/Microsoft.DesktopVirtualization/hostPools/WPNS-AVD'
.\Invoke-AVDSessionHostReport.ps1 -HostPoolResourceId $hpId `
    -StorageAccountName stavdreports -StorageResourceGroupName rg-avd -ContainerName reports
~~~

Add `-CollectOnly` to re-download the last run's reports without re-running the hosts.
Output lands in `.\AVDReports\<timestamp>-<host>.json` plus a merged
`AVDMonitoring-<timestamp>.csv`, with a worst-status-per-host summary table.
