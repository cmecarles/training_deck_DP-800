# Instructor-Examiner guide — Query Store Hints 1

Companion to [query_store_hints_1.md](query_store_hints_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

**Specific to this question.** This is a multi-part lab question with six numbered statements, S1 to S6. Take Part 1 one statement at a time, then Part 2, then Part 3. The learner may have run the lab already; if so, ask what they observed before you confirm anything. Query Store identifiers in this file are the ones the reference run produced (query 11, query 14, plan 19); tell the learner that their own numbers may differ and that the reasoning is what counts.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Optimize database performance.
- Task bullet: Execution plans, DMVs, Query Store, Query Performance Insight.
- What is tested: how Query Store hints are created, inspected and removed; which hints are rejected; that a hint replaces the statement's own OPTION clause; that a hint that cannot produce a plan is ignored and counted rather than failing the query; and that hints and forced plans exclude each other.

## 2. Scenario to read aloud

**Piece 1, the story.** "A harbour authority sells ferry passes and records every crossing in a SQL Server 2025 database called PierPass. The DBA switched Query Store on at database level, with operation mode READ underscore WRITE and query capture mode ALL, so that every statement is captured. The flush interval is sixty seconds and the statistics interval is one minute."

**Piece 2, the tables.** "There is a schema called Pier with two tables. Pier dot Passes has PassId, an integer primary key, Holder, a name, and PassType, a one-character code: D for day, M for month, A for annual. Pier dot Crossings has CrossingId, an integer primary key, PassId, a foreign key to Passes, Route, a three-letter code, NTH, STH or EST, CrossedAt, a datetime2, and Fare, a decimal. There are two thousand passes and two hundred thousand crossings, and a nonclustered index on Crossings dot PassId."

**Piece 3, the vendor procedure.** "The first procedure is Pier dot usp underscore RouteRevenue. It takes a parameter, at From, a datetime2. It joins Crossings to Passes on PassId, keeps crossings on or after at From, groups by Route, and returns the route, the count of crossings and the sum of fares, ordered by route. The last line of the query is a hard-coded hint: OPTION, open paren, LOOP JOIN, close paren. The procedure comes from a signed vendor package, so nobody can edit that text."

**Piece 4, the audit procedure.** "The second procedure is Pier dot usp underscore FaresAbovePass, with an integer parameter at MaxPass. It joins Crossings to Passes with a non-equality condition: c dot Fare greater than p dot PassId. It keeps passes with PassId less than or equal to at MaxPass, groups by PassId and returns the pass and a count. It has no hint."

**Piece 5, the lookup.** "The revenue procedure ran twice and the audit procedure ran once. The DBA flushed Query Store and joined sys dot query underscore store underscore query, sys dot query underscore store underscore query underscore text and sys dot query underscore store underscore plan. The output had two rows. Query 11 is usp RouteRevenue, plan 11, not forced, the join operator in the plan is Nested Loops, no Query Store hint text in the plan, two executions. Query 14 is usp FaresAbovePass, plan 14, not forced, Nested Loops, no hint, one execution. The view sys dot query underscore store underscore query underscore hints is empty."

**Piece 6, the six statements.** "Because the loop join makes the report slow, the DBA decides to use Query Store hints. Six statements run in order, each in its own batch.

- S1 calls sys dot sp underscore query underscore store underscore set underscore hints for query 11 with the string OPTION, open paren, OPTIMIZE FOR, open paren, at From equals the date 2026 01 01, close paren, close paren.
- S2 calls the same procedure for query 11 with the string OPTION, open paren, RECOMPILE, close paren. Right after it, the DBA executes usp RouteRevenue once and flushes Query Store.
- S3 calls set underscore hints for query 14 with OPTION, open paren, HASH JOIN, close paren. Right after it, the DBA executes usp FaresAbovePass once and flushes.
- S4 calls sp underscore query underscore store underscore force underscore plan for query 11 and plan 19. Plan 19 is the plan that the execution after S2 added to the catalog.
- S5 is two statements: sys dot sp underscore query underscore store underscore clear underscore hints for query 11, and then force plan again for query 11, plan 19.
- S6 calls set underscore hints for query 11 with OPTION, open paren, MAXDOP 1, close paren."

## 3. Setup script (reference only; do not read verbatim unless asked)

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
CREATE PROCEDURE Pier.usp_FaresAbovePass @MaxPass INT
AS
SELECT p.PassId, COUNT(*) AS Crossings
FROM Pier.Crossings AS c
JOIN Pier.Passes AS p ON c.Fare > p.PassId
WHERE p.PassId <= @MaxPass
GROUP BY p.PassId;
GO
EXEC Pier.usp_RouteRevenue @From = '20260101';
EXEC Pier.usp_RouteRevenue @From = '20260301';
EXEC Pier.usp_FaresAbovePass @MaxPass = 5;
EXEC sp_query_store_flush_db;
GO
-- S1
EXEC sys.sp_query_store_set_hints @query_id = 11, @query_hints = N'OPTION(OPTIMIZE FOR (@From = ''20260101''))';
-- S2  (then: EXEC Pier.usp_RouteRevenue @From = '20260101';  EXEC sp_query_store_flush_db;)
EXEC sys.sp_query_store_set_hints @query_id = 11, @query_hints = N'OPTION(RECOMPILE)';
-- S3  (then: EXEC Pier.usp_FaresAbovePass @MaxPass = 5;  EXEC sp_query_store_flush_db;)
EXEC sys.sp_query_store_set_hints @query_id = 14, @query_hints = N'OPTION(HASH JOIN)';
-- S4
EXEC sp_query_store_force_plan @query_id = 11, @plan_id = 19;
-- S5
EXEC sys.sp_query_store_clear_hints @query_id = 11;
EXEC sp_query_store_force_plan @query_id = 11, @plan_id = 19;
-- S6
EXEC sys.sp_query_store_set_hints @query_id = 11, @query_hints = N'OPTION(MAXDOP 1)';
```

## 4. The question (ask exactly this)

"Part 1. For each of the six statements, S1 to S6, tell me whether it succeeds or raises an error, and give the error number when it fails. One statement at a time, starting with S1."

"Part 2. Think about the execution of usp RouteRevenue right after S2. Which physical join operator does its plan use? And what does the plan XML carry, in the StmtSimple element, that proves where the hint came from?"

"Part 3. Now the execution of usp FaresAbovePass right after S3. Does it fail? And what does sys dot query underscore store underscore query underscore hints show for query 14 in query underscore hint underscore failure underscore count, last underscore query underscore hint underscore failure underscore reason, and the reason description?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Part 1**

| Stmt | Outcome | Detail |
|---|---|---|
| S1 | Fails, error 12455 | "Setting query hint(s) 'OPTIMIZE FOR' in Query Store is not supported." OPTIMIZE FOR with a value is unsupported; only OPTIMIZE FOR UNKNOWN is |
| S2 | Succeeds | hint id 1 for query 11, text OPTION(RECOMPILE), source User |
| S3 | Succeeds | hint id 2 for query 14, text OPTION(HASH JOIN). The string is accepted; whether it can produce a plan is only known at compile time |
| S4 | Fails, error 12458 | "Query with query_id 11 has query store hints. Query Store can't force a plan for it while it has hints." |
| S5 | Succeeds | hints cleared, then plan 19 forced, is_forced_plan = 1 |
| S6 | Fails, error 12457 | "Query with query_id 11 has forced plan. No hints can be applied to it while it has forced plan." |

**Part 2.** Hash Match, in a new plan, plan 19. The StmtSimple element has QueryStoreStatementHintText equal to OPTION(RECOMPILE), QueryStoreStatementHintId equal to 1, and QueryStoreStatementHintSource equal to User. The hard-coded LOOP JOIN was replaced, not merged, so the optimizer chose the join type freely.

**Part 3.** The execution succeeds, with the old Nested Loops plan 14, now at two executions. The catalog shows query_hint_failure_count 1, last_query_hint_failure_reason 8622, last_query_hint_failure_reason_desc NO_PLAN. A hash join needs an equality predicate, so the hint cannot produce a plan; it is ignored and counted, the query is not blocked.

## 6. Hint ladder (one hint per attempt, in order)

**S1**
1. "Look closely at the hint. It is OPTIMIZE FOR with a specific value. Is that the same hint as OPTIMIZE FOR UNKNOWN?"
2. "The set underscore hints procedure validates the hint names before storing anything. Some OPTION hints are on a do-not-support list. Which family does this one belong to?"
3. "It is an error, raised by the procedure itself, and nothing is stored. The number is in the twelve thousand four hundreds."

**S2**
1. "RECOMPILE is on the supported list. Does anything else stop the procedure from storing it?"
2. "Query 11 exists and has no forced plan. So what happens?"

**S3**
1. "Do not think about whether HASH JOIN can work on that query yet. Think only about what set underscore hints checks when it is called."
2. "The procedure checks the syntax and the hint name. HASH JOIN is a supported name. The engine finds out whether it can be applied only later, at compile time."

**S4**
1. "Before this statement runs, what is stored for query 11 in sys dot query underscore store underscore query underscore hints?"
2. "Plan forcing and Query Store hints are two ways to steer the same query. Can they both be active on one query?"
3. "They cannot. One of them is already there. The error number is 12458."

**S5**
1. "What does the first statement of S5 remove?"
2. "Once the hint is gone, is there still anything blocking force underscore plan?"

**S6**
1. "What is the state of query 11 after S5?"
2. "S4 failed because a hint blocked a forced plan. Think about the mirror image."
3. "Same rule, other direction. The error number is 12457."

**Part 2**
1. "The query text still says LOOP JOIN. But what reaches the optimizer: the text's OPTION clause plus the hint, or the hint instead of the text's OPTION clause?"
2. "The documentation says Query Store hints override hard-coded statement-level hints. If LOOP JOIN is no longer there, what join does the optimizer pick for a two hundred thousand row join with a group by?"
3. "For the proof, think of three attributes on StmtSimple whose names start with QueryStore Statement Hint."

**Part 3**
1. "A hash join needs an equality predicate. This join uses greater than. Can the optimizer build a hash join plan at all?"
2. "If that hint were written inside the query text, the query would fail with error 8622. Does a Query Store hint behave the same way, or more gently?"
3. "The query is never blocked by a Query Store hint. The hint is dropped for that compilation. Now look at the failure columns: what number and what description would you expect?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "S1 succeeds; OPTIMIZE FOR is a supported hint" | Confuses OPTIMIZE FOR UNKNOWN with OPTIMIZE FOR a value | "There are two forms of OPTIMIZE FOR. Which of them is on the supported list?" |
| "S3 fails with 8622" | Thinks the procedure compiles the query when the hint is set | "When is a hint tested against the query: when it is stored, or when the query is compiled?" |
| "After S2 the plan still uses Nested Loops; RECOMPILE only recompiles" | Believes the hint is appended to the text's OPTION clause | "Does the Query Store hint add to the OPTION clause, or take its place?" |
| "After S3 the audit query fails" | Believes an unusable hint is an error for the user | "The documentation says queries with Query Store hints always execute. So what happens to the hint instead?" |
| "S4 succeeds; plan forcing is independent of hints" | Does not know the mutual exclusion | "Check the catalog before S4. What is stored for query 11? Can that coexist with a forced plan?" |
| "S6 succeeds because the hint was cleared in S5" | Forgets that S5 also forced a plan | "S5 has two statements. What did the second one do to query 11?" |
| "The failure reason description is NO_INDEX" | Confuses plan-forcing failure reasons with hint failure reasons | "NO_INDEX is a plan forcing reason. Here the optimizer could not produce any plan with the hint. What would that be called?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the workflow first:

- **Find the query.** Query Store must be on. Join sys dot query underscore store underscore query to sys dot query underscore store underscore query underscore text to read the query_id. Capture mode ALL guarantees that every statement is captured; sp underscore query underscore store underscore flush underscore db makes the data visible at once.
- **Set the hint.** sys dot sp underscore query underscore store underscore set underscore hints takes at query_id and at query_hints, a string that must start with OPTION. Calling it again replaces the previous hint string for that query. An unknown query_id raises error 12402. A string without OPTION fails to parse with error 102. An unsupported hint, such as OPTIMIZE FOR with a value, USE PLAN, MAXRECURSION or any table hint, raises 12455, and nothing is stored. That is S1.
- **The hint replaces the OPTION clause.** Query Store hints override hard-coded statement hints and plan guides. The vendor's LOOP JOIN disappears from the optimizer's view, so the optimizer picks a Hash Match. The plan XML proves it with QueryStoreStatementHintText, QueryStoreStatementHintId and QueryStoreStatementHintSource on StmtSimple. That is S2 and Part 2.
- **A hint that cannot produce a plan is ignored, not fatal.** In the query text, HASH JOIN on an inequality join fails with 8622. As a Query Store hint the query still runs, the hint is dropped, and the catalog records query_hint_failure_count 1, reason 8622, description NO_PLAN. That is S3 and Part 3.
- **Hints and forced plans are exclusive.** With a hint stored, sp underscore query underscore store underscore force underscore plan fails with 12458. With a forced plan, set underscore hints fails with 12457. Clear one before using the other. That is S4, S5 and S6.

Then the side rules: hints persist and survive restarts; manual hints are exempt from Query Store cleanup; hints are skipped for statements that qualify for simple parameterization; and RECOMPILE inside a hint is ignored with warning 12461 when the database uses forced parameterization.

Memory hook: "Set it, the hint replaces the OPTION clause. Cannot plan it, the hint is dropped and counted. Forced plan or hint, never both."

## 9. Follow-up oral questions (optional)

1. "How do you remove the hint from query 14?" (EXEC sys dot sp underscore query underscore store underscore clear underscore hints at query_id equals 14.)
2. "Which source_desc values can appear in sys dot query underscore store underscore query underscore hints?" (User for hints you set, and CE feedback for hints created automatically by cardinality estimation feedback.)
3. "Where would a Query Store hint fit in an Azure SQL Database scenario where the application text is generated by an ORM?" (Exactly there: the hint is applied by query_id without touching the generated text; Query Performance Insight in the portal can help find the query, and the same set underscore hints procedure applies the hint.)

## 10. References

- Query Store hints overview: https://learn.microsoft.com/en-us/sql/relational-databases/performance/query-store-hints
- sys.sp_query_store_set_hints, with the supported and unsupported hint lists: https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sys-sp-query-store-set-hints-transact-sql
- sys.sp_query_store_clear_hints: https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sys-sp-query-store-clear-hints-transact-sql
- sys.query_store_query_hints catalog view: https://learn.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-query-store-query-hints-transact-sql
- sp_query_store_force_plan: https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-query-store-force-plan-transact-sql
- Query hints (OPTION clause): https://learn.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-query
- Monitor performance by using the Query Store: https://learn.microsoft.com/en-us/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
