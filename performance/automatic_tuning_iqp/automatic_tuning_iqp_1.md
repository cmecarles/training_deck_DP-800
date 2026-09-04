# SQL Server question — Automatic Tuning and IQP 1

## Statement

A quarry operator moved its haulage database to a SQL Server 2025 instance (Standard Developer Edition) last month, keeping the compatibility level it had on the old server. Next quarter the database moves again, to **Azure SQL Database**.

```sql
CREATE DATABASE QuarryOps;
GO
ALTER DATABASE QuarryOps SET COMPATIBILITY_LEVEL = 140;
GO
USE QuarryOps;
GO
CREATE SCHEMA Pit;
GO
CREATE TABLE Pit.Loads
(
    LoadId INT          NOT NULL IDENTITY PRIMARY KEY,
    SiteId INT          NOT NULL,
    Tons   DECIMAL(8,2) NOT NULL
);
INSERT INTO Pit.Loads (SiteId, Tons)
SELECT n % 7 + 1, (n % 40) + 12.5
FROM (SELECT TOP (20000) ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n
      FROM sys.all_columns AS a CROSS JOIN sys.all_columns AS b) AS x;
GO
CREATE FUNCTION Pit.fn_Levy (@Tons DECIMAL(8,2))
RETURNS DECIMAL(10,4)
AS
BEGIN
    RETURN @Tons * 0.035;
END;
GO
SELECT name, value, is_value_default
FROM sys.database_scoped_configurations
WHERE name IN ('TSQL_SCALAR_UDF_INLINING', 'BATCH_MODE_ON_ROWSTORE', 'DEFERRED_COMPILATION_TV',
               'PARAMETER_SENSITIVE_PLAN_OPTIMIZATION', 'DOP_FEEDBACK', 'CE_FEEDBACK',
               'OPTIONAL_PARAMETER_OPTIMIZATION')
ORDER BY configuration_id;
```

| name | value | is_value_default |
|---|---|---|
| TSQL_SCALAR_UDF_INLINING | 1 | 1 |
| BATCH_MODE_ON_ROWSTORE | 1 | 1 |
| DEFERRED_COMPILATION_TV | 1 | 1 |
| PARAMETER_SENSITIVE_PLAN_OPTIMIZATION | 1 | 1 |
| CE_FEEDBACK | 1 | 1 |
| DOP_FEEDBACK | 1 | 1 |
| OPTIONAL_PARAMETER_OPTIMIZATION | 1 | 1 |

`sys.sql_modules.is_inlineable` is 1 for `Pit.fn_Levy`. Yet the plan of the levy report `SELECT SUM(Pit.fn_Levy(Tons)) FROM Pit.Loads WHERE SiteId = 3;`, read back from `sys.dm_exec_query_plan`, still contains a `UserDefinedFunction` node — the function is being invoked row by row, not inlined. Query Store is enabled in `READ_WRITE` mode.

Requirements:

1. Make **every intelligent query processing feature the engine supports** active for `QuarryOps` — scalar UDF inlining, batch mode on rowstore, table variable deferred compilation, parameter sensitive plan optimization, DOP and CE feedback, optional parameter plan optimization — without editing any query.
2. One report, `Pit.usp_SiteLevy`, regresses when its UDF is inlined. Disable inlining **for that one statement only**; every other caller of `Pit.fn_Levy` must keep the inlined form.
3. After the move to Azure SQL Database, the DBA wants automatic plan-regression correction **and** automatic index creation, but an index must **never** be dropped automatically. Use the fewest statements that achieve this on the new database.

Which script meets all three requirements?

### a.

```sql
ALTER DATABASE QuarryOps SET COMPATIBILITY_LEVEL = 170;
GO
-- inside Pit.usp_SiteLevy
SELECT SUM(Pit.fn_Levy(Tons)) FROM Pit.Loads WHERE SiteId = @SiteId
OPTION (USE HINT ('DISABLE_TSQL_SCALAR_UDF_INLINING'));
GO
-- Azure SQL Database, after migration
ALTER DATABASE CURRENT SET AUTOMATIC_TUNING (CREATE_INDEX = ON);
```

### b.

```sql
ALTER DATABASE SCOPED CONFIGURATION SET TSQL_SCALAR_UDF_INLINING = ON;
ALTER DATABASE SCOPED CONFIGURATION SET BATCH_MODE_ON_ROWSTORE = ON;
ALTER DATABASE SCOPED CONFIGURATION SET DEFERRED_COMPILATION_TV = ON;
ALTER DATABASE SCOPED CONFIGURATION SET PARAMETER_SENSITIVE_PLAN_OPTIMIZATION = ON;
GO
ALTER FUNCTION Pit.fn_Levy (@Tons DECIMAL(8,2)) RETURNS DECIMAL(10,4)
WITH INLINE = OFF AS BEGIN RETURN @Tons * 0.035; END;
GO
-- Azure SQL Database, after migration
ALTER DATABASE CURRENT SET AUTOMATIC_TUNING = INHERIT;
```

### c.

```sql
ALTER DATABASE QuarryOps SET COMPATIBILITY_LEVEL = 170;
GO
ALTER DATABASE SCOPED CONFIGURATION SET TSQL_SCALAR_UDF_INLINING = OFF;
GO
-- Azure SQL Database, after migration
ALTER DATABASE CURRENT SET AUTOMATIC_TUNING (FORCE_LAST_GOOD_PLAN = ON, CREATE_INDEX = ON, DROP_INDEX = OFF);
```

### d.

```sql
ALTER DATABASE QuarryOps SET COMPATIBILITY_LEVEL = 170;
GO
-- inside Pit.usp_SiteLevy
SELECT SUM(Pit.fn_Levy(Tons)) FROM Pit.Loads WHERE SiteId = @SiteId
OPTION (USE HINT ('DISABLE_SCALAR_UDF_INLINING'));
GO
-- Azure SQL Database, after migration
ALTER DATABASE CURRENT SET AUTOMATIC_TUNING (CREATE_INDEX = ON, DROP_INDEX = ON);
```

## Correct Answer

**a**

## Explanation

Two separate switchboards are being confused in this scenario. **Intelligent query processing** features are gated by the database **compatibility level** and can be *turned off* per database (`ALTER DATABASE SCOPED CONFIGURATION`) or per query (`OPTION (USE HINT ('DISABLE_...'))`). **Automatic tuning** is a different feature (Query-Store-driven plan correction, plus index management in Azure SQL Database) with its own catalog view and defaults.

### Why option a is correct

- **Requirement 1 — compatibility level, not the scoped configurations.** The catalog already shows every IQP configuration at its default `1`; they are *enable* switches, and they only matter once the compatibility level unlocks the feature: "You can make workloads automatically eligible for intelligent query processing by enabling the applicable database compatibility level for the database." At level 140 the database gets adaptive joins, interleaved execution and batch-mode memory grant feedback only. Scalar UDF inlining, batch mode on rowstore, table variable deferred compilation and row-mode memory grant feedback need **150**; parameter sensitive plan optimization, CE feedback and DOP feedback need **160**; optional parameter plan optimization needs **170** (CE feedback for expressions: 160 on SQL Server 2025, 170 on Azure SQL Database). Verified: at level 140 the levy plan contains a `UserDefinedFunction` node; after `SET COMPATIBILITY_LEVEL = 170` (nothing else changed) the same query's plan has **no** `UserDefinedFunction` node — the function is inlined. CE feedback, DOP feedback and memory-grant-feedback persistence additionally "require Query Store to be enabled ... and in `READ_WRITE` mode", which the scenario already has.
- **Requirement 2 — a query-level hint.** `USE HINT ('DISABLE_TSQL_SCALAR_UDF_INLINING')` "disables scalar UDF inlining" for that statement. Verified: with the hint the plan shows the `UserDefinedFunction` node again while an unhinted call still inlines. The list of valid names is `sys.dm_exec_valid_use_hints` (36 names on this build, including `DISALLOW_BATCH_MODE`, `DISABLE_DEFERRED_COMPILATION_TV`, `DISABLE_CE_FEEDBACK`, `DISABLE_DOP_FEEDBACK`, `DISABLE_OPTIMIZED_PLAN_FORCING`, `DISABLE_PARAMETER_SNIFFING`, `FORCE_LEGACY_CARDINALITY_ESTIMATION` and `QUERY_OPTIMIZER_COMPATIBILITY_LEVEL_100 ... _170`). "Some `USE HINT` hints might conflict with ... database scoped configuration settings. In this case, the query level hint (`USE HINT`) always takes precedence." If the procedure text could not be edited, `sys.sp_query_store_set_hints` would attach the same `OPTION (USE HINT (...))` to the query through Query Store.
- **Requirement 3 — Azure defaults do most of the work.** "By default, new servers inherit Azure defaults for automatic tuning settings. Azure defaults are set to `FORCE_LAST_GOOD_PLAN` enabled, `CREATE_INDEX` disabled, and `DROP_INDEX` disabled", and a new database inherits from its server. Plan correction is therefore already ON and index dropping already OFF; the single statement `ALTER DATABASE CURRENT SET AUTOMATIC_TUNING (CREATE_INDEX = ON)` adds the missing piece. Automatically applied recommendations are validated ("from 30 minutes to 72 hours") and reverted if performance regresses; recommendations applied by hand through T-SQL get no such validation.

### Why option b is wrong

The four `ALTER DATABASE SCOPED CONFIGURATION ... = ON` statements set values that are **already** `1`; they are no-ops, and the compatibility level stays at 140, so nothing is inlined and no 150/160/170 feature appears. `ALTER FUNCTION ... WITH INLINE = OFF` disables inlining for **every** caller of `Pit.fn_Levy` (and flips `is_inlineable` to 0), violating "that one statement only". `SET AUTOMATIC_TUNING = INHERIT` re-applies the server's settings — the Azure defaults — which leave `CREATE_INDEX` OFF, so automatic index creation never starts.

### Why option c is wrong

The compatibility-level change is right and the Azure statement is valid (explicitly setting all three options is a legitimate, if verbose, way to get the same state as option a). The fault is requirement 2: `ALTER DATABASE SCOPED CONFIGURATION SET TSQL_SCALAR_UDF_INLINING = OFF` is **database-wide** — verified: after it, the unhinted levy query's plan shows the `UserDefinedFunction` node again, i.e. no caller is inlined any more, while `sys.sql_modules.is_inlineable` stays 1 (the column reports whether the function *can* be inlined, not whether the database allows it). A database-scoped switch cannot express "this one statement".

### Why option d is wrong

This is the subtle distractor. `DISABLE_SCALAR_UDF_INLINING` is not a hint name — the engine rejects the whole statement:

```text
Msg 10715, Level 15, State 1
'DISABLE_SCALAR_UDF_INLINING' is not a valid hint.
```

The real name carries the `TSQL_` prefix (`DISABLE_TSQL_SCALAR_UDF_INLINING`), which is why `sys.dm_exec_valid_use_hints` exists. And `DROP_INDEX = ON` violates requirement 3: the option "drops unused (over the last 90 days) and duplicate indexes" — it is only *unique* indexes and those backing primary-key/unique constraints that "are never dropped"; ordinary nonclustered indexes are fair game. Note also that the option "can be automatically disabled when queries with index hints are present in the workload".

### Other facts verified on this run

- `USE HINT ('QUERY_OPTIMIZER_COMPATIBILITY_LEVEL_140')` on the 170 database also produced a plan with the `UserDefinedFunction` node — the hint makes the optimizer behave "as if the query was compiled with database compatibility level *n*" — but it is the wrong tool for requirement 2 because it also switches the cardinality-estimation model, batch mode on rowstore and every other optimizer behaviour of that level; and it "doesn't affect other features of SQL Server that might depend on the database compatibility level, such as the availability of certain database features".
- `sys.database_automatic_tuning_options` in `QuarryOps` on this SQL Server returns one row: `FORCE_LAST_GOOD_PLAN | desired_state_desc DEFAULT | actual_state_desc OFF | reason_desc AUTO_CONFIGURED`. SQL Server has no `CREATE_INDEX`/`DROP_INDEX`: `ALTER DATABASE QuarryOps SET AUTOMATIC_TUNING (CREATE_INDEX = ON)` fails with `Msg 102 ... Incorrect syntax near 'CREATE_INDEX'`, and `SET AUTOMATIC_TUNING = INHERIT` with `Msg 102 ... Incorrect syntax near 'INHERIT'` (there is no server to inherit from). Even `FORCE_LAST_GOOD_PLAN = ON` is refused on this edition — `Msg 15707 Automatic Tuning is available only in the Enterprise and Developer editions of SQL Server.` On Azure SQL Managed Instance only `FORCE_LAST_GOOD_PLAN` is supported; SQL database in Fabric enables `CREATE_INDEX` automatically. `reason_desc` values are `DISABLED`, `QUERY_STORE_OFF`, `QUERY_STORE_READ_ONLY`, `NOT_SUPPORTED`; recommendations surface in `sys.dm_db_tuning_recommendations`.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output. The Azure SQL Database automatic-tuning defaults are taken from the official documentation (conceptual part).

## DP-800 Exam Rule to Remember

```text
IQP is unlocked by COMPATIBILITY_LEVEL, then switched OFF (never ON) per scope:
   140: adaptive joins, interleaved execution, batch-mode memory grant feedback
   150: scalar UDF inlining, batch mode on rowstore, table variable deferred compilation, row-mode MGF
   160: parameter sensitive plan optimization, CE feedback, DOP feedback     (feedback needs Query Store RW)
   170: optional parameter plan optimization (OPPO)
   database: ALTER DATABASE SCOPED CONFIGURATION SET <FEATURE> = OFF      (sys.database_scoped_configurations)
   query:    OPTION (USE HINT ('DISABLE_<FEATURE>'))  -> names in sys.dm_exec_valid_use_hints; hint wins over DSC
             Msg 10715 '<name>' is not a valid hint   -> look the name up, don't guess it

Automatic tuning (Query Store based):
   SQL Server / MI:   FORCE_LAST_GOOD_PLAN only (Enterprise/Developer)     sys.database_automatic_tuning_options
   Azure SQL DB:      + CREATE_INDEX, DROP_INDEX;  defaults FLGP ON, CREATE_INDEX OFF, DROP_INDEX OFF, INHERIT from server
                      DROP_INDEX drops unused/duplicate NONUNIQUE indexes only; validation 30 min - 72 h, auto-revert
```

A scoped configuration showing `value = 1` is not proof the feature is running — check the compatibility level first; and "one statement only" always means a query hint (or a Query Store hint), never a database-wide switch or an `ALTER FUNCTION`.
