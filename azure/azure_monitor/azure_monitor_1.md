# SQL Server question — Azure Monitor 1

## Statement

An airline's fare engine runs on an Azure SQL Database named `SkyFare` (vCore, General Purpose). Two callers hit it: an ASP.NET Core pricing API instrumented with the Azure Monitor OpenTelemetry distro (workspace-based **Application Insights**), and a **Data API builder** container that serves a partner REST API.

Operations reports intermittent deadlocks and slow partner calls and asks you to recommend the monitoring configuration. The design must satisfy **all** of the following:

1. Deadlock events, blocking events, query-store runtime statistics and errors from `SkyFare` must be queryable with **KQL**, joinable with the application telemetry, and retained for 90 days.
2. For every failed or slow pricing request it must be possible to see the **SQL calls the request made** (duration, success, target) and correlate them with the request by `operation_Id`, with no custom telemetry code.
3. The DAB container must send its **requests, dependencies and traces to the same Application Insights resource**.
4. An alert must fire when `cpu_percent` exceeds 90% for 5 minutes. It must evaluate the **platform metric directly** — not a copy ingested into logs — with near-real-time latency and no per-query evaluation cost.
5. The raw resource logs must also be kept for **7 years** at the lowest possible cost; nobody needs to query that copy interactively.
6. No agent or monitoring virtual machine may be deployed, and only supported (non-retired) features may be used.

Which configuration should you recommend?

### a.

- Diagnostic setting #1 on `SkyFare`: categories `Deadlocks`, `Blocks`, `QueryStoreRuntimeStatistics`, `Errors` → **Send to Log Analytics workspace** `law-skyfare` (90-day retention).
- Diagnostic setting #2 on `SkyFare`: the same categories → **Archive to a storage account** with a 7-year lifecycle policy.
- Application Insights: rely on the OpenTelemetry distro's automatic `Microsoft.Data.SqlClient` dependency collection; query `AppDependencies | where DependencyType == "SQL"` joined to `AppRequests` on `OperationId`.
- DAB: `"runtime": { "telemetry": { "application-insights": { "enabled": true, "connection-string": "@env('APPINSIGHTS_CONNECTION_STRING')" } } }`.
- **Metric alert** rule on `SkyFare`, signal `cpu_percent`, condition greater than 90, aggregation Average, granularity 1 minute, frequency 1 minute, look-back 5 minutes.

### b.

- Diagnostic setting on `SkyFare`: categories `Deadlocks`, `Blocks`, `QueryStoreRuntimeStatistics`, `Errors` → **Archive to a storage account** (7-year lifecycle). Analysts run KQL against the storage container from the Log Analytics workspace.
- Application Insights, DAB telemetry key and the metric alert exactly as in option a.

### c.

- Diagnostic settings #1 and #2, Application Insights and DAB telemetry exactly as in option a, plus `Basic` metrics sent to the workspace.
- **Log search alert** rule on `law-skyfare`, evaluation frequency 5 minutes, query:

  ```kusto
  AzureMetrics
  | where ResourceProvider == "MICROSOFT.SQL" and MetricName == "cpu_percent"
  | summarize AggregatedValue = avg(Average) by bin(TimeGenerated, 5m)
  | where AggregatedValue > 90
  ```

### d.

- Diagnostic setting #1 on `SkyFare`: categories `SQLInsights` and `Basic` metrics → Log Analytics workspace `law-skyfare`; diagnostic setting #2 to a storage account as in option a.
- Application Insights and the metric alert exactly as in option a.
- DAB: `"runtime": { "telemetry": { "open-telemetry": { "enabled": true, "endpoint": "@env('APPINSIGHTS_CONNECTION_STRING')" } } }`.

## Correct Answer

**a**

## Explanation

This is a destination-and-signal matching exercise. Azure SQL Database emits **metrics** (`Basic`) and **resource-log categories** (`SQLInsights`, `AutomaticTuning`, `QueryStoreRuntimeStatistics`, `QueryStoreWaitStatistics`, `Errors`, `DatabaseWaitStatistics`, `Timeouts`, `Blocks`, `Deadlocks`, plus the audit categories). A diagnostic setting routes any subset of them to up to three destination types — **Log Analytics workspace** (KQL, alerts, joins), **Azure Storage** (cheap archive, retention policy, not queryable by Log Analytics) and **Event Hubs** (streaming to SIEM/third-party or Power BI). A resource can have several diagnostic settings, which is how one set of logs reaches both a workspace and an archive.

### Why option a is correct

- **Requirement 1** — the four categories land in the `AzureDiagnostics` table of `law-skyfare` (`Category == "Deadlocks"`, `"Blocks"`, `"QueryStoreRuntimeStatistics"`, `"Errors"`), where they can be joined with `AppRequests`/`AppDependencies` from the workspace-based Application Insights resource. Workspace retention is set to 90 days.
- **Requirement 2** — the OpenTelemetry distro auto-instruments `Microsoft.Data.SqlClient`; every SQL call becomes a dependency record (`DependencyType == "SQL"`, duration, success, target, `OperationId`) with no code. The KQL join on `OperationId` is the documented pattern (classic tables: `dependencies | join requests on operation_Id`).
- **Requirement 3** — the DAB key is `runtime.telemetry.application-insights.connection-string` (`enabled` defaults to `true` once the object exists). `@env()` keeps the connection string out of the config file.
- **Requirement 4** — a **metric alert** evaluates the pre-computed platform metric `cpu_percent` on the `Microsoft.Sql/servers/databases` resource, with evaluation frequency down to 1 minute, is stateful by default and is charged per monitored time series, not per query evaluation.
- **Requirement 5** — the second diagnostic setting archives the same categories to blob storage under `insights-logs-<category>` containers; a lifecycle/retention policy keeps them 7 years at archive-tier cost.
- **Requirement 6** — nothing is installed on a VM. (SQL Insights (preview), which relied on a Telegraf monitoring VM, was **retired on 31 December 2024**; Azure SQL Analytics is a legacy workspace solution "no longer in active development". The current estate-level option with nothing for you to deploy is **database watcher** (Microsoft-managed collection, managed-identity login), which stores data in Azure Data Explorer/Fabric Real-Time Intelligence rather than a Log Analytics workspace — a valid extra, but not required here.)

### Why option b is wrong

Storage-only violates requirement 1. Azure Monitor Logs **does not read data from a storage account**: KQL, workspace retention, log alerts and joins with Application Insights all require the logs to be ingested into a Log Analytics workspace. A storage archive is the right answer for the 7-year copy (requirement 5) — but only as a *second* diagnostic setting, not instead of the workspace.

### Why option c is wrong

The subtle distractor: everything is correct except the alert type. A **log search alert** on `AzureMetrics` evaluates a *copy* of the metric that first has to be ingested into the workspace (only after `Basic` metrics are streamed there), runs a scheduled KQL query (billed per evaluation interval) and inherits log-ingestion latency. Requirement 4 explicitly asks for the platform metric evaluated directly, near real time, without per-query cost — the definition of a **metric alert**. Use log search alerts when the condition needs KQL logic over logs (for example "a `Deadlocks` record appeared in the last 5 minutes"), not for a plain platform metric threshold.

### Why option d is wrong

Two faults:

- The diagnostic setting sends `SQLInsights` (Intelligent Insights root-cause analyses) and `Basic` metrics but **not** the `Deadlocks`, `Blocks`, `QueryStoreRuntimeStatistics` or `Errors` categories, so requirement 1 fails. Intelligent Insights may *mention* a deadlock pattern, but the deadlock events themselves are a separate category.
- The DAB block uses `runtime.telemetry.open-telemetry.endpoint`, which expects an **OTLP collector URL** (gRPC `:4317` or HTTP/protobuf `:4318`). An Application Insights connection string is not an OTLP endpoint; DAB's Application Insights export is configured with `runtime.telemetry.application-insights.connection-string`, so requirement 3 fails.

### What the recommended configuration lets operations run

Once option a is in place, the deadlock investigation is a single workspace query that joins database-side resource logs with app-side telemetry:

```kusto
// Deadlock events from the database (resource log category "Deadlocks")
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.SQL" and Category == "Deadlocks"
| project TimeGenerated, Resource, deadlock_xml_s
| join kind=inner (
    // Failed SQL dependencies recorded by the pricing API in the same minute
    AppDependencies
    | where DependencyType == "SQL" and Success == false
    | project TimeGenerated, OperationId, Name, DurationMs, ResultCode
  ) on $left.TimeGenerated == $right.TimeGenerated
| join kind=leftouter (AppRequests | project OperationId, Url, RequestDurationMs = DurationMs) on OperationId
| order by TimeGenerated desc
```

A **log search alert** is the right tool for "a deadlock happened in the last 5 minutes" (`AzureDiagnostics | where Category == "Deadlocks" | where TimeGenerated > ago(5m)`, fire when the result count is greater than 0), while the CPU threshold stays a **metric alert**. Both kinds can be attached to the same action group. The DAB container appears in the same Application Insights resource as its own cloud role, so its partner-API requests and the SQL dependencies they generate are visible on the Application map next to the pricing API.

Two rows in the storage account, one per category container, illustrate the archive layout the second diagnostic setting produces:

```text
insights-logs-deadlocks/resourceId=/SUBSCRIPTIONS/.../DATABASES/SKYFARE/y=2026/m=09/d=03/h=10/m=00/PT1H.json
insights-logs-blocks/resourceId=/SUBSCRIPTIONS/.../DATABASES/SKYFARE/y=2026/m=09/d=03/h=10/m=00/PT1H.json
```

Conceptual question (Azure / tooling); not executed against an engine.

## DP-800 Exam Rule to Remember

```text
WHAT the database emits            WHERE it goes (diagnostic setting)           WHY
Basic metrics (cpu_percent, dtu,   → Log Analytics workspace  (AzureMetrics /   KQL, joins, log alerts,
 storage_percent, deadlock, ...)     AzureDiagnostics tables)                     retention up to 2 years (interactive)
Log categories: SQLInsights,       → Storage account          (insights-logs-*)  cheapest long-term archive,
 QueryStoreRuntimeStatistics,                                                    NOT queryable by KQL
 QueryStoreWaitStatistics, Errors, → Event Hubs                                  stream to SIEM / custom pipeline
 DatabaseWaitStatistics, Timeouts,
 Blocks, Deadlocks, AutomaticTuning   (several settings per resource are allowed)
```

- **Application Insights** = app-side view: auto-collected SQL dependency telemetry (`AppDependencies`, `DependencyType == "SQL"`) correlated with `AppRequests` by `OperationId`. **DAB** joins it with `runtime.telemetry.application-insights.connection-string` (OpenTelemetry has its own `open-telemetry.endpoint` for OTLP collectors; Log Analytics ingestion uses `azure-log-analytics` with a DCR/DCE).
- **Metric alert** → platform metric, near real time, 1-minute frequency, priced per time series. **Log search alert** → KQL over a workspace/App Insights, scheduled evaluation, priced per evaluation, needed when the signal only exists in logs (deadlock events, errors, Intelligent Insights text).
- Retired/legacy: SQL Insights (preview, monitoring VM) retired 31 Dec 2024; Azure SQL Analytics is legacy. Current estate monitoring with no VM/agent to deploy: **database watcher** (managed identity, ADX/Fabric store).
