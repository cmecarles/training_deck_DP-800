# Instructor-Examiner guide — Table-Valued Functions 1

Companion to [table_valued_functions_1.md](table_valued_functions_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a nine-statement walkthrough: three function definitions, F1 to F3, then six queries, Q1 to Q6, then one final query. Take them in order. For the queries that return rows, ask for the rows one nurse at a time if the learner prefers. Note that F2 fails, so the function AllShifts never exists; no later query uses it.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Implement programmability objects.
- Task bullet: Create table-valued functions.
- What is tested: the difference between an inline and a multi-statement table-valued function, the ORDER BY rule inherited from views, CROSS APPLY versus OUTER APPLY versus JOIN for correlated calls, how the optimizer estimates each type, and which type is updatable.

## 2. Scenario to read aloud

**Piece 1, the story.** "ShiftPlanner is the rostering database of a hospital. Nurses belong to a ward and work shifts of eight or twelve hours. The planners want reusable, parameterized views of the data, and they write table-valued functions."

**Piece 2, the tables.** "Two tables in a schema called Roster. Roster dot Nurse has NurseID, integer, primary key; Name, text up to thirty; and Ward, a three-character code. Roster dot Shift has ShiftID, integer, primary key; NurseID, integer, a foreign key to Nurse; ShiftDay, a date; and Hours, a TINYINT."

**Piece 3, the data.** "Four nurses. Nurse 1, Ines, ward ICU. Nurse 2, Joel, ICU. Nurse 3, Kara, ER. Nurse 4, Liam, ER. Six shifts. Shift 10, nurse 1, September first 2026, twelve hours. Shift 11, nurse 1, September second, eight hours. Shift 12, nurse 1, September fourth, twelve hours. Shift 13, nurse 2, September first, eight hours. Shift 14, nurse 3, September third, twelve hours. Shift 15, nurse 3, September fifth, twelve hours. So Ines has three shifts, Joel one, Kara two, and Liam none."

**Piece 4, functions F1 and F2.** "F1 creates Roster dot LastShifts with two parameters, at NurseID and at N. It RETURNS TABLE, and the body is a single RETURN of a SELECT: TOP open paren at N close paren, ShiftID, ShiftDay, Hours from Shift where NurseID equals at NurseID, ORDER BY ShiftDay descending. F2 creates Roster dot AllShifts with one parameter, at NurseID. Also RETURNS TABLE, single RETURN of a SELECT: ShiftID, ShiftDay, Hours from Shift where NurseID equals at NurseID, ORDER BY ShiftDay descending. No TOP this time."

**Piece 5, function F3.** "F3 creates Roster dot HoursSummary with one parameter, at Ward. It RETURNS at t TABLE with four columns: NurseID, Name, TotalHours integer, and Band, one character. The body is BEGIN, END. First, an INSERT into at t of NurseID, Name and ISNULL of SUM of Hours, else zero, from Nurse LEFT JOIN Shift on NurseID, where Ward equals at Ward, grouped by NurseID and Name. Second, an UPDATE of at t setting Band to H when TotalHours is at least twenty-four, M when TotalHours is greater than zero, else L. Then a bare RETURN."

**Piece 6, queries Q1 to Q3.** "Q1 selects Name, ShiftDay and Hours from Nurse as n, CROSS APPLY Roster dot LastShifts with n dot NurseID and two, alias ls, ordered by Name then ShiftDay. Q2 is the same but with OUTER APPLY and the number one, ordered by Name. Q3 selects Name and ShiftDay from Nurse as n, INNER JOIN Roster dot LastShifts with n dot NurseID and two, alias ls, ON one equals one."

**Piece 7, queries Q4 to Q6 and the final query.** "Q4 selects NurseID, Name, TotalHours and Band from Roster dot HoursSummary with ER, ordered by NurseID. Q5 is an UPDATE of Roster dot LastShifts with one and three, setting Hours to nine where ShiftID is eleven. Q6 is an UPDATE of Roster dot HoursSummary with ICU, setting Band to X. Finally, a query selects ShiftID and Hours from Shift where NurseID is one, ordered by ShiftID."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE ShiftPlanner;
GO
USE ShiftPlanner;
GO
CREATE SCHEMA Roster;
GO
CREATE TABLE Roster.Nurse
(
    NurseID INT          NOT NULL PRIMARY KEY,
    Name    NVARCHAR(30) NOT NULL,
    Ward    CHAR(3)      NOT NULL
);
CREATE TABLE Roster.Shift
(
    ShiftID  INT     NOT NULL PRIMARY KEY,
    NurseID  INT     NOT NULL REFERENCES Roster.Nurse (NurseID),
    ShiftDay DATE    NOT NULL,
    Hours    TINYINT NOT NULL
);
INSERT INTO Roster.Nurse (NurseID, Name, Ward)
VALUES (1, N'Ines', 'ICU'), (2, N'Joel', 'ICU'), (3, N'Kara', 'ER'), (4, N'Liam', 'ER');
INSERT INTO Roster.Shift (ShiftID, NurseID, ShiftDay, Hours) VALUES
  (10, 1, '2026-09-01', 12), (11, 1, '2026-09-02', 8), (12, 1, '2026-09-04', 12),
  (13, 2, '2026-09-01', 8),
  (14, 3, '2026-09-03', 12), (15, 3, '2026-09-05', 12);
GO

-- F1
CREATE FUNCTION Roster.LastShifts (@NurseID INT, @N INT)
RETURNS TABLE
AS
RETURN
    SELECT TOP (@N) ShiftID, ShiftDay, Hours
    FROM Roster.Shift
    WHERE NurseID = @NurseID
    ORDER BY ShiftDay DESC;

-- F2
CREATE FUNCTION Roster.AllShifts (@NurseID INT)
RETURNS TABLE
AS
RETURN
    SELECT ShiftID, ShiftDay, Hours
    FROM Roster.Shift
    WHERE NurseID = @NurseID
    ORDER BY ShiftDay DESC;

-- F3
CREATE FUNCTION Roster.HoursSummary (@Ward CHAR(3))
RETURNS @t TABLE (NurseID INT, Name NVARCHAR(30), TotalHours INT, Band CHAR(1))
AS
BEGIN
    INSERT INTO @t (NurseID, Name, TotalHours)
    SELECT n.NurseID, n.Name, ISNULL(SUM(s.Hours), 0)
    FROM Roster.Nurse AS n
    LEFT JOIN Roster.Shift AS s ON s.NurseID = n.NurseID
    WHERE n.Ward = @Ward
    GROUP BY n.NurseID, n.Name;

    UPDATE @t SET Band = CASE WHEN TotalHours >= 24 THEN 'H' WHEN TotalHours > 0 THEN 'M' ELSE 'L' END;
    RETURN;
END;

-- Q1
SELECT n.Name, ls.ShiftDay, ls.Hours
FROM Roster.Nurse AS n
CROSS APPLY Roster.LastShifts(n.NurseID, 2) AS ls
ORDER BY n.Name, ls.ShiftDay;

-- Q2
SELECT n.Name, ls.ShiftDay, ls.Hours
FROM Roster.Nurse AS n
OUTER APPLY Roster.LastShifts(n.NurseID, 1) AS ls
ORDER BY n.Name;

-- Q3
SELECT n.Name, ls.ShiftDay
FROM Roster.Nurse AS n
INNER JOIN Roster.LastShifts(n.NurseID, 2) AS ls ON 1 = 1;

-- Q4
SELECT NurseID, Name, TotalHours, Band FROM Roster.HoursSummary('ER') ORDER BY NurseID;

-- Q5
UPDATE Roster.LastShifts(1, 3) SET Hours = 9 WHERE ShiftID = 11;

-- Q6
UPDATE Roster.HoursSummary('ICU') SET Band = 'X';

-- Final
SELECT ShiftID, Hours FROM Roster.Shift WHERE NurseID = 1 ORDER BY ShiftID;
```

## 4. The question (ask exactly this)

"For F1, F2 and F3, and then for Q1 to Q6, tell me whether each succeeds or raises an error, and give the exact result set of every query that returns one. One at a time, starting with F1."

After all nine: "Now tell me the result of the final query: ShiftID and Hours for nurse 1, ordered by ShiftID."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

| Stmt | Outcome | Detail |
|---|---|---|
| F1 | Succeeds | Inline TVF, type IF |
| F2 | Fails, Msg 1033 | The ORDER BY clause is invalid in views, inline functions, derived tables, subqueries, and common table expressions, unless TOP, OFFSET or FOR XML is also specified |
| F3 | Succeeds | Multi-statement TVF, type TF |
| Q1 | Succeeds | 5 rows, below |
| Q2 | Succeeds | 4 rows, below |
| Q3 | Fails, Msg 4104 | The multi-part identifier "n.NurseID" could not be bound |
| Q4 | Succeeds | 2 rows, below |
| Q5 | Succeeds | 1 row updated in Roster dot Shift through the inline TVF |
| Q6 | Fails, Msg 270 | Object 'Roster.HoursSummary' cannot be modified |

Q1, CROSS APPLY, two most recent shifts per nurse, Liam dropped:

| Name | ShiftDay | Hours |
|---|---|---|
| Ines | 2026-09-02 | 8 |
| Ines | 2026-09-04 | 12 |
| Joel | 2026-09-01 | 8 |
| Kara | 2026-09-03 | 12 |
| Kara | 2026-09-05 | 12 |

Q2, OUTER APPLY, latest shift per nurse, Liam kept with NULLs:

| Name | ShiftDay | Hours |
|---|---|---|
| Ines | 2026-09-04 | 12 |
| Joel | 2026-09-01 | 8 |
| Kara | 2026-09-05 | 12 |
| Liam | NULL | NULL |

Q4:

| NurseID | Name | TotalHours | Band |
|---|---|---|---|
| 3 | Kara | 24 | H |
| 4 | Liam | 0 | L |

Final query:

| ShiftID | Hours |
|---|---|
| 10 | 12 |
| 11 | 9 |
| 12 | 12 |

## 6. Hint ladder (one hint per attempt, in order)

**F1**
1. "RETURNS TABLE with a single RETURN SELECT. Which kind of table-valued function is that, and does it have an ORDER BY problem?"
2. "ORDER BY inside an inline function follows the view rule. Is there a TOP in F1?"

**F2**
1. "Compare F2 with F1. What is missing from the SELECT?"
2. "An inline TVF is expanded like a view. What does a view require before it accepts ORDER BY?"
3. "Without TOP, OFFSET or FOR XML, ORDER BY in a view body is a compile error. Same error number as for views."

**F3**
1. "RETURNS at t TABLE with a column list, BEGIN, END, INSERT, UPDATE, bare RETURN. Is that a valid shape for a function?"
2. "That is the multi-statement form. It is allowed to run procedural statements against the table variable."

**Q1**
1. "CROSS APPLY runs the function once per nurse. What happens to a nurse whose function call returns no rows?"
2. "Ines has three shifts. TOP two ordered by ShiftDay descending picks which two?"
3. "Liam has no shifts. Does he appear?"

**Q2**
1. "Now OUTER APPLY, and only one shift per nurse. What is the difference from CROSS APPLY for a nurse with no shifts?"
2. "Which shift is the latest for each of Ines, Joel and Kara? And what does Liam's row look like?"

**Q3**
1. "The function argument is n dot NurseID, a column of the left table. Can the right side of a JOIN reference a column of the left side?"
2. "Only APPLY allows the right operand to see the left row. What kind of error is it when a name cannot be resolved: runtime or compile time?"

**Q4**
1. "Ward ER has Kara and Liam. Sum Kara's hours. What about Liam, who has no shifts, under a LEFT JOIN with ISNULL?"
2. "Apply the band rule: twenty-four or more is H, above zero is M, otherwise L."

**Q5**
1. "An inline TVF is expanded like a view. Are views updatable when they select from one table with direct column mapping?"
2. "If the update goes through, which base table changes, and which row?"

**Q6**
1. "What does a multi-statement TVF return: a base table or a table variable that exists only during the call?"
2. "Can DML target something that vanishes after the call? What error says an object cannot be modified?"

**Final query**
1. "Which statement changed anything in Roster dot Shift? Only one did."
2. "Shift 11 belonged to nurse 1 with eight hours. What is it now?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "F1 fails, ORDER BY is not allowed in functions" | Forgets the TOP exception | "The rule has exceptions. Read the SELECT in F1 once more. What is right after SELECT?" |
| "F2 succeeds, functions are not views" | Does not know inline TVFs inherit the view rules | "An inline TVF is expanded into the calling query exactly like a view. What does that imply for ORDER BY?" |
| "F3 fails because a function cannot run UPDATE" | Confuses the side-effect rule with table variable DML | "The side-effect rule forbids changing base tables. Is at t a base table?" |
| "Q1 has six rows, including Liam" | Treats CROSS APPLY like OUTER APPLY | "CROSS APPLY behaves like an inner join. What happens to a left row with an empty right side?" |
| "Q1 shows Ines on 09-01 and 09-02" | Applies TOP ascending | "The ORDER BY inside the function is ShiftDay descending. Which two dates come first?" |
| "Q2 has three rows" | Treats OUTER APPLY like CROSS APPLY | "OUTER APPLY behaves like a left join. What about Liam?" |
| "Q3 succeeds because ON one equals one is always true" | Thinks the problem is the join predicate | "The problem is not the ON clause. Look at the function argument. Can the right side of a JOIN see n dot NurseID?" |
| "Q4 has only Kara" | Forgets the LEFT JOIN and ISNULL | "The join is a LEFT JOIN from Nurse. Does Liam get a row, and what total?" |
| "Q5 fails, functions are read-only" | Assumes no TVF is updatable | "One of the two kinds of TVF is expanded like a view. Are simple views updatable?" |
| "Q6 succeeds, it updates the table variable" | Thinks the table variable is reachable from outside | "The table variable exists only inside the call. What is left to modify once the function has returned?" |
| "Final query shows 11 with 8 hours" | Thought Q5 failed | "Reconsider Q5. If it succeeded, which base table did it change?" |

## 8. Teaching notes (after the answer is complete or revealed)

Start with the two kinds of table-valued function:

- **Inline TVF, F1.** RETURNS TABLE and a body that is a single RETURN SELECT. The column list is inferred from the SELECT. It is expanded into the calling query exactly like a view, so the optimizer sees the base tables and uses their statistics. The actual plan for Q1 touches Roster dot Shift directly with a TopN Sort per nurse and contains no function operator. It inherits the view rules: ORDER BY only with TOP, OFFSET or FOR XML, so F2 fails with 1033. Even with TOP, the ORDER BY inside only chooses which rows to return; it does not guarantee the order of the outer result, which is why Q1 and Q2 add their own ORDER BY. Wrapping an inline body in BEGIN END is also invalid, error 178.
- **Multi-statement TVF, F3.** RETURNS at t TABLE with a column list, procedural logic that fills the variable, and a bare RETURN. It is opaque to the optimizer: the plan contains a Table-valued function operator that materializes at t. Historically that operator carried a fixed guess of one hundred rows. Since SQL Server 2017, compatibility level 140 and higher, interleaved execution pauses optimization, runs the function, and re-optimizes with the real count. On this build the plan for a join on HoursSummary shows IsInterleavedExecuted equals 1 with EstimateRows 2; with the hint DISABLE underscore INTERLEAVED underscore EXECUTION underscore TVF it estimates 100. Prefer inline whenever one SELECT will do; use multi-statement only for real procedural logic.

Then APPLY versus JOIN:

- A TVF whose argument comes from another table is a correlated table expression. It must sit on the right side of CROSS APPLY, which drops left rows with no match like an inner join, or OUTER APPLY, which keeps them with NULLs like a left join. A regular JOIN cannot reference columns of the left input in its right operand, so Q3 fails to bind n dot NurseID with error 4104. That is a compile-time error, whatever the ON clause says. Q3 becomes valid by replacing INNER JOIN ON one equals one with CROSS APPLY.
- Q1 walkthrough: Ines has three shifts, TOP two ordered descending gives 09-04 and 09-02; Joel has one; Kara has two; Liam has none and disappears. Q2 asks for one shift per nurse and keeps Liam with NULLs. Q1 could also be written with ROW underscore NUMBER partitioned by NurseID ordered by ShiftDay descending and a filter rn less than or equal to two, but CROSS APPLY with TOP is the idiomatic TVF form.

Then Q4: for ward ER, Kara's LEFT JOIN sums twelve plus twelve, band H. Liam has no shifts and keeps ISNULL zero, band L. The band is computed by the second statement, something an inline TVF could only do with a CASE inside its SELECT.

Then updatability: because an inline TVF is expanded like a view, it is updatable under the view rules, one base table and direct column mapping. Q5 changes shift 11 from eight to nine hours in Roster dot Shift, which is why the final query shows 11 with 9. A multi-statement TVF returns a table variable that vanishes after the call, so DML against it is rejected with error 270.

Memory hook: "Inline is a view with parameters: same ORDER BY rule, same updatability, same statistics. Multi-statement is a black box. Correlated calls need APPLY."

## 9. Follow-up oral questions (optional)

1. "How do you fix F2 without changing its meaning?" (Remove the ORDER BY from the function and order in the calling query; or, if a row limit is intended, add TOP or OFFSET.)
2. "Which question would point you to an inline TVF as the answer?" (Any question asking which function type gives the best plan or lets the optimizer use base-table statistics.)
3. "What does interleaved execution change for a multi-statement TVF, and from which compatibility level?" (It replaces the fixed one-hundred-row estimate with the real row count; compatibility level 140 and higher.)

## 10. References

- CREATE FUNCTION, inline and multi-statement table-valued functions: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-function-transact-sql
- FROM clause, including CROSS APPLY and OUTER APPLY: https://learn.microsoft.com/en-us/sql/t-sql/queries/from-transact-sql
- User-defined functions overview: https://learn.microsoft.com/en-us/sql/relational-databases/user-defined-functions/user-defined-functions
- Intelligent query processing, interleaved execution for multi-statement TVFs: https://learn.microsoft.com/en-us/sql/relational-databases/performance/intelligent-query-processing-details
- ORDER BY clause restriction in views and inline functions: https://learn.microsoft.com/en-us/sql/t-sql/queries/select-order-by-clause-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
