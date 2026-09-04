# Instructor-Examiner guide — Database Configurations 1

Companion to [database_configurations_1.md](database_configurations_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d, each a script of four or five statements. Read all four findings, the constraint, and all four options before taking an answer. When the learner answers, ask them to map each statement of their chosen option to a finding.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Optimize database performance.
- Task bullet: Configure database-scoped configurations and database options.
- What is tested: choosing database-scoped settings over instance-wide ones for a single tenant on a shared instance, and knowing what READ_COMMITTED_SNAPSHOT does versus ALLOW_SNAPSHOT_ISOLATION.

## 2. Scenario to read aloud

**Piece 1, the story.** "An accounting SaaS runs about thirty customer databases on one SQL Server 2025 instance. Sixteen cores. The instance-level max degree of parallelism is zero. One of the databases, LedgerPulse, was just migrated from a SQL Server 2014 server and, as part of the migration, upgraded to compatibility level 170."

**Piece 2, the current settings.** "After the migration the team queries sys dot database underscore scoped underscore configurations in LedgerPulse. Six rows, all at their default values. MAXDOP is zero. LEGACY underscore CARDINALITY underscore ESTIMATION is zero, that is off. PARAMETER underscore SNIFFING is one, on. OPTIMIZE underscore FOR underscore AD underscore HOC underscore WORKLOADS is zero, off. PARAMETER underscore SENSITIVE underscore PLAN underscore OPTIMIZATION is one. And DOP underscore FEEDBACK is one. The is underscore value underscore default column is one on every row."

**Piece 3, finding 1.** "After a week in production the team has four findings and one hard constraint. Finding 1. The month-end consolidation query is twenty times slower than on the old server. Comparing plans shows that the new cardinality estimator, the compatibility level 170 model, badly misestimates a join in this query. The plan is the same for every parameter value, so it is not a parameter sniffing problem. The other twenty-nine databases benefit from the new estimator and must keep it."

**Piece 4, finding 2.** "Finding 2. The reporting tool used by this customer sends thousands of single-use ad hoc statements. LedgerPulse alone accounts for two gigabytes of single-use plans in the plan cache."

**Piece 5, finding 3.** "Finding 3. Dashboard readers are blocked by writers for seconds at a time. The application uses the default isolation level and cannot be changed. No SET TRANSACTION ISOLATION LEVEL, no hints."

**Piece 6, finding 4.** "Finding 4. Report queries in LedgerPulse go parallel across all sixteen cores and starve the other tenants. This customer's queries must be limited to four cores, while the other databases keep the instance default."

**Piece 7, the constraint.** "The constraint. LedgerPulse must stay at compatibility level 170, because its new stored procedures use REGEXP underscore LIKE. And no instance-wide setting may change."

**Piece 8, option a.** "Option a. Five statements. ALTER DATABASE LedgerPulse SET COMPATIBILITY underscore LEVEL equals 120. Then sp underscore configure show advanced options 1, reconfigure. Then sp underscore configure optimize for ad hoc workloads 1, reconfigure. Then sp underscore configure max degree of parallelism 4, reconfigure. And ALTER DATABASE LedgerPulse SET ALLOW underscore SNAPSHOT underscore ISOLATION ON."

**Piece 9, option b.** "Option b. USE LedgerPulse, then four statements. ALTER DATABASE SCOPED CONFIGURATION SET LEGACY underscore CARDINALITY underscore ESTIMATION equals ON. ALTER DATABASE SCOPED CONFIGURATION SET OPTIMIZE underscore FOR underscore AD underscore HOC underscore WORKLOADS equals ON. ALTER DATABASE SCOPED CONFIGURATION SET MAXDOP equals 4. And ALTER DATABASE LedgerPulse SET READ underscore COMMITTED underscore SNAPSHOT ON WITH ROLLBACK IMMEDIATE."

**Piece 10, option c.** "Option c. DBCC TRACEON, open paren, 9481, comma, minus 1, close paren. Then USE LedgerPulse. Then ALTER DATABASE SCOPED CONFIGURATION SET OPTIMIZE underscore FOR underscore AD underscore HOC underscore WORKLOADS equals ON. Then SET MAXDOP equals 4. And ALTER DATABASE LedgerPulse SET ALLOW underscore SNAPSHOT underscore ISOLATION ON."

**Piece 11, option d.** "Option d. USE LedgerPulse. ALTER DATABASE SCOPED CONFIGURATION SET PARAMETER underscore SNIFFING equals OFF. Then SET MAXDOP equals 4. Then ALTER DATABASE LedgerPulse SET AUTOMATIC underscore TUNING, open paren, FORCE underscore LAST underscore GOOD underscore PLAN equals ON, comma, CREATE underscore INDEX equals ON, close paren. And ALTER DATABASE LedgerPulse SET READ underscore COMMITTED underscore SNAPSHOT ON WITH ROLLBACK IMMEDIATE."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE LedgerPulse;
GO
ALTER DATABASE LedgerPulse SET COMPATIBILITY_LEVEL = 170;
GO
USE LedgerPulse;
GO
SELECT configuration_id, name, value, is_value_default
FROM sys.database_scoped_configurations
WHERE name IN ('MAXDOP', 'LEGACY_CARDINALITY_ESTIMATION', 'PARAMETER_SNIFFING',
               'OPTIMIZE_FOR_AD_HOC_WORKLOADS', 'PARAMETER_SENSITIVE_PLAN_OPTIMIZATION', 'DOP_FEEDBACK')
ORDER BY configuration_id;
```

| configuration_id | name | value | is_value_default |
|---|---|---|---|
| 1 | MAXDOP | 0 | 1 |
| 2 | LEGACY_CARDINALITY_ESTIMATION | 0 | 1 |
| 3 | PARAMETER_SNIFFING | 1 | 1 |
| 13 | OPTIMIZE_FOR_AD_HOC_WORKLOADS | 0 | 1 |
| 28 | PARAMETER_SENSITIVE_PLAN_OPTIMIZATION | 1 | 1 |
| 37 | DOP_FEEDBACK | 1 | 1 |

Option a:

```sql
ALTER DATABASE LedgerPulse SET COMPATIBILITY_LEVEL = 120;
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'optimize for ad hoc workloads', 1; RECONFIGURE;
EXEC sp_configure 'max degree of parallelism', 4; RECONFIGURE;
ALTER DATABASE LedgerPulse SET ALLOW_SNAPSHOT_ISOLATION ON;
```

Option b:

```sql
USE LedgerPulse;
ALTER DATABASE SCOPED CONFIGURATION SET LEGACY_CARDINALITY_ESTIMATION = ON;
ALTER DATABASE SCOPED CONFIGURATION SET OPTIMIZE_FOR_AD_HOC_WORKLOADS = ON;
ALTER DATABASE SCOPED CONFIGURATION SET MAXDOP = 4;
ALTER DATABASE LedgerPulse SET READ_COMMITTED_SNAPSHOT ON WITH ROLLBACK IMMEDIATE;
```

Option c:

```sql
DBCC TRACEON (9481, -1);
USE LedgerPulse;
ALTER DATABASE SCOPED CONFIGURATION SET OPTIMIZE_FOR_AD_HOC_WORKLOADS = ON;
ALTER DATABASE SCOPED CONFIGURATION SET MAXDOP = 4;
ALTER DATABASE LedgerPulse SET ALLOW_SNAPSHOT_ISOLATION ON;
```

Option d:

```sql
USE LedgerPulse;
ALTER DATABASE SCOPED CONFIGURATION SET PARAMETER_SNIFFING = OFF;
ALTER DATABASE SCOPED CONFIGURATION SET MAXDOP = 4;
ALTER DATABASE LedgerPulse SET AUTOMATIC_TUNING (FORCE_LAST_GOOD_PLAN = ON, CREATE_INDEX = ON);
ALTER DATABASE LedgerPulse SET READ_COMMITTED_SNAPSHOT ON WITH ROLLBACK IMMEDIATE;
```

## 4. The question (ask exactly this)

"Which script addresses all four findings within the constraint? Option a, compatibility level 120, instance-wide sp underscore configure for ad hoc workloads and MAXDOP 4, and ALLOW SNAPSHOT ISOLATION. Option b, database-scoped LEGACY CARDINALITY ESTIMATION, OPTIMIZE FOR AD HOC WORKLOADS and MAXDOP 4, and READ COMMITTED SNAPSHOT. Option c, global trace flag 9481, database-scoped ad hoc and MAXDOP 4, and ALLOW SNAPSHOT ISOLATION. Option d, database-scoped PARAMETER SNIFFING OFF and MAXDOP 4, AUTOMATIC TUNING with FORCE LAST GOOD PLAN and CREATE INDEX, and READ COMMITTED SNAPSHOT. Give me one letter, and map each statement to a finding."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct: b.** Every knob is database-scoped. LEGACY_CARDINALITY_ESTIMATION ON gives this database the old estimator while it stays at level 170 (finding 1). OPTIMIZE_FOR_AD_HOC_WORKLOADS ON at database scope stores plan stubs on first execution (finding 2). READ_COMMITTED_SNAPSHOT ON changes what the default isolation level does, so readers stop blocking with no application change (finding 3). Database-scoped MAXDOP 4 caps this database only (finding 4). Afterwards the catalog shows MAXDOP 4, LEGACY CE 1, ad hoc 1, and sys.databases shows compatibility 170, is_read_committed_snapshot_on 1, snapshot_isolation_state OFF.

- **a is wrong.** Dropping to compatibility level 120 breaks the constraint and REGEXP_LIKE. Both sp_configure changes are instance-wide and throttle all thirty tenants. ALLOW_SNAPSHOT_ISOLATION only permits sessions that explicitly ask for SNAPSHOT; the default-isolation readers keep taking shared locks.
- **c is wrong.** Trace flag 9481 does select the legacy estimator, but DBCC TRACEON with minus 1 is global, for every database, and is not persisted across restarts. The database-scoped configuration was introduced in 2016 to replace it. Also repeats the ALLOW_SNAPSHOT_ISOLATION mistake.
- **d is wrong.** PARAMETER_SNIFFING OFF does nothing for a misestimate that is the same for every parameter. AUTOMATIC_TUNING with CREATE_INDEX is Azure SQL Database only; on SQL Server it is a syntax error, message 102, and FORCE_LAST_GOOD_PLAN fails on this edition with 15707 and would need Query Store anyway. Nothing addresses finding 2.

## 6. Hint ladder (one hint per attempt, in order)

1. "Every finding is about one database on a shared instance. So what scope must every change have? Which statements in the options are instance-wide?"
2. "Look at option a's first line. What does dropping the compatibility level do to REGEXP underscore LIKE? Which constraint does that break?"
3. "Option c starts with a DBCC TRACEON and a minus one. What does the minus one mean, and does it respect finding 1's demand that the other twenty-nine databases keep the new estimator?"
4. "Finding 3 says the application stays at the default isolation level. There are two database options with snapshot in the name. One changes what the default level does; the other only allows sessions that explicitly ask for SNAPSHOT. Which one helps here?"
5. "Two options left, b and d. Finding 1 says the plan is the same for every parameter value. Does turning parameter sniffing off help a misestimate that is not about parameters? And is CREATE underscore INDEX in AUTOMATIC underscore TUNING valid on SQL Server at all?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, go back to level 120 like the old server" | Treats compatibility level as the only way to get the old estimator | "What does the constraint say about the level, and why? Is there a database-scoped setting that gives the old estimator without changing the level?" |
| "c, trace flag 9481 is the legacy CE switch" | Forgets the minus one makes it global | "Who is affected by DBCC TRACEON with minus one? Just LedgerPulse, or everyone?" |
| "ALLOW SNAPSHOT ISOLATION fixes the readers" | Confuses the two snapshot options | "The application never asks for SNAPSHOT. Does ALLOW underscore SNAPSHOT underscore ISOLATION change what READ COMMITTED does?" |
| "d, parameter sniffing off fixes the bad plan" | Misreads finding 1 | "Finding 1 says the plan is the same for every parameter. Is that a sniffing problem?" |
| "d, automatic tuning will create the missing index" | Thinks CREATE_INDEX exists on SQL Server | "On which platform is CREATE underscore INDEX an automatic tuning option? What does SQL Server say when you try it?" |
| "sp underscore configure for MAXDOP is fine, four is a good number" | Ignores the other tenants | "Who else gets capped at four cores if you set it with sp underscore configure?" |

## 8. Teaching notes (after the answer is complete or revealed)

The theme is scope. Every finding is about one database on a shared instance, so every knob must be database-scoped. SQL Server offers two families: ALTER DATABASE SCOPED CONFIGURATION SET, visible in sys.database_scoped_configurations, for optimizer and execution settings; and ALTER DATABASE SET, visible in sys.databases, for database options. Instance-wide look-alikes exist for most of them and are the trap: sp_configure max degree of parallelism, sp_configure optimize for ad hoc workloads, DBCC TRACEON 9481 or 4199 with minus 1.

Why b works, finding by finding:

- **Finding 1.** LEGACY_CARDINALITY_ESTIMATION ON makes the optimizer use the SQL Server 2012-era estimator for this database while the compatibility level stays 170, so REGEXP_LIKE keeps working. The other databases keep the default. Per query, the same effect is OPTION USE HINT FORCE_LEGACY_CARDINALITY_ESTIMATION, but the scenario forbids touching the application.
- **Finding 2.** OPTIMIZE_FOR_AD_HOC_WORKLOADS ON, database-scoped since SQL Server 2019, stores only a small compiled-plan stub on first execution and the full plan only if the batch runs again. It is the per-database twin of the sp_configure option.
- **Finding 3.** READ_COMMITTED_SNAPSHOT ON changes what the default isolation level does: readers get the last committed version from the version store instead of waiting for a writer's X lock. The application keeps running at read committed and stops blocking. WITH ROLLBACK IMMEDIATE is needed because the switch requires exclusive access. ALLOW_SNAPSHOT_ISOLATION only permits sessions that explicitly request SNAPSHOT; verified that after turning it on, default-isolation readers still take shared locks.
- **Finding 4.** Database-scoped MAXDOP 4 caps parallelism for queries in LedgerPulse; the instance value still governs the others. Values are validated; MAXDOP 100000 fails with error 12108.

Why the others fail: a drops to 120 (accepted by the engine, but the constraint forbids it), uses instance-wide sp_configure, and picks the wrong snapshot option. c uses a global trace flag, not persisted across restarts, that the 2016 database-scoped configuration was designed to replace, and repeats the snapshot mistake. d turns off parameter sniffing for a non-sniffing problem, uses CREATE_INDEX which is Azure SQL Database only and a syntax error on SQL Server, FORCE_LAST_GOOD_PLAN which needs Enterprise or Developer edition and Query Store and only reverts detected regressions, and does nothing about the ad hoc plan bloat.

Memory hook: "One database on a shared instance means database-scoped, never sp_configure or global trace flags. RCSI changes the default; ALLOW_SNAPSHOT only allows. Legacy CE without leaving the level."

## 9. Follow-up oral questions (optional)

1. "If you could touch the query, how would you get the legacy estimator for just that one statement?" (OPTION with USE HINT FORCE_LEGACY_CARDINALITY_ESTIMATION.)
2. "After option b, what does sys.databases show for LedgerPulse's snapshot_isolation_state_desc?" (OFF. Only is_read_committed_snapshot_on is 1.)
3. "Why does READ_COMMITTED_SNAPSHOT ON need WITH ROLLBACK IMMEDIATE?" (The option requires exclusive access to the database to switch; ROLLBACK IMMEDIATE kicks out the other sessions.)

## 10. References

- ALTER DATABASE SCOPED CONFIGURATION: https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-database-scoped-configuration-transact-sql
- ALTER DATABASE SET options, including READ_COMMITTED_SNAPSHOT and ALLOW_SNAPSHOT_ISOLATION: https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-database-transact-sql-set-options
- ALTER DATABASE compatibility level: https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-database-transact-sql-compatibility-level
- Cardinality estimation: https://learn.microsoft.com/en-us/sql/relational-databases/performance/cardinality-estimation-sql-server
- Automatic tuning: https://learn.microsoft.com/en-us/sql/relational-databases/automatic-tuning/automatic-tuning
- Snapshot isolation in SQL Server: https://learn.microsoft.com/en-us/dotnet/framework/data/adonet/sql/snapshot-isolation-in-sql-server
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
