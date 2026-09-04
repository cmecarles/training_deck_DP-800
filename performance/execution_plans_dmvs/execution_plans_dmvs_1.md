# SQL Server question — Execution Plans and DMVs 1

## Statement

A courier company stores consignments in a SQL Server 2025 database named `CourierLane`. The instance uses a Windows collation, but `TrackingCode` was created with the legacy SQL collation the old system used:

```sql
CREATE DATABASE CourierLane;
GO
ALTER DATABASE CourierLane SET COMPATIBILITY_LEVEL = 170;
GO
USE CourierLane;
GO
CREATE SCHEMA Ship;
GO
CREATE TABLE Ship.Consignments
(
    ConsignmentId INT          NOT NULL PRIMARY KEY,
    TrackingCode  VARCHAR(20)  COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
    ShipDate      DATE         NOT NULL,
    WeightKg      DECIMAL(7,2) NOT NULL,
    DestZip       CHAR(5)      NOT NULL
);
GO
-- 200,000 rows: TRK-00000001 .. TRK-00200000, ship dates spread over 2025-2026
INSERT INTO Ship.Consignments (ConsignmentId, TrackingCode, ShipDate, WeightKg, DestZip)
SELECT n, 'TRK-' + RIGHT('00000000' + CAST(n AS VARCHAR(8)), 8),
       DATEADD(DAY, n % 730, '20250101'), (n % 500) / 10.0 + 0.5,
       RIGHT('00000' + CAST(n % 99999 AS VARCHAR(5)), 5)
FROM (SELECT TOP (200000) ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n
      FROM sys.all_columns a CROSS JOIN sys.all_columns b) AS x;
GO
CREATE NONCLUSTERED INDEX IX_Consignments_TrackingCode ON Ship.Consignments (TrackingCode);
CREATE NONCLUSTERED INDEX IX_Consignments_ShipDate    ON Ship.Consignments (ShipDate) INCLUDE (WeightKg);
GO
```

The tracking web page (a .NET app that passes strings as `SqlDbType.NVarChar`) runs **Q-A**; a support script runs **Q-B**; the monthly tonnage report runs **Q-C**; and a developer wrote **Q-D** as an experiment:

```sql
-- Q-A  (parameter as sent by the web app)
DECLARE @code NVARCHAR(20) = N'TRK-00042500';
SELECT /* Q-A */ ConsignmentId, ShipDate, DestZip FROM Ship.Consignments WHERE TrackingCode = @code;
GO
-- Q-B
DECLARE @code VARCHAR(20) = 'TRK-00042500';
SELECT /* Q-B */ ConsignmentId, ShipDate, DestZip FROM Ship.Consignments WHERE TrackingCode = @code;
GO
-- Q-C
DECLARE @y INT = 2026, @m INT = 3;
SELECT /* Q-C */ SUM(WeightKg) AS TotalKg FROM Ship.Consignments WHERE YEAR(ShipDate) = @y AND MONTH(ShipDate) = @m;
GO
-- Q-D
DECLARE @from DATE = '20260301', @to DATE = '20260401';
SELECT /* Q-D */ SUM(WeightKg) AS TotalKg FROM Ship.Consignments WHERE ShipDate >= @from AND ShipDate < @to;
GO
```

All four return the expected rows (Q-A and Q-B both return consignment 42500; Q-C and Q-D both return `215796.60`). The DBA then inspects the cached plans through the DMVs, searching the plan XML for the access operator and for optimizer warnings:

```sql
WITH XMLNAMESPACES (DEFAULT 'http://schemas.microsoft.com/sqlserver/2004/07/showplan')
SELECT SUBSTRING(st.text, CHARINDEX('/* Q-', st.text) + 3, 3) AS query_tag,
       qs.execution_count, qs.total_logical_reads,
       qp.query_plan.value('(//RelOp[@PhysicalOp="Index Seek" or @PhysicalOp="Index Scan"
                                    or @PhysicalOp="Clustered Index Scan"]/@PhysicalOp)[1]', 'nvarchar(40)') AS access_op,
       qp.query_plan.exist('//RelOp[@LogicalOp="Key Lookup"]')        AS has_key_lookup,
       qp.query_plan.exist('//Warnings/PlanAffectingConvert')          AS convert_warning,
       qp.query_plan.value('(//Warnings/PlanAffectingConvert/@ConvertIssue)[1]', 'nvarchar(40)') AS convert_issue
FROM sys.dm_exec_query_stats AS qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle)    AS st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) AS qp
WHERE st.text LIKE '%/* Q-_ */%' AND st.text NOT LIKE '%dm_exec_query_stats%'
ORDER BY query_tag;
```

| query_tag | execution_count | total_logical_reads | access_op | has_key_lookup | convert_warning | convert_issue |
|---|---|---|---|---|---|---|
| Q-A | 1 | 652 | Index Scan | 1 | 1 | Seek Plan |
| Q-B | 1 | 6 | Index Seek | 1 | 0 | NULL |
| Q-C | 1 | 448 | Index Scan | 0 | 0 | NULL |
| Q-D | 1 | 22 | Index Seek | 0 | 0 | NULL |

The DBA must make **Q-A and Q-C** perform like Q-B and Q-D. **No index may be created, dropped, or altered** (the table is replicated and the index set is frozen). Which change achieves that?

### a.

Add `WITH (FORCESEEK)` to the `FROM` clause of Q-A and Q-C so the optimizer is forced to seek the existing indexes.

### b.

Change the column to Unicode so that no conversion is needed — `ALTER TABLE Ship.Consignments ALTER COLUMN TrackingCode NVARCHAR(20) NOT NULL;` — and rewrite Q-C as `WHERE FORMAT(ShipDate, 'yyyyMM') = '202603'` so that only one function call is evaluated per row.

### c.

Send the tracking code as `SqlDbType.VarChar` (so the parameter is `VARCHAR(20)`, as in Q-B), and rewrite Q-C as a half-open range on the column itself, as in Q-D:

```sql
DECLARE @from DATE = DATEFROMPARTS(@y, @m, 1), @to DATE = DATEADD(MONTH, 1, DATEFROMPARTS(@y, @m, 1));
SELECT SUM(WeightKg) AS TotalKg FROM Ship.Consignments WHERE ShipDate >= @from AND ShipDate < @to;
```

### d.

Add `OPTION (RECOMPILE)` to Q-A and Q-C and run `UPDATE STATISTICS Ship.Consignments WITH FULLSCAN`, so that each execution is optimized with the actual parameter values and accurate row estimates.

## Correct Answer

**c**

## Explanation

Both slow queries have the same root cause: their predicate is **not SARGable** (search-argument-able). An index seek needs a predicate of the form `column <op> value` where the column is used bare, with a value of the column's own type. Q-A wraps the column in an implicit conversion; Q-C wraps it in `YEAR()`/`MONTH()`. In both cases the optimizer cannot compute a key range, so it reads the whole index (`Index Scan`) and evaluates the expression on every row.

### Why option c is correct

- **Q-A.** `TrackingCode` is `VARCHAR` with a **SQL collation**; the parameter is `NVARCHAR`. Data-type precedence says `NVARCHAR` wins, so SQL Server converts the *column* on every row: the plan XML carries the warning `<PlanAffectingConvert ConvertIssue="Seek Plan" Expression="CONVERT_IMPLICIT(nvarchar(20),[CourierLane].[Ship].[Consignments].[TrackingCode],0)=CONVERT_IMPLICIT(nvarchar(20),[@code],0)"/>` — literally "this conversion affected the seek plan". The result is an `Index Scan` of `IX_Consignments_TrackingCode` (652 logical reads) followed by a `Key Lookup` (the nonclustered index does not cover `ShipDate`/`DestZip`, so `Nested Loops` + a clustered-index lookup fetch the row). Passing the value as `VARCHAR(20)` (Q-B) removes the conversion: `Index Seek`, 6 logical reads, no warning. (With a *Windows* collation the optimizer can often still seek through a computed range for this mismatch; with a SQL collation, as here, it cannot — which is why the column's collation is part of the scenario.)
- **Q-C.** `YEAR(ShipDate) = @y AND MONTH(ShipDate) = @m` hides the column inside functions, so `IX_Consignments_ShipDate` can only be scanned (448 reads, every row evaluated). The half-open range `ShipDate >= @from AND ShipDate < @to` (Q-D) is the SARGable rewrite: `Index Seek`, 22 reads, identical result `215796.60`, and no `Key Lookup` because `WeightKg` is an included column. `sys.dm_db_index_usage_stats` confirms the picture: after the four queries, `IX_Consignments_TrackingCode` and `IX_Consignments_ShipDate` each show `user_seeks = 1, user_scans = 1`, and the clustered index shows `user_lookups = 2` (one per tracking-code query).

Neither change touches an index, so the constraint is respected.

### Why option a is wrong

`FORCESEEK` cannot manufacture a seek where no seekable predicate exists. Verified on both queries:

```text
Msg 8622, Level 16, State 1
Query processor could not produce a query plan because of the hints defined in this query.
Resubmit the query without specifying any hints and without using SET FORCEPLAN.
```

Hints constrain the optimizer; they do not repair a predicate. This is the subtle distractor because `FORCESEEK` *does* work when a seek is *possible* but not chosen.

### Why option b is wrong

The `ALTER COLUMN` violates the constraint and fails anyway, because the column is the key of an existing index: `Msg 5074: The index 'IX_Consignments_TrackingCode' is dependent on column 'TrackingCode'.` followed by `Msg 4922: ALTER TABLE ALTER COLUMN TrackingCode failed because one or more objects access this column.` Even if the index were dropped and re-created, doubling the column to Unicode doubles its storage for a code that is pure ASCII. And `FORMAT(ShipDate, 'yyyyMM') = '202603'` is still a function on the column — still non-SARGable, still a scan — and `FORMAT` is one of the slowest scalar functions in T-SQL.

### Why option d is wrong

`OPTION (RECOMPILE)` and fresh statistics improve **estimates**; they do not change what is **possible**. Cardinality is not the problem — Q-A's plan already knows it will return one row; it simply cannot seek for a converted column. After recompiling with perfect statistics, Q-A still has the `PlanAffectingConvert` warning and the scan, and Q-C still evaluates `YEAR()` on 200,000 rows. Recompile-per-execution also adds CPU to a query that the web page runs thousands of times a day.

### Estimated vs actual plans, and what the DMVs record

Two further facts were verified for this scenario:

- `SET SHOWPLAN_XML ON` (which must be the only statement in its batch) returns the **estimated** plan without executing the query: a fifth query `Q-E` (`WHERE DestZip = '00042'`) run under it produced no row in `sys.dm_exec_query_stats` (`execution_count` is populated only by executions) — yet its compilation **did** register a suggestion in `sys.dm_db_missing_index_details` (`[CourierLane].[Ship].[Consignments]`, `equality_columns = [DestZip]`), because missing-index information is collected at optimization time. `SET STATISTICS XML ON` executes the query and returns the **actual** plan, with `RunTimeInformation` (actual rows, actual executions) that the estimated plan lacks.
- Q-A and Q-C generated **no** missing-index suggestion: the optimizer does not recommend indexes for predicates it cannot seek. Absence of a suggestion is not evidence that a query is well indexed.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

## DP-800 Exam Rule to Remember

```text
Seek requires a SARGable predicate:  column  <op>  value-of-the-column's-type
   NOT sargable -> scan:  YEAR(col) = ..., FORMAT(col) = ..., col + 1 = ..., LIKE '%x',
                          varchar column = nvarchar value (SQL collation) -> CONVERT_IMPLICIT on the column
   Fix the query, not the optimizer: half-open ranges, matching parameter types.

Reading a plan:  Index Seek (+ Key Lookup if not covering)  vs  Index/Clustered Index Scan
                 Warnings: PlanAffectingConvert (ConvertIssue="Seek Plan"), spills, missing index
                 Nested Loops = small outer input / lookups; Hash Match = large unsorted inputs
Estimated plan:  SET SHOWPLAN_XML ON  (no execution, no RunTimeInformation, no dm_exec_query_stats row)
Actual plan:     SET STATISTICS XML ON (executes; actual rows / executions per operator)
DMVs:            dm_exec_query_stats + dm_exec_sql_text + dm_exec_query_plan  (cached plans, reads, CPU)
                 dm_db_index_usage_stats (seeks/scans/lookups)   dm_db_missing_index_details (never for non-SARGable)
Hints (FORCESEEK, RECOMPILE) cannot make a non-seekable predicate seekable: Msg 8622.
```
