# Instructor-Examiner guide — Query Store 1

Companion to [query_store_1.md](query_store_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a, b, c and d. Read all four options before taking an answer. The two catalog snapshots in pieces 5 and 7 are the evidence; repeat them whenever asked, especially the plan numbers, the read counts and the failure reason NO underscore INDEX. The question has two constraints, least disruption and keep the history; remind the learner of both if they ask. Accept the answer as a letter.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked. Say "sp underscore query underscore store underscore force underscore plan" in full the first time, then "the force plan procedure" is fine.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Optimize database performance.
- Task bullet: Monitor and troubleshoot performance using execution plans, DMVs, Query Store and Query Performance Insight.
- What is tested: the Query Store catalog views, what plan forcing does and what a forcing failure with reason NO_INDEX means, that QUERY_STORE CLEAR wipes history, and that Query Performance Insight is an Azure SQL Database portal feature that does not exist for on-premises SQL Server.

## 2. Scenario to read aloud

**Piece 1, the story.** "A toll-road operator records every plaza passage in a SQL Server 2025 database named TollGate. It runs on premises; a migration to Azure SQL Database is planned for next year. The DBA enabled Query Store when the database was created."

**Piece 2, the Query Store settings.** "The database is set to compatibility level 170. Query Store is ON with OPERATION underscore MODE READ underscore WRITE, QUERY underscore CAPTURE underscore MODE ALL, a data flush interval of 60 seconds, an interval length of 1 minute, and WAIT underscore STATS underscore CAPTURE underscore MODE ON. The DBA checked sys dot database underscore query underscore store underscore options and saw actual state READ underscore WRITE, capture mode ALL, readonly reason zero, wait stats ON."

**Piece 3, the table and procedure.** "There is a schema Toll and a table Toll dot Passages with PassageId, an integer primary key; PlazaId, an integer; PassedAt, a DATETIME2; VehicleClass, a TINYINT; and Amount, a DECIMAL six comma two. One hundred thousand passages are inserted, spread over two hundred plazas. A procedure Toll dot usp underscore PlazaRevenue takes a PlazaId and returns VehicleClass and the sum of Amount as Revenue, grouped by VehicleClass, for that plaza."

**Piece 4, what happened.** "The procedure ran three times. Then a performance engineer created a nonclustered index IX underscore Passages underscore PlazaId on Passages on PlazaId, including VehicleClass and Amount. The procedure ran twice more. The DBA ran sp underscore query underscore store underscore flush underscore db and queried the Query Store catalog views: sys dot query underscore store underscore query joined to sys dot query underscore store underscore plan joined to sys dot query underscore store underscore runtime underscore stats, filtered to the procedure."

**Piece 5, the first snapshot.** "Two rows came back, both for query id 7. Plan 7: not forced, zero failures, reason NONE, three executions, three hundred sixty-one average logical reads, and no index seek in the plan. Plan 9: not forced, zero failures, reason NONE, two executions, four average logical reads, and it has an index seek."

**Piece 6, the forcing.** "To protect the procedure against future regressions, the DBA forced plan 9 with sp underscore query underscore store underscore force underscore plan, query id 7, plan id 9. The catalog then showed plan 9 with is underscore forced underscore plan equal to one, zero failures, reason NONE."

**Piece 7, the incident and second snapshot.** "A week later, a nightly maintenance script dropped the index IX underscore Passages underscore PlazaId by mistake. The procedure keeps returning correct results, but the plaza dashboard is slow again. The catalog now reads: plan 7, not forced, zero failures, reason NONE. Plan 9, forced, force failure count one, last force failure reason 8712, description NO underscore INDEX. The DBA must get the procedure back to the four-read plan with the least disruption and without losing the Query Store history collected so far."

**Piece 8, option a.** "Option a: run sp underscore query underscore store underscore unforce underscore plan for query 7, plan 9, and then sp underscore query underscore store underscore force underscore plan for query 7, plan 7."

**Piece 9, option b.** "Option b: run ALTER DATABASE TollGate SET QUERY underscore STORE CLEAR, and then sp underscore query underscore store underscore force underscore plan for query 7, plan 9."

**Piece 10, option c.** "Option c: re-create the nonclustered index IX underscore Passages underscore PlazaId on Toll dot Passages on PlazaId, including VehicleClass and Amount, with the same definition, and leave the forced plan in place."

**Piece 11, option d.** "Option d: open Query Performance Insight for TollGate in the Azure portal, find query 7 in the top-consuming list, and apply the force plan recommendation shown for it."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE TollGate;
GO
ALTER DATABASE TollGate SET COMPATIBILITY_LEVEL = 170;
ALTER DATABASE TollGate SET QUERY_STORE = ON
(
    OPERATION_MODE = READ_WRITE,
    QUERY_CAPTURE_MODE = ALL,
    DATA_FLUSH_INTERVAL_SECONDS = 60,
    INTERVAL_LENGTH_MINUTES = 1,
    WAIT_STATS_CAPTURE_MODE = ON
);
GO
USE TollGate;
GO
SELECT actual_state_desc, query_capture_mode_desc, readonly_reason, wait_stats_capture_mode_desc
FROM sys.database_query_store_options;
-- READ_WRITE | ALL | 0 | ON
GO
CREATE SCHEMA Toll;
GO
CREATE TABLE Toll.Passages
(
    PassageId    INT          NOT NULL PRIMARY KEY,
    PlazaId      INT          NOT NULL,
    PassedAt     DATETIME2(0) NOT NULL,
    VehicleClass TINYINT      NOT NULL,
    Amount       DECIMAL(6,2) NOT NULL
);
GO
-- 100,000 passages spread over 200 plazas
INSERT INTO Toll.Passages (PassageId, PlazaId, PassedAt, VehicleClass, Amount)
SELECT n, n % 200 + 1, DATEADD(SECOND, n, '20260301'), n % 5 + 1, (n % 5 + 1) * 2.50
FROM (SELECT TOP (100000) ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n
      FROM sys.all_columns a CROSS JOIN sys.all_columns b) AS x;
GO
CREATE PROCEDURE Toll.usp_PlazaRevenue @PlazaId INT
AS
SELECT VehicleClass, SUM(Amount) AS Revenue
FROM Toll.Passages
WHERE PlazaId = @PlazaId
GROUP BY VehicleClass;
GO
```

Index added after three executions:

```sql
CREATE NONCLUSTERED INDEX IX_Passages_PlazaId ON Toll.Passages (PlazaId) INCLUDE (VehicleClass, Amount);
```

Catalog query:

```sql
EXEC sp_query_store_flush_db;
SELECT q.query_id, p.plan_id, OBJECT_NAME(q.object_id) AS object_name,
       p.is_forced_plan, p.force_failure_count, p.last_force_failure_reason_desc,
       rs.count_executions, CAST(rs.avg_logical_io_reads AS INT) AS avg_logical_reads,
       CAST(p.query_plan AS XML).exist('declare default element namespace
            "http://schemas.microsoft.com/sqlserver/2004/07/showplan"; //RelOp[@PhysicalOp="Index Seek"]') AS has_seek
FROM sys.query_store_query AS q
JOIN sys.query_store_plan AS p ON p.query_id = q.query_id
JOIN sys.query_store_runtime_stats AS rs ON rs.plan_id = p.plan_id
WHERE q.object_id = OBJECT_ID('Toll.usp_PlazaRevenue')
ORDER BY p.plan_id;
```

First snapshot:

| query_id | plan_id | is_forced_plan | force_failure_count | last_force_failure_reason_desc | count_executions | avg_logical_reads | has_seek |
|---|---|---|---|---|---|---|---|
| 7 | 7 | 0 | 0 | NONE | 3 | 361 | 0 |
| 7 | 9 | 0 | 0 | NONE | 2 | 4 | 1 |

Forcing:

```sql
EXEC sp_query_store_force_plan @query_id = 7, @plan_id = 9;
```

After the index was dropped:

| plan_id | is_forced_plan | force_failure_count | last_force_failure_reason | last_force_failure_reason_desc |
|---|---|---|---|---|
| 7 | 0 | 0 | 0 | NONE |
| 9 | 1 | 1 | 8712 | NO_INDEX |

Options:

```sql
-- a
EXEC sp_query_store_unforce_plan @query_id = 7, @plan_id = 9;
EXEC sp_query_store_force_plan   @query_id = 7, @plan_id = 7;

-- b
ALTER DATABASE TollGate SET QUERY_STORE CLEAR;
EXEC sp_query_store_force_plan @query_id = 7, @plan_id = 9;

-- c
CREATE NONCLUSTERED INDEX IX_Passages_PlazaId
    ON Toll.Passages (PlazaId) INCLUDE (VehicleClass, Amount);
-- and leave the forced plan in place

-- d: Query Performance Insight in the Azure portal, apply the "force plan" recommendation
```

## 4. The question (ask exactly this)

"The DBA must get the procedure back to the four-read plan with the least disruption and without losing the Query Store history collected so far. Which action should the DBA take?

- a. Unforce plan 9, then force plan 7.
- b. Clear Query Store with ALTER DATABASE SET QUERY underscore STORE CLEAR, then force plan 9 again.
- c. Re-create the index IX underscore Passages underscore PlazaId with the same definition, and leave the forced plan in place.
- d. Open Query Performance Insight for TollGate in the Azure portal and apply the force plan recommendation for query 7."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: c.**

A forced plan is not a copy of the index; it is a plan shape the optimizer must reproduce at compile time. When the index disappeared, forcing failed. Forcing failures are not errors for the user: the engine falls back to a normal compilation, so correct results but slow, increments force_failure_count, and records the reason, 8712 equals NO_INDEX. Re-creating the index with the same definition makes the forced plan reproducible again. Verified: after CREATE INDEX and two more executions, plan 9's execution count rose from 2 to 4 at 4 logical reads each, while force_failure_count 1 and NO_INDEX remain as history of the incident. Nothing was unforced, nothing was cleared.

Why the wrong options are wrong:

- a: forces the slow plan. Plan 7 is the clustered-index-scan plan with 361 reads and no seek. It stops the failures by pinning the dashboard at the regressed speed, and prevents a seek even after the index is restored.
- b: QUERY_STORE CLEAR deletes everything, queries, plans, runtime and wait statistics, violating the history requirement. The second statement then fails with error 12402, query 7 not found in the Query Store. Even after repopulation, with no index there is no seek plan to force.
- d: Query Performance Insight is an Azure SQL Database portal feature built on Query Store. TollGate is on-premises SQL Server; there is no portal blade for it, and QPI does not exist for Managed Instance either. Even in Azure SQL Database QPI shows queries; forcing comes from automatic tuning FORCE_LAST_GOOD_PLAN or from sp_query_store_force_plan, and a dropped index is not a plan regression that FORCE_LAST_GOOD_PLAN can undo.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start from the evidence. What does the failure reason NO underscore INDEX tell you about why plan 9 can no longer be used?"
2. "Is a forced plan a stored copy of the index, or a shape the optimizer must rebuild each time it compiles? What does that shape need in order to be rebuilt?"
3. "The question has two constraints: least disruption, and keep the history. Which option throws the history away? That one is out."
4. "Where does TollGate run today, on premises or in Azure? Which option assumes an Azure portal feature?"
5. "Two options remain. One of them changes which plan is forced. Look at the read counts of plan 7 and plan 9. Is plan 7 the plan you want to pin?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, unforce the failing plan and force the one that works" | Confuses the plan without failures with the good plan | "Check the runtime stats. How many logical reads does plan 7 have, and does it seek?" |
| "b, clear the bad state and force the good plan again" | Does not know CLEAR wipes all history | "What exactly does QUERY underscore STORE CLEAR remove? And after that, does query 7 still exist to be forced?" |
| "d, the portal recommends the fix" | Thinks Query Performance Insight is available everywhere | "Which platform offers Query Performance Insight? Where is TollGate running right now?" |
| "c is not enough, the forced plan is broken and must be re-forced" | Thinks a forcing failure permanently disables the forced plan | "Is the plan still marked is underscore forced underscore plan equal to one? What happens at the next compile once the index exists again?" |
| "The failure count will reset after the fix" | Expects counters to be cleared | "Are those counters a current state, or a history of what happened?" |
| "The procedure must have been erroring all week" | Thinks a forcing failure is a user error | "The scenario says correct results, just slow. What does the engine do when forcing fails?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain what Query Store persists:

- Per query, sys dot query underscore store underscore query keyed by query id, linked to the query text. Every plan the optimizer produced, sys dot query underscore store underscore plan with plan id, is underscore forced underscore plan, force underscore failure underscore count and last underscore force underscore failure underscore reason. The runtime statistics of each plan per time interval, sys dot query underscore store underscore runtime underscore stats, plus sys dot query underscore store underscore wait underscore stats when wait-stats capture is on. That is what made the regression visible: the same query id 7 has a scan plan, 7, with 361 reads, and a seek plan, 9, with 4 reads, and the seek plan only exists because the index exists.

Then plan forcing and forcing failures:

- sp underscore query underscore store underscore force underscore plan pins a plan id to a query id. Undo with sp underscore query underscore store underscore unforce underscore plan. Always read the runtime statistics before choosing the plan id to force; the plan without failures is not the same as the good plan. That is option a's mistake.
- A forcing failure is not an error. The engine compiles a normal plan, the user gets correct results, and the catalog records force_failure_count and a reason: NO_INDEX, an index referenced by the forced plan does not exist; also NO_PLAN, NO_TABLE, NO_CONSTRAINT, NO_PARTITION_SCHEME, NO_SEMANTICS, NO_STATISTICS, NO_INDEX_HINT and GENERAL_FAILURE. Fix the cause and the forced plan works again; the counters keep the history and are not reset by a later success. That is option c.

Then clearing and read-only:

- ALTER DATABASE SET QUERY_STORE CLEAR wipes queries, plans, runtime and wait statistics. Forcing a query id afterwards fails with error 12402, query not found; a wrong plan id for an existing query fails with 12406. That is option b.
- OPERATION_MODE can be set to READ_ONLY deliberately, readonly_reason zero, and Query Store goes read-only on its own when MAX_STORAGE_SIZE_MB is exhausted, readonly_reason non-zero. Check sys dot database underscore query underscore store underscore options. Capture modes: ALL records every query, AUTO, the default, skips infrequent and insignificant ones, CUSTOM lets you set thresholds, NONE stops capturing new queries. sp underscore query underscore store underscore flush underscore db writes in-memory statistics to disk at once; otherwise they appear after DATA_FLUSH_INTERVAL_SECONDS, default 900.

Then Query Performance Insight:

- An Azure SQL Database portal feature, single and pooled databases, built on top of Query Store. It charts top resource-consuming queries by CPU, duration and execution count, and long-running queries; it requires Query Store to be active and prompts to enable it when it is off or read-only; it annotates charts with Database Advisor recommendations. It does not exist for on-premises SQL Server or for Azure SQL Managed Instance. Plan forcing in Azure SQL Database comes from automatic tuning, FORCE_LAST_GOOD_PLAN, which reverts a plan regression detected by Query Store, or from sp underscore query underscore store underscore force underscore plan, not from a QPI button. That is option d.

Memory hook: "Forcing failure is not an error: fallback plan, counter, reason. NO INDEX means put the index back. CLEAR wipes history. QPI is Azure SQL Database only."

## 9. Follow-up oral questions (optional)

1. "After the index is re-created, what will force underscore failure underscore count show for plan 9?" (Still 1, with NO_INDEX; the counters are history and are not reset by later success.)
2. "What would happen if the DBA ran the force plan procedure with plan id 999 for query 7, without clearing?" (Error 12406, plan not found in the Query Store for that query.)
3. "Next year, after migrating to Azure SQL Database, which feature would automatically revert a plan regression detected by Query Store?" (Automatic tuning with FORCE_LAST_GOOD_PLAN equals ON.)

## 10. References

- Monitoring performance by using the Query Store: https://learn.microsoft.com/en-us/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store
- Best practices for managing the Query Store, including capture modes and read-only state: https://learn.microsoft.com/en-us/sql/relational-databases/performance/best-practice-with-the-query-store
- Query Store usage scenarios, plan forcing and forcing failures: https://learn.microsoft.com/en-us/sql/relational-databases/performance/query-store-usage-scenarios
- sys.query_store_plan, force failure reasons: https://learn.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-query-store-plan-transact-sql
- sp_query_store_force_plan: https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-query-store-force-plan-transact-sql
- sp_query_store_unforce_plan: https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-query-store-unforce-plan-transact-sql
- ALTER DATABASE SET options, QUERY_STORE CLEAR: https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-database-transact-sql-set-options
- Query Performance Insight for Azure SQL Database: https://learn.microsoft.com/en-us/azure/azure-sql/database/query-performance-insight-use
- Automatic tuning, FORCE_LAST_GOOD_PLAN: https://learn.microsoft.com/en-us/azure/azure-sql/database/automatic-tuning-overview
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
