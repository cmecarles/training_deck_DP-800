# Instructor-Examiner guide — Blocking and Deadlocks 1

Companion to [blocking_deadlocks_1.md](blocking_deadlocks_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Read all four options before taking an answer. Ask the learner for one letter and one sentence on why. If the letter is wrong, do not say which letter is right; go to the hint ladder.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Optimize database performance.
- Task bullet: Identify and resolve blocking and deadlocks.
- What is tested: reading a blocking chain in the DMVs, recognising a lock cycle from the deadlock report, and knowing which changes remove a deadlock versus which only change who loses.

## 2. Scenario to read aloud

**Piece 1, the story.** "A logistics hub tracks its loading bays in a SQL Server 2025 database called WarehouseDock. A double-length truck needs two bays at once, so a workflow reserves both bays inside one transaction. There are two workflows. The dispatch screen reserves bay 1 and then bay 2. The yard tablet, written by another team, reserves bay 2 and then bay 1. In production the two sometimes collide, and the DBA wants to reproduce that."

**Piece 2, the table.** "There is one table, in a schema called Dock, named Bays. It has three columns. BayId, an integer, the primary key. Status, a varchar of ten, not null. And TruckRef, a varchar of twenty, which allows null. Two rows are inserted: bay 1 with status FREE, and bay 2 with status FREE. TruckRef is null on both."

**Piece 3, session A.** "The DBA opens two sessions. Session A is spid 66. It plays the dispatch screen. It begins a transaction. It updates Dock dot Bays, setting Status to BUSY and TruckRef to TRK dash A, where BayId equals 1. Then it runs WAITFOR DELAY of eight seconds, to mimic a slow user. Then it updates the same table, same values, where BayId equals 2. Then it commits."

**Piece 4, session B.** "Session B is spid 68. It plays the yard tablet, and it starts one second after A. Its first line is SET DEADLOCK underscore PRIORITY LOW, because the tablet's connection uses a low priority setting. Then it begins a transaction. It updates Dock dot Bays, setting Status to BUSY and TruckRef to TRK dash B, where BayId equals 2. Then WAITFOR DELAY of three seconds. Then it updates the same table with the same values where BayId equals 1. Then it commits."

**Piece 5, the blocking snapshot.** "Five seconds in, a third session queries sys dot dm underscore exec underscore requests for sessions 66 and 68. It returns two rows. Session 66 has blocking session id zero, wait type WAITFOR, status suspended, command WAITFOR. Session 68 has blocking session id 66, wait type LCK underscore M underscore X, a wait resource that starts with KEY colon 17, status suspended, command UPDATE."

**Piece 6, the lock detail.** "sys dot dm underscore os underscore waiting underscore tasks shows the same edge: session 68 is blocked by 66, wait type LCK underscore M underscore X, waiting for about two seconds. sys dot dm underscore tran underscore locks shows session 66 holding a KEY X lock on bay 1, granted, plus an intent exclusive on the page and an intent exclusive on the object. Session 68 holds a KEY X lock on bay 2, granted, and its request for a KEY X lock on bay 1 is in WAIT."

**Piece 7, the deadlock.** "Three seconds later, session A's WAITFOR ends and A asks for bay 2. Session B's batch stops with message 1205, level 13, state 51. The text says: Transaction, process ID 68, was deadlocked on lock resources with another process and has been chosen as the deadlock victim. Rerun the transaction. Session A prints A committed, and the Bays table ends with both rows BUSY and TRK dash A."

**Piece 8, the deadlock report.** "The xml underscore deadlock underscore report event in the system underscore health session confirms it. The victim is the process with priority minus five, which is spid 68. The other process has priority zero. Both resources are keylock entries on WarehouseDock dot Dock dot Bays, on the primary key index. Each key is owned in mode X by one process and requested in mode X by the other."

**Piece 9, the requirement.** "The company wants the deadlock eliminated. Not a different victim. Eliminated. And the application's isolation semantics must not change. Four changes are proposed. I will read them one at a time."

**Piece 10, option a.** "Option a. Add SET DEADLOCK underscore PRIORITY HIGH at the start of the dispatch-screen workflow, that is session A's code, so that the dispatch transaction always wins."

**Piece 11, option b.** "Option b. Enable row versioning for the database so that the two updates no longer block each other. The statement is ALTER DATABASE WarehouseDock SET READ underscore COMMITTED underscore SNAPSHOT ON WITH ROLLBACK IMMEDIATE."

**Piece 12, option c.** "Option c. Prevent the lock from being escalated to the table by disabling lock escalation on the table. The statement is ALTER TABLE Dock dot Bays SET, open paren, LOCK underscore ESCALATION equals DISABLE, close paren."

**Piece 13, option d.** "Option d. Make both workflows update the bays in the same order, ascending BayId. Keep each transaction as short as possible, with no user interaction or delays between the two updates. And make the client retry the whole transaction when it receives error 1205."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE WarehouseDock;
GO
USE WarehouseDock;
GO
CREATE SCHEMA Dock;
GO
CREATE TABLE Dock.Bays
(
    BayId    INT         NOT NULL PRIMARY KEY,
    Status   VARCHAR(10) NOT NULL,
    TruckRef VARCHAR(20) NULL
);
INSERT INTO Dock.Bays (BayId, Status) VALUES (1, 'FREE'), (2, 'FREE');
GO
```

```sql
-- Session A (spid 66)                                -- Session B (spid 68)
BEGIN TRAN;                                           SET DEADLOCK_PRIORITY LOW;
UPDATE Dock.Bays SET Status = 'BUSY', TruckRef = 'TRK-A'   BEGIN TRAN;
    WHERE BayId = 1;                                  UPDATE Dock.Bays SET Status = 'BUSY', TruckRef = 'TRK-B'
WAITFOR DELAY '00:00:08';                                 WHERE BayId = 2;
UPDATE Dock.Bays SET Status = 'BUSY', TruckRef = 'TRK-A'   WAITFOR DELAY '00:00:03';
    WHERE BayId = 2;                                  UPDATE Dock.Bays SET Status = 'BUSY', TruckRef = 'TRK-B'
COMMIT;                                                   WHERE BayId = 1;
                                                      COMMIT;
```

```sql
-- Third session, five seconds in
SELECT session_id, blocking_session_id, wait_type, wait_resource, status, command
FROM sys.dm_exec_requests WHERE session_id IN (66, 68);
```

| session_id | blocking_session_id | wait_type | wait_resource | status | command |
|---|---|---|---|---|---|
| 66 | 0 | WAITFOR | | suspended | WAITFOR |
| 68 | 66 | LCK_M_X | KEY: 17:72057594047234048 (8194443284a0) | suspended | UPDATE |

```text
Msg 1205, Level 13, State 51, Line 8
Transaction (Process ID 68) was deadlocked on lock resources with another process and has been
chosen as the deadlock victim. Rerun the transaction.
```

Option statements:

```sql
-- a
SET DEADLOCK_PRIORITY HIGH;
-- b
ALTER DATABASE WarehouseDock SET READ_COMMITTED_SNAPSHOT ON WITH ROLLBACK IMMEDIATE;
-- c
ALTER TABLE Dock.Bays SET (LOCK_ESCALATION = DISABLE);
-- d: same access order (ascending BayId), short transactions, retry on 1205
```

## 4. The question (ask exactly this)

"The company wants the deadlock eliminated, not just a different victim, and the application's isolation semantics must not change. Which change achieves that? Option a, deadlock priority HIGH on session A. Option b, READ COMMITTED SNAPSHOT on the database. Option c, LOCK ESCALATION DISABLE on the table. Option d, same update order, short transactions, and retry on 1205. Give me one letter, and one sentence on why."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct: d.** Updating the bays in the same order removes the cycle: B's first update waits for A's KEY X on bay 1, which is plain blocking, and B continues once A commits. Short transactions shrink the window, and a retry on error 1205 handles the residual case. Isolation semantics are untouched.

- **a is wrong.** DEADLOCK_PRIORITY only chooses the victim. B is already LOW (priority minus five) and already loses. The cycle still forms every time the timing overlaps.
- **b is wrong.** RCSI fixes reader versus writer blocking. Both statements are UPDATEs, and writers still take X locks under RCSI, so the lock graph is identical and the deadlock reproduces. It also changes the database's default isolation semantics, which the requirement forbids.
- **c is wrong.** Each transaction holds one KEY lock plus intent locks. Escalation happens at roughly five thousand locks per statement on one object. There is no escalation here, so disabling it changes nothing.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start from the deadlock report. Two keylock resources, each owned in mode X by one process and requested in mode X by the other. What shape is that? What has to change so that shape can never form?"
2. "Three of the options change something about the engine's behaviour. One option changes what the application does. Which one attacks the cycle itself?"
3. "Think about option a. Session B is already LOW and is already the victim. If A becomes HIGH, who loses? And does the deadlock still happen?"
4. "Think about option c. Look at sys dot dm underscore tran underscore locks again. How many KEY locks does each session hold? Is anything being escalated to the table?"
5. "Two options left. One of them changes the database's isolation semantics. The requirement says that is not allowed. And ask yourself: does row versioning help two UPDATEs on the same row, or only a SELECT against an UPDATE?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, make dispatch win with HIGH priority" | Thinks victim selection stops the deadlock | "Would the cycle still form? And who receives 1205 then?" |
| "b, snapshot stops the blocking" | Believes RCSI removes writer versus writer locks | "Under RCSI, does an UPDATE still need an X lock on the row it changes?" |
| "b, and isolation is fine because it is still read committed" | Misses that RCSI changes the default isolation semantics | "Read the requirement again. Does turning on READ COMMITTED SNAPSHOT change what a SELECT sees?" |
| "c, escalation is causing a table lock" | Confuses intent locks with escalation | "Check the lock list. Is there an OBJECT X lock anywhere, or only OBJECT IX?" |
| "d, but retry alone would be enough" | Treats retry as the fix rather than the safety net | "Retry handles the failure. Which part of option d stops the failure from happening in the first place?" |
| "Add NOLOCK to the updates" | Thinks NOLOCK works on write targets | "Is NOLOCK allowed on the target table of an UPDATE? There is an error number for that." |

## 8. Teaching notes (after the answer is complete or revealed)

Explain blocking versus deadlock first:

- **Blocking** is one session waiting for another. It ends when the holder commits or rolls back. You see it in sys.dm_exec_requests as a non-zero blocking_session_id and a wait type that starts with LCK_M, and in sys.dm_os_waiting_tasks and sys.dm_tran_locks.
- **Deadlock** is a cycle. A holds bay 1's KEY X and wants bay 2. B holds bay 2's KEY X and wants bay 1. X is incompatible with X, S and U. The lock monitor wakes about every five seconds, detects the cycle, picks a victim, rolls that transaction back and returns error 1205. The message says "Rerun the transaction" for a reason.

Then why d works:

- **Same access order** removes the cycle. If both update bay 1 first, B simply waits on bay 1 until A commits, then takes bay 2, which A already released. Nobody ever holds what the other wants.
- **Short transactions** shrink the window. Holding an X lock across an eight-second WAITFOR, or across a user's screen, turned a sub-millisecond conflict into a certain deadlock. Locks are held until COMMIT or ROLLBACK.
- **Retry on 1205** handles the rare leftover. 1205 is a transient error; a TRY CATCH that checks ERROR_NUMBER() = 1205 and re-runs the whole transaction is the standard pattern.

Then why the others fail:

- **a.** DEADLOCK_PRIORITY ranges from LOW (minus five) through NORMAL (zero) to HIGH (ten), or any integer from minus ten to ten. The lowest priority loses; on a tie, the cheapest to roll back loses. It is a policy about who loses, not a cure. B is already the victim.
- **b.** RCSI lets readers see the last committed version instead of waiting for a writer. Writers still take U and X locks; two uncommitted versions of one row cannot coexist. Two UPDATEs deadlock exactly as before. SNAPSHOT isolation would not fix it either; it turns the second writer's wait into update conflict error 3960. And both change the isolation semantics, which the requirement forbids.
- **c.** Escalation to a table lock is attempted at about five thousand locks per statement on one object, and again every 1,250 further locks, or under lock memory pressure. Here each session holds one KEY X plus PAGE IX and OBJECT IX. Nothing escalates, so DISABLE changes nothing.

Two adjacent facts: NOLOCK on the target of an UPDATE fails with error 1065. And the deadlock graph is captured automatically as the xml_deadlock_report event in the always-on system_health extended events session; no trace flag 1222 is needed to read it after the fact.

Memory hook: "Same order, short transaction, retry on 1205. Priority picks the loser; snapshot helps readers; escalation needs thousands of locks."

## 9. Follow-up oral questions (optional)

1. "If both sessions had DEADLOCK_PRIORITY NORMAL, how does the engine choose the victim?" (The transaction that is cheapest to roll back, the one with the fewest log records.)
2. "Where do you read the deadlock graph after the fact, without having set up anything in advance?" (The xml_deadlock_report event in the system_health extended events session.)
3. "Under SNAPSHOT isolation, what happens to the second writer when the first commits?" (It gets update conflict error 3960 instead of waiting on the lock.)

## 10. References

- Deadlocks guide, victim selection and DEADLOCK_PRIORITY: https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-deadlocks-guide
- Transaction locking and row versioning guide, lock compatibility and escalation: https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-locking-and-row-versioning-guide
- SET DEADLOCK_PRIORITY: https://learn.microsoft.com/en-us/sql/t-sql/statements/set-deadlock-priority-transact-sql
- sys.dm_exec_requests: https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-requests-transact-sql
- sys.dm_tran_locks: https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-tran-locks-transact-sql
- ALTER TABLE, LOCK_ESCALATION option: https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-table-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
