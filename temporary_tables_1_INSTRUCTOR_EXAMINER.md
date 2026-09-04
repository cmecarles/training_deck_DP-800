# Instructor-Examiner guide — Temporary Tables 1

Companion to [temporary_tables_1.md](temporary_tables_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Read all four options before taking an answer. The options differ in three things: which batches fail, whether batch H works, and the RowsSeen numbers in the log. A good way by voice: walk batches A to I with the learner, then match to an option. Remember that a failing batch does not stop the session.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Design and implement database objects.
- Task bullet: Use temporary tables and table variables.
- What is tested: the scope and lifetime of local temp tables created in a procedure versus at session level, shadowing when a nested procedure creates a temp table with the same name, global temp tables surviving the creating procedure, and table variables being invisible outside their batch.

## 2. Scenario to read aloud

**Piece 1, the story.** "The payroll department runs its monthly batch in a database called PayrollHub. The batch procedures exchange intermediate data through temporary objects, and a developer wants to know exactly which of them can see what. A setup script runs first and completes without error. Then the same session runs nine batches, A to I. A batch that fails does not stop the session."

**Piece 2, the log table.** "One permanent table, in a schema called Pay, named RunLog. LogID, an integer IDENTITY, primary key. Source, text up to forty. RowsSeen, an integer. Every procedure writes one row here saying who it is and how many rows it counted."

**Piece 3, the first two procedures.** "Pay dot usp underscore LoadAdjustments creates a local temp table, hash Batch, with EmpID and Amount, inserts two rows, employees 7 and 8, and logs LoadAdjustments with COUNT star from hash Batch. Pay dot usp underscore Summarize creates nothing. It only logs Summarize with COUNT star from hash Batch."

**Piece 4, the other two procedures.** "Pay dot usp underscore Stage creates a global temp table, double hash Stage, with one column EmpID, and inserts four rows, 1 to 4. Pay dot usp underscore RunPayroll creates its own local temp table hash Batch, same two columns, inserts three rows, employees 1, 2 and 3. Then it executes usp underscore LoadAdjustments, then usp underscore Summarize, and finally logs RunPayroll with COUNT star from hash Batch."

**Piece 5, batches A to D.** "Batch A executes usp underscore RunPayroll. Batch B executes usp underscore Summarize on its own. Batch C, at session level, creates a local temp table hash Batch with the same two columns and inserts one row, employee 10, twelve fifty. Batch D executes usp underscore Summarize again."

**Piece 6, batches E to I.** "Batch E declares a table variable at Recent with one column EmpID and inserts one row. Batch F, a new batch, selects COUNT star as RecentRows from at Recent. Batch G executes usp underscore Stage. Batch H selects COUNT star as StagedRows from double hash Stage. Batch I selects LogID, Source and RowsSeen from Pay dot RunLog ordered by LogID."

**Piece 7, option a.** "Option a says batches B, F and H fail. Batch I returns four rows: LoadAdjustments 2, Summarize 3, RunPayroll 3, Summarize 1."

**Piece 8, option b.** "Option b says batches B and F fail. Batch H returns StagedRows 4. Batch I returns four rows: LoadAdjustments 5, Summarize 5, RunPayroll 5, Summarize 1."

**Piece 9, option c.** "Option c says batches B and F fail. Batch H returns StagedRows 4. Batch I returns four rows: LoadAdjustments 2, Summarize 3, RunPayroll 3, Summarize 1."

**Piece 10, option d.** "Option d says batches B, D and F fail. Batch H returns StagedRows 4. Batch I returns only three rows: LoadAdjustments 2, Summarize 3, RunPayroll 3."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE PayrollHub;
GO
USE PayrollHub;
GO
CREATE SCHEMA Pay;
GO
CREATE TABLE Pay.RunLog
(
    LogID    int IDENTITY(1,1) PRIMARY KEY,
    Source   nvarchar(40) NOT NULL,
    RowsSeen int          NOT NULL
);
GO
CREATE PROCEDURE Pay.usp_LoadAdjustments
AS
BEGIN
    CREATE TABLE #Batch (EmpID int NOT NULL, Amount decimal(9,2) NOT NULL);
    INSERT INTO #Batch VALUES (7, 50.00), (8, 75.00);
    INSERT INTO Pay.RunLog (Source, RowsSeen)
    SELECT N'LoadAdjustments', COUNT(*) FROM #Batch;
END;
GO
CREATE PROCEDURE Pay.usp_Summarize
AS
BEGIN
    INSERT INTO Pay.RunLog (Source, RowsSeen)
    SELECT N'Summarize', COUNT(*) FROM #Batch;
END;
GO
CREATE PROCEDURE Pay.usp_Stage
AS
BEGIN
    CREATE TABLE ##Stage (EmpID int NOT NULL);
    INSERT INTO ##Stage VALUES (1), (2), (3), (4);
END;
GO
CREATE PROCEDURE Pay.usp_RunPayroll
AS
BEGIN
    CREATE TABLE #Batch (EmpID int NOT NULL, Amount decimal(9,2) NOT NULL);
    INSERT INTO #Batch VALUES (1, 1000.00), (2, 1100.00), (3, 900.00);

    EXEC Pay.usp_LoadAdjustments;
    EXEC Pay.usp_Summarize;

    INSERT INTO Pay.RunLog (Source, RowsSeen)
    SELECT N'RunPayroll', COUNT(*) FROM #Batch;
END;
GO

-- Batch A
EXEC Pay.usp_RunPayroll;
GO
-- Batch B
EXEC Pay.usp_Summarize;
GO
-- Batch C
CREATE TABLE #Batch (EmpID int NOT NULL, Amount decimal(9,2) NOT NULL);
INSERT INTO #Batch VALUES (10, 12.50);
GO
-- Batch D
EXEC Pay.usp_Summarize;
GO
-- Batch E
DECLARE @Recent TABLE (EmpID int NOT NULL);
INSERT INTO @Recent VALUES (1);
GO
-- Batch F
SELECT COUNT(*) AS RecentRows FROM @Recent;
GO
-- Batch G
EXEC Pay.usp_Stage;
GO
-- Batch H
SELECT COUNT(*) AS StagedRows FROM ##Stage;
GO
-- Batch I
SELECT LogID, Source, RowsSeen
FROM Pay.RunLog
ORDER BY LogID;
GO
```

Options as shown to the learner:

```text
a. B, F, H fail.     Batch I: (1, LoadAdjustments, 2), (2, Summarize, 3), (3, RunPayroll, 3), (4, Summarize, 1)
b. B, F fail. H = 4. Batch I: (1, LoadAdjustments, 5), (2, Summarize, 5), (3, RunPayroll, 5), (4, Summarize, 1)
c. B, F fail. H = 4. Batch I: (1, LoadAdjustments, 2), (2, Summarize, 3), (3, RunPayroll, 3), (4, Summarize, 1)
d. B, D, F fail. H = 4. Batch I: (1, LoadAdjustments, 2), (2, Summarize, 3), (3, RunPayroll, 3)
```

## 4. The question (ask exactly this)

"Which batches raise an error, and what do batches H and I return? Option a: B, F and H fail, and the log shows 2, 3, 3, 1. Option b: B and F fail, H returns four, and the log shows 5, 5, 5, 1. Option c: B and F fail, H returns four, and the log shows 2, 3, 3, 1. Option d: B, D and F fail, H returns four, and the log shows only three rows, 2, 3, 3. Which one, a, b, c or d?"

If the learner prefers to reason first: "Let's walk the batches. Start with batch A: what does each of the three procedures count?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: c.**

Batch by batch:

| Batch | Outcome | Detail |
|---|---|---|
| A | Succeeds, logs 3 rows | RunPayroll creates hash Batch with 3 rows. LoadAdjustments creates its own hash Batch that shadows the caller's, inserts 2, logs LoadAdjustments 2, and its table is dropped on return. Summarize creates nothing, sees the caller's 3-row table, logs Summarize 3. RunPayroll logs RunPayroll 3 |
| B | Fails, Msg 208 | Invalid object name '#Batch'. RunPayroll's table was dropped when it returned. Runtime error thanks to deferred name resolution. Nothing logged |
| C | Succeeds | Session-level hash Batch with 1 row; survives across GO until session end |
| D | Succeeds, logs 1 row | Summarize sees the session's hash Batch, logs Summarize 1 |
| E | Succeeds | DECLARE and INSERT of at Recent share one batch |
| F | Fails, Msg 1087 | Must declare the table variable "@Recent". Compile-time error; the whole batch is rejected |
| G | Succeeds | Creates double hash Stage with 4 rows |
| H | Succeeds, StagedRows = 4 | A global temp table is not dropped when the creating procedure returns; it lives until the creating session ends and nothing references it |
| I | Succeeds | Four rows below |

Batch I:

| LogID | Source | RowsSeen |
|---|---|---|
| 1 | LoadAdjustments | 2 |
| 2 | Summarize | 3 |
| 3 | RunPayroll | 3 |
| 4 | Summarize | 1 |

Why each wrong option is wrong:

- **a** — Every log row is right, but it claims batch H fails by applying the local-temp-table lifetime rule to a global temp table. Global temp tables are dropped when the creating session ends, not when the creating procedure returns. The subtle distractor.
- **b** — Assumes one hash Batch per session, so LoadAdjustments' two rows land in RunPayroll's table and every count is five. In reality the nested CREATE TABLE hash Batch creates a distinct inner table that shadows the caller's. Counts are 2, 3, 3.
- **d** — Treats the session-level hash Batch from batch C as batch-scoped like a table variable, so it predicts batch D fails. A local temp table created outside any procedure persists for the session, across GO, and is visible to procedures the session calls. Batch D logs Summarize 1.

## 6. Hint ladder (one hint per attempt, in order)

1. "Three scoping models are in play: local temp tables, global temp tables, and table variables. For each one, who can see it, and when is it dropped? Start with the local temp table created inside a procedure."
2. "In batch A, LoadAdjustments runs CREATE TABLE hash Batch while its caller already has a hash Batch. Is that an error, or does it get its own table? If its own, what does it count?"
3. "Summarize creates no table. Which hash Batch does it find when RunPayroll calls it? That gives the second log row. And after RunPayroll returns, does its hash Batch still exist for batch B?"
4. "Batch C creates hash Batch at session level, outside any procedure. How long does that one live? Can a procedure called later in the session see it? That decides batch D."
5. "Batch E and F: a table variable lives in one batch. Is batch F the same batch as batch E? And what kind of error is an undeclared variable, compile-time or runtime?"
6. "Batch H: double hash Stage was created inside usp underscore Stage, which has returned. Does a global temp table die with the procedure, or with the session? That decides between the last two options."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "Batch A fails, you cannot create hash Batch twice" | Thinks the name is unique per session | "A nested procedure may create a temp table with the same name as its caller. What does it get?" |
| "b, the counts are five" | Believes both procedures share one hash Batch | "If LoadAdjustments creates its own table, where do its two rows go? And what happens to that table when it returns?" |
| "Batch B succeeds, hash Batch still exists" | Thinks a procedure's temp table outlives the procedure | "Who created that hash Batch, and when does a temp table created in a procedure get dropped?" |
| "d, batch D fails because batch C's table ended with GO" | Treats a session temp table like a table variable | "Batch C created the table at session level. Does GO end the session?" |
| "Batch F works, at Recent was declared in the session" | Thinks table variables are session-scoped | "A variable belongs to the batch that declares it. Is batch F the same batch as batch E?" |
| "a, double hash Stage is gone when usp underscore Stage returns" | Applies the local rule to a global table | "That rule is right for hash local tables. Is it the same for double hash global tables?" |
| "Batch H fails because another session did not reference it" | Misreads the global drop rule | "The table lives while the creating session is open. Is the creating session still open in batch H?" |
| "Batch B is a compile error like batch F" | Does not know deferred name resolution | "The procedure was created fine even though hash Batch did not exist. So when is the missing table detected?" |

## 8. Teaching notes (after the answer is complete or revealed)

Give the scope table first:

- **Local temp table created in a procedure.** Visible to that procedure and to every procedure nested below it. Dropped when the creating procedure returns. That is why batch A works and batch B fails with Msg 208.
- **Local temp table created at session level.** Visible to all later batches in the session and to procedures the session calls. Dropped at session end or by an explicit DROP. That is why batch D works and logs Summarize 1.
- **Global temp table.** Visible to all sessions. Dropped when the creating session ends and no other task references it. Not when the creating procedure returns. That is why batch H returns four.
- **Table variable.** Scoped to the single batch, procedure or function that declares it. Never visible to nested procedures. Gone at the end of the batch. That is why batch F fails with Msg 1087.

Then the two trap behaviours:

- **Shadowing.** A nested procedure may run CREATE TABLE hash X while its caller already has hash X. It gets its own inner table that shadows the caller's for the duration of the call, and that inner table is dropped on return. So LoadAdjustments counts two, Summarize, which creates nothing, counts the caller's three, and RunPayroll still counts three. Never five.
- **Compile-time versus runtime.** Referencing a table variable from another batch is a compile-time error, Msg 1087, that rejects the whole batch before any statement runs. It cannot be caught by TRY CATCH in the same batch. Referencing a vanished temp table inside a procedure is a runtime error, Msg 208, because deferred name resolution let the procedure be created without the table existing.

Then batch A step by step: RunPayroll makes hash Batch with three rows. LoadAdjustments makes its own hash Batch, inserts two, logs 2, returns, and its table disappears; the caller's three rows were never touched. Summarize finds the nearest enclosing hash Batch, the caller's, and logs 3. RunPayroll logs 3.

Memory hook: "Visibility flows downward only. Lifetime belongs to the creator: hash dies with the proc, double hash dies with the session, at variable dies with the batch."

## 9. Follow-up oral questions (optional)

1. "How would you make usp underscore Summarize safe to call on its own?" (Check OBJECT underscore ID of tempdb dot dot hash Batch and create or skip, or pass the data as a table-valued parameter instead of relying on an ambient temp table.)
2. "Could a different session read double hash Stage while the first session is still open?" (Yes. Global temp tables are visible to all sessions.)
3. "Why did CREATE PROCEDURE usp underscore Summarize succeed although hash Batch did not exist at creation time?" (Deferred name resolution: table references in a procedure are resolved at execution, not at creation.)

## 10. References

- CREATE TABLE, temporary tables section: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-table-transact-sql
- Table variables, DECLARE at local underscore variable: https://learn.microsoft.com/en-us/sql/t-sql/data-types/table-transact-sql
- Deferred name resolution and compilation: https://learn.microsoft.com/en-us/sql/relational-databases/stored-procedures/create-a-stored-procedure
- tempdb database: https://learn.microsoft.com/en-us/sql/relational-databases/databases/tempdb-database
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
