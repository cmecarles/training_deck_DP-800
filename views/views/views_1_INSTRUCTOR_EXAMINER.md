# Instructor-Examiner guide — Views 1

Companion to [views_1.md](views_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

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

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Implement programmability objects.
- Task bullet: Create views.
- What is tested: which modifications through a view succeed, what `WITH CHECK OPTION` blocks, what `SCHEMABINDING` blocks, and why `ORDER BY` is rejected inside a view.

## 2. Scenario to read aloud

**Piece 1, the story.** "A university keeps its course catalog in a SQL Server database called CampusReg. The registrar's office lets academic advisors see and change only the Computer Science part of the catalog, and only through a view. So the advisors never touch the table directly; every insert, update or delete they run goes through that view."

**Piece 2, the table.** "There is one table, in a schema called Acad, named Course. It has five columns. CourseID, an integer, the primary key. Dept, a four-character code such as CS or MATH. Title, a text column of up to sixty characters. Credits, a small whole number. And Seats, a small integer that has a default value of thirty."

**Piece 3, the data.** "Four rows are inserted. Course 101, department CS, title Programming One, three credits, one hundred twenty seats. Course 102, CS, Databases, four credits, eighty seats. Course 205, MATH, Linear Algebra, three credits, sixty seats. And course 301, CS, Compilers, six credits, forty seats."

**Piece 4, the first view.** "Now the first view, called Acad dot CoreCatalog. It is created WITH SCHEMABINDING. It selects four columns from Course: CourseID, Dept, Title and Credits. It does not expose Seats. Its WHERE clause keeps only rows where Dept equals CS and Credits is less than or equal to four. And the view definition ends WITH CHECK OPTION."

**Piece 5, the second view.** "The second view is Acad dot DeptStats. It groups Course by Dept and returns Dept and a count of rows called CourseCount. Nothing else."

**Piece 6, the eight statements.** "After that, eight statements run in order, each in its own batch, in one session. I will read them one at a time.

- S1 inserts through CoreCatalog: course 110, CS, Operating Systems, four credits.
- S2 inserts through CoreCatalog: course 210, MATH, Calculus Two, four credits.
- S3 updates CoreCatalog: sets Credits to five where CourseID is 102.
- S4 updates CoreCatalog: sets Title to Advanced Compilers where CourseID is 301.
- S5 updates CoreCatalog: sets Credits to two where CourseID is 101.
- S6 alters the base table: ALTER TABLE Acad dot Course, ALTER COLUMN Title, to NVARCHAR one hundred twenty, NOT NULL.
- S7 creates a third view, RankedCatalog, that selects CourseID, Title and Credits from Course, with an ORDER BY Credits descending inside the view definition.
- S8 deletes from DeptStats where Dept equals MATH."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE CampusReg;
GO
USE CampusReg;
GO
CREATE SCHEMA Acad;
GO
CREATE TABLE Acad.Course
(
    CourseID INT          NOT NULL PRIMARY KEY,
    Dept     CHAR(4)      NOT NULL,
    Title    NVARCHAR(60) NOT NULL,
    Credits  TINYINT      NOT NULL,
    Seats    SMALLINT     NOT NULL CONSTRAINT DF_Course_Seats DEFAULT (30)
);
GO
INSERT INTO Acad.Course (CourseID, Dept, Title, Credits, Seats) VALUES
  (101, 'CS',   N'Programming I',  3, 120),
  (102, 'CS',   N'Databases',      4,  80),
  (205, 'MATH', N'Linear Algebra', 3,  60),
  (301, 'CS',   N'Compilers',      6,  40);
GO
CREATE VIEW Acad.CoreCatalog
WITH SCHEMABINDING
AS
SELECT CourseID, Dept, Title, Credits
FROM Acad.Course
WHERE Dept = 'CS' AND Credits <= 4
WITH CHECK OPTION;
GO
CREATE VIEW Acad.DeptStats
AS
SELECT Dept, COUNT(*) AS CourseCount
FROM Acad.Course
GROUP BY Dept;
GO
-- S1
INSERT INTO Acad.CoreCatalog (CourseID, Dept, Title, Credits) VALUES (110, 'CS', N'Operating Systems', 4);
-- S2
INSERT INTO Acad.CoreCatalog (CourseID, Dept, Title, Credits) VALUES (210, 'MATH', N'Calculus II', 4);
-- S3
UPDATE Acad.CoreCatalog SET Credits = 5 WHERE CourseID = 102;
-- S4
UPDATE Acad.CoreCatalog SET Title = N'Advanced Compilers' WHERE CourseID = 301;
-- S5
UPDATE Acad.CoreCatalog SET Credits = 2 WHERE CourseID = 101;
-- S6
ALTER TABLE Acad.Course ALTER COLUMN Title NVARCHAR(120) NOT NULL;
-- S7
CREATE VIEW Acad.RankedCatalog AS SELECT CourseID, Title, Credits FROM Acad.Course ORDER BY Credits DESC;
-- S8
DELETE FROM Acad.DeptStats WHERE Dept = 'MATH';
```

## 4. The question (ask exactly this)

"For each of the eight statements, S1 to S8, tell me whether it succeeds or raises an error. For the ones that succeed, tell me how many rows are affected. Let's go one at a time, starting with S1."

After all eight: "Now tell me the final contents of the Course table, ordered by CourseID: for each row, the CourseID, Dept, Title, Credits and Seats."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

| Stmt | Outcome | Detail |
|---|---|---|
| S1 | Succeeds, 1 row | Row 110 inserted; Seats filled by the default, 30 |
| S2 | Fails, error 550 | Row would not be visible through the view (Dept is MATH), blocked by WITH CHECK OPTION |
| S3 | Fails, error 550 | Credits 5 would push row 102 outside the view's predicate |
| S4 | Succeeds, 0 rows | Course 301 has 6 credits, so the view cannot see it; nothing matches, no error |
| S5 | Succeeds, 1 row | Credits 2 still satisfies the predicate |
| S6 | Fails, errors 5074 and 4922 | SCHEMABINDING on CoreCatalog blocks ALTER COLUMN on Title, even a widening |
| S7 | Fails, error 1033 | ORDER BY is invalid in a view unless TOP, OFFSET or FOR XML is present |
| S8 | Fails, error 4403 | A view with GROUP BY and an aggregate is not updatable |

Final table:

| CourseID | Dept | Title | Credits | Seats |
|---|---|---|---|---|
| 101 | CS | Programming I | 2 | 120 |
| 102 | CS | Databases | 4 | 80 |
| 110 | CS | Operating Systems | 4 | 30 |
| 205 | MATH | Linear Algebra | 3 | 60 |
| 301 | CS | Compilers | 6 | 40 |

Only S1 and S5 changed data.

## 6. Hint ladder (one hint per attempt, in order)

**S1**
1. "The view does not show the Seats column, and Seats is NOT NULL. Does the table have a way to fill it anyway?"
2. "Remember the DEFAULT constraint on Seats. What value does the engine use when a column is omitted and it has a default?"
3. "Check the row against the view's WHERE clause: CS, four credits. Does it pass?"

**S2**
1. "Look at the department of the new row, and then look again at the view's WHERE clause."
2. "The view definition ends with a clause that only matters for modifications. Which clause is that, and what does it do?"
3. "WITH CHECK OPTION requires every inserted or updated row to remain visible through the view. Is a MATH row visible through CoreCatalog?"

**S3**
1. "Row 102 is visible before the update. What about after Credits becomes five?"
2. "Same clause as in S2. It checks the row after the change, not before."

**S4**
1. "Before deciding, ask: can the view even see course 301? Check its credits."
2. "When an UPDATE through a view targets a row the view filters out, is that an error, or just no rows affected?"
3. "It is not an error. Think about what rowcount an UPDATE returns when its WHERE clause matches nothing."

**S5**
1. "Apply the view's predicate to the row after the change: CS with two credits."
2. "Two is less than or equal to four. So does CHECK OPTION object?"

**S6**
1. "This statement changes the table, not the view. But one of the views was created with a special option that ties it to the table's schema. Which option?"
2. "SCHEMABINDING prevents changes to referenced columns of the base table. Is Title referenced by CoreCatalog?"
3. "Even a widening from sixty to one hundred twenty characters is blocked. It is an error, with two messages."

**S7**
1. "Look at the last clause of the view definition. Is that clause allowed in a view on its own?"
2. "ORDER BY inside a view is only accepted together with TOP, OFFSET or FOR XML. None of those is present."

**S8**
1. "What does DeptStats contain: plain columns, or an aggregate with GROUP BY?"
2. "Can the engine map a deleted row of a grouped view back to specific base rows? If not, the view is not updatable."

**Final table**
1. "Only two statements actually changed data. Which two succeeded with one row affected?"
2. "S1 added course 110. What Seats value did it get?"
3. "S5 changed the credits of course 101. Everything else, including course 301's title, is unchanged."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "S1 fails because Seats is NOT NULL and the view does not expose it" | Forgets DEFAULT constraints apply when a column is omitted | "Check the column definition of Seats once more. Is there anything that fills it automatically?" |
| "S2 succeeds and lands in the base table" | Ignores WITH CHECK OPTION | "That would be true without one particular clause. Which clause is on the view?" |
| "S4 raises an error" | Thinks touching an invisible row is an error | "Is a zero-row UPDATE an error in SQL Server?" |
| "S6 succeeds because widening is harmless" | Does not know SCHEMABINDING blocks any change to referenced columns | "SCHEMABINDING does not judge whether the change is harmless. What does it forbid?" |
| "S7 succeeds; views can be ordered" | Confuses TOP 100 PERCENT trick with plain ORDER BY | "Which keywords must accompany ORDER BY inside a view?" |
| "S8 deletes the MATH course" | Believes any view is updatable | "What makes a view non-updatable? Think about aggregates." |
| Final table shows course 301 titled Advanced Compilers | Forgot S4 affected zero rows | "Did S4 actually reach course 301? Recall how many rows it affected." |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the two-gate model:

- **Gate 1, is the view updatable for this statement?** Modifications through a view must touch one base table only, the modified columns must map directly to base columns, and the view must not use aggregates, GROUP BY, DISTINCT, or PIVOT. Otherwise error 4403. That is S8.
- **Gate 2, WITH CHECK OPTION.** Every row produced by an INSERT or UPDATE through the view must still satisfy the view's WHERE clause. Otherwise error 550. That is S2 and S3. Without the option, S2 would have succeeded silently and the row would be invisible through the view.

Then the side rules:

- A view only "sees" its own rows. DML through the view silently affects zero rows for filtered-out base rows. No error. That is S4.
- A hidden NOT NULL column is fine on INSERT through a view when it has a DEFAULT. That is S1.
- SCHEMABINDING freezes the referenced columns. ALTER COLUMN on Title fails with errors 5074 and 4922, even for a widening. Altering Seats, which the view does not reference, would succeed. That is S6.
- ORDER BY inside a view needs TOP, OFFSET or FOR XML to compile, error 1033 otherwise. And even with TOP 100 PERCENT, the order of rows returned from the view is not guaranteed. Only an ORDER BY on the outer query guarantees order. That is S7.

Memory hook: "Check option checks the row after the change. Schemabinding freezes the columns. Grouped views are read-only."

## 9. Follow-up oral questions (optional)

1. "If CoreCatalog had been created without WITH CHECK OPTION, what would S2 have done?" (Succeeds, inserts a MATH row into the base table that the view never shows.)
2. "If S6 had altered the Seats column instead of Title, would it succeed?" (Yes. Seats is not referenced by the schemabound view.)
3. "How can you make a grouped view like DeptStats accept a DELETE?" (Only with an INSTEAD OF trigger on the view.)

## 10. References

- CREATE VIEW, including WITH CHECK OPTION, SCHEMABINDING and updatability rules: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-view-transact-sql
- Modify data through a view: https://learn.microsoft.com/en-us/sql/relational-databases/views/modify-data-through-a-view
- ORDER BY clause and its restriction in views: https://learn.microsoft.com/en-us/sql/t-sql/queries/select-order-by-clause-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
