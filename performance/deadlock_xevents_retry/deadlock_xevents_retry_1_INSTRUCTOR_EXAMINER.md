# Instructor-Examiner guide — Deadlock Extended Events and Retry 1

Companion to [deadlock_xevents_retry_1.md](deadlock_xevents_retry_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

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

**Specific to this question.** This is a multi-part lab question about two concurrent sessions. It is a timing scenario, so read pieces 5 and 6 slowly and offer to repeat the order of the updates. The learner may have run it already; ask what they observed before confirming. Part 1 has four sub-answers, Part 2 has three; accept them in any order but check each one.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Optimize database performance.
- Task bullet: Isolation levels and concurrency controls; monitoring with Extended Events and DMVs.
- What is tested: how the engine chooses a deadlock victim, priority first and then rollback cost; what error 1205 looks like; what TRY CATCH sees after a deadlock and how a correct retry loop is written; and where the deadlock graph is recorded and how it is read.

## 2. Scenario to read aloud

**Piece 1, the story.** "A sawmill schedules cutting work orders in a SQL Server 2025 database called MillQueue. There is a planning screen used by the office, and a yard tablet used next to the saws. Both can assign work orders to a crew, and one afternoon they deadlocked."

**Piece 2, the tables.** "There is a schema called Mill with two tables. Mill dot WorkOrders has OrderId, an integer primary key, Species, a string, Status, a string, and Crew, a nullable string. Two rows: order 1, OAK, QUEUED, and order 2, PINE, QUEUED. Mill dot CutLog has CutId, an integer primary key, OrderId and Boards, both integers. It holds five thousand rows, all for order 2, all with Boards equal to zero."

**Piece 3, the procedure with retry.** "The planning screen calls a procedure, Mill dot usp underscore AssignPair, with parameters at First, at Second, at Crew and at Delay. It loops up to three attempts. Inside a TRY it begins a transaction, updates the first order to ASSIGNED with the crew, waits for at Delay, updates the second order the same way, commits, prints the crew name and committed on attempt, and returns. In the CATCH it saves ERROR underscore NUMBER, XACT underscore STATE and at at TRANCOUNT into variables. If XACT underscore STATE is not zero it rolls back. Then, if the error is 1205 and attempts remain, it prints that this attempt was the deadlock victim, together with the saved XACT underscore STATE and TRANCOUNT values, waits half a second, and loops again. Any other error, or the last attempt, is rethrown with THROW."

**Piece 4, the Extended Events session.** "The DBA also creates an Extended Events session on the server called MillDeadlocks. It adds the event sqlserver dot xml underscore deadlock underscore report and an event underscore file target named MillDeadlocks dot xel, with startup state on, and starts it."

**Piece 5, session A and session B, run 1.** "Two sessions run at the same time. Session A is the planning screen. It calls usp AssignPair with First equal to 1, Second equal to 2, crew CREW dash A, and a delay of four seconds. So A updates order 1, waits four seconds, then updates order 2. Session B is the yard tablet, starting one second after A. It has no retry logic. In one transaction it updates all five thousand CutLog rows for order 2, adding one to Boards, then updates order 2 to ASSIGNED for CREW dash B, waits one second, then updates order 1 for CREW dash B, commits, and prints B committed. Both sessions use default READ COMMITTED and default deadlock priority."

**Piece 6, run 2.** "Afterwards the DBA resets the data: both orders back to QUEUED with a null crew, and Boards back to zero everywhere. Then the experiment is repeated as run 2 with one change only: session A's batch starts with SET DEADLOCK underscore PRIORITY HIGH before the EXEC."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE MillQueue;
GO
USE MillQueue;
GO
CREATE SCHEMA Mill;
GO
CREATE TABLE Mill.WorkOrders
(
    OrderId INT         NOT NULL PRIMARY KEY,
    Species VARCHAR(10) NOT NULL,
    Status  VARCHAR(10) NOT NULL,
    Crew    VARCHAR(10) NULL
);
CREATE TABLE Mill.CutLog
(
    CutId   INT NOT NULL PRIMARY KEY,
    OrderId INT NOT NULL,
    Boards  INT NOT NULL
);
INSERT INTO Mill.WorkOrders (OrderId, Species, Status) VALUES (1, 'OAK', 'QUEUED'), (2, 'PINE', 'QUEUED');
INSERT INTO Mill.CutLog (CutId, OrderId, Boards)
SELECT n, 2, 0
FROM (SELECT TOP (5000) ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n
      FROM sys.all_columns a CROSS JOIN sys.all_columns b) AS x;
GO
CREATE PROCEDURE Mill.usp_AssignPair
    @First INT, @Second INT, @Crew VARCHAR(10), @Delay CHAR(8) = '00:00:00'
AS
SET NOCOUNT ON;
DECLARE @attempt INT = 1, @maxAttempts INT = 3;
WHILE @attempt <= @maxAttempts
BEGIN
    BEGIN TRY
        BEGIN TRAN;
        UPDATE Mill.WorkOrders SET Status = 'ASSIGNED', Crew = @Crew WHERE OrderId = @First;
        WAITFOR DELAY @Delay;
        UPDATE Mill.WorkOrders SET Status = 'ASSIGNED', Crew = @Crew WHERE OrderId = @Second;
        COMMIT;
        PRINT CONCAT(@Crew, ': committed on attempt ', @attempt);
        RETURN 0;
    END TRY
    BEGIN CATCH
        DECLARE @err INT = ERROR_NUMBER(), @xs INT = XACT_STATE(), @tc INT = @@TRANCOUNT;
        IF @xs <> 0 ROLLBACK;
        IF @err = 1205 AND @attempt < @maxAttempts
        BEGIN
            PRINT CONCAT(@Crew, ': attempt ', @attempt, ' was the deadlock victim (error ', @err,
                         ', XACT_STATE ', @xs, ', @@TRANCOUNT ', @tc, '); retrying');
            SET @attempt += 1;
            WAITFOR DELAY '00:00:00.500';
        END
        ELSE
            THROW;
    END CATCH
END;
PRINT CONCAT(@Crew, ': gave up after ', @maxAttempts, ' attempts');
RETURN 1;
GO
CREATE EVENT SESSION MillDeadlocks ON SERVER
ADD EVENT sqlserver.xml_deadlock_report
ADD TARGET package0.event_file (SET filename = N'MillDeadlocks.xel', max_file_size = 5, max_rollover_files = 2)
WITH (STARTUP_STATE = ON);
ALTER EVENT SESSION MillDeadlocks ON SERVER STATE = START;
GO
-- Session A (run 1)
EXEC Mill.usp_AssignPair @First = 1, @Second = 2, @Crew = 'CREW-A', @Delay = '00:00:04';
-- Session A (run 2)
SET DEADLOCK_PRIORITY HIGH;
EXEC Mill.usp_AssignPair @First = 1, @Second = 2, @Crew = 'CREW-A', @Delay = '00:00:04';
-- Session B (both runs, starts one second after A)
-- yard tablet: no retry logic
BEGIN TRAN;
UPDATE Mill.CutLog SET Boards = Boards + 1 WHERE OrderId = 2;
UPDATE Mill.WorkOrders SET Status = 'ASSIGNED', Crew = 'CREW-B' WHERE OrderId = 2;
WAITFOR DELAY '00:00:01';
UPDATE Mill.WorkOrders SET Status = 'ASSIGNED', Crew = 'CREW-B' WHERE OrderId = 1;
COMMIT;
PRINT 'B committed';
-- Reading the graph
SELECT CAST(event_data AS XML) AS ev
FROM sys.fn_xe_file_target_read_file('system_health*.xel', NULL, NULL, NULL)
WHERE object_name = 'xml_deadlock_report';
```

## 4. The question (ask exactly this)

"Part 1, run 1. Which session is chosen as the deadlock victim, and by what rule? What does each session print? And what are the final rows of Mill dot WorkOrders and the sum of Boards in CutLog?"

"Part 2, run 2, with session A at DEADLOCK PRIORITY HIGH. Which session is the victim now? What does session B receive, error number and text? And what are the final rows of WorkOrders and the sum of Boards?"

"Part 3. In the CATCH block of the victim in run 1, what values do XACT underscore STATE and at at TRANCOUNT have? And why must the procedure roll back and then re-run the whole transaction, rather than only the statement that failed?"

"Part 4. Without the MillDeadlocks session, where is the deadlock graph already recorded? Which function reads it from disk? And which two attributes of each process element explain the victim choice in runs 1 and 2?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Part 1.** Victim: session A. Both sessions have deadlock priority 0, so the engine picks the transaction that is cheaper to roll back, measured in log bytes written: A has logused 300, B has logused 600304 because of the five-thousand-row CutLog update. Session A prints two lines: "CREW-A: attempt 1 was the deadlock victim (error 1205, XACT_STATE -1, @@TRANCOUNT 1); retrying" and then "CREW-A: committed on attempt 2". Session B prints its row counts and "B committed", with no error. Final state: order 1 OAK ASSIGNED CREW-A, order 2 PINE ASSIGNED CREW-A, because A's retry ran after B committed and overwrote CREW-B; sum of Boards is 5000.

**Part 2.** Victim: session B. A has priority 5, B has 0; priority is compared before cost, so the process with far more log is killed. B's batch ends with "Msg 1205, Level 13, State 51, Line 6. Transaction (Process ID 61) was deadlocked on lock resources with another process and has been chosen as the deadlock victim. Rerun the transaction." A prints "CREW-A: committed on attempt 1". Final state: both orders ASSIGNED CREW-A; sum of Boards is 0, because B's whole transaction, including the CutLog update, was rolled back.

**Part 3.** XACT_STATE is minus 1 and @@TRANCOUNT is 1. Inside TRY CATCH the 1205 error does not end the batch; the victim's changes are undone but the transaction stays open in an uncommittable, doomed, state, so the CATCH must ROLLBACK before it can begin a new transaction. The rollback undid the first UPDATE too, and the other session has committed in the meantime, so retrying only the failed statement would leave order 1 unassigned and would act on stale assumptions; the retry must start again at BEGIN TRAN.

**Part 4.** In the always-on system_health Extended Events session, event xml_deadlock_report, kept in a ring buffer and in system_health underscore star dot xel files in the instance Log folder. Read with sys dot fn underscore xe underscore file underscore target underscore read underscore file, first argument 'system_health*.xel', the other three NULL, filtering object_name equal to xml_deadlock_report. The two attributes are priority, 0 and 0 in run 1, 5 and 0 in run 2, and logused, 300 for A and 600304 for B.

## 6. Hint ladder (one hint per attempt, in order)

**Part 1, the victim**
1. "Both sessions have the same deadlock priority. What is the tie-breaker rule?"
2. "The engine kills the transaction that is cheapest to roll back. How does it measure cost?"
3. "Cost is log bytes written. Which session updated five thousand rows before the deadlock?"

**Part 1, what is printed**
1. "The victim is inside a procedure with a retry loop. What does the CATCH print, and what happens on the next attempt?"
2. "By the time A retries, has B finished? So does the second attempt block or deadlock?"
3. "B was never a victim. What is the last line of B's batch?"

**Part 1, the final state**
1. "Two transactions both assigned both orders. Which one committed last?"
2. "Did B's CutLog update commit or roll back in run 1?"

**Part 2**
1. "Now the priorities differ. Which is checked first, priority or cost?"
2. "HIGH maps to 5, NORMAL to 0. The lowest number loses. Who is that?"
3. "B has no TRY CATCH. What reaches B's client, and what happens to the rest of B's batch, including the five thousand updated rows?"

**Part 3**
1. "In a CATCH block after a deadlock, is the transaction already gone, or still counted?"
2. "Think of the three values XACT_STATE can return: 1, 0, minus 1. Which one means open but uncommittable?"
3. "For the why: after a rollback, is the first UPDATE still in place? And could the data have changed while the victim was waiting?"

**Part 4**
1. "There is a session that is always running on every SQL Server instance. What is its name?"
2. "The system_health session writes to two targets. One is in memory, the other is a file. Which function reads xel files?"
3. "In the graph, each process element has a priority and a size of log written. What are the attribute names?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "B is the victim in run 1 because it started later" | Thinks arrival order decides | "The engine does not look at who arrived first. What does it compare when priorities are equal?" |
| "A is the victim in run 1 because it waited longer" | Thinks wait time decides | "Wait time is not the rule either. Think about what each transaction has written to the log." |
| "In run 1 the final crew is CREW-B because B committed" | Forgets that A retries and commits afterwards | "B committed, yes. But what did A do after its rollback, and when?" |
| "In run 2 B is still fine, it just retries" | Forgets B has no retry code | "Look at B's batch again. Is there a TRY CATCH or a loop anywhere?" |
| "HIGH is 10" | Confuses the range with the keyword values | "The range is minus ten to ten. The keywords map to three specific numbers inside it. Which?" |
| "XACT_STATE is 0 because the engine already rolled the victim back" | Thinks the rollback completes before the CATCH runs | "Inside TRY CATCH the transaction is doomed, not closed. Which XACT_STATE value describes that?" |
| "You only need to retry the UPDATE that got the 1205" | Does not realise the rollback undid the whole transaction | "After ROLLBACK, is the first UPDATE still applied? What must the retry include?" |
| "You must enable trace flag 1222 to see the graph" | Old habit from before Extended Events | "That was the SQL Server 2005 way. Which session records xml_deadlock_report on its own today?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the victim rule as two steps:

- **Step 1, priority.** SET DEADLOCK_PRIORITY takes LOW, NORMAL, HIGH or an integer from minus ten to ten. LOW is minus 5, NORMAL is 0 and the default, HIGH is 5. The session with the lowest value is the victim. Run 2 shows it: A at 5 wins even though B has written two thousand times more log.
- **Step 2, cost.** With equal priorities the engine kills the transaction that is cheaper to roll back, measured as log bytes written so far. That is the logused attribute in the deadlock graph: 300 for A, 600304 for B. Run 1 shows it: A loses despite arriving first. Priority changes who loses; it never removes the deadlock.

Then the retry pattern:

- Error 1205 is transient and its text says "Rerun the transaction". Outside TRY CATCH it aborts the batch and rolls back. Inside TRY CATCH the CATCH block runs with the transaction still counted, @@TRANCOUNT 1, but doomed, XACT_STATE minus 1. Only ROLLBACK is allowed; a COMMIT would fail. So the pattern is: test ERROR_NUMBER equals 1205, roll back if XACT_STATE is not zero, wait briefly, and loop again from BEGIN TRAN, with a maximum number of attempts, and THROW anything else.
- Retry the whole unit of work, never a single statement. The rollback removed every change of the victim, and the other session may have committed in between; the retry in run 1 silently overwrote CREW-B, which is why business logic should re-read state inside the retried transaction if that matters.

Then reading the graph:

- system_health captures xml_deadlock_report automatically, in a ring buffer and in system_health underscore star dot xel files. sys dot fn underscore xe underscore file underscore target underscore read underscore file reads the files; sys dot dm underscore xe underscore session underscore targets exposes the ring buffer. The XML has a victim-list, a process-list with spid, priority, logused, waitresource, lockMode, isolationlevel and the input buffer, and a resource-list of keylock elements with owner and waiter modes. Timestamps are UTC. A dedicated session with an event_file target keeps history longer; its file is flushed after MAX_DISPATCH_LATENCY, thirty seconds by default, so a report may appear in the ring buffer before it appears in the file.

Memory hook: "Lowest priority dies; on a tie the lightest log dies. Catch 1205, roll back the doomed transaction, and rerun everything from BEGIN TRAN. The graph is already in system_health."

## 9. Follow-up oral questions (optional)

1. "What single code change would remove this deadlock instead of choosing a different victim?" (Make both workflows update the orders in the same order, ascending OrderId, and keep the transaction short; then B simply blocks until A commits.)
2. "If both sessions ran at DEADLOCK_PRIORITY LOW, who would be the victim in run 1?" (Still A: equal priorities, so the cheaper rollback, 300 bytes of log against 600304.)
3. "What does the procedure do if a third attempt also deadlocks?" (The CATCH rethrows error 1205 with THROW because attempt is not less than maxAttempts; the caller sees the deadlock error and the final PRINT and RETURN 1 are never reached.)

## 10. References

- SET DEADLOCK_PRIORITY, values and victim selection: https://learn.microsoft.com/en-us/sql/t-sql/statements/set-deadlock-priority-transact-sql
- Deadlocks guide, including the deadlock graph and system_health: https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-deadlocks-guide
- Use the system_health session: https://learn.microsoft.com/en-us/sql/relational-databases/extended-events/use-the-system-health-session
- sys.fn_xe_file_target_read_file: https://learn.microsoft.com/en-us/sql/relational-databases/system-functions/sys-fn-xe-file-target-read-file-transact-sql
- CREATE EVENT SESSION: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-event-session-transact-sql
- TRY...CATCH, including uncommittable transactions: https://learn.microsoft.com/en-us/sql/t-sql/language-elements/try-catch-transact-sql
- XACT_STATE: https://learn.microsoft.com/en-us/sql/t-sql/functions/xact-state-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
