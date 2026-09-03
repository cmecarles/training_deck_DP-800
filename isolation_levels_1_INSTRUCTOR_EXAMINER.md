# Instructor-Examiner guide — Isolation Levels 1

Companion to [isolation_levels_1.md](isolation_levels_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Read the timeline T1 to T6 slowly, then all four options, before taking an answer. Each option describes what happens at T5, what happens at T6, and the final row. Offer to repeat the timeline as often as needed.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Optimize database performance.
- Task bullet: Implement isolation levels and concurrency controls.
- What is tested: how SNAPSHOT isolation detects an update conflict, when the error is raised, and how that differs from a lost update, from commit-time validation and from a deadlock.

## 2. Scenario to read aloud

**Piece 1, the story.** "A concert hall runs its ticket booking system on Azure SQL Database, in a database called Ticketing. Two customers, Alex and Dana, try to book the same seat, number 101, at almost the same time, from two separate sessions."

**Piece 2, the table.** "Seat inventory is in one table, Box dot Seats. Three columns. SeatId, an integer, the primary key. Status, text up to twenty characters, NOT NULL, holding either Available or Booked. And BookedBy, text up to fifty characters, nullable. One row is inserted: seat 101, status Available, BookedBy NULL."

**Piece 3, the database settings.** "The database uses the Azure SQL Database default settings for row versioning, and this has been verified in sys dot databases. READ underscore COMMITTED underscore SNAPSHOT is ON, and ALLOW underscore SNAPSHOT underscore ISOLATION is ON. No table hints are used, no lock timeout is configured, and no other session touches the table."

**Piece 4, the timeline.** "The statements run in exactly this order, and every step completes or fails before the next one starts, except where the question itself is what happens.

- T1. Session 1, customer Alex, runs SET TRANSACTION ISOLATION LEVEL SNAPSHOT, then BEGIN TRANSACTION.
- T2. Session 1 selects Status and BookedBy from Box dot Seats where SeatId is 101. It gets Available and NULL.
- T3. Session 2, customer Dana, at the default READ COMMITTED level, runs BEGIN TRANSACTION and then updates seat 101 to Status Booked, BookedBy Dana. It completes immediately.
- T4. Session 2 commits. Completes at T4.
- T5. Session 1 updates seat 101 to Status Booked, BookedBy Alex.
- T6. Session 1 runs COMMIT TRANSACTION."

**Piece 5, option a.** "Option a. At T5 the update succeeds immediately, because under SNAPSHOT isolation Session 1 operates on its own row version of seat 101. At T6 the commit succeeds and the last write wins. Final row: Booked, Alex. Dana's booking is silently overwritten."

**Piece 6, option b.** "Option b. At T5 the update succeeds and writes a new row version, but the conflict with Session 2's change is detected when Session 1 tries to commit. The commit at T6 fails with an update-conflict error and the transaction is rolled back. Final row: Booked, Dana."

**Piece 7, option c.** "Option c. At T5 the update fails immediately with error 3960, Snapshot isolation transaction aborted due to update conflict, and Session 1's transaction is automatically rolled back. T6 no longer has an open transaction to commit. Final row: Booked, Dana. Alex's booking attempt must be retried in a new transaction."

**Piece 8, option d.** "Option d. At T5 the two sessions form a deadlock. Session 1 waits for the row modified by Session 2, while Session 2 waits for the row version Session 1 read at T2. The engine chooses Session 1 as the deadlock victim and terminates it with error 1205. Final row: Booked, Dana."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE TABLE Box.Seats
(
    SeatId    int NOT NULL PRIMARY KEY,
    Status    nvarchar(20) NOT NULL,   -- N'Available' or N'Booked'
    BookedBy  nvarchar(50) NULL
);
GO

INSERT INTO Box.Seats (SeatId, Status, BookedBy)
VALUES (101, N'Available', NULL);
GO

SELECT is_read_committed_snapshot_on,   -- returns 1 (ON: Azure SQL Database default)
       snapshot_isolation_state_desc    -- returns 'ON' (Azure SQL Database default)
FROM sys.databases
WHERE name = N'Ticketing';
```

Session 1 (Alex):

```sql
-- T1
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
BEGIN TRANSACTION;

-- T2
SELECT Status, BookedBy
FROM Box.Seats
WHERE SeatId = 101;
-- Returns: Status = N'Available', BookedBy = NULL

-- T5
UPDATE Box.Seats
SET Status = N'Booked', BookedBy = N'Alex'
WHERE SeatId = 101;

-- T6
COMMIT TRANSACTION;
```

Session 2 (Dana), default READ COMMITTED:

```sql
-- T3
BEGIN TRANSACTION;

UPDATE Box.Seats
SET Status = N'Booked', BookedBy = N'Dana'
WHERE SeatId = 101;
-- Completes immediately at T3

-- T4
COMMIT TRANSACTION;
-- Completes at T4
```

| Time | Session 1 (SNAPSHOT) | Session 2 (READ COMMITTED) |
|------|---|---|
| T1 | SET SNAPSHOT; BEGIN TRANSACTION | |
| T2 | SELECT seat 101 → Available, NULL | |
| T3 | | BEGIN TRANSACTION; UPDATE seat 101 → Dana |
| T4 | | COMMIT TRANSACTION |
| T5 | UPDATE seat 101 → Alex | |
| T6 | COMMIT TRANSACTION | |

## 4. The question (ask exactly this)

"What happens in Session 1 at T5 and T6, and what is the final committed state of seat 101? Choose a, b, c or d."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: c.**

At T5 the UPDATE fails immediately with Msg 3960, Level 16, State 2, "Snapshot isolation transaction aborted due to update conflict." The whole Session 1 transaction is rolled back automatically. At T6 there is no open transaction, so COMMIT has nothing to commit (attempting it raises Msg 3902, no corresponding BEGIN TRANSACTION). Final row: SeatId 101, Status Booked, BookedBy Dana. Alex must retry in a new transaction.

| Option | Why it is wrong |
|---|---|
| a | Describes a lost update. Row versioning applies to reads; a snapshot transaction is never allowed to write over a newer committed version. Option a is what would happen under READ COMMITTED with RCSI, which has no conflict detection |
| b | Puts the validation at commit time. For disk-based tables the conflict is detected when the conflicting UPDATE or DELETE statement executes, never at COMMIT. Commit-time validation exists for memory-optimized tables, not here |
| d | A deadlock needs a cycle of waiting. Session 1's SELECT took no locks at all, and Session 2 has already committed at T4, so at T5 only one transaction is active. Nobody waits on anybody |

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with T2. Under SNAPSHOT isolation, does the SELECT take any locks on the row? And what does that mean for Session 2 at T3?"
2. "Session 2 was not blocked, and by T4 it has committed. So at T5, who else holds a lock on seat 101? If nobody does, can there be any waiting at all? That rules out one option."
3. "Option d is out. Now think about the word optimistic. SNAPSHOT lets readers proceed without locks, but every write is validated against something. Against what?"
4. "The engine compares the latest committed version of the row with the moment Session 1's snapshot began. The row was changed and committed after that moment. Is Session 1 allowed to write over it? That rules out another option."
5. "Option a is out. Two options remain, and they differ only in when the conflict is detected: at the UPDATE statement, or at COMMIT. For a disk-based table, which one is it?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, snapshot means Session 1 works on its own copy, last writer wins" | Confuses versioned reads with unchecked writes | "Versioning applies to reads. Does SNAPSHOT let a transaction write over a row that someone else committed after the snapshot began?" |
| "b, optimistic systems validate at commit" | Transplants commit-time validation from other engines | "For a disk-based table in SQL Server, which statement detects the conflict: the UPDATE or the COMMIT?" |
| "d, two sessions on one row, must be a deadlock" | Thinks any conflict is a deadlock | "A deadlock needs two sessions each waiting on the other. At T5, is Session 2 still active? Did Session 1's SELECT take any lock?" |
| "Session 1's UPDATE blocks and waits for Session 2" | Ignores that Session 2 committed at T4 | "Session 2's lock was released at T4. Is there anything left to wait for at T5?" |
| "Session 2 is blocked at T3 by Session 1's read" | Thinks snapshot reads hold shared locks | "What locks does a SELECT acquire under SNAPSHOT isolation?" |
| "The error is 1205" | Mixes up the deadlock error with the update-conflict error | "1205 is the deadlock victim error. Is this a deadlock?" |

## 8. Teaching notes (after the answer is complete or revealed)

The one behaviour that separates SNAPSHOT from every locking isolation level: writes are validated with optimistic conflict detection. Walk the timeline:

- **T1.** SET SNAPSHOT is legal because ALLOW_SNAPSHOT_ISOLATION is ON, the Azure SQL Database default. BEGIN TRANSACTION alone does not fix the snapshot yet. A snapshot transaction logically starts at its first data access.
- **T2.** The SELECT is the first data access. From now on Session 1 sees the database as it was at this instant. The read takes no shared locks.
- **T3.** Session 2 requests an exclusive lock. Nobody holds a lock on the row, so it is not blocked. The old image, Available and NULL, goes to the version store so Session 1's view stays consistent.
- **T4.** Session 2 commits and releases its lock. The current committed row is Booked, Dana.
- **T5.** Session 1 tries to update. The engine checks the latest committed version and finds it was modified by a transaction that committed after Session 1's snapshot began. Update conflict. There is nothing to wait for, so the statement fails immediately with Msg 3960, and the whole transaction is rolled back automatically.
- **T6.** No open transaction. COMMIT has nothing to commit; attempting it raises Msg 3902.

Why each wrong option is wrong:

- **Option a** is a lost update, precisely what snapshot conflict detection prevents. It would happen if Session 1 ran under READ COMMITTED with RCSI, which gives statement-level versioned reads but no conflict detection. That is why a booking system on RCSI alone needs UPDLOCK, a rowversion check, or SNAPSHOT.
- **Option b** puts the check at COMMIT. For disk-based tables the conflicting statement itself fails. The only timing variation: if Session 2 had still been uncommitted at T5, Session 1 would block on the exclusive lock and then get 3960 the moment Session 2 committed, or succeed if Session 2 rolled back. Still at the statement.
- **Option d** needs mutual waiting. Snapshot reads hold no locks, and Session 2 finished at T4. A reader-writer deadlock could happen under REPEATABLE READ, where both SELECTs hold shared locks and both UPDATEs try to convert them.

Decision points to memorize:

- Azure SQL Database defaults: READ_COMMITTED_SNAPSHOT ON and ALLOW_SNAPSHOT_ISOLATION ON. SQL Server on-premises defaults both to OFF.
- The snapshot starts at the first data access, not at BEGIN TRANSACTION.
- Conflicting writer still active: the snapshot updater blocks, then gets 3960 when the writer commits. Conflicting writer already committed: immediate 3960.
- Mitigations: retry logic for error 3960, or take the lock early with SELECT WITH UPDLOCK to trade optimism for blocking.

Memory hook: "Snapshot readers never block. Snapshot writers get 3960 at the statement, not at commit."

## 9. Follow-up oral questions (optional)

1. "If Session 1 had used the default READ COMMITTED level with RCSI on, what would have happened at T5 and T6?" (The UPDATE locks and modifies the current row, the COMMIT succeeds, Dana is silently overwritten by Alex: a lost update.)
2. "If Session 2 had not yet committed when Session 1 ran its UPDATE at T5, what would Session 1 do?" (Block on Session 2's exclusive lock, then fail with 3960 when Session 2 commits, or succeed if Session 2 rolls back.)
3. "What is the simplest way to make Session 1's read take the lock early so the second booker waits instead of failing later?" (SELECT with the UPDLOCK hint at T2.)

## 10. References

- SET TRANSACTION ISOLATION LEVEL: https://learn.microsoft.com/en-us/sql/t-sql/statements/set-transaction-isolation-level-transact-sql
- Snapshot isolation in SQL Server, including update conflicts: https://learn.microsoft.com/en-us/dotnet/framework/data/adonet/sql/snapshot-isolation-in-sql-server
- Transaction locking and row versioning guide: https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-locking-and-row-versioning-guide
- ALTER DATABASE SET options, READ_COMMITTED_SNAPSHOT and ALLOW_SNAPSHOT_ISOLATION: https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-database-transact-sql-set-options
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
