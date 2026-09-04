# Instructor-Examiner guide — Azure SQL Auditing to Log Analytics 1

Companion to [azure_sql_auditing_log_analytics_1.md](azure_sql_auditing_log_analytics_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**This is a hands-on Azure lab question.** Before anything else, ask: "Have you already run this lab in your own subscription?" If yes, go through A1 to A3, W1 to W4 and the two KQL queries, and ask what the learner observed at each step before you quiz; use the observations to anchor the discussion. If no, walk through the provisioning, the workload and the queries in words using section 2, so the question can still be answered from the documented facts.

**This is a multiple-choice question.** Read all four options, pieces 9 to 12, before taking an answer. Option b is long; read it slowly and offer to repeat it.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "dash dash" for `--`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line from section 3 only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Implement data security and compliance.
- Task bullet: Configure auditing for Azure SQL Database and route audit and resource logs to Log Analytics.
- What is tested: that server-level and database-level audit policies run side by side, where audit records land in Log Analytics and which fields distinguish them, what a failed Entra login does and does not produce, and why the auto-created diagnostic setting must not be deleted.

## 2. Scenario to read aloud

**Piece 1, the story.** "A county court office stores hearing dockets in an Azure SQL Database called CourtDocket. Compliance wants every batch and every login attempt audited at the server level, plus a database-level audit of reads of the Docket schema. Both must land in one Log Analytics workspace where they can be queried with KQL. The DBA also wants Query Store runtime statistics and deadlock reports in the same workspace. This is a hands-on lab: provision, generate activity, query the workspace, then answer a multiple-choice question."

**Piece 2, the provisioning.** "An Azure CLI script. Region West Europe, a random suffix, a resource group rg dash dp800 dash courtdocket dash suffix. az monitor log dash analytics workspace create, SKU PerGB2018, retention thirty days. The script saves the workspace resource id in LAW underscore ID and the workspace customer id, a GUID, in LAW underscore CID. This time the SQL server is created with a SQL admin, dash u docketadmin and dash p a password, because the lab needs a failed SQL login later. A firewall rule for your IP. az sql db create makes CourtDocket in the Basic DTU tier with local backup redundancy. The script saves the database resource id in DB underscore ID."

**Piece 3, A1 and A2, the audit policies.** "A1 is az sql server audit dash policy update on the server with dash dash state Enabled, dash dash lats Enabled, meaning Log Analytics target state, and dash dash lawri set to the workspace resource id. No actions are given, so the default action groups apply. A2 is az sql db audit dash policy update on the database, same state and workspace flags, plus dash dash actions with two entries: FAILED underscore DATABASE underscore AUTHENTICATION underscore GROUP, and the string SELECT ON SCHEMA colon colon Docket BY public."

**Piece 4, A3, the diagnostic setting.** "A3 is az monitor diagnostic dash settings create named ds dash courtdocket, on the database resource id, with dash dash workspace set to the workspace id, and dash dash logs as a JSON array of two objects: category QueryStoreRuntimeStatistics enabled true, and category Deadlocks enabled true. Then az monitor diagnostic dash settings list on the database, output as a table. The comment says: note the extra setting named SQLSecurityAuditEvents underscore followed by a GUID-like suffix."

**Piece 5, the schema and data.** "Connected with sqlcmd as docketadmin, you create a schema Docket and two tables. Docket dot Hearings has HearingId, an integer primary key; CaseNo, varchar twelve; Room, a tinyint; and HeldOn, a date. Docket dot Rooms has Room, a tinyint primary key, and Name, varchar twenty. Two rooms: one, Courtroom A; two, Courtroom B. Two hearings: hearing 1, case CV dash 2026 dash 0101, room 1, held on the tenth of September 2026. Hearing 2, case CR dash 2026 dash 0202, room 2, held on the eleventh."

**Piece 6, the workload.** "W1 is a read of the audited schema: SELECT CaseNo and room Name FROM Docket dot Hearings JOIN Docket dot Rooms on Room. W2 is a write that must read the schema to find its row: UPDATE Docket dot Hearings SET Room equals 1 WHERE HearingId equals 2. W3, from bash, is sqlcmd as docketadmin with a wrong password, running SELECT 1. That is a failed SQL login. W4 is sqlcmd with the right password, running SELECT COUNT star FROM Docket dot Hearings."

**Piece 7, the first KQL query.** "After a few minutes, because ingestion is not instantaneous, you run az monitor log dash analytics query with dash w the workspace customer id and dash t PT2H, meaning the last two hours. The first KQL: AzureDiagnostics, pipe, where Category equals equals the string SQLSecurityAuditEvents, pipe, project event underscore time underscore t, is underscore server underscore level underscore audit underscore s, action underscore name underscore s, server underscore principal underscore name underscore s, database underscore name underscore s, succeeded underscore s, statement underscore s, pipe, order by event time ascending."

**Piece 8, the second KQL query.** "The second KQL: AzureDiagnostics, pipe, summarize count by Category and is underscore server underscore level underscore audit underscore s."

**Piece 9, option a.** "Option a says: because A2 configured a database-level policy, it replaces the server-level policy for CourtDocket. The workspace receives only the SELECT ON SCHEMA Docket and failed-authentication events for this database, and the server-level BATCH underscore COMPLETED underscore GROUP rows stop for it."

**Piece 10, option b.** "Option b says: both audits run side by side and both write to the AzureDiagnostics table with Category equal to SQLSecurityAuditEvents. W1 therefore yields a row from the server audit, is underscore server underscore level underscore audit underscore s true, batch completed, and a row from the database audit, false, the SELECT action on the schema. W2 is captured by the database audit too, because the UPDATE with a WHERE uses the SELECT permission on Docket dot Hearings. W3 produces a failed-authentication row, whereas a failed Microsoft Entra login would produce none. statement underscore s is truncated at four thousand characters. The QueryStoreRuntimeStatistics and Deadlocks categories land in the same AzureDiagnostics table, distinguished by Category, and the deadlock report is in deadlock underscore xml underscore s."

**Piece 11, option c.** "Option c says: audit records go to a dedicated resource-specific table named SQLSecurityAuditEvents. AzureDiagnostics only receives the two diagnostic-setting categories, which is why the second query returns rows for QueryStoreRuntimeStatistics and Deadlocks but none for auditing."

**Piece 12, option d.** "Option d says: the extra SQLSecurityAuditEvents underscore diagnostic setting listed by A3's last command is a leftover of the portal experience and can be deleted. az sql server audit dash policy show still reports state Enabled, so audit records keep flowing to the workspace."

## 3. Setup script (reference only; do not read verbatim unless asked)

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

```bash
az monitor log-analytics query -w $LAW_CID -t PT2H --analytics-query '
AzureDiagnostics
| where Category == "SQLSecurityAuditEvents"
| project event_time_t, is_server_level_audit_s, action_name_s, server_principal_name_s, database_name_s, succeeded_s, statement_s
| order by event_time_t asc' -o table

az monitor log-analytics query -w $LAW_CID -t PT2H --analytics-query '
AzureDiagnostics | summarize count() by Category, is_server_level_audit_s' -o table
```

Cleanup: `az group delete -n $RG --yes --no-wait`

## 4. The question (ask exactly this)

"Which statement about what appears in the workspace is correct? Option a, option b, option c, or option d? I will repeat any option if you like."

Options in full:

- **a.** Because A2 configured a database-level policy, it **replaces** the server-level policy for `CourtDocket`: the workspace receives only the `SELECT ON SCHEMA::Docket` and failed-authentication events for this database, and the server-level `BATCH_COMPLETED_GROUP` rows stop for it.
- **b.** Both audits run **side by side** and both write to the `AzureDiagnostics` table with `Category == "SQLSecurityAuditEvents"`. W1 therefore yields a row from the server audit (`is_server_level_audit_s = "true"`, batch completed) *and* a row from the database audit (`"false"`, the `SELECT` action on the schema); W2 is captured by the database audit too, because the `UPDATE ... WHERE` uses the `SELECT` permission on `Docket.Hearings`. W3 produces a failed-authentication row, whereas a failed **Microsoft Entra** login would produce none. `statement_s` is truncated at 4,000 characters. The `QueryStoreRuntimeStatistics` and `Deadlocks` categories land in the same `AzureDiagnostics` table, distinguished by `Category` (the deadlock report is in `deadlock_xml_s`).
- **c.** Audit records go to a dedicated resource-specific table named `SQLSecurityAuditEvents`; `AzureDiagnostics` only receives the two diagnostic-setting categories, which is why the second query returns rows for `QueryStoreRuntimeStatistics` and `Deadlocks` but none for auditing.
- **d.** The extra `SQLSecurityAuditEvents_*` diagnostic setting listed by A3's last command is a leftover of the portal experience and can be deleted; `az sql server audit-policy show` still reports `state: Enabled`, so audit records keep flowing to the workspace.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct: b.**

- **a is wrong.** There is no override. Enabling a database policy adds a second audit that exists side by side with the server audit; it does not detach the database from the server policy. Only disabling the server policy changes what the server audit captures.
- **c is wrong.** Auditing to Log Analytics writes to the AzureDiagnostics table with Category SQLSecurityAuditEvents (plus DevOpsOperationsAudit for Microsoft support operations). A resource-specific dedicated table exists only for diagnostic settings created with export-to-resource-specific or destination type Dedicated, which A3 did not do. The second query returns rows for all three categories.
- **d is wrong.** The auto-created SQLSecurityAuditEvents_ diagnostic setting is critical for auditing to external targets. If it is deleted, auditing fails silently and no records reach the workspace, even though the policy still reports Enabled. The documentation recommends an activity-log alert on diagnostic-setting deletion.

Facts behind b: the default action groups of a policy are BATCH_COMPLETED_GROUP, SUCCESSFUL_DATABASE_AUTHENTICATION_GROUP and FAILED_DATABASE_AUTHENTICATION_GROUP; the flag is_server_level_audit_s tells the two audits apart; an UPDATE with a WHERE exercises the SELECT permission; a failed Entra login never reaches the database so no audit record exists; statement is capped at 4,000 characters; the deadlock XML is in deadlock_xml_s.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with the relationship between a server policy and a database policy. Does the documentation say one wins, or that they coexist?"
2. "Option a says the server audit stops for this database. If the two audits coexist, can that be right?"
3. "Now the destination. When the audit target is Log Analytics, which table receives the audit rows, and does A3 ask for a dedicated table anywhere?"
4. "Option c claims a dedicated table. A3 has no export-to-resource-specific flag, so where do the rows actually go?"
5. "Two options remain, b and d. Option d says a diagnostic setting created automatically can be deleted without effect. What does the documentation say happens to auditing when it is deleted, and does the policy state reflect it?"
6. "The remaining option makes several claims: side-by-side audits, one table with a flag, W2 counting as a read, Entra failures missing, four thousand characters, and the deadlock column. Check them one by one; they all match the documentation."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, the more specific policy wins" | Applies a precedence model that does not exist | "Is there a precedence rule between server and database auditing, or a coexistence rule?" |
| "c, resource-specific tables are the modern default" | Thinks dedicated tables are automatic | "Which flag on a diagnostic setting selects a dedicated table, and did A3 use it?" |
| "d, the policy says Enabled so it is fine" | Trusts the policy state over the plumbing | "Who actually delivers the records to the workspace: the policy object, or the diagnostic setting? What happens if the delivery path is gone?" |
| "b is wrong because W2 is an UPDATE, not a SELECT" | Does not know audit fires on permission checks | "To update where HearingId equals 2, what must the engine do first, and which permission does that use?" |
| "b is wrong because a failed Entra login would also be audited" | Does not know where Entra credentials are verified | "Where is an Entra password checked, and does the database ever get touched when it fails?" |
| "b is wrong because Query Store statistics go to a different table" | Thinks each category has its own table | "Without a dedicated-table flag, where do all diagnostic-setting categories land, and how are they told apart?" |

## 8. Teaching notes (after the answer is complete or revealed)

Azure SQL Database auditing has no CREATE SERVER AUDIT. Two policies exist:

- **Server policy** applies to all existing and newly created databases on the server. `az sql server audit-policy update --state Enabled --lats Enabled --lawri <workspace id>`. With no actions given, the default groups are BATCH_COMPLETED_GROUP, SUCCESSFUL_DATABASE_AUTHENTICATION_GROUP and FAILED_DATABASE_AUTHENTICATION_GROUP. That is A1.
- **Database policy** exists side by side with the server audit. `az sql db audit-policy update ... --actions` replaces the defaults with a custom list, like a CREATE DATABASE AUDIT SPECIFICATION on SQL Server. That is A2. No precedence; each audit evaluates its own list and writes its own records. That is why a is wrong.
- Other targets use the same command with `--bsts Enabled --storage-account` for blob storage or `--ehts Enabled --ehari` for Event Hubs.

Where the records land:

- Log Analytics target: the AzureDiagnostics table, Category SQLSecurityAuditEvents. Field names carry type suffixes: `_s` string, `_d` number, `_t` datetime. is_server_level_audit_s tells the two audits apart; also action_name_s, server_principal_name_s, database_name_s, succeeded_s, statement_s, client_ip_s, application_name_s. The server-level records come from the re-architected single extended-event session for the whole server; on storage targets they go to the master folder. That is why c is wrong: a dedicated table only appears with `--export-to-resource-specific true` or `--log-analytics-destination-type Dedicated`.
- W1 yields two rows: server audit, batch completed, flag true; database audit, SELECT on the schema, flag false. W2 is captured by the database audit as a read, because UPDATE with a WHERE checks the SELECT permission on Docket.Hearings; the UPDATE itself is not in A2's list. The server audit records the batch.
- W3, a SQL authentication failure, is checked at the database and captured by FAILED_DATABASE_AUTHENTICATION_GROUP. A failed Entra login never reaches the database, so no audit record; those live in the Entra sign-in logs.
- statement and data_sensitivity_information are capped at 4,000 characters.
- A3's categories are resource logs of the database, exported by a diagnostic setting into the same AzureDiagnostics table, distinguished by Category: QueryStoreRuntimeStatistics with OperationName QueryStoreRuntimeStatisticsEvent and query_hash_s; Deadlocks with deadlock_xml_s. Other categories: SQLInsights, AutomaticTuning, QueryStoreWaitStatistics, Errors, DatabaseWaitStatistics, Timeouts, Blocks.

The silent failure:

- When auditing targets Event Hubs or Log Analytics, Azure creates an extra diagnostic setting named SQLSecurityAuditEvents_ followed by a suffix. It is critical. Delete it and auditing fails silently: no records reach the target, and `az sql server audit-policy show` still says Enabled. Hence the recommendation to create an activity-log alert on diagnostic-setting deletion. That is why d is wrong.

Memory hook: "Server and database audits coexist. One table, AzureDiagnostics, one category, one flag. Audited means permission used. Entra failures never arrive. Delete the auto setting and auditing dies quietly."

## 9. Follow-up oral questions (optional)

1. "How would you send the two resource-log categories to a dedicated table instead of AzureDiagnostics?" (Add --export-to-resource-specific true, or --log-analytics-destination-type Dedicated, to the diagnostic setting.)
2. "Where would you look for a failed Microsoft Entra sign-in to this database?" (The Entra sign-in logs; the attempt never reaches the database, so SQL auditing has nothing.)
3. "What is the recommended safeguard against someone deleting the SQLSecurityAuditEvents diagnostic setting?" (An activity-log alert on diagnostic-setting deletion; the audit policy state alone will not warn you.)

## 10. References

- Auditing for Azure SQL Database and Azure Synapse Analytics: https://learn.microsoft.com/en-us/azure/azure-sql/database/auditing-overview
- Set up auditing for Azure SQL Database: https://learn.microsoft.com/en-us/azure/azure-sql/database/auditing-setup
- Analyze audit logs and reports: https://learn.microsoft.com/en-us/azure/azure-sql/database/auditing-analyze-audit-logs
- Diagnostic telemetry for Azure SQL Database (resource log categories): https://learn.microsoft.com/en-us/azure/azure-sql/database/metrics-diagnostic-telemetry-logging-streaming-export-configure
- az sql server audit-policy: https://learn.microsoft.com/en-us/cli/azure/sql/server/audit-policy
- az sql db audit-policy: https://learn.microsoft.com/en-us/cli/azure/sql/db/audit-policy
- az monitor diagnostic-settings: https://learn.microsoft.com/en-us/cli/azure/monitor/diagnostic-settings
- AzureDiagnostics table and resource-specific mode: https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/resource-logs
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
