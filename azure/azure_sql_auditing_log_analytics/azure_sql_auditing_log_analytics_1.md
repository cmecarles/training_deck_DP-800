# SQL Server question — Azure SQL Auditing to Log Analytics 1

## Statement

A county court office stores hearing dockets in an **Azure SQL Database** named `CourtDocket`. Compliance wants every batch and every login attempt audited **at the server level**, plus a **database-level** audit of reads of the `Docket` schema, both landing in one **Log Analytics** workspace where they can be queried with KQL; the DBA also wants Query Store runtime statistics and deadlock reports in the same workspace.

This is a **hands-on** question: provision, generate activity, query the workspace, then answer the multiple-choice question.

### Provisioning (Azure CLI, bash; `az login` first)

```bash
LOCATION="westeurope"
SUFFIX=$RANDOM
RG="rg-dp800-courtdocket-$SUFFIX"
SQL="sql-courtdocket-$SUFFIX"
DB="CourtDocket"
LAW="law-courtdocket-$SUFFIX"
SQL_ADMIN="docketadmin"; SQL_PWD="Str0ng!Passw0rd#2026"      # SQL auth stays enabled: the lab needs a failed SQL login
MYIP=$(curl -s https://api.ipify.org)

az group create -n $RG -l $LOCATION
az monitor log-analytics workspace create -g $RG -n $LAW -l $LOCATION --sku PerGB2018 --retention-time 30
LAW_ID=$(az monitor log-analytics workspace show -g $RG -n $LAW --query id -o tsv)
LAW_CID=$(az monitor log-analytics workspace show -g $RG -n $LAW --query customerId -o tsv)

az sql server create -g $RG -n $SQL -l $LOCATION -u $SQL_ADMIN -p "$SQL_PWD"
az sql server firewall-rule create -g $RG -s $SQL -n client --start-ip-address $MYIP --end-ip-address $MYIP
az sql db create -g $RG -s $SQL -n $DB -e Basic --backup-storage-redundancy Local         # Basic DTU is enough here
SQL_FQDN=$(az sql server show -g $RG -n $SQL --query fullyQualifiedDomainName -o tsv)
DB_ID=$(az sql db show -g $RG -s $SQL -n $DB --query id -o tsv)

# A1: server-level auditing (default action groups) -> Log Analytics
az sql server audit-policy update -g $RG -n $SQL --state Enabled --lats Enabled --lawri $LAW_ID
# A2: database-level auditing with a custom action list -> the same workspace
az sql db audit-policy update -g $RG -s $SQL -n $DB --state Enabled --lats Enabled --lawri $LAW_ID \
  --actions FAILED_DATABASE_AUTHENTICATION_GROUP "SELECT ON SCHEMA::Docket BY public"
# A3: diagnostic setting for two resource-log categories of the database
az monitor diagnostic-settings create -n ds-courtdocket --resource $DB_ID --workspace $LAW_ID \
  --logs '[{"category":"QueryStoreRuntimeStatistics","enabled":true},{"category":"Deadlocks","enabled":true}]'
az monitor diagnostic-settings list --resource $DB_ID -o table       # note the extra SQLSecurityAuditEvents_* setting
```

Schema and workload (`sqlcmd -S $SQL_FQDN -d $DB -U $SQL_ADMIN -P "$SQL_PWD" -I`):

```sql
CREATE SCHEMA Docket;
GO
CREATE TABLE Docket.Hearings (HearingId INT NOT NULL PRIMARY KEY, CaseNo VARCHAR(12) NOT NULL, Room TINYINT NOT NULL, HeldOn DATE NOT NULL);
CREATE TABLE Docket.Rooms (Room TINYINT NOT NULL PRIMARY KEY, Name VARCHAR(20) NOT NULL);
INSERT INTO Docket.Rooms VALUES (1, 'Courtroom A'), (2, 'Courtroom B');
INSERT INTO Docket.Hearings VALUES (1, 'CV-2026-0101', 1, '20260910'), (2, 'CR-2026-0202', 2, '20260911');
GO
-- W1 a read of the audited schema
SELECT h.CaseNo, r.Name FROM Docket.Hearings AS h JOIN Docket.Rooms AS r ON r.Room = h.Room;
-- W2 a write that must read the schema to find its rows
UPDATE Docket.Hearings SET Room = 1 WHERE HearingId = 2;
GO
```

```bash
# W3 a failed SQL login (wrong password) and W4 a successful one
sqlcmd -S $SQL_FQDN -d $DB -U $SQL_ADMIN -P "wrong-password" -Q "SELECT 1;"
sqlcmd -S $SQL_FQDN -d $DB -U $SQL_ADMIN -P "$SQL_PWD" -Q "SELECT COUNT(*) FROM Docket.Hearings;"
```

Wait a few minutes (Azure Monitor ingestion is not instantaneous), then query (`az monitor log-analytics query` lives in the `log-analytics` extension, which the CLI offers to install):

```bash
az monitor log-analytics query -w $LAW_CID -t PT2H --analytics-query '
AzureDiagnostics
| where Category == "SQLSecurityAuditEvents"
| project event_time_t, is_server_level_audit_s, action_name_s, server_principal_name_s, database_name_s, succeeded_s, statement_s
| order by event_time_t asc' -o table

az monitor log-analytics query -w $LAW_CID -t PT2H --analytics-query '
AzureDiagnostics | summarize count() by Category, is_server_level_audit_s' -o table
```

Which statement about what appears in the workspace is **correct**?

### a.

Because A2 configured a database-level policy, it **replaces** the server-level policy for `CourtDocket`: the workspace receives only the `SELECT ON SCHEMA::Docket` and failed-authentication events for this database, and the server-level `BATCH_COMPLETED_GROUP` rows stop for it.

### b.

Both audits run **side by side** and both write to the `AzureDiagnostics` table with `Category == "SQLSecurityAuditEvents"`. W1 therefore yields a row from the server audit (`is_server_level_audit_s = "true"`, batch completed) *and* a row from the database audit (`"false"`, the `SELECT` action on the schema); W2 is captured by the database audit too, because the `UPDATE ... WHERE` uses the `SELECT` permission on `Docket.Hearings`. W3 produces a failed-authentication row, whereas a failed **Microsoft Entra** login would produce none. `statement_s` is truncated at 4,000 characters. The `QueryStoreRuntimeStatistics` and `Deadlocks` categories land in the same `AzureDiagnostics` table, distinguished by `Category` (the deadlock report is in `deadlock_xml_s`).

### c.

Audit records go to a dedicated resource-specific table named `SQLSecurityAuditEvents`; `AzureDiagnostics` only receives the two diagnostic-setting categories, which is why the second query returns rows for `QueryStoreRuntimeStatistics` and `Deadlocks` but none for auditing.

### d.

The extra `SQLSecurityAuditEvents_*` diagnostic setting listed by A3's last command is a leftover of the portal experience and can be deleted; `az sql server audit-policy show` still reports `state: Enabled`, so audit records keep flowing to the workspace.

**Cost and cleanup.** Basic DTU is about five euros a month (cents for this lab); the workspace bills per GB ingested (a few KB here). Delete everything at the end:

```bash
az group delete -n $RG --yes --no-wait
```

## Correct Answer

**b**

## Explanation

### Why option b is correct

- **Server and database policies coexist.** "Server auditing policies apply to all existing and newly created databases on this server" and "If server auditing is enabled, the database-configured audit exists side-by-side with the server audit." There is no precedence: each audit evaluates its own action list, and each writes its own records. The default action groups of a policy are `BATCH_COMPLETED_GROUP`, `SUCCESSFUL_DATABASE_AUTHENTICATION_GROUP` and `FAILED_DATABASE_AUTHENTICATION_GROUP` (that is what A1 uses); `--actions` on A2 replaces them with a custom list, exactly like a `CREATE DATABASE AUDIT SPECIFICATION` on SQL Server.
- **One table, one category, one flag.** For a Log Analytics destination the audit events are written "to the `AzureDiagnostics` table with the category `SQLSecurityAuditEvents`"; the field `is_server_level_audit_s` ("Flag indicating if this audit is at the server level") is what tells the two audits apart, alongside `action_name_s`, `server_principal_name_s`, `database_name_s`, `succeeded_s`, `statement_s`, `client_ip_s`, `application_name_s` (Log Analytics names carry the `_s`/`_d`/`_t` type suffixes of the blob-format fields). The server-level records are the re-architected (July 2025) single extended-event session for the whole server; on storage targets they are written to the `master` folder.
- **W2 is an audited read.** An object/schema-level `SELECT` action fires whenever the `SELECT` permission on the object is checked, and an `UPDATE ... WHERE HearingId = 2` must read the table to find its row — the same behaviour the on-premises audit question of this deck verified. The `UPDATE` itself is not in A2's action list, so the database audit records the *read*, and the server audit records the batch.
- **Failed logins.** W3 is a SQL-authentication failure; the gateway routes it to the database, the password is checked there, and `FAILED_DATABASE_AUTHENTICATION_GROUP` captures it. "When using Microsoft Entra authentication, failed logins records don't appear in the SQL audit log ... the credentials are verified before attempting to use that user to sign into the requested database. In the case of failure, the requested database is never accessed, so no auditing occurs" — those attempts live in the Entra sign-in logs instead.
- **Truncation and categories.** "Azure SQL Database and Azure Synapse audit logs capture up to 4,000 characters in the `statement` and `data_sensitivity_information` fields." A3's categories are resource logs of the database, exported by a diagnostic setting; in the workspace they share `AzureDiagnostics` (the default, dynamic-schema destination) and are distinguished by `Category` (`QueryStoreRuntimeStatistics`, `OperationName = QueryStoreRuntimeStatisticsEvent`, `query_hash_s`, ...; `Deadlocks` with `deadlock_xml_s`).

### Why option a is wrong

There is no override: enabling a database policy adds a second audit, it does not detach the database from the server policy. Only disabling the server policy (or using database-level auditing everywhere, the documented recommendation for large OLTP estates to keep each database's log volume separate) changes what the server audit captures.

### Why option c is wrong

Auditing to Log Analytics uses the `AzureDiagnostics` table with `Category == "SQLSecurityAuditEvents"` (and `DevOpsOperationsAudit` for Microsoft support operations). A resource-specific ("dedicated") table exists only for diagnostic settings created with `--export-to-resource-specific true` (or `--log-analytics-destination-type Dedicated`), which A3 did not do; the second query returns rows for all three categories.

### Why option d is wrong

"When auditing is configured with Azure external monitors (for example, Event Hubs or Log Analytics) as the target, an additional diagnostic settings resource named `SQLSecurityAuditEvents_XXXX-XXXX-XXX` is created, which is critical for the proper functioning of auditing. If the diagnostic settings are deleted, either intentionally or unintentionally, the auditing functionality will fail silently, and audit logs won't be sent to the target location." The policy still reads `Enabled`, which is why the documentation recommends an activity-log alert on diagnostic-setting deletion.

Hands-on question (Azure subscription required); the T-SQL fragments that do not depend on Azure were checked on SQL Server 2025 RTM 17.0.1000.7; Azure-side behaviour is taken from the official documentation.

## DP-800 Exam Rule to Remember

```text
Azure SQL Database auditing (no CREATE SERVER AUDIT):
  server policy  -> all databases, existing and new           az sql server audit-policy update
  database policy-> side by side with the server audit         az sql db audit-policy update
     --state Enabled  --lats Enabled --lawri <workspace id> | --bsts Enabled --storage-account | --ehts Enabled --ehari
     --actions <groups/actions>   default: BATCH_COMPLETED_GROUP, SUCCESSFUL/FAILED_DATABASE_AUTHENTICATION_GROUP
  Log Analytics: AzureDiagnostics | where Category == "SQLSecurityAuditEvents"
     is_server_level_audit_s, action_name_s, server_principal_name_s, database_name_s, succeeded_s, statement_s (<= 4000 chars)
     auto-created SQLSecurityAuditEvents_* diagnostic setting: delete it and auditing fails SILENTLY
  audited = permission used (UPDATE ... WHERE -> SELECT action); Entra login failures never reach the audit
Resource logs (diagnostic settings, az monitor diagnostic-settings create --logs '[{category,enabled}]'):
  SQLInsights, AutomaticTuning, QueryStoreRuntimeStatistics, QueryStoreWaitStatistics, Errors,
  DatabaseWaitStatistics, Timeouts, Blocks, Deadlocks (deadlock_xml_s) -> same AzureDiagnostics table, by Category
```
