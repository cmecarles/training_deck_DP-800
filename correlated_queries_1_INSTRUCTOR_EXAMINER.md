# Instructor-Examiner guide — Correlated queries 1

Companion to [correlated_queries_1.md](correlated_queries_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a result-prediction question with three queries. The learner must name the rows, not just the row counts. Take Query 1, then Query 2, then Query 3. The NULL vineyard on harvest 106 is the whole point of Query 1; make sure the learner has heard it clearly before asking.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Write Transact-SQL queries.
- Task bullet: Write queries using subqueries, including correlated subqueries, and handle NULL under three-valued logic.
- What is tested: why NOT IN collapses to zero rows when the subquery returns a NULL, why correlated NOT EXISTS does not, and how a correlated scalar subquery keeps ties.

## 2. Scenario to read aloud

**Piece 1, the story.** "A group of vineyards tracks its harvest yields in a SQL Server database called VineYield. Each harvest row records the vineyard, the year, the grape variety and the tons picked. One harvest came from a leased plot that has not yet been registered as a vineyard, so its VineyardID is NULL. The column is deliberately nullable."

**Piece 2, the tables.** "There is a schema called Farm with two tables. Farm dot Vineyard has three columns: VineyardID, an integer, the primary key. VineyardName, text. And Region, text. Farm dot Harvest has five columns: HarvestID, integer, primary key. VineyardID, an integer that allows NULL and is a foreign key to Vineyard. HarvestYear, a small integer. GrapeVariety, text. And TonsPicked, a decimal with two places."

**Piece 3, the vineyards.** "Four vineyards. Vineyard 1, Stonebrook, in Rioja. Vineyard 2, Mirador, in Priorat. Vineyard 3, Coldwell, in Duero. Vineyard 4, Larkspur, in Rioja."

**Piece 4, the harvests.** "Six harvests. Harvest 101, vineyard 1, year 2024, Tempranillo, eight point four zero tons. Harvest 102, vineyard 1, year 2025, Tempranillo, nine point one zero tons. Harvest 103, vineyard 2, year 2024, Garnacha, six point seven five tons. Harvest 104, vineyard 2, year 2025, Carinena, six point seven five tons. Harvest 105, vineyard 3, year 2024, Verdejo, five point two zero tons. And harvest 106, vineyard NULL, year 2025, Garnacha, four point zero zero tons. That is the leased plot. Notice that Larkspur, vineyard 4, has no harvest at all."

**Piece 5, Query 1.** "The estate manager asks: which vineyards recorded no harvest at all in 2025? Developer A writes Query 1 with NOT IN. It selects VineyardName from Vineyard, aliased v, where v dot VineyardID is NOT IN a subquery. The subquery selects h dot VineyardID from Harvest, aliased h, where HarvestYear equals 2025. Ordered by VineyardName."

**Piece 6, Query 2.** "Developer B writes Query 2 with a correlated NOT EXISTS. It selects VineyardName from Vineyard, aliased v, where NOT EXISTS, open paren, select 1 from Harvest aliased h where h dot VineyardID equals v dot VineyardID and h dot HarvestYear equals 2025, close paren. Ordered by VineyardName."

**Piece 7, Query 3.** "A third report lists, for every vineyard, its best harvest ever, meaning the harvest with the most tons. Query 3 selects VineyardName, HarvestYear, GrapeVariety and TonsPicked. It inner joins Vineyard, aliased v, to Harvest, aliased h, on h dot VineyardID equals v dot VineyardID. The WHERE clause keeps rows where h dot TonsPicked equals a correlated scalar subquery: select MAX of h2 dot TonsPicked from Harvest aliased h2 where h2 dot VineyardID equals v dot VineyardID. Ordered by VineyardName, then HarvestYear."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE VineYield;
GO
USE VineYield;
GO
CREATE SCHEMA Farm;
GO
CREATE TABLE Farm.Vineyard
(
    VineyardID   INT          NOT NULL PRIMARY KEY,
    VineyardName NVARCHAR(40) NOT NULL,
    Region       NVARCHAR(40) NOT NULL
);
GO
CREATE TABLE Farm.Harvest
(
    HarvestID    INT           NOT NULL PRIMARY KEY,
    VineyardID   INT           NULL REFERENCES Farm.Vineyard(VineyardID),
    HarvestYear  SMALLINT      NOT NULL,
    GrapeVariety NVARCHAR(40)  NOT NULL,
    TonsPicked   DECIMAL(6,2)  NOT NULL
);
GO
INSERT INTO Farm.Vineyard (VineyardID, VineyardName, Region) VALUES
  (1, N'Stonebrook', N'Rioja'),
  (2, N'Mirador',    N'Priorat'),
  (3, N'Coldwell',   N'Duero'),
  (4, N'Larkspur',   N'Rioja');
GO
INSERT INTO Farm.Harvest (HarvestID, VineyardID, HarvestYear, GrapeVariety, TonsPicked) VALUES
  (101, 1,    2024, N'Tempranillo', 8.40),
  (102, 1,    2025, N'Tempranillo', 9.10),
  (103, 2,    2024, N'Garnacha',    6.75),
  (104, 2,    2025, N'Carinena',    6.75),
  (105, 3,    2024, N'Verdejo',     5.20),
  (106, NULL, 2025, N'Garnacha',    4.00);   -- leased plot, no vineyard yet
GO
-- Query 1 (developer A, NOT IN)
SELECT v.VineyardName
FROM Farm.Vineyard AS v
WHERE v.VineyardID NOT IN
      (SELECT h.VineyardID
       FROM Farm.Harvest AS h
       WHERE h.HarvestYear = 2025)
ORDER BY v.VineyardName;
-- Query 2 (developer B, correlated NOT EXISTS)
SELECT v.VineyardName
FROM Farm.Vineyard AS v
WHERE NOT EXISTS
      (SELECT 1
       FROM Farm.Harvest AS h
       WHERE h.VineyardID  = v.VineyardID
         AND h.HarvestYear = 2025)
ORDER BY v.VineyardName;
-- Query 3 (correlated scalar subquery)
SELECT v.VineyardName, h.HarvestYear, h.GrapeVariety, h.TonsPicked
FROM Farm.Vineyard AS v
JOIN Farm.Harvest  AS h
  ON h.VineyardID = v.VineyardID
WHERE h.TonsPicked =
      (SELECT MAX(h2.TonsPicked)
       FROM Farm.Harvest AS h2
       WHERE h2.VineyardID = v.VineyardID)
ORDER BY v.VineyardName, h.HarvestYear;
```

## 4. The question (ask exactly this)

"Predict the exact result set of each query: every row, every value, in order. Let's take them one at a time. First, Query 1 with NOT IN: how many rows does it return, and which vineyard names?"

Then: "Query 2 with NOT EXISTS: how many rows, and which names? And if the two queries differ, why?"

Then: "Query 3, the best harvest report: how many rows per vineyard, what are the rows, and which vineyards are absent?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Query 1 returns 0 rows.** An empty result set, no error. Not even Larkspur. The subquery returns the set 1, 2, NULL. NOT IN expands to an AND chain that includes "not equal to NULL", which is UNKNOWN, so the predicate is never TRUE for any vineyard.

**Query 2 returns 2 rows:**

| VineyardName |
|---|
| Coldwell |
| Larkspur |

They differ by two rows because NOT EXISTS is evaluated once per outer row and only ever returns TRUE or FALSE; the NULL harvest row simply fails to match and is ignored.

**Query 3 returns 4 rows:**

| VineyardName | HarvestYear | GrapeVariety | TonsPicked |
|---|---|---|---|
| Coldwell | 2024 | Verdejo | 5.20 |
| Mirador | 2024 | Garnacha | 6.75 |
| Mirador | 2025 | Carinena | 6.75 |
| Stonebrook | 2025 | Tempranillo | 9.10 |

Mirador appears twice because its two harvests tie at 6.75. Larkspur is absent because the inner join finds no harvest for vineyard 4. Harvest 106 with NULL vineyard joins to nothing and is invisible as well.

## 6. Hint ladder (one hint per attempt, in order)

**Query 1**
1. "Before judging the outer query, write down what the subquery returns. List the VineyardID of every 2025 harvest, including harvest 106."
2. "So the list is 1, 2 and NULL. Now expand NOT IN for one vineyard, say Coldwell: three is not equal to 1, AND three is not equal to 2, AND three is not equal to NULL. What is the last comparison worth?"
3. "That last comparison is UNKNOWN. An AND chain with an UNKNOWN in it can be FALSE or UNKNOWN, but can it ever be TRUE?"
4. "A WHERE clause keeps only rows where the predicate is TRUE. Apply that to all four vineyards."

**Query 2**
1. "NOT EXISTS does not compare against a list. It asks a yes-or-no question once per vineyard. Ask it for Stonebrook first."
2. "Stonebrook and Mirador each have a 2025 harvest, so they are out. Now Coldwell: does any harvest row have VineyardID equal to 3 and year 2025?"
3. "For Coldwell, harvest 106 is tested with NULL equals 3. That is UNKNOWN, so the row is not returned by the subquery. Does a row that is simply not returned hurt anything?"
4. "EXISTS is always TRUE or FALSE, never UNKNOWN. Which two vineyards have an empty subquery?"

**Query 3**
1. "Start with the inner join. Which vineyards have at least one harvest row to join to? One of the four does not."
2. "For each remaining vineyard, the subquery computes the maximum tons for that vineyard only. What is the max for Stonebrook, for Mirador, for Coldwell?"
3. "Mirador's max is six point seven five. How many Mirador harvests equal that value?"
4. "An equals-the-max filter keeps every tied row. Count the rows: Stonebrook one, Mirador two, Coldwell one."
5. "Order the rows by VineyardName, then by HarvestYear."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "Query 1 returns Coldwell and Larkspur" | Ignores the NULL in the subquery list | "Look again at harvest 106. What VineyardID does it contribute to the subquery's list?" |
| "Query 1 raises an error because of the NULL" | Expects the engine to complain | "Does three-valued logic raise errors, or does it just change which rows pass the WHERE?" |
| "Query 1 returns Larkspur only" | Thinks NULL only affects vineyards that have harvests | "Expand NOT IN for Larkspur: four not equal to 1, AND not equal to 2, AND not equal to NULL. Can that be TRUE?" |
| "Query 2 also returns zero rows, same as Query 1" | Believes NOT IN and NOT EXISTS are always equivalent | "They are equivalent only under one condition about the subquery column. Is that condition met here?" |
| "Query 2 returns three rows, including the NULL plot" | Confuses the harvest side with the vineyard side | "The outer query lists vineyards. Is the leased plot a vineyard?" |
| "Query 3 returns three rows, one per vineyard with harvests" | Thinks the filter picks a single row like ROW_NUMBER | "Check Mirador. Its two harvests have the same tons. Does an equals comparison choose between them?" |
| "Query 3 includes Larkspur with NULLs" | Treats the join as an outer join | "What kind of join is it? What happens to a vineyard with no matching harvest?" |
| "Query 3 includes harvest 106" | Forgets NULL never matches in a join | "NULL equals four, NULL equals anything, is UNKNOWN. Does that row join to any vineyard?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the three-valued logic trap first:

- **NOT IN with a NULL in the list is never TRUE.** x NOT IN (a, b, NULL) means x not equal a AND x not equal b AND x not equal NULL. The last term is UNKNOWN. An AND chain with an UNKNOWN can be FALSE or UNKNOWN but never TRUE. WHERE keeps only TRUE, so the result is empty. No error is raised, which is exactly what makes the bug dangerous. That is Query 1.
- **Plain IN still works with NULL.** 1 IN (1, 2, NULL) is TRUE because IN expands to an OR chain, and one TRUE decides an OR. Only the negated form collapses.
- **Correlated NOT EXISTS is immune.** It asks a yes-or-no question once per outer row. A subquery row that does not match, such as harvest 106 against Coldwell, is simply not returned. EXISTS and NOT EXISTS are always TRUE or FALSE, never UNKNOWN. That is Query 2, which returns the two semantically correct rows. NOT IN and NOT EXISTS are equivalent only when the subquery column is guaranteed non-nullable.
- **Fixes for the NOT IN version.** Add AND h dot VineyardID IS NOT NULL inside the subquery, or use EXCEPT, which treats two NULLs as not distinct and so never empties the result.

Then the correlated scalar subquery:

- **Equals the correlated MAX keeps ties.** For each joined row, the subquery computes the max tons for that row's vineyard. Every row equal to the max survives. Mirador has two rows at six point seven five, so both appear. This behaves like RANK equals 1, not like ROW underscore NUMBER equals 1. Substituting ROW underscore NUMBER would silently drop one Mirador row unless a deterministic tie-breaker is added.
- **An inner join drops groups with no detail rows.** Larkspur has no harvest, so it never reaches the WHERE. Harvest 106 with NULL vineyard joins to nothing either.
- **Alias scoping.** Inside a subquery, a name resolves first against the subquery's own FROM, and only then outward to the outer aliases. That is why h2 dot VineyardID equals v dot VineyardID works. Re-using the outer alias for the inner table would silently de-correlate the predicate. Always qualify columns in subqueries.

Memory hook: "NOT IN plus one NULL equals zero rows and no error. NOT EXISTS never says UNKNOWN. Equals-the-max keeps ties."

## 9. Follow-up oral questions (optional)

1. "What one-line change inside Query 1's subquery makes it return the same two rows as Query 2?" (Add AND h dot VineyardID IS NOT NULL to the subquery's WHERE clause.)
2. "If Query 3 were rewritten with a window function and a filter of rank equals 1, which function reproduces the four rows exactly: RANK or ROW underscore NUMBER?" (RANK. ROW underscore NUMBER would keep only one arbitrary Mirador row.)
3. "Does plain IN suffer the same problem as NOT IN when the list contains a NULL?" (No. IN expands to an OR chain, so a single TRUE match still decides it; it merely cannot match the NULL itself.)

## 10. References

- Subqueries, including correlated subqueries and NOT IN with NULL: https://learn.microsoft.com/en-us/sql/relational-databases/performance/subqueries
- IN (Transact-SQL), remarks on NULL values: https://learn.microsoft.com/en-us/sql/t-sql/language-elements/in-transact-sql
- EXISTS (Transact-SQL): https://learn.microsoft.com/en-us/sql/language-elements/exists-transact-sql
- EXCEPT and INTERSECT, distinct-based NULL handling: https://learn.microsoft.com/en-us/sql/t-sql/language-elements/set-operators-except-and-intersect-transact-sql
- RANK (Transact-SQL): https://learn.microsoft.com/en-us/sql/t-sql/functions/rank-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
