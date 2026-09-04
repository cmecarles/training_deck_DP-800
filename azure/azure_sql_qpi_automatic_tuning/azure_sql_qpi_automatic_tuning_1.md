# SQL Server question — Azure SQL Query Performance Insight and Automatic Tuning 1

## Statement

A container terminal keeps one million crane movements in an **Azure SQL Database** named `CraneYard`. A stored procedure that reports tonnage per port suffers a classic parameter-sniffing regression, and the operations team wants to see how Azure SQL Database surfaces and repairs it: Query Store, **Query Performance Insight**, and **automatic tuning**.

This is a **hands-on** question: provision, generate the regression, inspect the catalog, then answer the multiple-choice question.

### Provisioning (Azure CLI, bash; `az login` first)

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

Connect with sqlcmd (Go): `sqlcmd -S $SQL_FQDN -d $DB --authentication-method ActiveDirectoryDefault -I` (ODBC sqlcmd: `-G -U "$ADMIN_UPN" -I`).

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

### The regression workload and the inspection

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

Then open the database in the Azure portal: **Intelligent Performance › Query Performance Insight** and **Intelligent Performance › Automatic tuning**.

Which statement about this lab is **correct**?

### a.

`sys.database_automatic_tuning_options` on the new database shows `FORCE_LAST_GOOD_PLAN` with `desired_state_desc = DEFAULT` / `actual_state_desc = ON` and `CREATE_INDEX`, `DROP_INDEX` with `DEFAULT` / `OFF`, because the database inherits the server, which inherits the Azure defaults. If the engine generates a plan-regression recommendation for `usp_PortTonnage`, it is applied by the system while `FORCE_LAST_GOOD_PLAN` is on (`state.currentValue` moves `Verifying` → `Success` or `Reverted`), whereas a plan forced by hand with the script from `details` gets no automatic verification or rollback. The T3 PATCH is rejected with 409 because `Default` is not allowed for an option while `desiredState` is `Custom`.

### b.

Query Performance Insight reads `sys.dm_exec_query_stats`, so the regression is visible on the blade immediately after W3 even if Query Store is `READ_ONLY`; Query Store is only needed for the *Performance recommendations* blade.

### c.

T2's first statement fails on Azure SQL Database with `Msg 15707 Automatic Tuning is available only in the Enterprise and Developer editions of SQL Server.`; automatic tuning can only be switched on from the portal or the REST API, never with T-SQL.

### d.

After `SET AUTOMATIC_TUNING (CREATE_INDEX = ON)`, `sys.dm_db_tuning_recommendations` immediately lists a `CREATE_INDEX` recommendation covering `(PortCode) INCLUDE (Weight)`, because the DMV is populated from the missing-index DMVs each time a plan with a missing-index hint is compiled.

**Cost and cleanup.** Serverless GP 1 vCore is billed per second of activity; the load takes a few minutes. Delete everything at the end:

```bash
az group delete -n $RG --yes --no-wait
```

## Correct Answer

**a**

## Explanation

### What the lab shows

Query Store is **enabled by default** on every new Azure SQL Database (it cannot be enabled for `master`), so T1 already returns `actual_state_desc = READ_WRITE`; the default capture mode is `AUTO` (skips infrequent, insignificant queries — the lab switches to `ALL` so that 20 executions are surely captured), `stale_query_threshold_days` is 30 and size-based cleanup is `AUTO`. After W1–W3 the flushed Query Store holds **one `query_id` with two plans** for the procedure. Executed on SQL Server 2025 with the same script, the catalog returned:

| plan_id | count_executions | avg_cpu_us | avg_reads | has_seek |
|---|---|---|---|---|
| 6 | 20 | 2,522 | 166 | 1 |
| 7 | 21 | 1,258,446 – 1,831,024 | 3,229 | 0 |

Plan 7 (compiled for `ZZZZZ`, a clustered index scan) is ~500× more CPU than plan 6 for the same `P0007` calls. That is exactly the pattern **automatic plan correction** looks for: the same `query_id`, a new plan whose CPU per execution is significantly worse than the last known good plan.

### Why option a is correct

- **Defaults and inheritance.** New servers inherit the *Azure defaults* `FORCE_LAST_GOOD_PLAN` enabled, `CREATE_INDEX` disabled, `DROP_INDEX` disabled, and every database inherits its server unless overridden. In `sys.database_automatic_tuning_options`, `desired_state_desc` is what *you* set (`DEFAULT` = inherit), `actual_state_desc` is what runs, and `reason_desc` explains a difference (`QUERY_STORE_OFF`, `QUERY_STORE_READ_ONLY`, `DISABLED` = disabled by the system, `NOT_SUPPORTED`). `ALTER DATABASE CURRENT SET AUTOMATIC_TUNING = INHERIT` (T2, second statement) puts every option back to `DEFAULT`; `AUTO` applies Azure defaults and `CUSTOM` requires you to set each option; `(FORCE_LAST_GOOD_PLAN = ON, CREATE_INDEX = ON, DROP_INDEX = OFF)` sets them individually.
- **What happens to a recommendation.** `sys.dm_db_tuning_recommendations` (permission `VIEW SERVER PERFORMANCE STATE`) exposes `type = FORCE_LAST_GOOD_PLAN`, `reason`, `score`, a `state` JSON (`currentValue` ∈ Active, Verifying, Success, Reverted, Expired; `reason` such as `AutomaticTuningOptionNotEnabled`, `LastGoodPlanForced`, `PlanForcedByUser`, `SchemaChanged`) and a `details` JSON with `planForceDetails.regressedPlanId` / `recommendedPlanId` and `implementationDetails.script` (`exec sp_query_store_force_plan @query_id = ..., @plan_id = ...`). With the option **on**, the system applies the recommendation and *validates* it (30 minutes to 72 hours depending on execution frequency), reverting on regression. Applied manually through T-SQL, "the automatic performance validation and reversal mechanisms are not available". Whether a recommendation appears during your lab window is up to the engine's detection cadence; the states and the verification rule are deterministic.
- **The REST/portal contract.** `PATCH .../automaticTuning/current?api-version=2021-11-01` with `desiredState` `Auto | Inherit | Custom` and per-option `desiredState` `On | Off | Default` is what the **Automatic tuning** blade calls. The documented error list includes `409 DefaultAdvisorStateNotAllowedInCustomDbMode — DEFAULT advisor state is not allowed in CUSTOM mode`, which is precisely T3's body. The GET response reports `actualState`, `reasonCode` and `reasonDesc` (`AutoConfigured`, `InheritedFromServer`, `QueryStoreOff`, ...) per option — including a fourth option, `maintainIndex`.

### Why option b is wrong

Query Performance Insight is built **on Query Store**: "Query Performance Insight requires that Query Store is active on your database ... If Query Store is not running, the Azure portal will prompt you to enable it." When Query Store is read-only (out of space) the blade shows "Query Store is not properly configured on this database". It also needs "a couple hours of data" and shows only the top 5–20 queries by CPU, duration or execution count (no DDL). The Performance-recommendations blade is the Database Advisor, a different consumer of the same telemetry.

### Why option c is wrong

`Msg 15707` is what the **local SQL Server 2025 Standard Developer** edition returns for `ALTER DATABASE ... SET AUTOMATIC_TUNING (FORCE_LAST_GOOD_PLAN = ON)` (verified verbatim), because on SQL Server automatic plan correction is an Enterprise/Developer feature. Azure SQL Database has no editions; the T-SQL is the documented way to configure the option, and Managed Instance supports *only* T-SQL (and only `FORCE_LAST_GOOD_PLAN`).

### Why option d is wrong

`CREATE_INDEX` recommendations come from the Database Advisor's workload analysis, not from a per-compilation copy of `sys.dm_db_missing_index_details`; they are produced only after the service has observed the workload, are withheld when the index would push storage over 90 % or the table exceeds 10 GB, and are applied only at times of low utilization with automatic verification and rollback. Nothing appears "immediately", and index recommendations never reach `sys.dm_db_tuning_recommendations` on SQL Server, whose only option is `FORCE_LAST_GOOD_PLAN`.

Hands-on question (Azure subscription required); the T-SQL fragments that do not depend on Azure were checked on SQL Server 2025 RTM 17.0.1000.7; Azure-side behaviour is taken from the official documentation.

## DP-800 Exam Rule to Remember

```text
Azure SQL Database, per database:
  Query Store ON by default (READ_WRITE, capture AUTO, 30-day cleanup) -> feeds BOTH QPI and automatic tuning
  Query Performance Insight = portal, Intelligent Performance; top CPU/duration/count; needs hours of Query Store data;
                              read-only/full Query Store -> "not properly configured"
  Automatic tuning options: FORCE_LAST_GOOD_PLAN (Azure default ON) | CREATE_INDEX (OFF) | DROP_INDEX (OFF) [| MAINTAIN_INDEX]
     inheritance: database -> server -> Azure defaults;  T-SQL: SET AUTOMATIC_TUNING = AUTO|INHERIT|CUSTOM
                                                                SET AUTOMATIC_TUNING (opt = ON|OFF|DEFAULT, ...)
     REST/portal: PATCH .../automaticTuning/current  desiredState Auto|Inherit|Custom  (Custom + Default -> 409)
  sys.database_automatic_tuning_options: desired vs actual state, reason_desc (QUERY_STORE_OFF/READ_ONLY, DISABLED)
  sys.dm_db_tuning_recommendations: state.currentValue Active|Verifying|Success|Reverted|Expired,
                                    details.planForceDetails + implementationDetails.script
  system-applied = verified (30 min-72 h) and auto-reverted; applied by hand via T-SQL = no verification
SQL Server: FLGP only, Enterprise/Developer (Standard -> Msg 15707); Managed Instance: FLGP via T-SQL only
```
