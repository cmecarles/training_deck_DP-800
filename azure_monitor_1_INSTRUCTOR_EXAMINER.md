# Instructor-Examiner guide — Azure Monitor 1

Companion to [azure_monitor_1.md](azure_monitor_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. The options share many pieces, so read the six requirements and all four options in full before taking an answer. Ask for one letter and one sentence per rejected option. This is a conceptual Azure question: describe diagnostic settings, JSON keys and alert rules precisely in words.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Monitor database performance and health.
- Task bullet: Configure Azure Monitor, diagnostic settings, Application Insights and alerts for Azure SQL Database.
- What is tested: matching each signal to the right destination, Log Analytics workspace versus storage account, metric alert versus log search alert, automatic SQL dependency collection in Application Insights, and the Data API builder telemetry keys.

## 2. Scenario to read aloud

**Piece 1, the story.** "An airline's fare engine runs on an Azure SQL Database named SkyFare, vCore, General Purpose tier. Two callers hit it. First, an ASP dot NET Core pricing API, instrumented with the Azure Monitor OpenTelemetry distro and a workspace-based Application Insights resource. Second, a Data API builder container that serves a partner REST API. Operations reports intermittent deadlocks and slow partner calls, and asks you to recommend the monitoring configuration."

**Piece 2, requirements one to three.** "The design must satisfy all six requirements. One: deadlock events, blocking events, query store runtime statistics and errors from SkyFare must be queryable with KQL, joinable with the application telemetry, and retained for ninety days. Two: for every failed or slow pricing request, it must be possible to see the SQL calls the request made, duration, success and target, and correlate them with the request by operation underscore Id, with no custom telemetry code. Three: the DAB container must send its requests, dependencies and traces to the same Application Insights resource."

**Piece 3, requirements four to six.** "Four: an alert must fire when cpu underscore percent exceeds ninety percent for five minutes. It must evaluate the platform metric directly, not a copy ingested into logs, with near real time latency and no per-query evaluation cost. Five: the raw resource logs must also be kept for seven years at the lowest possible cost. Nobody needs to query that copy interactively. Six: no agent or monitoring virtual machine may be deployed, and only supported, non-retired features may be used."

**Piece 4, option a.** "Option a has five parts. First, diagnostic setting number one on SkyFare, with categories Deadlocks, Blocks, QueryStoreRuntimeStatistics and Errors, sent to a Log Analytics workspace called law dash skyfare, with ninety-day retention. Second, diagnostic setting number two on SkyFare, same categories, archived to a storage account with a seven-year lifecycle policy. Third, Application Insights relies on the OpenTelemetry distro's automatic Microsoft dot Data dot SqlClient dependency collection; you query the AppDependencies table where DependencyType equals SQL, joined to AppRequests on OperationId. Fourth, in the DAB config file, under runtime, telemetry, application dash insights, enabled true and connection dash string set from the environment variable APPINSIGHTS underscore CONNECTION underscore STRING. Fifth, a metric alert rule on SkyFare, signal cpu underscore percent, condition greater than ninety, aggregation Average, granularity one minute, frequency one minute, look-back five minutes."

**Piece 5, option b.** "Option b has one diagnostic setting only, with the same four categories, archived to a storage account with a seven-year lifecycle. Analysts run KQL against the storage container from the Log Analytics workspace. Application Insights, the DAB telemetry key and the metric alert are exactly as in option a."

**Piece 6, option c.** "Option c keeps diagnostic settings one and two, Application Insights and the DAB telemetry exactly as in option a, and also sends Basic metrics to the workspace. The difference is the alert. It is a log search alert rule on law dash skyfare, evaluation frequency five minutes, with a KQL query on the AzureMetrics table, where ResourceProvider equals MICROSOFT dot SQL and MetricName equals cpu underscore percent, summarize the average of the Average column by five-minute bins, and keep bins where the value is greater than ninety."

**Piece 7, option d.** "Option d changes two things. Diagnostic setting number one sends the categories SQLInsights and Basic metrics to the workspace, and diagnostic setting number two goes to a storage account as in option a. Application Insights and the metric alert are as in option a. The DAB config uses runtime, telemetry, open dash telemetry, enabled true, and endpoint set from the environment variable APPINSIGHTS underscore CONNECTION underscore STRING."

## 3. Setup script (reference only; do not read verbatim unless asked)

Option a DAB telemetry block:

```json
"runtime": { "telemetry": { "application-insights": { "enabled": true, "connection-string": "@env('APPINSIGHTS_CONNECTION_STRING')" } } }
```

Option c log search alert query:

```kusto
AzureMetrics
| where ResourceProvider == "MICROSOFT.SQL" and MetricName == "cpu_percent"
| summarize AggregatedValue = avg(Average) by bin(TimeGenerated, 5m)
| where AggregatedValue > 90
```

Option d DAB telemetry block:

```json
"runtime": { "telemetry": { "open-telemetry": { "enabled": true, "endpoint": "@env('APPINSIGHTS_CONNECTION_STRING')" } } }
```

Deadlock investigation query once option a is in place:

```kusto
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.SQL" and Category == "Deadlocks"
| project TimeGenerated, Resource, deadlock_xml_s
| join kind=inner (
    AppDependencies
    | where DependencyType == "SQL" and Success == false
    | project TimeGenerated, OperationId, Name, DurationMs, ResultCode
  ) on $left.TimeGenerated == $right.TimeGenerated
| join kind=leftouter (AppRequests | project OperationId, Url, RequestDurationMs = DurationMs) on OperationId
| order by TimeGenerated desc
```

Storage archive layout produced by the second diagnostic setting:

```text
insights-logs-deadlocks/resourceId=/SUBSCRIPTIONS/.../DATABASES/SKYFARE/y=2026/m=09/d=03/h=10/m=00/PT1H.json
insights-logs-blocks/resourceId=/SUBSCRIPTIONS/.../DATABASES/SKYFARE/y=2026/m=09/d=03/h=10/m=00/PT1H.json
```

## 4. The question (ask exactly this)

"Which configuration should you recommend?

a. Diagnostic setting one on SkyFare, categories Deadlocks, Blocks, QueryStoreRuntimeStatistics, Errors, sent to Log Analytics workspace law dash skyfare with ninety-day retention. Diagnostic setting two, same categories, archived to a storage account with a seven-year lifecycle policy. Application Insights relies on the OpenTelemetry distro's automatic Microsoft dot Data dot SqlClient dependency collection; query AppDependencies where DependencyType equals SQL joined to AppRequests on OperationId. DAB: runtime, telemetry, application dash insights, enabled true, connection dash string from the environment variable APPINSIGHTS underscore CONNECTION underscore STRING. A metric alert rule on SkyFare, signal cpu underscore percent, greater than ninety, aggregation Average, granularity one minute, frequency one minute, look-back five minutes.

b. Diagnostic setting on SkyFare, same four categories, archived to a storage account with a seven-year lifecycle. Analysts run KQL against the storage container from the Log Analytics workspace. Application Insights, DAB telemetry key and the metric alert exactly as in option a.

c. Diagnostic settings one and two, Application Insights and DAB telemetry exactly as in option a, plus Basic metrics sent to the workspace. A log search alert rule on law dash skyfare, evaluation frequency five minutes, with a query on AzureMetrics where ResourceProvider equals MICROSOFT dot SQL and MetricName equals cpu underscore percent, summarize average of Average by five-minute bins, where the aggregated value is greater than ninety.

d. Diagnostic setting one on SkyFare, categories SQLInsights and Basic metrics, to the workspace; diagnostic setting two to a storage account as in option a. Application Insights and the metric alert exactly as in option a. DAB: runtime, telemetry, open dash telemetry, enabled true, endpoint from the environment variable APPINSIGHTS underscore CONNECTION underscore STRING.

Which letter, and why do the other three fail?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

| Option | Verdict | Why |
|---|---|---|
| a | Correct | Two diagnostic settings, workspace for KQL and 90-day retention, storage for the 7-year archive. OpenTelemetry distro auto-instruments Microsoft.Data.SqlClient, giving AppDependencies rows with DependencyType SQL correlated by OperationId. DAB key runtime.telemetry.application-insights.connection-string. Metric alert on the platform metric cpu_percent, 1-minute frequency, priced per time series. No VM. |
| b | Wrong | Storage-only. Azure Monitor Logs cannot read a storage account; KQL, retention, log alerts and joins require ingestion into a workspace. Requirement 1 fails. |
| c | Wrong | Alert type. A log search alert on AzureMetrics evaluates a copy of the metric that must first be ingested, runs scheduled KQL billed per evaluation, and inherits log latency. Requirement 4 asks for the platform metric evaluated directly, which is a metric alert. |
| d | Wrong | Diagnostic setting sends SQLInsights and Basic only, not Deadlocks, Blocks, QueryStoreRuntimeStatistics or Errors, so requirement 1 fails. DAB open-telemetry.endpoint expects an OTLP collector URL, port 4317 gRPC or 4318 HTTP, not an Application Insights connection string, so requirement 3 fails. |

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with requirement one, KQL over deadlocks and blocks. Which destination of a diagnostic setting can Log Analytics actually query: a workspace, a storage account, or both?"
2. "Now look at which categories each option sends to the workspace. One option sends SQLInsights and Basic metrics only. Are deadlock events inside the SQLInsights category?"
3. "Requirement three is about the DAB config. There are two different telemetry objects: application dash insights takes a connection string, open dash telemetry takes an endpoint. What kind of value does an OTLP endpoint expect?"
4. "Requirement four says: evaluate the platform metric directly, near real time, no per-query cost. There are two alert families in Azure Monitor. Which one runs a scheduled KQL query over ingested data, and which one reads the metric itself?"
5. "Only one option uses two diagnostic settings, the application dash insights DAB key, and a metric alert. Which letter is that?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "Option b, storage is cheaper and KQL can read blobs" | Thinks Log Analytics queries external storage | "Where does Azure Monitor Logs read from when you run KQL? Can it open a blob container?" |
| "Option b, one setting is enough" | Does not know a resource can have several diagnostic settings | "Requirements one and five want two different destinations. Can one resource have more than one diagnostic setting?" |
| "Option c, the log alert also checks CPU" | Confuses a log search alert on AzureMetrics with a metric alert | "Requirement four says not a copy ingested into logs. Which table does that query read, and how did the data get there?" |
| "Option c is cheaper because it evaluates every five minutes" | Does not know log alerts are billed per evaluation and metric alerts per time series | "Which alert family is charged per query evaluation? Read requirement four again." |
| "Option d, SQLInsights covers deadlocks" | Confuses Intelligent Insights text with the Deadlocks category | "Intelligent Insights may mention a deadlock pattern. Is that the same as the deadlock event records themselves?" |
| "Option d, OpenTelemetry works with App Insights" | Thinks any OTel endpoint accepts a connection string | "The open dash telemetry endpoint key expects a collector URL on port 4317 or 4318. Is an Application Insights connection string a URL of that kind?" |
| "Option a needs custom code for SQL dependencies" | Does not know the distro auto-instruments SqlClient | "Which client library does the Azure Monitor OpenTelemetry distro instrument out of the box?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the destination-and-signal model:

- **What the database emits.** Basic metrics, such as cpu underscore percent, dtu, storage underscore percent and deadlock. And resource log categories: SQLInsights, AutomaticTuning, QueryStoreRuntimeStatistics, QueryStoreWaitStatistics, Errors, DatabaseWaitStatistics, Timeouts, Blocks, Deadlocks, plus the audit categories.
- **Where a diagnostic setting can send them.** A Log Analytics workspace, into the AzureMetrics and AzureDiagnostics tables, for KQL, joins, log alerts and interactive retention. A storage account, into insights dash logs dash category containers, the cheapest long-term archive but not queryable by KQL. Event Hubs, to stream to a SIEM or custom pipeline. A resource may have several diagnostic settings, which is how one set of logs reaches both a workspace and an archive. That is requirements 1 and 5, and why option b fails.
- **Application Insights.** The Azure Monitor OpenTelemetry distro auto-instruments Microsoft dot Data dot SqlClient. Every SQL call becomes an AppDependencies row with DependencyType SQL, duration, success, target and OperationId. Join to AppRequests on OperationId. In the classic tables, dependencies join requests on operation underscore Id. That is requirement 2.
- **DAB telemetry keys.** runtime dot telemetry dot application dash insights dot connection dash string sends to Application Insights; enabled defaults to true once the object exists, and at env keeps the secret out of the file. runtime dot telemetry dot open dash telemetry dot endpoint is for an OTLP collector URL. There is also azure dash log dash analytics with a data collection rule and endpoint. That is requirement 3 and why option d fails.
- **Alert families.** A metric alert evaluates the platform metric on the Microsoft dot Sql slash servers slash databases resource, frequency down to one minute, stateful by default, priced per monitored time series. A log search alert runs scheduled KQL over a workspace, priced per evaluation, and inherits ingestion latency. Use log search alerts when the signal exists only in logs, for example "a Deadlocks record appeared in the last five minutes". That is requirement 4 and why option c fails.
- **Retired and legacy.** SQL Insights preview, which needed a Telegraf monitoring VM, was retired on 31 December 2024. Azure SQL Analytics is a legacy workspace solution. The current estate-level option with nothing to deploy is database watcher, Microsoft-managed collection with a managed identity, storing in Azure Data Explorer or Fabric Real-Time Intelligence. That is requirement 6.

Memory hook: "Workspace for KQL, storage for the archive, metric alert for a metric, log alert for a log, application dash insights key for DAB."

## 9. Follow-up oral questions (optional)

1. "You want an alert when a deadlock record appears in the last five minutes. Which alert family, and roughly what query?" (A log search alert; AzureDiagnostics where Category equals Deadlocks and TimeGenerated greater than ago five minutes, fire when the count is greater than zero.)
2. "Which DAB telemetry object would you use to send to an OTLP collector on port 4317?" (runtime dot telemetry dot open dash telemetry, key endpoint.)
3. "Name the current estate-level Azure SQL monitoring feature that needs no VM and where it stores data." (Database watcher; Azure Data Explorer or Fabric Real-Time Intelligence.)

## 10. References

- Monitor Azure SQL Database with Azure Monitor, diagnostic settings and log categories: https://learn.microsoft.com/en-us/azure/azure-sql/database/monitoring-sql-database-azure-monitor
- Azure SQL Database monitoring data reference: https://learn.microsoft.com/en-us/azure/azure-sql/database/monitoring-sql-database-azure-monitor-reference
- Diagnostic settings in Azure Monitor: https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/diagnostic-settings
- Metric alerts: https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-types
- Azure Monitor OpenTelemetry distro for .NET: https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-enable
- Data API builder telemetry with Application Insights: https://learn.microsoft.com/en-us/azure/data-api-builder/deployment/how-to-use-application-insights
- Data API builder configuration reference, runtime telemetry: https://learn.microsoft.com/en-us/azure/data-api-builder/reference-configuration
- Database watcher: https://learn.microsoft.com/en-us/azure/azure-sql/database-watcher-overview
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
