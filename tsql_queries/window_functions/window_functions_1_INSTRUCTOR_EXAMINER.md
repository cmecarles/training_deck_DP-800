# Instructor-Examiner guide — Window Functions 1

Companion to [window_functions_1.md](window_functions_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a write-the-query question: the learner is shown a result grid and must dictate the query that produces it. Take it in three parts: the Place column, the CharityBanked column, and the final ORDER BY. Let the learner dictate the query in words; you check each part against section 5. Several equivalent forms are accepted. The learner may ask you to repeat the expected grid, piece 5, as often as needed.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Write advanced T-SQL code.
- Task bullet: Use window functions.
- What is tested: choosing RANK over ROW underscore NUMBER and DENSE underscore RANK from the numbering pattern, knowing that an aggregate with ORDER BY defaults to a RANGE frame that includes peers, and keeping ties inside the window while removing them in the presentation ORDER BY.

## 2. Scenario to read aloud

**Piece 1, the story.** "The city marathon uses a SQL Server database called RaceDay for its timing system. As runners cross the finish line, the timing mats write one row per finisher. The script I will describe is the complete history of the database. After it runs, a single query is executed and returns a result grid. Your job is to write that query."

**Piece 2, the table.** "One table, in a schema called Timing, named Finishers. BibNumber, integer, primary key. RunnerName, text up to sixty. Category, VARCHAR ten, with a CHECK constraint allowing Elite, Masters or Open. FinishTime, a TIME with zero fractional seconds. And CharityRaised, a decimal with two decimals."

**Piece 3, the Elite runners.** "Ten rows are inserted, deliberately out of order. Six are Elite. Bib 101, Amara Diallo, two hours six minutes five seconds, three hundred raised. Bib 118, Lucas Meyer, two hours seven twenty-two, two hundred. Bib 122, Priya Nair, two hours eight thirty, two hundred fifty. Bib 130, Owen Carter, also two hours eight thirty, one hundred. Bib 149, Tomas Silva, two hours ten eighteen, fifty. And bib 205, Kenji Sato, two hours six minutes five seconds, the same time as Amara, one hundred fifty."

**Piece 4, the Masters runners.** "Four are Masters. Bib 210, Grace Kim, two hours twenty minutes four seconds, one hundred twenty. Bib 217, Ivan Petrov, two hours twenty-one exactly, two hundred. Bib 224, Nadia Farah, two hours twenty-two eleven, sixty. And bib 233, Hana Yusuf, two hours twenty-one exactly, same as Ivan, eighty. So there are three exact ties: Amara and Kenji, Priya and Owen, Ivan and Hana."

**Piece 5, the expected grid.** "The result has six columns: Category, BibNumber, RunnerName, FinishTime, Place, and CharityBanked. Ten rows, in this order. Elite 101 Amara, place 1, banked four fifty. Elite 205 Kenji, place 1, banked four fifty. Elite 118 Lucas, place 3, banked six fifty. Elite 122 Priya, place 4, banked one thousand. Elite 130 Owen, place 4, banked one thousand. Elite 149 Tomas, place 6, banked one thousand fifty. Masters 210 Grace, place 1, banked one twenty. Masters 217 Ivan, place 2, banked four hundred. Masters 233 Hana, place 2, banked four hundred. Masters 224 Nadia, place 4, banked four sixty."

**Piece 6, the meaning of the columns.** "Place is the official finishing position inside each category. The mats recorded exact ties, and runners who cross together share the same official place. CharityBanked is what the charity dashboard showed at the moment each runner crossed the line: the total money raised so far by that runner's category. Runners who cross together are credited together, so tied runners see the same figure."

**Piece 7, the constraints.** "Three constraints. One: the query must read Timing dot Finishers exactly once. No self-joins, no correlated subqueries against the same table. Two: no hard-coded runner data, no literals for names, bibs, times, places or amounts. Three: the result must be deterministic, the same row order on every execution, on any instance."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE RaceDay;
GO
USE RaceDay;
GO
CREATE SCHEMA Timing;
GO
CREATE TABLE Timing.Finishers (
    BibNumber     INT           NOT NULL PRIMARY KEY,
    RunnerName    NVARCHAR(60)  NOT NULL,
    Category      VARCHAR(10)   NOT NULL
                  CHECK (Category IN ('Elite','Masters','Open')),
    FinishTime    TIME(0)       NOT NULL,
    CharityRaised DECIMAL(8,2)  NOT NULL
);
GO
INSERT INTO Timing.Finishers
    (BibNumber, RunnerName, Category, FinishTime, CharityRaised)
VALUES
    (101, N'Amara Diallo', 'Elite',   '02:06:05', 300.00),
    (118, N'Lucas Meyer',  'Elite',   '02:07:22', 200.00),
    (122, N'Priya Nair',   'Elite',   '02:08:30', 250.00),
    (130, N'Owen Carter',  'Elite',   '02:08:30', 100.00),
    (149, N'Tomas Silva',  'Elite',   '02:10:18',  50.00),
    (205, N'Kenji Sato',   'Elite',   '02:06:05', 150.00),
    (210, N'Grace Kim',    'Masters', '02:20:04', 120.00),
    (217, N'Ivan Petrov',  'Masters', '02:21:00', 200.00),
    (224, N'Nadia Farah',  'Masters', '02:22:11',  60.00),
    (233, N'Hana Yusuf',   'Masters', '02:21:00',  80.00);
GO
```

Expected result:

```text
Category  BibNumber  RunnerName    FinishTime  Place  CharityBanked
--------  ---------  ------------  ----------  -----  -------------
Elite     101        Amara Diallo  02:06:05    1      450.00
Elite     205        Kenji Sato    02:06:05    1      450.00
Elite     118        Lucas Meyer   02:07:22    3      650.00
Elite     122        Priya Nair    02:08:30    4      1000.00
Elite     130        Owen Carter   02:08:30    4      1000.00
Elite     149        Tomas Silva   02:10:18    6      1050.00
Masters   210        Grace Kim     02:20:04    1      120.00
Masters   217        Ivan Petrov   02:21:00    2      400.00
Masters   233        Hana Yusuf    02:21:00    2      400.00
Masters   224        Nadia Farah   02:22:11    4      460.00
```

## 4. The question (ask exactly this)

"Write the query that returns exactly that grid, under the three constraints. Let's build it in three parts. Part one: which window function and which window produce the Place column? Part two: which expression produces CharityBanked, and what frame does it use? Part three: what is the final ORDER BY that makes the row order deterministic?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

Reference query:

```sql
SELECT
    f.Category,
    f.BibNumber,
    f.RunnerName,
    f.FinishTime,
    RANK() OVER (PARTITION BY f.Category
                 ORDER BY f.FinishTime)      AS Place,
    SUM(f.CharityRaised)
        OVER (PARTITION BY f.Category
              ORDER BY f.FinishTime)         AS CharityBanked
FROM Timing.Finishers AS f
ORDER BY f.Category, f.FinishTime, f.BibNumber;
```

- **Part one:** RANK OVER PARTITION BY Category ORDER BY FinishTime. Ties on FinishTime are deliberately left as ties. Elite places 1, 1, 3, 4, 4, 6 and Masters 1, 2, 2, 4: shared values with gaps, which is RANK. ROW underscore NUMBER never repeats; DENSE underscore RANK has no gaps.
- **Part two:** SUM of CharityRaised OVER PARTITION BY Category ORDER BY FinishTime, with no explicit frame, or with the explicit default RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW. RANGE includes peers, so tied rows show the same running total: 450 and 450, 1000 and 1000, 400 and 400.
- **Part three:** ORDER BY Category, FinishTime, BibNumber. The unique tiebreaker BibNumber goes only in the outermost ORDER BY, never inside the windows.

Accepted equivalents: the explicit RANGE frame on the SUM; the same two window columns computed in a CTE with the ORDER BY outside; a named window with the WINDOW clause, SQL Server 2022 or later at compatibility level 160, RANK OVER W and SUM OVER W with WINDOW W AS PARTITION BY Category ORDER BY FinishTime. RANK cannot take a frame at all.

Wrong variants and their symptoms:

- ROW underscore NUMBER for Place: 1, 2, 3, 4, 5, 6, and nondeterministic within ties.
- DENSE underscore RANK for Place: 1, 1, 2, 3, 3, 4 for Elite, 1, 2, 2, 3 for Masters.
- ROWS UNBOUNDED PRECEDING on the SUM: tied rows get different partial totals, one of Amara or Kenji shows 300 or 150 instead of 450, and which one is nondeterministic.
- BibNumber added inside the window ORDER BY: no peers remain, so SUM behaves row by row, Amara 300 and Kenji 450, and RANK degenerates into ROW underscore NUMBER.
- No ORDER BY in the SUM's OVER: every Elite row shows 1050, every Masters row 460.
- No PARTITION BY: Grace Kim gets place 7 and her banked figure starts from 1170.
- No BibNumber in the final ORDER BY: the order within each tied pair is an implementation accident; constraint three violated.

## 6. Hint ladder (one hint per attempt, in order)

**Part one, Place**
1. "Look at the Elite places: 1, 1, 3, 4, 4, 6. Two things stand out. Tied rows share a value, and after each tie there is a gap. Which ranking functions share values, and which leave gaps?"
2. "ROW underscore NUMBER never repeats. DENSE underscore RANK repeats but never skips. Which one repeats and skips?"
3. "Now the window. Grace Kim is place 1 although five Elite runners finished before her. What clause restarts the numbering per category? And what column orders runners within a category?"
4. "Should anything else go in the window ORDER BY besides FinishTime? Think about what happens to the ties if you add BibNumber there."

**Part two, CharityBanked**
1. "It is a running total per category. Which aggregate, with which window clauses, gives a running total?"
2. "Look at Amara and Kenji. Both show 450, which is 300 plus 150, each other's money included. What kind of frame includes the rows that tie with the current row?"
3. "When an aggregate has an ORDER BY in its OVER and no explicit frame, what is the default frame? Is it ROWS or RANGE?"
4. "RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW includes all peers of the current row. So you can leave the frame out, or write it explicitly. Either way, do not put BibNumber in that window's ORDER BY."

**Part three, final order**
1. "Constraint three says deterministic. Within each tied pair, what decides that 101 comes before 205? Nothing in the window does. Where must the tiebreaker go?"
2. "Category first, then FinishTime, then a unique column. Which column is unique?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "ROW underscore NUMBER for Place" | Ignores the shared values | "Would ROW underscore NUMBER ever give two runners the same place?" |
| "DENSE underscore RANK for Place" | Ignores the gaps | "After the two Elite runners at place 1, what place would DENSE underscore RANK give Lucas? The grid says 3." |
| "RANK with ORDER BY FinishTime, BibNumber" | Adds the tiebreaker inside the window | "If BibNumber is in the window ORDER BY, are Amara and Kenji still tied? What does RANK give them then?" |
| "SUM with ROWS UNBOUNDED PRECEDING" | Thinks ROWS is the safe running-total frame | "ROWS stops at the current row physically. Can Amara and Kenji both show 450 under ROWS?" |
| "SUM OVER PARTITION BY Category, no ORDER BY" | Confuses total with running total | "Without ORDER BY in the OVER, the frame is the whole partition. What would every Elite row show?" |
| "SUM with ORDER BY FinishTime, BibNumber" | Same tiebreaker trap for the aggregate | "With BibNumber inside, no two rows are peers. Work out Amara's total under that window." |
| "No PARTITION BY, the categories are already grouped by the ORDER BY" | Confuses presentation with window partition | "What place would Grace Kim get if the ranking ran over all ten runners?" |
| "Final ORDER BY Category, Place" | Not unique | "Two runners share place 1. What decides which comes first?" |
| "Use a self-join to compute the sums" | Violates constraint one | "Constraint one says read the table once. Which feature computes across rows without a join?" |

## 8. Teaching notes (after the answer is complete or revealed)

Start with the three ties: Elite 02:06:05, bibs 101 and 205; Elite 02:08:30, bibs 122 and 130; Masters 02:21:00, bibs 217 and 233. Everything difficult in this question flows from them. Insertion order is irrelevant; windows see PARTITION BY Category ORDER BY FinishTime.

- **Place is RANK.** Tied rows share a value, and gaps follow the ties: after two runners at 1 the next is 3, after two at 4 the last is 6, after two Masters at 2 Nadia is 4. ROW underscore NUMBER never repeats and would split the ties arbitrarily. DENSE underscore RANK compresses the gaps. RANK is one plus the number of rows strictly ahead in the partition. PARTITION BY Category restarts the numbering, so Grace Kim is 1 rather than 7.
- **CharityBanked is a running SUM with the default RANGE frame.** Per the OVER clause documentation, an aggregate with ORDER BY and no explicit frame defaults to RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW, and a RANGE frame ending at CURRENT ROW includes every row whose ORDER BY value equals the current row's, the peers. So each row sums all rows in the partition with FinishTime less than or equal to its own, tied rows included in both directions. That is exactly the dashboard semantics: runners who cross together are credited together. Elite: 450, 450, 650, 1000, 1000, 1050, total 1050. Masters: 120, 400, 400, 460, total 460. The signature of RANGE is equal running totals on tied rows; the pair jumps the total by both peers at once.
- **Why the naive alternatives fail.** ROWS is physical and stops at the current row; the two Elite winners could never both show 450, and which one shows the partial sum is nondeterministic. Adding BibNumber to the window ORDER BY is the subtlest trap: it removes all peers, so even the default RANGE frame behaves row by row, Amara 300 and Kenji 450, and RANK degenerates into ROW underscore NUMBER. Dropping ORDER BY from the SUM's OVER gives category totals everywhere. Dropping PARTITION BY runs both columns across the whole field.
- **Deterministic presentation.** The final ORDER BY Category, FinishTime, BibNumber sorts Elite before Masters, then by time, and resolves the three ties by bib: 101 before 205, 122 before 130, 217 before 233. Without BibNumber the order within each tied pair is an accident and constraint three is violated. This is the mirror image of the trap above: ties must survive inside OVER and be eliminated outside it.
- **Equivalent forms.** Explicit RANGE frame on the SUM. A CTE that computes the window columns with the ORDER BY outside. A named window with the WINDOW clause on SQL Server 2022 or later at compatibility level 160. Ranking functions accept no ROWS or RANGE clause at all, so for them the ORDER BY list is the only tie control.

Memory hook: "Inside OVER, keep the ties: they define peers. Outside, kill the ties: add a unique key. Equal totals on tied rows means RANGE; strictly increasing means ROWS."

## 9. Follow-up oral questions (optional)

1. "What would Place be for the Elite runners with DENSE underscore RANK?" (1, 1, 2, 3, 3, 4.)
2. "What would Grace Kim's CharityBanked be if PARTITION BY were removed?" (1170: the Elite total 1050 plus her 120.)
3. "Can you put a ROWS or RANGE frame on RANK?" (No. Ranking functions accept no frame clause; only the ORDER BY list controls ties.)

## 10. References

- OVER clause, including the default RANGE frame and peers: https://learn.microsoft.com/en-us/sql/t-sql/queries/select-over-clause-transact-sql
- RANK: https://learn.microsoft.com/en-us/sql/t-sql/functions/rank-transact-sql
- DENSE_RANK: https://learn.microsoft.com/en-us/sql/t-sql/functions/dense-rank-transact-sql
- ROW_NUMBER: https://learn.microsoft.com/en-us/sql/t-sql/functions/row-number-transact-sql
- SELECT WINDOW clause, named windows: https://learn.microsoft.com/en-us/sql/t-sql/queries/select-window-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
