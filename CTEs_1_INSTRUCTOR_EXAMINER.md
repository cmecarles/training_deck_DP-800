# Instructor-Examiner guide — CTEs 1

Companion to [CTEs_1.md](CTEs_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a, b, c and d. Read all four options, pieces 6 to 9, before taking an answer. The options are long tables of rows; describe them in words, and read individual rows only on request. Accept an answer by letter. If the learner names the number of rows instead of the letter, ask them to confirm the letter.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Write advanced T-SQL code.
- Task bullet: CTEs.
- What is tested: how a recursive CTE unfolds level by level, that `UNION ALL` never removes duplicates, and what `OPTION (MAXRECURSION n)` really counts and what happens when the limit is hit.

## 2. Scenario to read aloud

**Piece 1, the story.** "A mountain rescue organization tracks its incident command structure in a SQL Server database called OrgAtlas. Every member has a call sign. The chain of command is stored as edges, in a table called Org dot CommandLink. Each edge says that a leader directly commands a member."

**Piece 2, the Member table.** "The first table is Org dot Member. Three columns. MemberID, an integer, the primary key. CallSign, a varchar of twenty, not null and unique. And Duty, a varchar of thirty, not null."

**Piece 3, the CommandLink table.** "The second table is Org dot CommandLink. Two columns, LeaderID and MemberID, both integers, both foreign keys to Member. The primary key is the pair, LeaderID plus MemberID. So a leader can command many members, and a member can have more than one leader."

**Piece 4, the eight members.** "Eight members are inserted. Member 1, Cortina, Incident Commander. Member 2, Sella, Operations Lead. Member 3, Brenta, Medical Officer. Member 4, Marmol, Field Team Lead. Member 5, Pordoi, Drone Pilot. Member 6, Falzar, Base Medic. Member 7, Tofana, Rope Specialist. And member 8, Civetta, Field Medic."

**Piece 5, the eight edges.** "Eight edges are inserted, leader then member. 1 commands 2. 1 commands 3. 2 commands 4. 2 commands 5. 3 commands 6. 3 commands 8. 4 commands 7. And 4 commands 8. Notice that member 8, Civetta, appears on two edges. She reports both to Brenta, member 3, and to Marmol, member 4."

**Piece 6, the query.** "The duty officer runs a recursive CTE called Chain. The anchor selects from Member where MemberID equals 1. It returns MemberID, CallSign, the number zero as a column called Lvl, and a Path column, which is a slash followed by the call sign, cast to varchar of two hundred. Then UNION ALL. The recursive member joins Chain to CommandLink on LeaderID equals the chain row's MemberID, then joins Member on the edge's MemberID. It returns the member's MemberID and CallSign, the previous Lvl plus one, and the previous Path plus a slash plus the new call sign, again cast to varchar of two hundred. The outer query selects Lvl, MemberID, CallSign and Path from Chain, ordered by Path, with the hint OPTION open paren MAXRECURSION 3 close paren."

**Piece 7, option a.** "Option a says the statement fails with message 530, level 16, state 1: The statement terminated. The maximum recursion 3 has been exhausted before statement completion. The reasoning given is that the anchor counts as the first recursion level, so a hierarchy whose deepest member sits three joins below the commander needs MAXRECURSION 4."

**Piece 8, option b.** "Option b says nine rows are returned, sorted by Path. Level 0, member 1, Cortina, slash Cortina. Level 1, member 3, Brenta, slash Cortina slash Brenta. Level 2, member 8, Civetta, under Brenta. Level 2, member 6, Falzar, under Brenta. Level 1, member 2, Sella, slash Cortina slash Sella. Level 2, member 4, Marmol, under Sella. Level 3, member 8, Civetta, under Sella slash Marmol. Level 3, member 7, Tofana, under Sella slash Marmol. And level 2, member 5, Pordoi, under Sella. So Civetta appears twice, once at level 2 and once at level 3, with two different paths."

**Piece 9, option c.** "Option c says eight rows. The same as option b, but without the level 3 row for Civetta under Sella slash Marmol. The reasoning is that member 8 was already produced at level 2, and a recursive CTE emits each row only once."

**Piece 10, option d.** "Option d says seven rows. The rows of option b with levels 0, 1 and 2 only. The reasoning is that MAXRECURSION 3 silently stops the recursion after producing level 2, so the two level 3 rows are omitted, but no error is raised."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE OrgAtlas;
GO
USE OrgAtlas;
GO
CREATE SCHEMA Org;
GO
CREATE TABLE Org.Member
(
    MemberID INT         NOT NULL PRIMARY KEY,
    CallSign VARCHAR(20) NOT NULL UNIQUE,
    Duty     VARCHAR(30) NOT NULL
);
GO
CREATE TABLE Org.CommandLink
(
    LeaderID INT NOT NULL REFERENCES Org.Member(MemberID),
    MemberID INT NOT NULL REFERENCES Org.Member(MemberID),
    CONSTRAINT PK_CommandLink PRIMARY KEY (LeaderID, MemberID)
);
GO
INSERT INTO Org.Member (MemberID, CallSign, Duty) VALUES
  (1, 'Cortina', 'Incident Commander'),
  (2, 'Sella',   'Operations Lead'),
  (3, 'Brenta',  'Medical Officer'),
  (4, 'Marmol',  'Field Team Lead'),
  (5, 'Pordoi',  'Drone Pilot'),
  (6, 'Falzar',  'Base Medic'),
  (7, 'Tofana',  'Rope Specialist'),
  (8, 'Civetta', 'Field Medic');
GO
INSERT INTO Org.CommandLink (LeaderID, MemberID) VALUES
  (1, 2),
  (1, 3),
  (2, 4),
  (2, 5),
  (3, 6),
  (3, 8),
  (4, 7),
  (4, 8);
GO
WITH Chain AS
(
    SELECT m.MemberID,
           m.CallSign,
           0 AS Lvl,
           CAST('/' + m.CallSign AS VARCHAR(200)) AS Path
    FROM Org.Member AS m
    WHERE m.MemberID = 1

    UNION ALL

    SELECT m.MemberID,
           m.CallSign,
           c.Lvl + 1,
           CAST(c.Path + '/' + m.CallSign AS VARCHAR(200))
    FROM Chain AS c
    JOIN Org.CommandLink AS l ON l.LeaderID = c.MemberID
    JOIN Org.Member      AS m ON m.MemberID = l.MemberID
)
SELECT Lvl, MemberID, CallSign, Path
FROM Chain
ORDER BY Path
OPTION (MAXRECURSION 3);
```

## 4. The question (ask exactly this)

"What does the query return? Choose one option."

- a. The statement fails with Msg 530, Level 16, State 1, "The statement terminated. The maximum recursion 3 has been exhausted before statement completion.", because the anchor member counts as the first recursion level, so a hierarchy whose deepest member sits three joins below the commander needs MAXRECURSION 4.
- b. Nine rows: Cortina at level 0; Brenta and Sella at level 1; Civetta, Falzar, Marmol and Pordoi at level 2; Civetta and Tofana at level 3; each with its own path, sorted by Path.
- c. Eight rows, as in option b but without the row (3, 8, Civetta, /Cortina/Sella/Marmol/Civetta), because member 8 has already been produced at level 2 and a recursive CTE emits each row only once.
- d. Seven rows, the rows of option b with Lvl 0, 1 and 2 only. MAXRECURSION 3 means the recursion is silently stopped after producing level 2, so the two Lvl 3 rows are omitted but no error is raised.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: b.** Nine rows, the statement succeeds.

| Lvl | MemberID | CallSign | Path |
|---|---|---|---|
| 0 | 1 | Cortina | /Cortina |
| 1 | 3 | Brenta | /Cortina/Brenta |
| 2 | 8 | Civetta | /Cortina/Brenta/Civetta |
| 2 | 6 | Falzar | /Cortina/Brenta/Falzar |
| 1 | 2 | Sella | /Cortina/Sella |
| 2 | 4 | Marmol | /Cortina/Sella/Marmol |
| 3 | 8 | Civetta | /Cortina/Sella/Marmol/Civetta |
| 3 | 7 | Tofana | /Cortina/Sella/Marmol/Tofana |
| 2 | 5 | Pordoi | /Cortina/Sella/Pordoi |

How the recursion unfolds: anchor gives Cortina (level 0). Recursive execution 1 gives Sella and Brenta (level 1). Execution 2 gives Marmol, Pordoi, Falzar, Civetta (level 2). Execution 3 gives Tofana and Civetta again (level 3). Execution 4 finds no outgoing edges from the level 3 rows, returns zero rows, and ends the recursion normally. Three productive executions fit within MAXRECURSION 3.

Why each wrong option is wrong:

- a. The anchor is not a recursion. MAXRECURSION counts executions of the recursive member only, and the final empty execution does not count. The error text is real engine output, but it appears with MAXRECURSION 2, not 3.
- c. A recursive CTE uses UNION ALL, which never deduplicates. The engine has no memory of visited members. Civetta is reached by two different edges at two depths, so she appears twice with two different paths.
- d. Exceeding MAXRECURSION is an error, 530, never a silent truncation. And in this query the limit is not reached at all.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start by drawing the tree from Cortina. Who is at level 1? Who is at level 2? Who is at level 3? Count how many rows that gives, remembering that a member on two edges is reached twice."
2. "Look at the set operator between the anchor and the recursive member. Is it UNION, or UNION ALL? What does that tell you about duplicates?"
3. "Now think about MAXRECURSION. Does the anchor count as a recursion, or only the executions of the recursive part? And how many productive executions does this tree need to reach level 3?"
4. "Consider what happens when a recursive CTE does hit its MAXRECURSION limit. Is it a quiet stop, or an error? That eliminates one option."
5. "So the question is between the answer with nine rows and the answer with eight. The only difference is whether Civetta appears a second time, under Marmol. Does UNION ALL have any reason to drop that row?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, the anchor uses up one level" | Thinks MAXRECURSION counts the anchor | "Read the hint name again: MAX recursion. Is the anchor a recursion, or the starting point?" |
| "a, three joins deep needs four" | Confuses depth with number of executions | "How many times must the recursive member run to produce level 1, level 2 and level 3? Count them." |
| "c, the engine will not produce member 8 twice" | Believes recursive CTEs deduplicate or track visited rows | "Which set operator joins the anchor to the recursive member? Does that operator ever remove rows?" |
| "c, the primary key on CommandLink stops the duplicate" | Confuses a unique edge with a unique member | "The primary key is on the pair, leader and member. Are the two Civetta edges the same pair?" |
| "d, it just stops at the limit without complaint" | Thinks MAXRECURSION is a soft cap | "When the limit is exhausted, does the documentation describe a warning, or an error that terminates the statement?" |
| "d, seven rows because level 3 is over the limit" | Off by one on what MAXRECURSION 3 permits | "Level 3 rows come out of the third execution of the recursive member. Is three within a limit of three?" |
| "It fails with a type mismatch on Path" | Overlooks the CAST on both branches | "Both branches cast Path to varchar of two hundred. Is there still a mismatch?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain how a recursive CTE runs:

- **The anchor runs once.** It produces the starting rows, here Cortina at level 0. It is not counted as a recursion.
- **UNION ALL is mandatory.** Plain UNION is not permitted between the anchor and the recursive member, so there is no duplicate elimination. The recursion has no memory of which members it has visited. It only feeds the rows produced by the previous pass into the next pass. A member reachable by two edges at two depths appears twice, with two levels and two paths. That is Civetta at level 2 under Brenta and at level 3 under Marmol. That is also why a cyclic graph makes a recursive CTE spin forever, and why MAXRECURSION exists as a safety net.
- **The recursive member runs repeatedly.** Each pass consumes only the rows of the previous pass. Here: pass 1 gives level 1, pass 2 gives level 2, pass 3 gives level 3, and pass 4 returns zero rows, which terminates the recursion normally.
- **MAXRECURSION counts productive recursive executions only.** The anchor does not count and the final empty pass does not count. A tree whose deepest node is n joins below the anchor succeeds with MAXRECURSION n. The range is 0 to 32,767, 0 means unlimited, the default is 100.
- **Exceeding the limit is error 530.** The statement terminates and its effects are rolled back. A SELECT may return partial or no results, but it never succeeds silently. With MAXRECURSION 2 this same query fails with "The maximum recursion 2 has been exhausted before statement completion." There is no setting that makes a recursive CTE stop quietly at a chosen depth. To truncate cleanly, add a filter such as WHERE c dot Lvl is less than 3 to the recursive member, so it produces an empty set.
- **Side note on types.** The two CASTs to varchar of two hundred are required. Anchor and recursive member must produce identical types in every column. Without them the statement fails with error 240, types do not match between the anchor and the recursive part in column Path.
- **Side note on order.** ORDER BY Path is deterministic here because every path is unique, even for the duplicated member.

Memory hook: "Anchor runs once and does not count. UNION ALL never dedups. MAXRECURSION counts productive passes, and hitting it is error 530, never a quiet stop."

## 9. Follow-up oral questions (optional)

1. "If the hint were OPTION open paren MAXRECURSION 2 close paren, what would happen?" (Error 530, the statement terminates; the third productive pass is needed for level 3.)
2. "How would you make the query show Civetta only once, at her shallowest level?" (Add explicit logic in the outer query, for example keep the minimum Lvl per MemberID; the CTE itself will not do it.)
3. "What does MAXRECURSION 0 mean, and what is the server default?" (0 means no limit; the default is 100.)

## 10. References

- WITH common_table_expression, including recursive CTE rules and MAXRECURSION: https://learn.microsoft.com/en-us/sql/t-sql/queries/with-common-table-expression-transact-sql
- Query hints, MAXRECURSION range, default and error behaviour: https://learn.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-query
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
