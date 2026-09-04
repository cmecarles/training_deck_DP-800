# Instructor-Examiner guide — Optimized Locking 1

Companion to [optimized_locking_1.md](optimized_locking_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.
7. **This is a multiple-choice question.** Each option describes four observations, labelled one to four, plus a final state. Read all four options, pieces 8 to 11, before taking an answer. Take one letter as the answer. If the learner wants to reason step by step, let them talk through observations one to four, but the answer is one letter.
8. The timing matters. Make sure the learner has understood that session 1 holds its transaction open for twelve seconds, session 2 starts at three seconds, and the monitor query runs at six seconds.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Optimize and troubleshoot database solutions (20–25%).
- Skill: Resolve performance-related issues.
- Task bullet: Identify and resolve blocking and locking issues, including optimized locking.
- What is tested: what transaction ID locking holds until commit, how lock after qualification lets writers on different rows pass, how it evaluates predicates on the committed version, and what a blocked session looks like in the DMVs.

## 2. Scenario to read aloud

**Piece 1, the story.** "A regional rail operator allocates platform slots in a SQL Server 2025 database named RailSlots. Dispatchers complained that holding a platform for one train blocks other dispatchers who are working on other platforms. So the DBA decided to try optimized locking. The first attempt failed."

**Piece 2, the failed attempt.** "The DBA created the database and ran ALTER DATABASE RailSlots SET OPTIMIZED underscore LOCKING equals ON. The engine answered with message 12133: Optimized Locking cannot be enabled for this database because Accelerated Database Recovery is not enabled. Enable Accelerated Database Recovery and try again. Followed by message 5069, ALTER DATABASE statement failed."

**Piece 3, the prerequisites.** "So the DBA ran three ALTER DATABASE statements. First, SET ACCELERATED underscore DATABASE underscore RECOVERY equals ON. Second, SET READ underscore COMMITTED underscore SNAPSHOT ON WITH ROLLBACK IMMEDIATE. Third, SET OPTIMIZED underscore LOCKING equals ON. Then a check: DATABASEPROPERTYEX of RailSlots, IsOptimizedLockingOn, returns 1."

**Piece 4, the tables.** "In RailSlots, a schema Rail. Table Rail dot Slots has four columns. SlotId, an integer, the primary key. Platform, an integer, and note: there is no index on Platform. Status, a VARCHAR ten. TrainRef, a VARCHAR ten, nullable. Six rows, all with status FREE and no TrainRef: slots 1, 2 and 3 on platform 1. Slots 4 and 5 on platform 2. Slot 6 on platform 3. A second table, Rail dot Signals, is a heap with no index at all: SignalId, an integer, and Aspect, an integer. One row: signal 1, aspect 1."

**Piece 5, session 1.** "Three sessions run, all at the default READ COMMITTED isolation level. Session 1 is spid 61 and starts at time zero. It begins a transaction. It updates Rail dot Slots, setting Status to HELD and TrainRef to IC dash 201, where Platform equals 1. That touches three rows. Then it updates Rail dot Signals, setting Aspect to 2 where SignalId equals 1. Then it waits twelve seconds with WAITFOR DELAY. Then it commits."

**Piece 6, session 2.** "Session 2 is spid 102 and starts at three seconds, while session 1 is still waiting. It does four things. Observation one: it queries sys dot dm underscore tran underscore locks for the RailSlots database, for sessions other than itself, showing resource type, resource description, request mode and request status, filtered to resource types OBJECT, PAGE, KEY, RID and XACT. Then it sets LOCK underscore TIMEOUT to four thousand milliseconds. Observation two: begin tran, update Rail dot Slots set Status HELD, TrainRef RE dash 77, where Platform equals 2. Observation three: update Rail dot Signals set Aspect to 3 where Aspect equals 2. Then commit. Observation four: begin tran, update Rail dot Slots set Status HELD, TrainRef RE dash 78, where Platform equals 1. Then, if a transaction is still open, rollback."

**Piece 7, session 3.** "Session 3 is a monitor. At six seconds it queries sys dot dm underscore exec underscore requests for the RailSlots database, other sessions only, showing session id, blocking session id, wait type, wait resource and command."

**Piece 8, option a.** "Option a says. Observation one: session 1 holds only three locks. OBJECT IX on Rail dot Slots, OBJECT IX on Rail dot Signals, and one XACT X lock, described as XACT colon 18 colon 1272 colon 0. No KEY, PAGE or RID locks. Observation two completes immediately, two rows. Observation three completes immediately, zero rows. Observation four waits. Session 3 sees session 102 blocked by 61, wait type LCK underscore M underscore S underscore XACT underscore MODIFY, wait resource XACT 18 colon 1272 colon 0 followed by a KEY resource, command UPDATE. After four seconds session 2 gets message 1222, lock request time out period exceeded. Final state: slots 1 to 3 HELD by IC dash 201, slots 4 and 5 HELD by RE dash 77, slot 6 FREE, and Signals Aspect equals 2."

**Piece 9, option b.** "Option b says. Observation one: session 1 holds KEY X on slots 1, 2 and 3, PAGE IX, OBJECT IX on both tables, and RID X on the signal row. Observation two is blocked, because the scan of Rail dot Slots takes U locks row by row and hits session 1's X key locks, and fails after four seconds with message 1222. Observation three waits for session 1 and then updates one row. Observation four is blocked and times out. Final state: slots 1 to 3 HELD by IC dash 201, slots 4 to 6 FREE, Signals Aspect equals 3."

**Piece 10, option c.** "Option c says. Observation one: session 1 holds the three locks of option a. Observation two is blocked, because the XACT X lock is a transaction-wide lock on every table the transaction touched, so any write to Rail dot Slots must wait; session 2 gets message 1222 after four seconds. Observation three is blocked for the same reason. Observation four is blocked. Final state: slots 1 to 3 HELD by IC dash 201, slots 4 to 6 FREE, Signals Aspect equals 2."

**Piece 11, option d.** "Option d says. Observation one: session 1 holds the three locks of option a. Observation two completes immediately, two rows. Observation three waits for session 1 to commit, re-evaluates its predicate against the committed row, where Aspect is now 2, and updates one row. Observation four waits. Session 3 sees wait type LCK underscore M underscore X on a KEY resource, and session 2 gets message 1222 after four seconds. Final state: slots 1 to 3 HELD by IC dash 201, slots 4 and 5 HELD by RE dash 77, slot 6 FREE, Signals Aspect equals 3."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE RailSlots;
GO
ALTER DATABASE RailSlots SET OPTIMIZED_LOCKING = ON;
```

```text
Msg 12133, Level 16, State 2
Optimized Locking cannot be enabled for this database because Accelerated Database Recovery is not enabled.
Enable Accelerated Database Recovery and try again.
Msg 5069, Level 16, State 1
ALTER DATABASE statement failed.
```

```sql
ALTER DATABASE RailSlots SET ACCELERATED_DATABASE_RECOVERY = ON;
ALTER DATABASE RailSlots SET READ_COMMITTED_SNAPSHOT ON WITH ROLLBACK IMMEDIATE;
ALTER DATABASE RailSlots SET OPTIMIZED_LOCKING = ON;
GO
SELECT DATABASEPROPERTYEX('RailSlots', 'IsOptimizedLockingOn') AS IsOptimizedLockingOn;   -- 1
GO
USE RailSlots;
GO
CREATE SCHEMA Rail;
GO
CREATE TABLE Rail.Slots
(
    SlotId   INT         NOT NULL PRIMARY KEY,
    Platform INT         NOT NULL,          -- no index on Platform
    Status   VARCHAR(10) NOT NULL,
    TrainRef VARCHAR(10) NULL
);
INSERT INTO Rail.Slots (SlotId, Platform, Status) VALUES
    (1, 1, 'FREE'), (2, 1, 'FREE'), (3, 1, 'FREE'),
    (4, 2, 'FREE'), (5, 2, 'FREE'), (6, 3, 'FREE');
GO
CREATE TABLE Rail.Signals (SignalId INT NOT NULL, Aspect INT NOT NULL);   -- heap
INSERT INTO Rail.Signals VALUES (1, 1);
GO
```

```sql
-- Session 1 (spid 61), t = 0
BEGIN TRAN;
UPDATE Rail.Slots   SET Status = 'HELD', TrainRef = 'IC-201' WHERE Platform = 1;   -- 3 rows
UPDATE Rail.Signals SET Aspect = 2 WHERE SignalId = 1;
WAITFOR DELAY '00:00:12';
COMMIT;

-- Session 2 (spid 102), t = 3 s
SELECT request_session_id, resource_type, resource_description, request_mode, request_status
FROM sys.dm_tran_locks
WHERE resource_database_id = DB_ID('RailSlots') AND request_session_id <> @@SPID
  AND resource_type IN ('OBJECT', 'PAGE', 'KEY', 'RID', 'XACT');                     -- (i)
SET LOCK_TIMEOUT 4000;
BEGIN TRAN;
UPDATE Rail.Slots SET Status = 'HELD', TrainRef = 'RE-77' WHERE Platform = 2;       -- (ii)
UPDATE Rail.Signals SET Aspect = 3 WHERE Aspect = 2;                                -- (iii)
COMMIT;
BEGIN TRAN;
UPDATE Rail.Slots SET Status = 'HELD', TrainRef = 'RE-78' WHERE Platform = 1;       -- (iv)
IF @@TRANCOUNT > 0 ROLLBACK;

-- Session 3 (monitor), t = 6 s
SELECT session_id, blocking_session_id, wait_type, wait_resource, command
FROM sys.dm_exec_requests WHERE database_id = DB_ID('RailSlots') AND session_id <> @@SPID;
```

## 4. The question (ask exactly this)

"Which option describes what happens? Option a, b, c or d?"

- **a.** (i) Session 1 holds only three locks: `OBJECT IX` on `Rail.Slots`, `OBJECT IX` on `Rail.Signals`, and one `XACT X` lock (`XACT: 18:1272:0`), no `KEY`, `PAGE` or `RID` locks. (ii) completes immediately, 2 rows. (iii) completes immediately, **0 rows**. (iv) waits; session 3 sees `102 | 61 | LCK_M_S_XACT_MODIFY | XACT: 18:1272:0 KEY: 18:72057594047234048 (8194443284a0) | UPDATE`, and after 4 s session 2 gets `Msg 1222 Lock request time out period exceeded.` Final state: slots 1–3 `HELD/IC-201`, slots 4–5 `HELD/RE-77`, slot 6 `FREE`, `Signals.Aspect = 2`.
- **b.** (i) Session 1 holds `KEY X` on slots 1, 2 and 3, `PAGE IX`, `OBJECT IX` on both tables and `RID X` on the signal row. (ii) is blocked, the scan of `Rail.Slots` takes `U` locks row by row and hits session 1's `X` key locks, and fails after 4 s with `Msg 1222`. (iii) waits for session 1 and then updates 1 row. (iv) is blocked and times out. Final state: slots 1–3 `HELD/IC-201`, slots 4–6 `FREE`, `Signals.Aspect = 3`.
- **c.** (i) Session 1 holds the three locks of option a. (ii) is blocked: the `XACT X` lock is a transaction-wide lock on every table the transaction touched, so any write to `Rail.Slots` must wait; session 2 gets `Msg 1222` after 4 s. (iii) is blocked for the same reason. (iv) is blocked. Final state: slots 1–3 `HELD/IC-201`, slots 4–6 `FREE`, `Signals.Aspect = 2`.
- **d.** (i) Session 1 holds the three locks of option a. (ii) completes immediately, 2 rows. (iii) waits for session 1 to commit, re-evaluates its predicate against the committed row (`Aspect = 2`) and updates **1 row**. (iv) waits; session 3 sees `wait_type = LCK_M_X` on `KEY: 18:72057594047234048 (8194443284a0)`, and session 2 gets `Msg 1222` after 4 s. Final state: slots 1–3 `HELD/IC-201`, slots 4–5 `HELD/RE-77`, slot 6 `FREE`, `Signals.Aspect = 3`.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

| Observation | Outcome | Why |
|---|---|---|
| (i) | OBJECT IX on Slots, OBJECT IX on Signals, one XACT X. No KEY, PAGE or RID. | TID locking: page and row locks are released per row; only the X lock on the transaction ID is held until commit. |
| (ii) | Immediate, 2 rows | LAQ: the scan checks Platform on the latest committed version with no U locks; the three held rows do not qualify and are skipped. |
| (iii) | Immediate, 0 rows | LAQ checks Aspect equals 2 on the committed version, where Aspect is 1. Not qualified, skipped, no wait. |
| (iv) | Waits, then Msg 1222 after 4 s | Slots 1 to 3 qualify; session 2 asks for an S lock on session 1's TID. Wait type LCK_M_S_XACT_MODIFY, wait resource XACT: 18:1272:0 plus the KEY. |
| Final | Slots 1–3 HELD/IC-201, 4–5 HELD/RE-77, 6 FREE, Aspect 2 | Session 1 committed Aspect 2; session 2's signals update touched nothing. |

Why the wrong options are wrong:

- **b.** That is exactly the classic behaviour with OPTIMIZED_LOCKING OFF: KEY X, PAGE IX, RID X held to commit, U-lock scans that collide, and a requalified signals update to 3. It is what the DBA switched on optimized locking to avoid.
- **c.** The XACT X lock is not a table lock. It protects only the rows the transaction modified; other writers take an S lock on that TID only when they try to modify one of those rows. Rows that never qualified are never touched. Optimized locking holds fewer locks, not bigger ones, and avoids escalation.
- **d.** Requalification only happens for a row that qualified on the committed version and then changed before the lock was taken. A row whose committed version does not qualify is skipped without waiting, so (iii) is 0 rows and Aspect ends at 2. And the wait in (iv) is LCK_M_S_XACT_MODIFY on the XACT resource, not LCK_M_X on a key.

## 6. Hint ladder (one hint per attempt, in order)

1. "Optimized locking has two parts. One changes what is held until commit. The other changes how a writer decides which rows to lock. Name the two parts, then think about which observations each one affects."
2. "For observation one: with transaction ID locking, what single lock does the transaction keep until commit? Are the individual key, page and row locks still held?"
3. "For observation two: session 2 scans all six slots because Platform has no index. Under lock after qualification with RCSI, is the predicate checked with a U lock, or against the last committed version without any lock? What happens to a row that does not qualify?"
4. "For observation three: what is the committed value of Aspect while session 1 is still open? Is it 1 or 2? Check the predicate Aspect equals 2 against that committed value."
5. "That eliminates option b, which is the old behaviour, and option c, which treats the XACT lock as a table lock. Two options remain and they differ on observation three and on the wait type in observation four."
6. "Requalification only happens when a row first qualified on the committed version and then changed. Did the signal row qualify at all? And when a writer waits on another transaction's TID, does it ask for an X lock on a key, or an S lock on the XACT resource?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "b, the scan takes U locks and hits the X key locks" | Describes classic locking, forgets the feature is on | "That is how READ COMMITTED behaves without the feature. What does lock after qualification change about that scan?" |
| "c, the XACT lock protects everything the transaction touched" | Thinks the TID lock is table-wide | "Which rows carry session 1's transaction ID? Does slot 4 carry it?" |
| "d, session 2 waits and then sees Aspect equals 2" | Confuses requalification with re-scanning | "Requalification applies to a row that qualified first. Did the signal row qualify on its committed version?" |
| "d, the wait type is LCK_M_X on the key" | Knows the classic wait type only | "What resource does a writer wait on when the row is stamped with another transaction's ID? Look at the resource types listed in observation one." |
| "a, but observation two should be blocked because there is no index" | Believes the missing index forces blocking | "Without an index, session 2 scans all rows. Under LAQ, does scanning a held row require a lock?" |
| "Optimized locking cannot be on because the first ALTER failed" | Missed the prerequisites piece | "Recall what the DBA did after the failure. What did DATABASEPROPERTYEX return?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the two components:

- **Transaction ID locking.** Each row is labelled with the last transaction ID that modified it. Instead of many key or row identifier locks, one X lock on the TID resource protects all the rows the transaction modified. Page and row locks are still taken while each row is modified, but they are released as soon as that row is done. The only lock held until commit is the single X lock on the XACT resource. That is observation one: two OBJECT IX locks and one XACT X, nothing else. Intent object locks are unaffected. Lock escalation is avoided, lock memory drops.
- **Lock after qualification.** Without the feature, predicates are checked row by row by first taking a U row lock. With the feature and RCSI, predicates are checked on the latest committed version without any lock. A row that does not qualify is skipped. A row that qualifies is then locked, and if it changed since the check, the predicate is re-evaluated. If a qualifying row is stamped with an active transaction's TID, the writer asks for an S lock on that TID and waits.
- **Observation two.** Platform has no index, so session 2 scans all six rows. Slots 1 to 3 have committed Platform 1, so they do not qualify for Platform 2 and are skipped without waiting. Slots 4 and 5 update immediately. Two rows.
- **Observation three.** The committed Aspect is still 1. The predicate Aspect equals 2 fails on the committed version, so the row is skipped. Zero rows, no wait. This is the documented behaviour change: LAQ removes blocking but can lead to different results. Workloads that rely on strict execution order should use REPEATABLE READ or SERIALIZABLE, or the READCOMMITTEDLOCK hint.
- **Observation four.** Slots 1 to 3 do qualify for Platform 1. Session 2 requests an S lock on session 1's TID. The new wait type is LCK_M_S_XACT_MODIFY, the wait resource is XACT 18 colon 1272 colon 0 with the key underneath. dm_tran_locks shows session 102 requesting XACT in mode S with status WAIT. LOCK_TIMEOUT 4000 turns the wait into Msg 1222. Without it, session 2 would complete after session 1 commits. In a deadlock graph the same appears as xactlock elements.
- **Prerequisites and availability.** Optimized locking needs accelerated database recovery, Msg 12133 if missing, and ADR cannot be turned off while it is on, Msg 12134. LAQ needs RCSI and default READ COMMITTED; it is not used with UPDLOCK, READCOMMITTEDLOCK, XLOCK, HOLDLOCK, other isolation levels, columnstore, MERGE, OUTPUT into a table variable, or variable assignment. It is always on in Azure SQL Database, SQL database in Fabric and Managed Instance with the always-up-to-date or 2025 policy; off by default per user database in SQL Server 2025; not on SQL Server 2022; not used in tempdb.

Memory hook: "One X lock on the transaction ID. Predicates checked on the committed version, no U locks. Not qualified, skip. Qualified and held, wait on XACT."

## 9. Follow-up oral questions (optional)

1. "How would you make observation three block and then update one row, as the old behaviour did, while keeping optimized locking on?" (Add the READCOMMITTEDLOCK or UPDLOCK hint, or run it under REPEATABLE READ or SERIALIZABLE, so LAQ is not used.)
2. "With optimized locking on, what happens if you try ALTER DATABASE RailSlots SET ACCELERATED_DATABASE_RECOVERY OFF?" (It fails with Msg 12134. Disable optimized locking first.)
3. "Does transaction ID locking need read committed snapshot?" (No. TID locking works without RCSI. Only lock after qualification needs RCSI.)

## 10. References

- Optimized locking: https://learn.microsoft.com/en-us/sql/relational-databases/performance/optimized-locking
- Accelerated database recovery: https://learn.microsoft.com/en-us/sql/relational-databases/accelerated-database-recovery-concepts
- ALTER DATABASE SET options (OPTIMIZED_LOCKING, ACCELERATED_DATABASE_RECOVERY, READ_COMMITTED_SNAPSHOT): https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-database-transact-sql-set-options
- sys.dm_tran_locks: https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-tran-locks-transact-sql
- sys.dm_exec_requests: https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-requests-transact-sql
- Transaction locking and row versioning guide: https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-locking-and-row-versioning-guide
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
