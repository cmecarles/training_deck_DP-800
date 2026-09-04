# Instructor-Examiner guide — Window Functions 2

Companion to [window_functions_2.md](window_functions_2.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a hand-computation question: twelve rows of data, two queries, nineteen result rows. The learner must do the arithmetic in their head, so read the data slowly and repeat it whenever asked; the learner may want to write the twelve rows down. Take Q1 column by column, not row by row: ask for one column across all twelve rows, confirm it, then move to the next column. Then take Q2 the same way. Accept "null" and "no value" as the same answer. Accept "zero" for 0.0 and "minus one" for -1.0.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Write advanced T-SQL code.
- Task bullet: Use window functions: LAG, LEAD, FIRST_VALUE, LAST_VALUE, NTILE, PERCENT_RANK and CUME_DIST, with PARTITION BY, ORDER BY and frame clauses.
- What is tested: when the LAG and LEAD default applies and when a real NULL comes back, the default frame that makes LAST_VALUE echo the current row, the ROWS frame that fixes it, IGNORE NULLS, how NTILE splits a remainder, and the formulas for PERCENT_RANK and CUME_DIST with ties.

## 2. Scenario to read aloud

**Piece 1, the story.** "A solar cooperative stores the monthly energy readings of its rooftop panels in a SQL Server database called SunMeter. A reading is NULL when the panel's meter was offline that month. There is one table and two queries. You will need to compute the result of both queries by hand, so feel free to write the data down."

**Piece 2, the table.** "The table is Solar dot Readings, in a schema called Solar. It has three columns. PanelId, a two-character code, such as P1 or P2. ReadMonth, a date, always the first day of a month. And kWh, kilowatt-hours, a decimal six comma one, which can be NULL. The primary key is the pair PanelId and ReadMonth, so within one panel every month appears once."

**Piece 3, the data for panel P1.** "Twelve rows are inserted. Panel P1 has seven months, January to July twenty twenty-six. January, four hundred ten point zero. February, NULL. March, three hundred eighty point zero. April, three hundred ninety-five point zero. May, NULL. June, NULL. July, four hundred fifty point zero. Again, P1 in order: four ten, null, three eighty, three ninety-five, null, null, four fifty."

**Piece 4, the data for panel P2.** "Panel P2 has five months, January to May twenty twenty-six. January, NULL. February, three hundred eighty point zero. March, four hundred twenty point zero. April, NULL. May, four hundred five point zero. Again, P2 in order: null, three eighty, four twenty, null, four oh five."

**Piece 5, query Q1, the window.** "Query Q1 walks each panel's timeline. It selects PanelId, ReadMonth and kWh, plus six window columns. Every one of the six uses the same window: PARTITION BY PanelId, ORDER BY ReadMonth. So each panel is processed alone, in calendar order, and since ReadMonth is unique inside a panel there are no ties. The final ORDER BY of the query is PanelId then ReadMonth, so the twelve rows come out P1 January to July, then P2 January to May."

**Piece 6, Q1 columns one and two.** "The first window column is called PrevKwh. It is LAG of kWh, offset one, default zero. So the value one row before, and zero when there is no such row. The second is TwoAhead. It is LEAD of kWh, offset two, default minus one. So the value two rows after, and minus one when there is no such row. Neither LAG nor LEAD has a frame clause; they are not allowed one."

**Piece 7, Q1 columns three and four.** "The third window column is FirstKwh: FIRST underscore VALUE of kWh over the window, with no frame clause, so the default frame applies. The fourth is LastDefault: LAST underscore VALUE of kWh over the same window, again with no frame clause, so the default frame applies. The default frame, when there is an ORDER BY and no frame is written, is RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW."

**Piece 8, Q1 columns five and six.** "The fifth window column is LastFixed: LAST underscore VALUE of kWh over the window, with an explicit frame, ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING. The sixth is Carried: LAST underscore VALUE of kWh with IGNORE NULLS, over the window, with no frame clause, so the default frame again. IGNORE NULLS means the function skips NULL values when it picks the last value in the frame."

**Piece 9, query Q2.** "Query Q2 ranks the months that actually have a reading, across both panels together, no partition. Its WHERE clause keeps only rows where kWh is not null, which leaves seven rows. It selects PanelId, ReadMonth, kWh and three window columns. Tercile is NTILE of three, ordered by kWh descending, then PanelId, then ReadMonth. PctRank is PERCENT underscore RANK ordered by kWh ascending, cast to decimal five comma four. CumeDist is CUME underscore DIST ordered by kWh ascending, cast to decimal five comma four. The final ORDER BY of the query is kWh ascending, then PanelId, then ReadMonth."

**Piece 10, the seven non-null rows.** "The seven rows Q2 works on, in ascending kWh order, are: P1 March, three eighty. P2 February, three eighty. P1 April, three ninety-five. P2 May, four oh five. P1 January, four ten. P2 March, four twenty. P1 July, four fifty. Note the tie: two rows at three eighty. Tell me when you are ready for the question."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE SunMeter;
GO
USE SunMeter;
GO
CREATE SCHEMA Solar;
GO
CREATE TABLE Solar.Readings
(
    PanelId   CHAR(2)      NOT NULL,
    ReadMonth DATE         NOT NULL,
    kWh       DECIMAL(6,1) NULL,
    CONSTRAINT PK_Readings PRIMARY KEY (PanelId, ReadMonth)
);
GO
INSERT INTO Solar.Readings (PanelId, ReadMonth, kWh) VALUES
    ('P1', '2026-01-01', 410.0),
    ('P1', '2026-02-01', NULL),
    ('P1', '2026-03-01', 380.0),
    ('P1', '2026-04-01', 395.0),
    ('P1', '2026-05-01', NULL),
    ('P1', '2026-06-01', NULL),
    ('P1', '2026-07-01', 450.0),
    ('P2', '2026-01-01', NULL),
    ('P2', '2026-02-01', 380.0),
    ('P2', '2026-03-01', 420.0),
    ('P2', '2026-04-01', NULL),
    ('P2', '2026-05-01', 405.0);
GO
```

Query Q1:

```sql
-- Q1
SELECT
    r.PanelId,
    r.ReadMonth,
    r.kWh,
    LAG(r.kWh, 1, 0)   OVER (PARTITION BY r.PanelId ORDER BY r.ReadMonth) AS PrevKwh,
    LEAD(r.kWh, 2, -1) OVER (PARTITION BY r.PanelId ORDER BY r.ReadMonth) AS TwoAhead,
    FIRST_VALUE(r.kWh) OVER (PARTITION BY r.PanelId ORDER BY r.ReadMonth) AS FirstKwh,
    LAST_VALUE(r.kWh)  OVER (PARTITION BY r.PanelId ORDER BY r.ReadMonth) AS LastDefault,
    LAST_VALUE(r.kWh)  OVER (PARTITION BY r.PanelId ORDER BY r.ReadMonth
                             ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING) AS LastFixed,
    LAST_VALUE(r.kWh) IGNORE NULLS
                       OVER (PARTITION BY r.PanelId ORDER BY r.ReadMonth) AS Carried
FROM Solar.Readings AS r
ORDER BY r.PanelId, r.ReadMonth;
```

Query Q2:

```sql
-- Q2
SELECT
    r.PanelId,
    r.ReadMonth,
    r.kWh,
    NTILE(3) OVER (ORDER BY r.kWh DESC, r.PanelId, r.ReadMonth)        AS Tercile,
    CAST(PERCENT_RANK() OVER (ORDER BY r.kWh) AS DECIMAL(5,4))         AS PctRank,
    CAST(CUME_DIST()    OVER (ORDER BY r.kWh) AS DECIMAL(5,4))         AS CumeDist
FROM Solar.Readings AS r
WHERE r.kWh IS NOT NULL
ORDER BY r.kWh, r.PanelId, r.ReadMonth;
```

## 4. The question (ask exactly this)

"Give me the exact result set of Q1, twelve rows, and of Q2, seven rows: every value, including every NULL, in the row order produced. Let's take Q1 one column at a time. Start with PrevKwh: give me its value for each of the twelve rows, P1 January to July, then P2 January to May."

Then, in order: "Now TwoAhead, all twelve rows." "Now FirstKwh." "Now LastDefault." "Now LastFixed." "Now Carried."

Then: "Now Q2. First tell me the seven rows in the order the query returns them, panel and month and kWh. Then give me Tercile for each. Then PctRank. Then CumeDist."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Q1**, twelve rows, DECIMAL(6,1) values:

| PanelId | ReadMonth | kWh | PrevKwh | TwoAhead | FirstKwh | LastDefault | LastFixed | Carried |
|---|---|---|---|---|---|---|---|---|
| P1 | 2026-01-01 | 410.0 | 0.0 | 380.0 | 410.0 | 410.0 | 450.0 | 410.0 |
| P1 | 2026-02-01 | NULL | 410.0 | 395.0 | 410.0 | NULL | 450.0 | 410.0 |
| P1 | 2026-03-01 | 380.0 | NULL | NULL | 410.0 | 380.0 | 450.0 | 380.0 |
| P1 | 2026-04-01 | 395.0 | 380.0 | NULL | 410.0 | 395.0 | 450.0 | 395.0 |
| P1 | 2026-05-01 | NULL | 395.0 | 450.0 | 410.0 | NULL | 450.0 | 395.0 |
| P1 | 2026-06-01 | NULL | NULL | -1.0 | 410.0 | NULL | 450.0 | 395.0 |
| P1 | 2026-07-01 | 450.0 | NULL | -1.0 | 410.0 | 450.0 | 450.0 | 450.0 |
| P2 | 2026-01-01 | NULL | 0.0 | 420.0 | NULL | NULL | 405.0 | NULL |
| P2 | 2026-02-01 | 380.0 | NULL | NULL | NULL | 380.0 | 405.0 | 380.0 |
| P2 | 2026-03-01 | 420.0 | 380.0 | 405.0 | NULL | 420.0 | 405.0 | 420.0 |
| P2 | 2026-04-01 | NULL | 420.0 | -1.0 | NULL | NULL | 405.0 | 420.0 |
| P2 | 2026-05-01 | 405.0 | NULL | -1.0 | NULL | 405.0 | 405.0 | 405.0 |

Column by column, for checking:

- PrevKwh: 0.0, 410.0, NULL, 380.0, 395.0, NULL, NULL; then 0.0, NULL, 380.0, 420.0, NULL. The 0.0 appears only on the first row of each panel. Every other NULL is a real NULL read from the previous row.
- TwoAhead: 380.0, 395.0, NULL, NULL, 450.0, -1.0, -1.0; then 420.0, NULL, 405.0, -1.0, -1.0. The -1.0 appears only on the last two rows of each panel.
- FirstKwh: 410.0 on all seven P1 rows; NULL on all five P2 rows.
- LastDefault: equals kWh on every row: 410.0, NULL, 380.0, 395.0, NULL, NULL, 450.0; NULL, 380.0, 420.0, NULL, 405.0.
- LastFixed: 450.0 on all seven P1 rows; 405.0 on all five P2 rows.
- Carried: 410.0, 410.0, 380.0, 395.0, 395.0, 395.0, 450.0; NULL, 380.0, 420.0, 420.0, 405.0.

**Q2**, seven rows:

| PanelId | ReadMonth | kWh | Tercile | PctRank | CumeDist |
|---|---|---|---|---|---|
| P1 | 2026-03-01 | 380.0 | 3 | 0.0000 | 0.2857 |
| P2 | 2026-02-01 | 380.0 | 3 | 0.0000 | 0.2857 |
| P1 | 2026-04-01 | 395.0 | 2 | 0.3333 | 0.4286 |
| P2 | 2026-05-01 | 405.0 | 2 | 0.5000 | 0.5714 |
| P1 | 2026-01-01 | 410.0 | 1 | 0.6667 | 0.7143 |
| P2 | 2026-03-01 | 420.0 | 1 | 0.8333 | 0.8571 |
| P1 | 2026-07-01 | 450.0 | 1 | 1.0000 | 1.0000 |

Arithmetic, n = 7:

- NTILE(3) over kWh descending: seven rows into three buckets, sizes 3, 2, 2. 450, 420, 410 get 1; 405, 395 get 2; 380, 380 get 3.
- PERCENT_RANK = (RANK - 1) / (n - 1) = (RANK - 1) / 6. The two 380s have RANK 1: 0. 395 has RANK 3: 2/6 = 0.3333. 405: 3/6 = 0.5000. 410: 4/6 = 0.6667. 420: 5/6 = 0.8333. 450: 6/6 = 1.0000.
- CUME_DIST = (rows with kWh less than or equal to current) / 7. The two 380s: 2/7 = 0.2857. 395: 3/7 = 0.4286. 405: 4/7 = 0.5714. 410: 5/7 = 0.7143. 420: 6/7 = 0.8571. 450: 7/7 = 1.0000.

## 6. Hint ladder (one hint per attempt, in order)

**PrevKwh**
1. "LAG looks one row back inside the panel. What does the third argument, the default, do? When exactly is it used?"
2. "The default is used only when the previous row does not exist. It is not a COALESCE. If the previous row exists but holds NULL, what comes back?"
3. "So zero only on the first month of each panel. For P1 March, the previous row is P1 February. What is February's value?"
4. "Check every row whose previous month is offline: P1 March, P1 June, P1 July, P2 February, P2 May. What do they get?"

**TwoAhead**
1. "LEAD with offset two looks two rows forward inside the panel. Which rows have no row two positions ahead?"
2. "Only the last two rows of each panel get the default minus one. Every other row reads the real value two months ahead, even if that value is NULL."
3. "P1 January looks at P1 March. P1 February looks at P1 April. P1 March looks at P1 May. What is P1 May's value?"

**FirstKwh**
1. "With ORDER BY and no frame, the frame runs from the start of the partition to the current row. Where does FIRST underscore VALUE look?"
2. "Always the first row of the panel. What is P1 January's value? What is P2 January's value?"
3. "P2's first month is offline. Does FIRST underscore VALUE skip a NULL by itself, or only when told to?"

**LastDefault**
1. "This is the trap column. Same default frame, from the start of the partition to the current row. Which row is the last row of a frame that ends at the current row?"
2. "The last row of the frame is the current row itself. So what does LAST underscore VALUE return on each row?"
3. "It simply echoes kWh, including the NULLs."

**LastFixed**
1. "Now the frame is ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING. Where does the frame end?"
2. "At the last row of the panel. What is P1's last month, and P2's last month?"
3. "The value is the same on every row of the panel."

**Carried**
1. "Default frame again, start of partition to current row. But IGNORE NULLS is on. So it is the last non-null value up to and including the current row."
2. "For P1 February, which is offline, what is the last non-null value so far? For P1 May and June?"
3. "For P2 January, the frame contains only one row, and it is NULL. Is there any non-null value to return?"

**Q2 row order**
1. "The WHERE clause drops the NULL months. How many rows remain? The ORDER BY is kWh ascending, then PanelId, then ReadMonth."
2. "Two rows tie at three eighty. Which comes first, P1 or P2?"

**Tercile**
1. "NTILE of three over seven rows. Seven does not divide by three. Which buckets get the extra row: the first ones or the last ones?"
2. "The first buckets take the remainder, so the sizes are three, two, two. The ordering is kWh descending. Which three values are the largest?"
3. "Bucket one: four fifty, four twenty, four ten. Bucket two: the next two. Bucket three: the two three eighties."

**PctRank**
1. "PERCENT underscore RANK is RANK minus one, divided by n minus one. Here n is seven. What is the lowest possible value, and who gets it?"
2. "The two three eighties share RANK one, so both get zero. What RANK does three ninety-five get: two or three?"
3. "RANK skips after a tie, so three ninety-five is RANK three: two sixths. Then three sixths, four sixths, five sixths, six sixths."

**CumeDist**
1. "CUME underscore DIST is the number of rows with a value less than or equal to the current one, divided by n. It counts the current row and its peers."
2. "How many rows are less than or equal to three eighty? Both three eighties count. So two out of seven for both."
3. "Then three sevenths, four sevenths, five sevenths, six sevenths, seven sevenths. Round to four decimals."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "PrevKwh for P1 March is zero" | Thinks the LAG default replaces NULL | "Does the previous row exist? The default only covers a missing row." |
| "PrevKwh for P1 January is NULL" | Forgets the default | "There is a third argument to LAG. When is it used?" |
| "TwoAhead for P1 March is minus one" | Confuses a NULL value two rows ahead with a missing row | "Two rows after P1 March is P1 May. Does that row exist?" |
| "FirstKwh for P2 is three eighty" | Assumes FIRST_VALUE skips NULL | "Without IGNORE NULLS, is a NULL a value like any other?" |
| "LastDefault is four fifty for all P1 rows" | Forgets the default frame ends at the current row | "Write out the default frame. Where does it end? Which row is last in it?" |
| "LastFixed equals kWh" | Swaps the two LAST_VALUE columns | "Look at the frame on this column. Where does it end?" |
| "Carried for P2 January is zero or three eighty" | Looks ahead, or invents a default | "The frame ends at the current row. Is there any non-null value at or before P2 January?" |
| "Carried for P1 May is NULL" | Forgets IGNORE NULLS | "This column has IGNORE NULLS. What is the last non-null value up to May?" |
| "Terciles are sized two, two, three" | Puts the remainder in the last bucket | "Which buckets take the extra rows in NTILE, the first or the last?" |
| "PctRank for three ninety-five is one sixth" | Uses dense rank | "PERCENT underscore RANK uses RANK, not DENSE underscore RANK. After a tie at rank one, what is the next rank?" |
| "PctRank for the two three eighties is one seventh" | Mixes up PERCENT_RANK and CUME_DIST | "One of the two functions always starts at zero. Which one?" |
| "CumeDist for the first three eighty is one seventh" | Counts only rows strictly below plus itself, not the peer | "CUME underscore DIST counts every row less than or equal to the current value. How many rows are less than or equal to three eighty?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the six Q1 columns as three rules, then the three Q2 columns as two formulas.

- **LAG and LEAD: the default covers a missing row, not a NULL row.** `LAG(expr, offset, default)` returns the value from the row offset positions before. The default is used only when that row does not exist. P1 January and P2 January have no previous row, so 0.0. P1 March's previous row exists but is NULL, so NULL. The same for LEAD: only the last two rows of each panel have no row two ahead, so -1.0; every other NULL is a real offline month. To skip offline months you would write `LAG(kWh, 1, 0) IGNORE NULLS`, which would give P1 July 395.0 and P1 March 410.0.
- **The default frame and the LAST_VALUE trap.** With ORDER BY and no frame, the frame is RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW. FIRST_VALUE is harmless: the frame always starts at row one of the partition, so P1 gets 410.0 everywhere and P2 gets NULL everywhere, because a NULL is a value unless you say IGNORE NULLS. LAST_VALUE is the trap: the last row of a frame that ends at the current row is the current row, so LastDefault just echoes kWh. Had ReadMonth contained ties, the RANGE frame would include the peers and return an arbitrary peer, worse, not better. The fix is a frame that reaches the end: ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING, or ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING. That is LastFixed: 450.0 for P1, 405.0 for P2.
- **IGNORE NULLS with the default frame is carry-forward.** LAST_VALUE IGNORE NULLS over start-to-current gives the last non-null value up to and including this row: P1 410.0 carries through February, 395.0 carries through May and June; P2 January is NULL because nothing non-null exists yet. IGNORE NULLS and RESPECT NULLS exist since SQL Server 2022 for FIRST_VALUE, LAST_VALUE, LAG and LEAD only.
- **Two compile-time gates.** Omitting ORDER BY from LAG, LEAD, FIRST_VALUE, LAST_VALUE or any ranking function is error 4112, "The function must have an OVER clause with ORDER BY." Adding a ROWS or RANGE frame to LAG, LEAD or a ranking or distribution function is error 10752, "The function may not have a window frame." Only aggregates and FIRST_VALUE and LAST_VALUE accept a frame.
- **NTILE with a remainder.** Seven rows into three buckets: the first buckets take the extra rows, sizes 3, 2, 2. The window ORDER BY adds PanelId and ReadMonth after kWh DESC because the two 380s tie; NTILE never shares a bucket value between tied rows, so without a unique order a tie could straddle a boundary nondeterministically.
- **PERCENT_RANK versus CUME_DIST.** Both are over ORDER BY kWh alone, so the two 380s are peers and get identical values. PERCENT_RANK = (RANK - 1) / (n - 1): starts at 0, ends at 1, counts rows strictly below, and RANK skips after a tie, so 395 is rank 3. CUME_DIST = (rows less than or equal to current) / n: never 0, ends at 1, counts the current row and its peers, so 2/7 for the tied pair. Both return FLOAT; the CAST to DECIMAL(5,4) only fixes the display.

Memory hook: "The default is for a missing row, not a NULL. LAST_VALUE needs UNBOUNDED FOLLOWING or it is just the current row. NTILE front-loads the remainder. PERCENT_RANK starts at zero, CUME_DIST never does."

## 9. Follow-up oral questions (optional)

1. "If you wrote LAG of kWh with IGNORE NULLS and default zero, what would PrevKwh be for P1 July?" (395.0, the last non-null value before July.)
2. "What error do you get if you add ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW to the LAG call?" (Error 10752, the function LAG may not have a window frame.)
3. "In Q2, what would CUME_DIST be for the two three eighties if only one of them existed, so n were six?" (1/6, about 0.1667. PERCENT_RANK would still be 0.)

## 10. References

- LAG: https://learn.microsoft.com/en-us/sql/t-sql/functions/lag-transact-sql
- LEAD: https://learn.microsoft.com/en-us/sql/t-sql/functions/lead-transact-sql
- FIRST_VALUE, including IGNORE NULLS: https://learn.microsoft.com/en-us/sql/t-sql/functions/first-value-transact-sql
- LAST_VALUE, including IGNORE NULLS: https://learn.microsoft.com/en-us/sql/t-sql/functions/last-value-transact-sql
- OVER clause, PARTITION BY, ORDER BY and the ROWS and RANGE frame, default frame: https://learn.microsoft.com/en-us/sql/t-sql/queries/select-over-clause-transact-sql
- NTILE: https://learn.microsoft.com/en-us/sql/t-sql/functions/ntile-transact-sql
- PERCENT_RANK: https://learn.microsoft.com/en-us/sql/t-sql/functions/percent-rank-transact-sql
- CUME_DIST: https://learn.microsoft.com/en-us/sql/t-sql/functions/cume-dist-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
