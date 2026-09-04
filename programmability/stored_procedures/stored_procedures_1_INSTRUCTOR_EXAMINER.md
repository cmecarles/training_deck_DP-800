# Instructor-Examiner guide — Stored Procedures 1

Companion to [stored_procedures_1.md](stored_procedures_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is an eleven-statement walkthrough. The statements change the data, so the learner must keep track of which claims are closed as they go; if they lose track, tell them the current state of the table but not the outcome of the statement under discussion. Take the statements strictly in order, S1 to S11, then the final table.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Implement programmability objects.
- Task bullet: Create stored procedures.
- What is tested: OUTPUT parameters that need the keyword at the call site, parameter defaults and error 201, integer return codes and the NULL-to-zero rule, the sp underscore prefix resolving to system procedures, SET NOCOUNT, EXECUTE AS OWNER, WITH RECOMPILE and plan caching, multiple result sets and INSERT EXEC.

## 2. Scenario to read aloud

**Piece 1, the story.** "WarrantyDesk is the database behind a home-appliance warranty call centre. Claims are opened, and when the refund is small enough, they are closed in bulk by product. The team writes several stored procedures and then runs eleven statements to exercise parameter passing, return codes, execution context and result sets."

**Piece 2, the table and data.** "One table, in a schema called Claims, named Claim. Four columns. ClaimID, integer, primary key. Product, text up to forty. Status, VARCHAR ten. Amount, decimal with two decimals. Five rows. Claim 1, Kettle, OPEN, forty. Claim 2, Toaster, OPEN, twenty-five. Claim 3, Blender, CLOSED, ninety. Claim 4, Kettle, OPEN, forty-two. Claim 5, Kettle, OPEN, seventy-five."

**Piece 3, procedure CloseClaims.** "Claims dot CloseClaims takes three parameters. At Product, NVARCHAR forty, no default. At MaxAmount, decimal, with a default of fifty. And at Closed, an integer, declared OUTPUT. The body sets NOCOUNT ON, then updates Claim, setting Status to CLOSED where Product equals at Product, Status is OPEN, and Amount is less than or equal to at MaxAmount. It sets at Closed to at at ROWCOUNT. If at Closed is zero it returns one. Otherwise it returns zero."

**Piece 4, procedures OpenReport, Peek and PeekAsOwner.** "Claims dot OpenReport has no parameters and does not set NOCOUNT. It selects ClaimID from Claim where Status is OPEN, and then updates Claim setting Amount equal to Amount where Product is Kettle. So a no-change update. Claims dot Peek selects USER underscore NAME as Ctx and COUNT star as Claims from Claim. Claims dot PeekAsOwner is the same select, but the procedure is created WITH EXECUTE AS OWNER."

**Piece 5, procedure TwoSets and the user.** "Claims dot TwoSets sets NOCOUNT ON, then selects ClaimID from Claim where Status is OPEN. On the very next line it declares at c as an integer equal to at at ROWCOUNT. Then it selects at c as OpenCount. Then it selects ClaimID from Claim where Status is CLOSED. So three selects. Finally, a user called Intern is created without login, and EXECUTE on the whole Claims schema is granted to Intern. Everything that follows is run by dbo, each statement in its own batch."

**Piece 6, statements S1 to S3.** "S1 declares at n as minus one and at rc. It runs EXEC at rc equals Claims dot CloseClaims with three positional arguments: Toaster, fifty, and at n. No OUTPUT keyword at the call. Then it selects at rc and at n. S2 declares the same two variables and calls CloseClaims with named parameters: at Product equals Kettle, at Closed equals at n OUTPUT. At MaxAmount is not passed. Then it selects at rc and at n. S3 is the same as S2 but with at Product equals Blender."

**Piece 7, statements S4 to S6.** "S4 declares at n and calls CloseClaims with at MaxAmount equals ten and at Closed equals at n OUTPUT. No at Product. S5 creates a procedure Claims dot Ping whose only statement is RETURN NULL. Then, in a new batch, declares at rc as ninety-nine, runs EXEC at rc equals Claims dot Ping, and selects at rc. S6 creates a procedure dbo dot sp underscore help that selects the string WarrantyDesk as Src. Then, in a new batch, it runs EXEC sp underscore help, with no arguments."

**Piece 8, statements S7 to S9.** "S7 runs EXEC Claims dot OpenReport. S8 does EXECUTE AS USER equals Intern, then EXEC Claims dot Peek, then EXEC Claims dot PeekAsOwner, then REVERT. S9 creates a procedure Claims dot ByStatus with one parameter at Status, created WITH RECOMPILE, that selects COUNT star as N from Claim where Status equals at Status. Then, in a new batch, it runs ByStatus with OPEN, and then selects COUNT star as CachedEntries from sys dot dm underscore exec underscore procedure underscore stats where object underscore id is the object id of Claims dot ByStatus."

**Piece 9, statements S10 and S11.** "S10 runs EXEC Claims dot TwoSets. S11 creates a temporary table hash Ids with one integer column ClaimID, then runs INSERT INTO hash Ids EXEC Claims dot TwoSets, then selects COUNT star as Captured from hash Ids."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE WarrantyDesk;
GO
USE WarrantyDesk;
GO
CREATE SCHEMA Claims;
GO
CREATE TABLE Claims.Claim
(
    ClaimID INT          NOT NULL PRIMARY KEY,
    Product NVARCHAR(40) NOT NULL,
    Status  VARCHAR(10)  NOT NULL,
    Amount  DECIMAL(8,2) NOT NULL
);
INSERT INTO Claims.Claim (ClaimID, Product, Status, Amount) VALUES
  (1, N'Kettle',  'OPEN',   40.00),
  (2, N'Toaster', 'OPEN',   25.00),
  (3, N'Blender', 'CLOSED', 90.00),
  (4, N'Kettle',  'OPEN',   42.00),
  (5, N'Kettle',  'OPEN',   75.00);
GO
CREATE PROCEDURE Claims.CloseClaims
    @Product   NVARCHAR(40),
    @MaxAmount DECIMAL(8,2) = 50.00,
    @Closed    INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    UPDATE Claims.Claim SET Status = 'CLOSED'
    WHERE Product = @Product AND Status = 'OPEN' AND Amount <= @MaxAmount;
    SET @Closed = @@ROWCOUNT;
    IF @Closed = 0 RETURN 1;
    RETURN 0;
END;
GO
CREATE PROCEDURE Claims.OpenReport
AS
    SELECT ClaimID FROM Claims.Claim WHERE Status = 'OPEN';
    UPDATE Claims.Claim SET Amount = Amount WHERE Product = N'Kettle';
GO
CREATE PROCEDURE Claims.Peek
AS
    SELECT USER_NAME() AS Ctx, COUNT(*) AS Claims FROM Claims.Claim;
GO
CREATE PROCEDURE Claims.PeekAsOwner
WITH EXECUTE AS OWNER
AS
    SELECT USER_NAME() AS Ctx, COUNT(*) AS Claims FROM Claims.Claim;
GO
CREATE PROCEDURE Claims.TwoSets
AS
BEGIN
    SET NOCOUNT ON;
    SELECT ClaimID FROM Claims.Claim WHERE Status = 'OPEN';
    DECLARE @c INT = @@ROWCOUNT;
    SELECT @c AS OpenCount;
    SELECT ClaimID FROM Claims.Claim WHERE Status = 'CLOSED';
END;
GO
CREATE USER Intern WITHOUT LOGIN;
GRANT EXECUTE ON SCHEMA::Claims TO Intern;
GO

-- S1
DECLARE @n INT = -1, @rc INT;
EXEC @rc = Claims.CloseClaims N'Toaster', 50.00, @n;
SELECT @rc AS rc, @n AS n;

-- S2
DECLARE @n INT = -1, @rc INT;
EXEC @rc = Claims.CloseClaims @Product = N'Kettle', @Closed = @n OUTPUT;
SELECT @rc AS rc, @n AS n;

-- S3
DECLARE @n INT = -1, @rc INT;
EXEC @rc = Claims.CloseClaims @Product = N'Blender', @Closed = @n OUTPUT;
SELECT @rc AS rc, @n AS n;

-- S4
DECLARE @n INT;
EXEC Claims.CloseClaims @MaxAmount = 10.00, @Closed = @n OUTPUT;

-- S5
CREATE PROCEDURE Claims.Ping AS RETURN NULL;
GO
DECLARE @rc INT = 99;
EXEC @rc = Claims.Ping;
SELECT @rc AS rc;

-- S6
CREATE PROCEDURE dbo.sp_help AS SELECT 'WarrantyDesk' AS Src;
GO
EXEC sp_help;

-- S7
EXEC Claims.OpenReport;

-- S8
EXECUTE AS USER = 'Intern';
EXEC Claims.Peek;
EXEC Claims.PeekAsOwner;
REVERT;

-- S9
CREATE PROCEDURE Claims.ByStatus @Status VARCHAR(10)
WITH RECOMPILE
AS
    SELECT COUNT(*) AS N FROM Claims.Claim WHERE Status = @Status;
GO
EXEC Claims.ByStatus 'OPEN';
SELECT COUNT(*) AS CachedEntries FROM sys.dm_exec_procedure_stats
WHERE object_id = OBJECT_ID('Claims.ByStatus');

-- S10
EXEC Claims.TwoSets;

-- S11
CREATE TABLE #Ids (ClaimID INT);
INSERT INTO #Ids EXEC Claims.TwoSets;
SELECT COUNT(*) AS Captured FROM #Ids;
```

## 4. The question (ask exactly this)

"For each of the eleven statements, S1 to S11, tell me whether it succeeds or raises an error, and give the exact result sets and informational messages it produces. One at a time, starting with S1."

After all eleven: "Now give me the final contents of Claims dot Claim, ordered by ClaimID: for each row, ClaimID, Product, Status and Amount."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

| Stmt | Outcome | Result / message |
|---|---|---|
| S1 | Succeeds | rc = 0, n = -1. Claim 2 closed, but at n is unchanged because the call omitted OUTPUT |
| S2 | Succeeds | rc = 0, n = 2. Claims 1 and 4 closed; claim 5 at 75.00 exceeds the default 50.00 |
| S3 | Succeeds | rc = 1, n = 0. Claim 3 was already closed, nothing to do |
| S4 | Fails, Msg 201 | Procedure or function 'CloseClaims' expects parameter '@Product', which was not supplied |
| S5 | Succeeds | Informational message 282, severity 10: The 'Ping' procedure attempted to return a status of NULL, which is not allowed. A status of 0 will be returned instead. Then rc = 0 |
| S6 | Succeeds | The system sp underscore help runs and lists the database objects. The WarrantyDesk row is never returned, not even with dbo dot sp underscore help or a three-part name |
| S7 | Succeeds | Result set ClaimID = 5, then messages (1 rows affected) and (3 rows affected) |
| S8 | Succeeds | Peek: Ctx = Intern, Claims = 5. PeekAsOwner: Ctx = dbo, Claims = 5 |
| S9 | Succeeds | N = 1, then CachedEntries = 0 |
| S10 | Succeeds | Three result sets: ClaimID = 5; OpenCount = 1; ClaimID = 1, 2, 3, 4 |
| S11 | Succeeds | (6 rows affected), then Captured = 6 |

Final table:

| ClaimID | Product | Status | Amount |
|---|---|---|---|
| 1 | Kettle | CLOSED | 40.00 |
| 2 | Toaster | CLOSED | 25.00 |
| 3 | Blender | CLOSED | 90.00 |
| 4 | Kettle | CLOSED | 42.00 |
| 5 | Kettle | OPEN | 75.00 |

## 6. Hint ladder (one hint per attempt, in order)

**S1**
1. "The procedure part is easy: Toaster, twenty-five, under fifty. Count what it closes and what it returns. Then look very carefully at the call site."
2. "At Closed is declared OUTPUT inside the procedure. Is the word OUTPUT also present where at n is passed in S1?"
3. "Without OUTPUT at the call, the procedure still runs, but the value never travels back. What was at n before the call?"

**S2**
1. "At MaxAmount is not passed. What value does the procedure use for it?"
2. "Apply fifty to the three open kettles: forty, forty-two, seventy-five. How many pass? This time OUTPUT is present at the call."

**S3**
1. "What is claim 3's status before S3? Does the WHERE clause find any open Blender?"
2. "When at Closed is zero, which RETURN statement runs?"

**S4**
1. "Which parameter of CloseClaims has no default? Is it supplied?"
2. "A required parameter that is missing is not a runtime problem. It is an error before the body runs. What error number?"

**S5**
1. "What type must a RETURN value in a procedure be?"
2. "NULL is not an integer status. The engine does not fail; it substitutes something and warns. What does it substitute?"
3. "Remember at rc started at ninety-nine. Does it stay at ninety-nine?"

**S6**
1. "The procedure is called sp underscore help. Does that name already exist somewhere, outside this database?"
2. "Names starting with sp underscore are looked up in a special order. Where does the engine look first?"
3. "Because a system procedure with that name exists, is the user's dbo dot sp underscore help ever reached, even with a schema prefix?"

**S7**
1. "Which claims are still OPEN after S1 to S3?"
2. "OpenReport does not set NOCOUNT. What message does the client get after each statement in the procedure?"
3. "The UPDATE sets Amount equal to Amount for the kettles. Does a no-change update still count rows?"

**S8**
1. "Peek has no EXECUTE AS clause. Whose name does USER underscore NAME return when Intern runs it?"
2. "Intern has only EXECUTE on the schema, no SELECT on the table. Why can Peek still read the table? Think about who owns the procedure and the table."
3. "PeekAsOwner switches context. Who owns the procedure?"

**S9**
1. "The first part is a count of open claims. How many are open now?"
2. "WITH RECOMPILE affects what happens to the plan after execution. Is it cached?"
3. "If nothing is cached, what does dm underscore exec underscore procedure underscore stats have for this object?"

**S10**
1. "How many SELECT statements are in TwoSets? Each returns a result set."
2. "The at at ROWCOUNT is read on the very next statement after the first SELECT. Which rows did that SELECT return?"

**S11**
1. "INSERT EXEC captures result sets into the table. Does it capture only the first one, or all of them?"
2. "Add up the rows of all three result sets from S10."

**Final table**
1. "Three statements closed claims: S1, S2 and S3. Which claims did each close, if any?"
2. "Did S7's UPDATE change any Amount?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "S1 gives n = 1" | Thinks OUTPUT in the declaration is enough | "Read the call in S1 again. Is there anything after at n?" |
| "S1 fails because OUTPUT is missing" | Thinks a mismatch is an error | "It is not an error. The procedure runs. What does not happen is the write-back." |
| "S2 gives n = 3" | Ignores the default at MaxAmount of fifty | "Claim 5 is seventy-five. What limit applies when at MaxAmount is not passed?" |
| "S4 succeeds, at Product is NULL" | Thinks missing parameters become NULL | "Only parameters with a default can be omitted. Does at Product have one?" |
| "S5 gives rc = NULL" or "rc = 99" | Does not know the NULL-to-zero rule | "The engine refuses a NULL status but does not raise an error. It substitutes a value and prints a message. Which value?" |
| "S6 returns the row WarrantyDesk" | Expects the local procedure to win | "Where does the engine look first for a name that starts with sp underscore?" |
| "S6 returns WarrantyDesk if you say dbo dot sp underscore help" | Believes schema qualification overrides the lookup | "Even with the schema or database prefix, the system procedure is found first. That is exactly why the prefix is discouraged." |
| "S7 shows no rows-affected messages" | Assumes NOCOUNT is on everywhere | "Which procedures set NOCOUNT ON? Is OpenReport one of them?" |
| "S7's UPDATE affects zero rows because nothing changes" | Confuses rows matched with rows changed | "A no-change update still counts the rows it touched. How many kettles are there?" |
| "S8 Peek fails, Intern has no SELECT" | Forgets ownership chaining | "Who owns Claims dot Peek and who owns Claims dot Claim? What happens when they are the same?" |
| "S9 CachedEntries = 1" | Does not know WITH RECOMPILE prevents caching | "What does WITH RECOMPILE on a procedure do with the plan after each run?" |
| "S11 captures 1 row" | Thinks INSERT EXEC takes only the first result set | "INSERT EXEC captures every result set the procedure returns, as long as each matches the target columns." |

## 8. Teaching notes (after the answer is complete or revealed)

Group the eleven statements by rule:

- **Parameters, S1 to S4.** An OUTPUT parameter writes back only if the call also says OUTPUT. S1 omitted it: the procedure closed claim 2 and set its local at Closed to one, but the caller's at n stayed at minus one, silently. S2 used named parameters with OUTPUT, skipped at MaxAmount so the default fifty applied, and got two. S3 found nothing to close, so RETURN 1 surfaced as rc equals one. S4 omitted at Product, which has no default, so error 201 fires before the body runs. Named and positional arguments can be mixed only while the positional ones come first. The keyword DEFAULT can be passed explicitly.
- **Return codes, S5.** RETURN in a procedure accepts only an integer. A string raises error 245, a decimal is truncated, and NULL is replaced by zero with informational message 282. A procedure that reaches its end returns zero. By convention non-zero means failure, which is why EXEC at rc equals proc, then IF at rc not equal to zero, is the classic pattern. Data belongs in OUTPUT parameters or result sets, never in the return code.
- **The sp underscore prefix, S6.** A name beginning with sp underscore is looked up first among the system stored procedures, which physically live in the Resource database and are exposed through master. Because a system sp underscore help exists, the user's dbo dot sp underscore help is never reached, not even with a two-part or three-part name. Only when no system procedure has that name does resolution fall back to the current database. That is the documented reason to avoid the prefix: a future system procedure with the same name breaks the application silently, and even non-conflicting names cost an extra probe of master.
- **NOCOUNT, S7.** OpenReport does not set NOCOUNT, so the client receives a rows-affected message per statement: one for the SELECT, which finds only claim 5 still open, and three for the UPDATE, which touches the three kettles without changing values. SET NOCOUNT ON removes those messages and reduces network traffic, but does not affect at at ROWCOUNT.
- **Execution context, S8.** Under EXECUTE AS USER Intern, Peek runs as the caller, so USER underscore NAME returns Intern. It can still read the table thanks to ownership chaining: procedure and table share the owner dbo, so no SELECT grant is needed. PeekAsOwner switches the context to dbo for the duration of the call. Use EXECUTE AS OWNER when the body needs permissions that ownership chaining does not cover, such as dynamic SQL, TRUNCATE, DDL, or objects with a different owner. The options are CALLER, the default, OWNER, SELF, and a named user.
- **Plan caching, S9.** WITH RECOMPILE on the procedure means it is compiled on every execution and its plan is never cached, so dm underscore exec underscore procedure underscore stats has no entry. An ordinary procedure such as Peek would show one after one execution. Use it when parameter sniffing makes one cached plan harmful. Lighter alternatives are EXEC WITH RECOMPILE for a single call, or OPTION RECOMPILE on the one problematic statement.
- **Result sets, S10 and S11.** A procedure returns one result set per SELECT it executes, so TwoSets returns three. At at ROWCOUNT is read on the very next statement; a PRINT or SET in between would reset it. INSERT EXEC captures all the result sets into the target table, so the single-column hash Ids receives one plus one plus four, six rows. Every result set must match the target column list or the insert fails.

Memory hook: "OUTPUT on both sides. RETURN is an integer, NULL becomes zero. sp underscore belongs to the system. INSERT EXEC takes every result set."

## 9. Follow-up oral questions (optional)

1. "How would you call CloseClaims for Kettle with the default max amount stated explicitly, in positional form?" (EXEC Claims dot CloseClaims N Kettle, DEFAULT, at n OUTPUT.)
2. "What happens if a procedure does RETURN 2.7?" (The value is truncated to 2.)
3. "In S8, what would Peek return if Intern had been granted nothing but EXECUTE and the table were owned by a different user than the procedure?" (Ownership chaining would break, and the SELECT would fail with a permission error, unless SELECT is granted or the procedure uses EXECUTE AS OWNER.)

## 10. References

- CREATE PROCEDURE, including EXECUTE AS and WITH RECOMPILE: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-procedure-transact-sql
- EXECUTE, including OUTPUT and return codes: https://learn.microsoft.com/en-us/sql/t-sql/language-elements/execute-transact-sql
- Return data from a stored procedure: https://learn.microsoft.com/en-us/sql/relational-databases/stored-procedures/return-data-from-a-stored-procedure
- RETURN: https://learn.microsoft.com/en-us/sql/t-sql/language-elements/return-transact-sql
- SET NOCOUNT: https://learn.microsoft.com/en-us/sql/t-sql/statements/set-nocount-transact-sql
- EXECUTE AS clause: https://learn.microsoft.com/en-us/sql/t-sql/statements/execute-as-clause-transact-sql
- Ownership chains: https://learn.microsoft.com/en-us/sql/relational-databases/security/authentication-access/ownership-chains
- sys.dm_exec_procedure_stats: https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-procedure-stats-transact-sql
- INSERT, including INSERT ... EXEC: https://learn.microsoft.com/en-us/sql/t-sql/statements/insert-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
