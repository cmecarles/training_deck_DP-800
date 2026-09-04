# SQL Server question — Query Store Hints 1

## Statement

A harbour authority sells ferry passes and records every crossing in a SQL Server 2025 database named `PierPass`. Query Store is enabled with `QUERY_CAPTURE_MODE = ALL` so that every statement is captured:

```sql
CREATE DATABASE PierPass;
GO
ALTER DATABASE PierPass SET COMPATIBILITY_LEVEL = 170;
ALTER DATABASE PierPass SET QUERY_STORE = ON
(
    OPERATION_MODE = READ_WRITE,
    QUERY_CAPTURE_MODE = ALL,
    DATA_FLUSH_INTERVAL_SECONDS = 60,
    INTERVAL_LENGTH_MINUTES = 1
);
GO
USE PierPass;
GO
SELECT actual_state_desc, query_capture_mode_desc FROM sys.database_query_store_options;
-- READ_WRITE | ALL
GO
CREATE SCHEMA Pier;
GO
CREATE TABLE Pier.Passes
(
    PassId   INT          NOT NULL PRIMARY KEY,
    Holder   NVARCHAR(40) NOT NULL,
    PassType CHAR(1)      NOT NULL   -- D = day, M = month, A = annual
);
CREATE TABLE Pier.Crossings
(
    CrossingId INT          NOT NULL PRIMARY KEY,
    PassId     INT          NOT NULL REFERENCES Pier.Passes (PassId),
    Route      CHAR(3)      NOT NULL,   -- NTH, STH, EST
    CrossedAt  DATETIME2(0) NOT NULL,
    Fare       DECIMAL(6,2) NOT NULL
);
GO
-- 2,000 passes; 200,000 crossings
INSERT INTO Pier.Passes (PassId, Holder, PassType)
SELECT n, CONCAT(N'Holder ', n), CASE n % 3 WHEN 0 THEN 'D' WHEN 1 THEN 'M' ELSE 'A' END
FROM (SELECT TOP (2000) ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n FROM sys.all_columns) AS x;
INSERT INTO Pier.Crossings (CrossingId, PassId, Route, CrossedAt, Fare)
SELECT n, CASE WHEN n % 100 = 0 THEN 7 ELSE n % 2000 + 1 END,
       CASE n % 3 WHEN 0 THEN 'NTH' WHEN 1 THEN 'STH' ELSE 'EST' END,
       DATEADD(MINUTE, n, '20260101'), (n % 4 + 1) * 3.25
FROM (SELECT TOP (200000) ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n
      FROM sys.all_columns a CROSS JOIN sys.all_columns b) AS x;
CREATE NONCLUSTERED INDEX IX_Crossings_PassId ON Pier.Crossings (PassId);
GO
-- vendor procedure: the LOOP JOIN hint is hard-coded and the signed package cannot be edited
CREATE PROCEDURE Pier.usp_RouteRevenue @From DATETIME2(0)
AS
SELECT c.Route, COUNT(*) AS Crossings, SUM(c.Fare) AS Revenue
FROM Pier.Crossings AS c
JOIN Pier.Passes AS p ON p.PassId = c.PassId
WHERE c.CrossedAt >= @From AND p.PassType <> 'X'
GROUP BY c.Route
ORDER BY c.Route
OPTION (LOOP JOIN);
GO
-- audit procedure: crossings whose fare exceeds the pass number (a non-equality join)
CREATE PROCEDURE Pier.usp_FaresAbovePass @MaxPass INT
AS
SELECT p.PassId, COUNT(*) AS Crossings
FROM Pier.Crossings AS c
JOIN Pier.Passes AS p ON c.Fare > p.PassId
WHERE p.PassId <= @MaxPass
GROUP BY p.PassId;
GO
```

`usp_RouteRevenue` ran twice and `usp_FaresAbovePass` once; the DBA then flushed Query Store and looked up the two statements and their plans (the `query_id` values below are the ones this run produced; with capture mode `ALL` the setup statements were captured too, so your numbers may differ):

```sql
EXEC sp_query_store_flush_db;
SELECT q.query_id, OBJECT_NAME(q.object_id) AS proc_name, p.plan_id, p.is_forced_plan,
       CAST(p.query_plan AS XML).value('declare default element namespace
            "http://schemas.microsoft.com/sqlserver/2004/07/showplan"; (//RelOp[@LogicalOp="Inner Join"]/@PhysicalOp)[1]', 'nvarchar(40)') AS join_operator,
       CAST(p.query_plan AS XML).value('declare default element namespace
            "http://schemas.microsoft.com/sqlserver/2004/07/showplan"; (//StmtSimple/@QueryStoreStatementHintText)[1]', 'nvarchar(200)') AS qs_hint_in_plan,
       (SELECT SUM(rs.count_executions) FROM sys.query_store_runtime_stats AS rs WHERE rs.plan_id = p.plan_id) AS executions
FROM sys.query_store_query AS q
JOIN sys.query_store_query_text AS qt ON qt.query_text_id = q.query_text_id
JOIN sys.query_store_plan AS p ON p.query_id = q.query_id
WHERE q.object_id IN (OBJECT_ID('Pier.usp_RouteRevenue'), OBJECT_ID('Pier.usp_FaresAbovePass'))
ORDER BY q.query_id, p.plan_id;
```

| query_id | proc_name | plan_id | is_forced_plan | join_operator | qs_hint_in_plan | executions |
|---|---|---|---|---|---|---|
| 11 | usp_RouteRevenue | 11 | 0 | Nested Loops | NULL | 2 |
| 14 | usp_FaresAbovePass | 14 | 0 | Nested Loops | NULL | 1 |

`sys.query_store_query_hints` is empty. The revenue report is slow because of the hard-coded loop join, so the DBA decides to use **Query Store hints**. The following statements run in order, each in its own batch, in one session:

```sql
-- S1
EXEC sys.sp_query_store_set_hints @query_id = 11, @query_hints = N'OPTION(OPTIMIZE FOR (@From = ''20260101''))';
-- S2  (followed by:  EXEC Pier.usp_RouteRevenue @From = '20260101';  EXEC sp_query_store_flush_db;)
EXEC sys.sp_query_store_set_hints @query_id = 11, @query_hints = N'OPTION(RECOMPILE)';
-- S3  (followed by:  EXEC Pier.usp_FaresAbovePass @MaxPass = 5;  EXEC sp_query_store_flush_db;)
EXEC sys.sp_query_store_set_hints @query_id = 14, @query_hints = N'OPTION(HASH JOIN)';
-- S4  (19 is the plan_id that the execution after S2 added to sys.query_store_plan)
EXEC sp_query_store_force_plan @query_id = 11, @plan_id = 19;
-- S5
EXEC sys.sp_query_store_clear_hints @query_id = 11;
EXEC sp_query_store_force_plan @query_id = 11, @plan_id = 19;
-- S6
EXEC sys.sp_query_store_set_hints @query_id = 11, @query_hints = N'OPTION(MAXDOP 1)';
```

**Part 1.** For each of S1–S6, does it succeed or raise an error? Give the error number.

**Part 2.** The execution of `usp_RouteRevenue` right after S2: which physical join operator does its plan use, and what does the plan XML carry (in the `StmtSimple` element) that proves where the hint came from?

**Part 3.** The execution of `usp_FaresAbovePass` right after S3: does it fail? What does `sys.query_store_query_hints` show for query 14 in `query_hint_failure_count`, `last_query_hint_failure_reason` and `last_query_hint_failure_reason_desc`?

## Correct Answer

**Part 1**

| Stmt | Outcome | Detail |
|---|---|---|
| S1 | Fails, error 12455 | `OPTIMIZE FOR (@var = value)` is not a supported Query Store hint (only `OPTIMIZE FOR UNKNOWN` is) |
| S2 | Succeeds | hint 1 created for query 11: `OPTION(RECOMPILE)` |
| S3 | Succeeds | hint 2 created for query 14: `OPTION(HASH JOIN)` (validity is checked at compile time, not now) |
| S4 | Fails, error 12458 | a query that has Query Store hints cannot have a plan forced |
| S5 | Succeeds | hints cleared, then plan 19 forced (`is_forced_plan = 1`) |
| S6 | Fails, error 12457 | a query with a forced plan cannot receive Query Store hints |

**Part 2.** A **Hash Match** join (new `plan_id 19`). The `StmtSimple` element carries `QueryStoreStatementHintText="OPTION(RECOMPILE)"`, `QueryStoreStatementHintId="1"` and `QueryStoreStatementHintSource="User"`. The hard-coded `OPTION (LOOP JOIN)` was **replaced**, not merged: the optimizer was free to choose the join type again.

**Part 3.** The execution **succeeds** (Nested Loops, plan 14 now at 2 executions). The catalog shows `query_hint_failure_count = 1`, `last_query_hint_failure_reason = 8622`, `last_query_hint_failure_reason_desc = NO_PLAN`: the hint was ignored, the query was not blocked.

## Explanation

Query Store hints (SQL Server 2022+, Azure SQL Database, Azure SQL Managed Instance) inject an `OPTION (...)` clause into a statement identified by its Query Store `query_id`, without touching the application text — the modern replacement for plan guides. The workflow is: run the query, find its `query_id` in `sys.query_store_query` joined to `sys.query_store_query_text`, call `sys.sp_query_store_set_hints @query_id, @query_hints`, inspect `sys.query_store_query_hints`, and remove with `sys.sp_query_store_clear_hints`. Hints persist in the database (they survive restarts and failovers) and are exempt from Query Store's automatic cleanup.

### S1 — unsupported hint (12455)

```text
Msg 12455, Level 16, State 2, Procedure sys.sp_query_store_set_hints, Line 1
Setting query hint(s) 'OPTIMIZE FOR' in Query Store is not supported.
```

The supported list is the statement-level `OPTION` hints: `{HASH|ORDER} GROUP`, `{CONCAT|HASH|MERGE} UNION`, `{LOOP|MERGE|HASH} JOIN`, `EXPAND VIEWS`, `FAST n`, `FORCE ORDER`, `IGNORE_NONCLUSTERED_COLUMNSTORE_INDEX`, `KEEP PLAN`, `KEEPFIXED PLAN`, `MAX_GRANT_PERCENT`, `MIN_GRANT_PERCENT`, `MAXDOP`, `NO_PERFORMANCE_SPOOL`, `OPTIMIZE FOR UNKNOWN`, `PARAMETERIZATION {SIMPLE|FORCED}`, `RECOMPILE`, `ROBUST PLAN`, `USE HINT('...')`. Documented as unsupported: `OPTIMIZE FOR (@var = value)`, `USE PLAN` (use plan forcing instead), `MAXRECURSION`, `DISABLE_DEFERRED_COMPILATION_TV`, `DISABLE_TSQL_SCALAR_UDF_INLINING`, and every **table** hint (`FORCESEEK`, `INDEX`, `NOLOCK`, ...). Verified on the same database: `N'OPTION(TABLE HINT(c, INDEX(0)))'` fails with `Msg 12455 ... Setting query hint(s) 'TABLE HINT' in Query Store is not supported.`, `N'OPTION(USE PLAN N''<x/>'')'` with `... 'USE PLAN' in Query Store is not supported.`; a string that does not start with `OPTION`, `N'MAXDOP 1'`, fails at parse time with `Msg 102, Level 15, State 1, Procedure sys.sp_query_store_set_hints, Line 1 — Incorrect syntax near 'MAXDOP'.`; and a `query_id` that does not exist fails with `Msg 12402, Level 11, State 5, Procedure sys.sp_query_store_set_hints — Query with provided query_id (999) is not found in the Query Store for database (20). Check the query_id value and rerun the command.` (State 6 from `sp_query_store_clear_hints`). Nothing is stored when the procedure fails: `sys.query_store_query_hints` stayed empty after S1.

### S2 — the hint replaces the statement's own OPTION clause

The Query Store hint string is the **whole** `OPTION` clause the optimizer will use; the documentation says Query Store hints "override hard-coded statement-level hints and plan guides". Verified after S2 and one execution:

| query_id | plan_id | join_operator | qs_hint_in_plan | qs_hint_source | qs_hint_id | executions |
|---|---|---|---|---|---|---|
| 11 | 11 | Nested Loops | NULL | NULL | NULL | 2 |
| 11 | 19 | Hash Match | OPTION(RECOMPILE) | User | 1 | 1 |
| 14 | 14 | Nested Loops | NULL | NULL | NULL | 1 |

and `sys.query_store_query_hints`: `query_hint_id 1, query_id 11, OPTION(RECOMPILE), query_hint_failure_count 0, NONE, source 0 / User`. The plan compiled under the hint is a Hash Match even though nothing in `OPTION(RECOMPILE)` says "hash": the vendor's `LOOP JOIN` simply no longer reaches the optimizer. This is the subtle distractor — "adding `RECOMPILE` cannot change the join type" is wrong because the hint set is replaced, not appended. The three attributes `QueryStoreStatementHintText`, `QueryStoreStatementHintId` and `QueryStoreStatementHintSource` on `StmtSimple` (visible in `SET STATISTICS XML`, `SET SHOWPLAN_XML` and the plan stored in `sys.query_store_plan`) are the proof that a plan was shaped by a Query Store hint; `source_desc` is `User` for hints you create and `CE feedback` for hints that the cardinality-estimation feedback feature creates on its own.

### S3 — a hint that cannot produce a plan is ignored, counted, and the query still runs

`HASH JOIN` needs an equality predicate; the audit procedure joins on `c.Fare > p.PassId`. Put in the text, `OPTION (HASH JOIN)` kills the query:

```text
Msg 8622, Level 16, State 1, Line 1
Query processor could not produce a query plan because of the hints defined in this query.
Resubmit the query without specifying any hints and without using SET FORCEPLAN.
```

As a Query Store hint the same conflict is **not** an error for the user: `sp_query_store_set_hints` accepts the string (S3 succeeds; validity is only known at compile time), the next execution compiles without the hint (Nested Loops, plan 14, executions 1 → 2), and the failure is recorded:

| query_hint_id | query_id | query_hint_text | query_hint_failure_count | last_query_hint_failure_reason | last_query_hint_failure_reason_desc | source_desc |
|---|---|---|---|---|---|---|
| 1 | 11 | OPTION(RECOMPILE) | 0 | 0 | NONE | User |
| 2 | 14 | OPTION(HASH JOIN) | 1 | 8622 | NO_PLAN | User |

"Except for the `ABORT_QUERY_EXECUTION` hint, queries with Query Store hints always execute"; when one hint in the string would prevent a plan, **all** hints in that string are ignored for that compilation.

### S4, S5, S6 — hints and forced plans are mutually exclusive

```text
Msg 12458, Level 16, State 1, Procedure sp_query_store_force_plan, Line 1
Query with query_id 11 has query store hints. Query Store can't force a plan for it while it has hints.
```

S5 clears the hint first and then forcing plan 19 succeeds (`is_forced_plan = 1`; a later execution ran plan 19 with `force_failure_count = 0` — a plan produced under a hint can be forced afterwards without the hint). S6 then hits the mirror image:

```text
Msg 12457, Level 16, State 1, Procedure sys.sp_query_store_set_hints, Line 1
Query with query_id 11 has forced plan. No hints can be applied to it while it has forced plan.
```

Choose one mechanism per query: **plan forcing** pins one exact plan shape (`sp_query_store_force_plan`, undone by `sp_query_store_unforce_plan`), **hints** constrain the optimizer but let it produce new plans as data or indexes change. Hints are also skipped for statements that qualify for simple parameterization, and `RECOMPILE` inside a hint is ignored with warning 12461 when the database uses forced parameterization.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

## DP-800 Exam Rule to Remember

```text
Query Store hints (SQL Server 2022+, Azure SQL DB / MI): change a plan WITHOUT changing the query text.
  1. Find query_id:  sys.query_store_query q JOIN sys.query_store_query_text qt  (needs Query Store ON;
                     QUERY_CAPTURE_MODE = ALL guarantees the statement is captured; flush with sp_query_store_flush_db)
  2. EXEC sys.sp_query_store_set_hints @query_id = n, @query_hints = N'OPTION(RECOMPILE, MAXDOP 1, USE HINT(''...''))'
       -> string must start with OPTION (else Msg 102); unsupported hint -> Msg 12455 (OPTIMIZE FOR @v=val,
          USE PLAN, MAXRECURSION, table hints); unknown query_id -> Msg 12402
       -> REPLACES the statement's own OPTION clause and any plan guide (does not merge with it)
  3. sys.query_store_query_hints: query_hint_text, query_hint_failure_count, last_query_hint_failure_reason(_desc)
       -> a hint that prevents a plan is IGNORED, the query still runs, failure_count++ (8622 = NO_PLAN)
  4. Proof in plan XML (StmtSimple): QueryStoreStatementHintText / HintId / HintSource (User | CE feedback)
  5. EXEC sys.sp_query_store_clear_hints @query_id = n
Hints and forced plans are exclusive:  hints present -> force_plan fails 12458;  forced plan -> set_hints fails 12457.
```
