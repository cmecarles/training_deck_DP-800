# Instructor-Examiner guide — Execution Plans and DMVs 1

Companion to [execution_plans_dmvs_1.md](execution_plans_dmvs_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. The DMV result table is the key evidence; read it row by row and offer to repeat it. Read all four options before taking an answer. When the learner picks a letter, ask them to name the root cause for Q-A and for Q-C separately.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Monitor, troubleshoot and optimize database solutions (20–25%).
- Skill: Troubleshoot and resolve performance issues.
- Task bullet: Analyse execution plans and dynamic management views to identify and fix non-SARGable predicates.
- What is tested: reading access operators and warnings from cached plans, recognising an implicit conversion and a function on a column as the cause of a scan, and knowing that hints and statistics cannot make a non-seekable predicate seekable.

## 2. Scenario to read aloud

**Piece 1, the story.** "A courier company stores consignments in a SQL Server 2025 database called CourierLane, at compatibility level 170. The instance uses a Windows collation, but one column, TrackingCode, was created with the legacy SQL collation the old system used."

**Piece 2, the table.** "There is a schema called Ship and one table, Ship dot Consignments, with five columns. ConsignmentId, an integer, the primary key. TrackingCode, a VARCHAR of twenty, with COLLATE SQL underscore Latin1 underscore General underscore CP1 underscore CI underscore AS, not null. ShipDate, a date. WeightKg, a decimal seven comma two. And DestZip, a char of five."

**Piece 3, the data and indexes.** "Two hundred thousand rows are loaded. Tracking codes run from TRK dash 00000001 to TRK dash 00200000. Ship dates are spread over 2025 and 2026. Two nonclustered indexes are then created. IX underscore Consignments underscore TrackingCode, on TrackingCode. And IX underscore Consignments underscore ShipDate, on ShipDate, INCLUDE WeightKg."

**Piece 4, the four queries.** "Four queries run. Q-A is what the tracking web page sends; it is a dot NET app that passes strings as SqlDbType NVarChar. So Q-A declares at code as NVARCHAR of twenty, value TRK dash 00042500, and selects ConsignmentId, ShipDate and DestZip WHERE TrackingCode equals at code. Q-B is a support script, identical except at code is declared as VARCHAR of twenty. Q-C is the monthly tonnage report: it declares at y equals 2026 and at m equals 3, and selects SUM of WeightKg WHERE YEAR of ShipDate equals at y AND MONTH of ShipDate equals at m. Q-D is a developer's experiment: it declares at from as the first of March 2026 and at to as the first of April 2026, and selects SUM of WeightKg WHERE ShipDate is greater than or equal to at from AND ShipDate is less than at to."

**Piece 5, the results.** "All four return the expected rows. Q-A and Q-B both return consignment 42500. Q-C and Q-D both return 215796 point 60."

**Piece 6, the DMV query.** "The DBA then inspects the cached plans. The query joins sys dot dm underscore exec underscore query underscore stats with dm underscore exec underscore sql underscore text and dm underscore exec underscore query underscore plan, and pulls from the plan XML: the first access operator, Index Seek, Index Scan or Clustered Index Scan; whether there is a Key Lookup; whether there is a PlanAffectingConvert warning; and the ConvertIssue attribute of that warning."

**Piece 7, the DMV result.** "Four rows. Q-A: one execution, 652 logical reads, access operator Index Scan, has key lookup 1, convert warning 1, convert issue Seek Plan. Q-B: one execution, 6 logical reads, Index Seek, has key lookup 1, no convert warning. Q-C: one execution, 448 logical reads, Index Scan, no key lookup, no warning. Q-D: one execution, 22 logical reads, Index Seek, no key lookup, no warning."

**Piece 8, the task.** "The DBA must make Q-A and Q-C perform like Q-B and Q-D. No index may be created, dropped or altered: the table is replicated and the index set is frozen. Four changes are proposed."

**Piece 9, option a.** "Option a. Add WITH, open paren, FORCESEEK, close paren, to the FROM clause of Q-A and Q-C so the optimizer is forced to seek the existing indexes."

**Piece 10, option b.** "Option b. Change the column to Unicode so no conversion is needed: ALTER TABLE Ship dot Consignments ALTER COLUMN TrackingCode NVARCHAR of twenty NOT NULL. And rewrite Q-C as WHERE FORMAT of ShipDate with the format string yyyyMM equals the string 202603, so that only one function call is evaluated per row."

**Piece 11, option c.** "Option c. Send the tracking code as SqlDbType VarChar, so the parameter is VARCHAR of twenty as in Q-B. And rewrite Q-C as a half-open range on the column itself, as in Q-D: declare at from as DATEFROMPARTS of at y, at m, 1, and at to as DATEADD of one month to that date; then WHERE ShipDate is greater than or equal to at from AND ShipDate is less than at to."

**Piece 12, option d.** "Option d. Add OPTION, open paren, RECOMPILE, close paren, to Q-A and Q-C, and run UPDATE STATISTICS Ship dot Consignments WITH FULLSCAN, so that each execution is optimized with the actual parameter values and accurate row estimates."

## 3. Setup script (reference only; do not read verbatim unless asked)

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

Option c's rewrite of Q-C:

```sql
DECLARE @from DATE = DATEFROMPARTS(@y, @m, 1), @to DATE = DATEADD(MONTH, 1, DATEFROMPARTS(@y, @m, 1));
SELECT SUM(WeightKg) AS TotalKg FROM Ship.Consignments WHERE ShipDate >= @from AND ShipDate < @to;
```

## 4. The question (ask exactly this)

"The DBA must make Q-A and Q-C perform like Q-B and Q-D, and no index may be created, dropped or altered. Which change achieves that? Option a, FORCESEEK on both queries. Option b, alter the column to NVARCHAR and rewrite Q-C with FORMAT. Option c, send the parameter as VarChar and rewrite Q-C as a half-open date range. Option d, OPTION RECOMPILE plus UPDATE STATISTICS WITH FULLSCAN. Give me one letter, and name the root cause for Q-A and for Q-C."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct: c.** Both slow queries have a non-SARGable predicate. Q-A compares a VARCHAR column under a SQL collation with an NVARCHAR parameter; data type precedence converts the column on every row, the plan carries PlanAffectingConvert with ConvertIssue Seek Plan, and the index is scanned, 652 reads, plus a key lookup. Passing VARCHAR removes the conversion: Index Seek, 6 reads. Q-C hides the column inside YEAR and MONTH, so the ShipDate index can only be scanned, 448 reads. The half-open range on the bare column is the SARGable rewrite: Index Seek, 22 reads, same result 215796.60, no key lookup because WeightKg is included. No index is touched.

- **a is wrong.** FORCESEEK cannot manufacture a seek where no seekable predicate exists. Both queries fail with error 8622: the query processor could not produce a plan because of the hints. Hints constrain the optimizer; they do not repair a predicate.
- **b is wrong.** The ALTER COLUMN violates the constraint and fails anyway, errors 5074 and 4922, because the column is the key of an existing index. FORMAT on the column is still a function on the column, still a scan, and FORMAT is one of the slowest scalar functions.
- **d is wrong.** RECOMPILE and fresh statistics improve estimates; they do not change what is possible. Cardinality is not the problem. After recompiling, Q-A still has the convert warning and the scan, and Q-C still evaluates YEAR on 200,000 rows. Recompile per execution also adds CPU to a query run thousands of times a day.

## 6. Hint ladder (one hint per attempt, in order)

1. "Compare Q-A with Q-B. The SQL text is the same. Only one thing differs. What is it, and what does the DMV row for Q-A say about a conversion?"
2. "Compare Q-C with Q-D. Same result. In which of them is the column used bare, and in which is it wrapped in a function? Which one seeks?"
3. "So both problems are about the shape of the predicate, not about the optimizer's estimates. Which options change the predicate, and which ones try to push the optimizer?"
4. "Option a forces a seek. Can the optimizer seek a range on a column it has to convert on every row? What does the engine say when a hint makes a plan impossible?"
5. "Option b starts with ALTER COLUMN on a column that is the key of an index. What does the constraint say, and what does the engine say? And is FORMAT still a function wrapped around the column?"
6. "Two options left. Does OPTION RECOMPILE change what the predicate looks like? Does perfect statistics let the engine seek on YEAR of ShipDate?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, FORCESEEK makes it use the index" | Thinks a hint can create a seek | "A seek needs a key range. Can the optimizer compute one for CONVERT of the column, or for YEAR of the column? What error do you get when a hint cannot be honoured?" |
| "b, make the column NVARCHAR so the types match" | Ignores the frozen index set | "The column is the key of IX underscore Consignments underscore TrackingCode. What happens to ALTER COLUMN on an indexed column? And what does the constraint say?" |
| "b, FORMAT is one call instead of two" | Counts function calls instead of asking whether the column is bare | "Is FORMAT still wrapped around ShipDate? Can the index seek through it?" |
| "d, the estimates are bad, recompile fixes it" | Confuses a cardinality problem with a SARGability problem | "Q-A already estimates one row. Is the problem how many rows, or whether a seek is possible at all?" |
| "Q-A is slow because of the key lookup" | Misreads the DMV row | "Q-B also has a key lookup and reads six pages. What is the real difference between the Q-A and Q-B rows?" |
| "The collation does not matter" | Misses why the scenario mentions it | "With a Windows collation the optimizer can often still seek through a computed range for this mismatch. Which collation does TrackingCode have?" |

## 8. Teaching notes (after the answer is complete or revealed)

The rule: an index seek needs a SARGable predicate, column, operator, value, with the column used bare and the value of the column's own type. Q-A wraps the column in an implicit conversion; Q-C wraps it in YEAR and MONTH. In both cases the optimizer cannot compute a key range, so it reads the whole index and evaluates the expression on every row.

- **Q-A.** TrackingCode is VARCHAR with a SQL collation; the parameter is NVARCHAR. Data type precedence says NVARCHAR wins, so the column is converted on every row. The plan XML carries PlanAffectingConvert with ConvertIssue Seek Plan and an expression showing CONVERT_IMPLICIT on the column: literally "this conversion affected the seek plan". Result: Index Scan of the TrackingCode index, 652 reads, then a Key Lookup because the index does not cover ShipDate and DestZip. Passing VARCHAR removes the conversion: Index Seek, 6 reads, no warning. With a Windows collation the optimizer can often still seek through a computed range for this mismatch; with a SQL collation it cannot, which is why the collation is part of the scenario.
- **Q-C.** YEAR and MONTH hide the column, so the ShipDate index is scanned, 448 reads, every row evaluated. The half-open range is the SARGable rewrite: Index Seek, 22 reads, identical result, no key lookup because WeightKg is an included column. sys.dm_db_index_usage_stats confirms it: each nonclustered index shows one seek and one scan, and the clustered index shows two lookups, one per tracking-code query.

Why the others fail: FORCESEEK gives error 8622 on both queries, because hints constrain the optimizer but do not repair a predicate; it only works when a seek is possible but not chosen. ALTER COLUMN fails with 5074 and 4922 because the index depends on the column, and FORMAT is still non-SARGable. RECOMPILE and FULLSCAN statistics improve estimates, not possibilities; the warning and the scan remain.

Estimated versus actual plans, and what the DMVs record: SET SHOWPLAN_XML ON, alone in its batch, returns the estimated plan without executing; a query run under it leaves no row in sys.dm_exec_query_stats, but its compilation does register a missing-index suggestion, because that information is collected at optimization time. SET STATISTICS XML ON executes and returns the actual plan with RunTimeInformation. And Q-A and Q-C generated no missing-index suggestion: the optimizer does not recommend indexes for predicates it cannot seek. Absence of a suggestion is not evidence that a query is well indexed.

Memory hook: "Bare column, matching type, half-open range. Fix the query, not the optimizer. Hints and statistics cannot seek what is not seekable."

## 9. Follow-up oral questions (optional)

1. "Why does Q-B still show a key lookup even though it seeks?" (The TrackingCode index does not cover ShipDate and DestZip, so each matched row is fetched from the clustered index.)
2. "What is the difference between SET SHOWPLAN_XML ON and SET STATISTICS XML ON?" (SHOWPLAN_XML returns the estimated plan without executing; STATISTICS XML executes and returns the actual plan with runtime row counts.)
3. "Does the missing-index DMV tell you that Q-C needs an index?" (No. The optimizer never suggests indexes for non-SARGable predicates; Q-C produced no suggestion.)

## 10. References

- Execution plan overview: https://learn.microsoft.com/en-us/sql/relational-databases/performance/execution-plans
- Showplan logical and physical operators reference: https://learn.microsoft.com/en-us/sql/relational-databases/showplan-logical-and-physical-operators-reference
- Data type precedence: https://learn.microsoft.com/en-us/sql/t-sql/data-types/data-type-precedence-transact-sql
- Table hints, FORCESEEK: https://learn.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-table
- sys.dm_exec_query_stats: https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-query-stats-transact-sql
- sys.dm_exec_query_plan: https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-query-plan-transact-sql
- sys.dm_db_index_usage_stats: https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-index-usage-stats-transact-sql
- SET SHOWPLAN_XML: https://learn.microsoft.com/en-us/sql/t-sql/statements/set-showplan-xml-transact-sql
- SET STATISTICS XML: https://learn.microsoft.com/en-us/sql/t-sql/statements/set-statistics-xml-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
