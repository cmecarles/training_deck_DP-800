# Instructor-Examiner guide — Azure SQL Query Performance Insight and Automatic Tuning 1

Companion to [azure_sql_qpi_automatic_tuning_1.md](azure_sql_qpi_automatic_tuning_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**This is a hands-on Azure lab question.** Before anything else, ask: "Have you already run this lab in your own subscription?" If yes, go through T1, W1 to W3, the Query Store inspection, T2 and T3, and ask what the learner observed at each step before you quiz; use the observations to anchor the discussion. If no, walk through the provisioning and the steps in words using section 2, so the question can still be answered from the documented facts.

**This is a multiple-choice question.** Read all four options, pieces 10 to 13, before taking an answer. Each option is a long statement; read it slowly and offer to repeat it.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "dash dash" for `--`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line from section 3 only when asked.

## 1. Exam skill covered

- Functional group: Monitor, troubleshoot and optimize database solutions (20–25%).
- Skill: Optimize database performance.
- Task bullet: Configure Query Store, Query Performance Insight and automatic tuning on Azure SQL Database.
- What is tested: the Azure defaults and inheritance of automatic tuning options, how the engine verifies and reverts a forced plan, what the portal blade and the REST API accept, and what Query Performance Insight depends on.

## 2. Scenario to read aloud

**Piece 1, the story.** "A container terminal keeps one million crane movements in an Azure SQL Database called CraneYard. A stored procedure that reports tonnage per port suffers a classic parameter-sniffing regression. The operations team wants to see how Azure SQL Database surfaces and repairs it, through Query Store, Query Performance Insight, and automatic tuning. This is a hands-on lab: provision, generate the regression, inspect the catalog, then answer a multiple-choice question."

**Piece 2, the provisioning.** "An Azure CLI script in bash. Region West Europe, a random suffix, a resource group rg dash dp800 dash craneyard dash suffix. az sql server create with dash dash enable dash ad dash only dash auth and the external admin flags pointing at you. A firewall rule for your IP. az sql db create makes CraneYard as General Purpose, Gen5, one vCore, Serverless, auto-pause sixty minutes, min capacity zero point five, local backup redundancy. The script also builds a variable DB underscore URL: the Azure Resource Manager URL of the database, management dot azure dot com, subscriptions, resource groups, providers Microsoft dot Sql, servers, databases. That URL is used later for the REST call."

**Piece 3, T1, what a new database reports.** "Connected with sqlcmd Go as yourself, step T1 runs two catalog queries. First, from sys dot database underscore query underscore store underscore options: actual underscore state underscore desc, query underscore capture underscore mode underscore desc, max storage size, interval length, stale query threshold days, and size-based cleanup mode. Second, from sys dot database underscore automatic underscore tuning underscore options: name, desired underscore state underscore desc, actual underscore state underscore desc and reason underscore desc. Then ALTER DATABASE CURRENT SET QUERY underscore STORE with QUERY underscore CAPTURE underscore MODE equals ALL and INTERVAL underscore LENGTH underscore MINUTES equals one, to capture everything in one-minute buckets."

**Piece 4, the table and data.** "A schema Cargo and a table Cargo dot Shipments with four columns: ShipmentId, an identity integer primary key. PortCode, char five. Weight, decimal nine comma two. ShippedOn, a date. One million rows are inserted. Port ZZZZZ owns ninety-nine percent of them. The remaining one percent is spread over one hundred ninety-nine small ports named P followed by four digits, such as P0007. Then a nonclustered index IX underscore Shipments underscore Port on PortCode alone. The comment says it is deliberately not covering. Finally a procedure Cargo dot usp underscore PortTonnage that takes a PortCode and returns PortCode, SUM of Weight as Tonnage and COUNT star as Shipments, grouped by PortCode."

**Piece 5, the regression workload.** "W1 executes the procedure twenty times for the small port P0007. That compiles a plan with an index seek and a key lookup. W2 runs sp underscore recompile on the procedure, which evicts the plan, then executes it once for the giant port ZZZZZ. That compiles a clustered index scan. W3 executes it twenty more times for P0007. The small port now reuses the scan plan. That is the regression."

**Piece 6, the Query Store inspection.** "Then sp underscore query underscore store underscore flush underscore db, and a query joining sys dot query underscore store underscore query, sys dot query underscore store underscore plan and sys dot query underscore store underscore runtime underscore stats, filtered to the procedure's object id. It shows query id, plan id, is underscore forced underscore plan, count of executions, average CPU in microseconds, average logical reads, and a flag has underscore seek computed by an XQuery exist over the plan XML looking for a RelOp whose PhysicalOp is Index Seek."

**Piece 7, T2, automatic tuning three ways.** "T2 runs ALTER DATABASE CURRENT SET AUTOMATIC underscore TUNING with CREATE underscore INDEX equals ON, DROP underscore INDEX equals OFF, FORCE underscore LAST underscore GOOD underscore PLAN equals ON. Then selects the tuning options. Then ALTER DATABASE CURRENT SET AUTOMATIC underscore TUNING equals INHERIT, and selects the options again. Then it queries sys dot dm underscore db underscore tuning underscore recommendations: name, type, reason, score, and JSON values pulled out of two JSON columns. From state: dollar dot currentValue and dollar dot reason. From details: planForceDetails dot regressedPlanId, planForceDetails dot recommendedPlanId, and implementationDetails dot script."

**Piece 8, T3, the REST call.** "T3 is az rest with method patch against the database URL followed by slash automaticTuning slash current, api-version 2021 dash 11 dash 01. The JSON body has properties with desiredState equal to Custom, and an options object with three entries: createIndex desiredState On, dropIndex desiredState Off, and forceLastGoodPlan desiredState Default. Note that word: Default, while the top-level mode is Custom. Then az rest get on the same URL, showing the properties."

**Piece 9, the portal.** "Finally, open the database in the Azure portal. Under Intelligent Performance there are two blades: Query Performance Insight, and Automatic tuning."

**Piece 10, option a.** "Option a says: sys dot database underscore automatic underscore tuning underscore options on the new database shows FORCE underscore LAST underscore GOOD underscore PLAN with desired state DEFAULT and actual state ON, and CREATE underscore INDEX and DROP underscore INDEX with DEFAULT and OFF, because the database inherits the server, which inherits the Azure defaults. If the engine generates a plan-regression recommendation for usp underscore PortTonnage, the system applies it while FORCE underscore LAST underscore GOOD underscore PLAN is on, and state dot currentValue moves from Verifying to Success or Reverted, whereas a plan forced by hand with the script from details gets no automatic verification or rollback. And the T3 PATCH is rejected with 409 because Default is not allowed for an option while desiredState is Custom."

**Piece 11, option b.** "Option b says: Query Performance Insight reads sys dot dm underscore exec underscore query underscore stats, so the regression is visible on the blade immediately after W3 even if Query Store is READ underscore ONLY. Query Store is only needed for the Performance recommendations blade."

**Piece 12, option c.** "Option c says: T2's first statement fails on Azure SQL Database with Msg 15707, Automatic Tuning is available only in the Enterprise and Developer editions of SQL Server. Automatic tuning can only be switched on from the portal or the REST API, never with T-SQL."

**Piece 13, option d.** "Option d says: after SET AUTOMATIC underscore TUNING with CREATE underscore INDEX equals ON, sys dot dm underscore db underscore tuning underscore recommendations immediately lists a CREATE underscore INDEX recommendation covering PortCode INCLUDE Weight, because the DMV is populated from the missing-index DMVs each time a plan with a missing-index hint is compiled."

## 3. Setup script (reference only; do not read verbatim unless asked)

```bash
LOCATION="westeurope"
SUFFIX=$RANDOM
RG="rg-dp800-craneyard-$SUFFIX"
SQL="sql-craneyard-$SUFFIX"
DB="CraneYard"
ADMIN_UPN=$(az ad signed-in-user show --query userPrincipalName -o tsv)
ADMIN_OID=$(az ad signed-in-user show --query id -o tsv)
MYIP=$(curl -s https://api.ipify.org)

az group create -n $RG -l $LOCATION
az sql server create -g $RG -n $SQL -l $LOCATION --enable-ad-only-auth \
  --external-admin-principal-type User --external-admin-name "$ADMIN_UPN" --external-admin-sid $ADMIN_OID
az sql server firewall-rule create -g $RG -s $SQL -n client --start-ip-address $MYIP --end-ip-address $MYIP
az sql db create -g $RG -s $SQL -n $DB -e GeneralPurpose -f Gen5 -c 1 --compute-model Serverless \
  --auto-pause-delay 60 --min-capacity 0.5 --backup-storage-redundancy Local
SQL_FQDN=$(az sql server show -g $RG -n $SQL --query fullyQualifiedDomainName -o tsv)
SUB=$(az account show --query id -o tsv)
DB_URL="https://management.azure.com/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.Sql/servers/$SQL/databases/$DB"
```

Connect: `sqlcmd -S $SQL_FQDN -d $DB --authentication-method ActiveDirectoryDefault -I` (ODBC sqlcmd: `-G -U "$ADMIN_UPN" -I`).

```sql
-- T1: what a brand-new database reports
SELECT actual_state_desc, query_capture_mode_desc, max_storage_size_mb, interval_length_minutes,
       stale_query_threshold_days, size_based_cleanup_mode_desc
FROM sys.database_query_store_options;
SELECT name, desired_state_desc, actual_state_desc, reason_desc FROM sys.database_automatic_tuning_options;
GO
ALTER DATABASE CURRENT SET QUERY_STORE (QUERY_CAPTURE_MODE = ALL, INTERVAL_LENGTH_MINUTES = 1);   -- capture everything, 1-minute buckets
GO
CREATE SCHEMA Cargo;
GO
CREATE TABLE Cargo.Shipments
(
    ShipmentId INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    PortCode   CHAR(5)       NOT NULL,
    Weight     DECIMAL(9,2)  NOT NULL,
    ShippedOn  DATE          NOT NULL
);
-- 1,000,000 rows: port ZZZZZ owns 99 % of them, 199 small ports share the rest
INSERT INTO Cargo.Shipments (PortCode, Weight, ShippedOn)
SELECT CASE WHEN n % 100 = 0 THEN CONCAT('P', RIGHT('0000' + CAST(n % 199 AS VARCHAR(4)), 4)) ELSE 'ZZZZZ' END,
       (n % 500) + 0.5, DATEADD(DAY, n % 365, '20260101')
FROM (SELECT TOP (1000000) ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n
      FROM sys.all_columns a CROSS JOIN sys.all_columns b CROSS JOIN sys.all_columns c) AS x;
CREATE NONCLUSTERED INDEX IX_Shipments_Port ON Cargo.Shipments (PortCode);     -- deliberately NOT covering
GO
CREATE PROCEDURE Cargo.usp_PortTonnage @PortCode CHAR(5)
AS
SELECT PortCode, SUM(Weight) AS Tonnage, COUNT(*) AS Shipments
FROM Cargo.Shipments WHERE PortCode = @PortCode GROUP BY PortCode;
GO
```

```sql
-- W1: compiled for a small port -> seek + key lookup, 20 runs
DECLARE @i INT = 0; WHILE @i < 20 BEGIN EXEC Cargo.usp_PortTonnage 'P0007'; SET @i += 1; END
GO
-- W2: evict the plan and recompile for the giant port -> clustered index scan
EXEC sp_recompile 'Cargo.usp_PortTonnage';
EXEC Cargo.usp_PortTonnage 'ZZZZZ';
GO
-- W3: the small port now reuses the scan plan: the regression, 20 runs
DECLARE @i INT = 0; WHILE @i < 20 BEGIN EXEC Cargo.usp_PortTonnage 'P0007'; SET @i += 1; END
GO
EXEC sp_query_store_flush_db;
SELECT q.query_id, p.plan_id, p.is_forced_plan, rs.count_executions,
       CAST(rs.avg_cpu_time AS INT) AS avg_cpu_us, CAST(rs.avg_logical_io_reads AS INT) AS avg_reads,
       CAST(p.query_plan AS XML).exist('declare default element namespace "http://schemas.microsoft.com/sqlserver/2004/07/showplan"; //RelOp[@PhysicalOp="Index Seek"]') AS has_seek
FROM sys.query_store_query q JOIN sys.query_store_plan p ON p.query_id = q.query_id
JOIN sys.query_store_runtime_stats rs ON rs.plan_id = p.plan_id
WHERE q.object_id = OBJECT_ID('Cargo.usp_PortTonnage') ORDER BY p.plan_id, rs.runtime_stats_interval_id;
GO
-- T2: automatic tuning, three ways
ALTER DATABASE CURRENT SET AUTOMATIC_TUNING (CREATE_INDEX = ON, DROP_INDEX = OFF, FORCE_LAST_GOOD_PLAN = ON);
SELECT name, desired_state_desc, actual_state_desc, reason_desc FROM sys.database_automatic_tuning_options;
ALTER DATABASE CURRENT SET AUTOMATIC_TUNING = INHERIT;
SELECT name, desired_state_desc, actual_state_desc, reason_desc FROM sys.database_automatic_tuning_options;
GO
SELECT name, type, reason, score, JSON_VALUE(state, '$.currentValue') AS current_state, JSON_VALUE(state, '$.reason') AS state_reason,
       JSON_VALUE(details, '$.planForceDetails.regressedPlanId') AS regressed_plan,
       JSON_VALUE(details, '$.planForceDetails.recommendedPlanId') AS recommended_plan,
       JSON_VALUE(details, '$.implementationDetails.script') AS script
FROM sys.dm_db_tuning_recommendations;
```

```bash
# T3: the same switch through the REST API (what the portal's "Automatic tuning" blade calls)
az rest --method patch --url "$DB_URL/automaticTuning/current?api-version=2021-11-01" \
  --body '{"properties":{"desiredState":"Custom","options":{"createIndex":{"desiredState":"On"},"dropIndex":{"desiredState":"Off"},"forceLastGoodPlan":{"desiredState":"Default"}}}}'
az rest --method get --url "$DB_URL/automaticTuning/current?api-version=2021-11-01" --query properties
```

Cleanup: `az group delete -n $RG --yes --no-wait`

## 4. The question (ask exactly this)

"Which statement about this lab is correct? Option a, option b, option c, or option d? I will repeat any option if you like."

Options in full:

- **a.** `sys.database_automatic_tuning_options` on the new database shows `FORCE_LAST_GOOD_PLAN` with `desired_state_desc = DEFAULT` / `actual_state_desc = ON` and `CREATE_INDEX`, `DROP_INDEX` with `DEFAULT` / `OFF`, because the database inherits the server, which inherits the Azure defaults. If the engine generates a plan-regression recommendation for `usp_PortTonnage`, it is applied by the system while `FORCE_LAST_GOOD_PLAN` is on (`state.currentValue` moves `Verifying` → `Success` or `Reverted`), whereas a plan forced by hand with the script from `details` gets no automatic verification or rollback. The T3 PATCH is rejected with 409 because `Default` is not allowed for an option while `desiredState` is `Custom`.
- **b.** Query Performance Insight reads `sys.dm_exec_query_stats`, so the regression is visible on the blade immediately after W3 even if Query Store is `READ_ONLY`; Query Store is only needed for the *Performance recommendations* blade.
- **c.** T2's first statement fails on Azure SQL Database with `Msg 15707 Automatic Tuning is available only in the Enterprise and Developer editions of SQL Server.`; automatic tuning can only be switched on from the portal or the REST API, never with T-SQL.
- **d.** After `SET AUTOMATIC_TUNING (CREATE_INDEX = ON)`, `sys.dm_db_tuning_recommendations` immediately lists a `CREATE_INDEX` recommendation covering `(PortCode) INCLUDE (Weight)`, because the DMV is populated from the missing-index DMVs each time a plan with a missing-index hint is compiled.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct: a.**

- **b is wrong.** Query Performance Insight is built on Query Store, not on sys.dm_exec_query_stats. It requires Query Store active and read-write; when Query Store is read-only the blade shows "Query Store is not properly configured on this database". It also needs a couple of hours of data. Performance recommendations is the Database Advisor, another consumer of the same telemetry.
- **c is wrong.** Msg 15707 is what local SQL Server 2025 Standard or Developer edition returns, because automatic plan correction is an Enterprise/Developer feature on SQL Server. Azure SQL Database has no editions; the T-SQL is the documented way, and Managed Instance supports only T-SQL and only FORCE_LAST_GOOD_PLAN.
- **d is wrong.** CREATE_INDEX recommendations come from the Database Advisor's workload analysis after observing the workload, not from a per-compilation copy of the missing-index DMVs. They are withheld when storage would exceed 90 percent or the table exceeds 10 GB, and applied only at low utilization with verification and rollback. Nothing appears immediately.

What the lab actually shows, for reference: T1 returns actual_state READ_WRITE, capture mode AUTO, stale threshold 30 days, size-based cleanup AUTO. After W1 to W3 the Query Store holds one query_id with two plans; on SQL Server 2025 the seek plan ran 20 times at about 2,522 microseconds CPU and 166 reads, has_seek 1; the scan plan ran 21 times at roughly 1.26 to 1.83 million microseconds and 3,229 reads, has_seek 0. About 500 times more CPU for the same P0007 calls.

## 6. Hint ladder (one hint per attempt, in order)

1. "Two of the four options make a claim about where a feature gets its data. Check each claim against what Query Store feeds on Azure SQL Database."
2. "Option c quotes an error message that mentions editions of SQL Server. Does Azure SQL Database have editions?"
3. "Option d says the recommendation appears immediately, from the missing-index DMVs. Who produces CREATE underscore INDEX recommendations on Azure, and when?"
4. "Option b says Query Performance Insight works even when Query Store is read-only. What does the portal show in that case?"
5. "Two options remain, a and one other. Look at option a's three claims: inheritance and defaults, verification of a system-applied plan versus a hand-forced plan, and the 409 on the PATCH with Default inside Custom. Are all three consistent with the documentation?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "b, because DMVs are live and Query Store lags" | Thinks the portal blade reads DMVs | "What does the documentation say Query Performance Insight requires to be active on the database?" |
| "c, I got Msg 15707 on my local instance" | Transfers a SQL Server edition rule to Azure | "You saw that on which product and edition? Does the same rule apply to Azure SQL Database?" |
| "d, the missing index is obvious after the scan" | Confuses the missing-index DMVs with Database Advisor recommendations | "The scan has a missing-index hint, yes. Is that what fills sys dot dm underscore db underscore tuning underscore recommendations, and how fast?" |
| "a is wrong because FORCE_LAST_GOOD_PLAN should be OFF on a new database" | Does not know the Azure defaults | "Which options are on and off in the Azure defaults that a new server inherits?" |
| "a is wrong because the PATCH should succeed, Default is a valid value" | Does not know the Custom mode rule | "Default is valid in general. Is it valid for an option when the top-level desiredState is Custom?" |
| "a is wrong because a hand-forced plan is also verified" | Thinks verification is tied to the plan, not to the actor | "Who verifies and reverts: the plan forcing mechanism itself, or the automatic tuning service that applied it?" |

## 8. Teaching notes (after the answer is complete or revealed)

Start with Query Store on Azure SQL Database:

- Query Store is enabled by default on every new Azure SQL Database, except master. T1 shows READ_WRITE, capture mode AUTO, 30-day stale threshold, size-based cleanup AUTO. The lab switches capture to ALL and one-minute intervals so twenty executions are surely captured. Query Store feeds both Query Performance Insight and automatic tuning.
- After W1 to W3 there is one query_id with two plans, and the scan plan is hundreds of times more expensive for the same small-port parameter. That is exactly the pattern automatic plan correction detects: same query, new plan, significantly worse CPU than the last known good plan.

Then automatic tuning, why a is right:

- **Defaults and inheritance.** Azure defaults are FORCE_LAST_GOOD_PLAN on, CREATE_INDEX off, DROP_INDEX off. A new server inherits those; a database inherits its server unless overridden. In `sys.database_automatic_tuning_options`, desired_state_desc is what you set, DEFAULT meaning inherit; actual_state_desc is what runs; reason_desc explains any difference: QUERY_STORE_OFF, QUERY_STORE_READ_ONLY, DISABLED by the system, NOT_SUPPORTED. `SET AUTOMATIC_TUNING = INHERIT` puts every option back to DEFAULT; AUTO applies the Azure defaults; CUSTOM requires each option set; the parenthesised form sets options individually.
- **What happens to a recommendation.** `sys.dm_db_tuning_recommendations` needs VIEW SERVER PERFORMANCE STATE and exposes type FORCE_LAST_GOOD_PLAN, reason, score, a state JSON with currentValue in Active, Verifying, Success, Reverted, Expired and a reason such as AutomaticTuningOptionNotEnabled, LastGoodPlanForced, PlanForcedByUser, SchemaChanged, plus a details JSON with planForceDetails regressedPlanId and recommendedPlanId and implementationDetails script, which is `exec sp_query_store_force_plan`. With the option on, the system applies it and validates it for 30 minutes to 72 hours depending on execution frequency, reverting on regression. Applied by hand through T-SQL, no automatic validation or reversal. Whether a recommendation appears during the lab window depends on the engine's detection cadence; the states and the rule are deterministic.
- **The REST and portal contract.** PATCH `.../automaticTuning/current?api-version=2021-11-01` with desiredState Auto, Inherit or Custom and per-option desiredState On, Off or Default is what the Automatic tuning blade calls. The documented error 409 DefaultAdvisorStateNotAllowedInCustomDbMode is exactly T3's body. The GET reports actualState, reasonCode and reasonDesc per option, including a fourth option, maintainIndex.

Why the others are wrong:

- b: Query Performance Insight requires Query Store active; read-only Query Store shows "not properly configured"; it needs hours of data and shows only the top 5 to 20 queries by CPU, duration or count.
- c: Msg 15707 is a SQL Server Standard edition message. Azure SQL Database accepts the T-SQL; Managed Instance accepts only T-SQL and only FORCE_LAST_GOOD_PLAN.
- d: CREATE_INDEX recommendations come from the Database Advisor after observing the workload, with size limits, low-utilization application, verification and rollback. On SQL Server the DMV only ever holds FORCE_LAST_GOOD_PLAN entries.

Memory hook: "Query Store feeds everything. Azure default: force last good plan on, indexes off. System-applied plans are verified; hand-forced plans are not. Custom plus Default equals 409."

## 9. Follow-up oral questions (optional)

1. "What is the difference between desired_state_desc and actual_state_desc in sys dot database underscore automatic underscore tuning underscore options?" (Desired is what you set, DEFAULT meaning inherit; actual is what runs; reason_desc explains any gap, for example QUERY_STORE_READ_ONLY.)
2. "How long does the system verify a plan it forced automatically, and what happens if performance regresses?" (30 minutes to 72 hours depending on execution frequency; it reverts the forced plan, and the state shows Reverted.)
3. "Which automatic tuning option does Azure SQL Managed Instance support, and through which interface?" (Only FORCE_LAST_GOOD_PLAN, and only through T-SQL.)

## 10. References

- Automatic tuning in Azure SQL Database and Managed Instance: https://learn.microsoft.com/en-us/azure/azure-sql/database/automatic-tuning-overview
- Enable automatic tuning in the portal, T-SQL and REST: https://learn.microsoft.com/en-us/azure/azure-sql/database/automatic-tuning-enable
- Query Performance Insight: https://learn.microsoft.com/en-us/azure/azure-sql/database/query-performance-insight-use
- Automatic plan correction and sys.dm_db_tuning_recommendations: https://learn.microsoft.com/en-us/sql/relational-databases/automatic-tuning/automatic-tuning
- sys.dm_db_tuning_recommendations: https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-tuning-recommendations-transact-sql
- sys.database_automatic_tuning_options: https://learn.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-database-automatic-tuning-options-transact-sql
- ALTER DATABASE SET options, AUTOMATIC_TUNING and QUERY_STORE: https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-database-transact-sql-set-options
- REST API, Database Automatic Tuning, Update: https://learn.microsoft.com/en-us/rest/api/sql/database-automatic-tuning/update
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
