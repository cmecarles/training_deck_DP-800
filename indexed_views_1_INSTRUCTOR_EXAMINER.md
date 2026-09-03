# Instructor-Examiner guide — Indexed Views 1

Companion to [indexed_views_1.md](indexed_views_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Read all four options before taking an answer. The four scripts differ in only one or two details each, so read each option slowly and offer to repeat it. If the learner asks "what is different between b and c", you may say that they differ in exactly one token, without saying which.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Implement programmability objects.
- Task bullet: Create views (indexed views).
- What is tested: the checklist a view must satisfy before a unique clustered index can be created on it, namely SCHEMABINDING, COUNT_BIG with GROUP BY, and no outer joins, and which error each missing item raises.

## 2. Scenario to read aloud

**Piece 1, the story.** "A wholesale fresh-produce market runs SQL Server with a database called FreshWholesale. Order lines arrive continuously. The analytics team wants revenue per produce item pre-aggregated and materialized by the engine, so they decide to create an indexed view."

**Piece 2, the first table.** "There is a schema called Mkt. The first table is Mkt dot Produce. Three columns. ProduceID, an integer, the primary key. ProduceName, text up to forty characters. Category, text up to twenty characters. All three are NOT NULL."

**Piece 3, the second table.** "The second table is Mkt dot OrderLine. Five columns, all NOT NULL. OrderID, an integer. LineNum, a small integer. ProduceID, an integer with a foreign key to Produce. Kg, a decimal with eight digits and two decimals. And PricePerKg, also decimal eight comma two. The primary key is the pair OrderID and LineNum, named PK underscore OrderLine."

**Piece 4, the test conditions.** "Each option below is run on its own, in a fresh copy of the database. The session has QUOTED underscore IDENTIFIER and ANSI underscore NULLS set to ON. Every option has two statements: a CREATE VIEW and then a CREATE UNIQUE CLUSTERED INDEX on that view. For exactly one option both statements succeed and you get a working indexed view."

**Piece 5, option a.** "Option a creates a view Mkt dot RevenueByProduce with no options on the CREATE VIEW line. It selects ProduceID, the SUM of Kg times PricePerKg as Revenue, and COUNT underscore BIG of star as LineCount, from Mkt dot OrderLine, grouped by ProduceID. Then it creates a unique clustered index IX underscore RevenueByProduce on the view, keyed on ProduceID."

**Piece 6, option b.** "Option b creates the same view Mkt dot RevenueByProduce, this time WITH SCHEMABINDING. It selects ProduceID, SUM of Kg times PricePerKg as Revenue, and plain COUNT of star as LineCount, from Mkt dot OrderLine, grouped by ProduceID. Then the same unique clustered index on ProduceID."

**Piece 7, option c.** "Option c creates the view Mkt dot RevenueByProduce WITH SCHEMABINDING. It selects ProduceID, SUM of Kg times PricePerKg as Revenue, and COUNT underscore BIG of star as LineCount, from Mkt dot OrderLine, grouped by ProduceID. Then the same unique clustered index on ProduceID."

**Piece 8, option d.** "Option d creates a different view, Mkt dot RevenueByCategory, WITH SCHEMABINDING. It selects p dot Category, SUM of ol dot Kg times ol dot PricePerKg as Revenue, and COUNT underscore BIG of star as LineCount. The FROM clause is Mkt dot Produce as p, LEFT JOIN Mkt dot OrderLine as ol, on ol dot ProduceID equals p dot ProduceID. It groups by p dot Category. Then it creates a unique clustered index IX underscore RevenueByCategory on the view, keyed on Category."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE FreshWholesale;
GO
USE FreshWholesale;
GO
CREATE SCHEMA Mkt;
GO
CREATE TABLE Mkt.Produce
(
    ProduceID   INT          NOT NULL PRIMARY KEY,
    ProduceName NVARCHAR(40) NOT NULL,
    Category    NVARCHAR(20) NOT NULL
);
GO
CREATE TABLE Mkt.OrderLine
(
    OrderID    INT          NOT NULL,
    LineNum    SMALLINT     NOT NULL,
    ProduceID  INT          NOT NULL REFERENCES Mkt.Produce(ProduceID),
    Kg         DECIMAL(8,2) NOT NULL,
    PricePerKg DECIMAL(8,2) NOT NULL,
    CONSTRAINT PK_OrderLine PRIMARY KEY (OrderID, LineNum)
);
GO
```

Option a:

```sql
CREATE VIEW Mkt.RevenueByProduce
AS
SELECT ProduceID,
       SUM(Kg * PricePerKg) AS Revenue,
       COUNT_BIG(*)         AS LineCount
FROM Mkt.OrderLine
GROUP BY ProduceID;
GO
CREATE UNIQUE CLUSTERED INDEX IX_RevenueByProduce
    ON Mkt.RevenueByProduce (ProduceID);
GO
```

Option b:

```sql
CREATE VIEW Mkt.RevenueByProduce
WITH SCHEMABINDING
AS
SELECT ProduceID,
       SUM(Kg * PricePerKg) AS Revenue,
       COUNT(*)             AS LineCount
FROM Mkt.OrderLine
GROUP BY ProduceID;
GO
CREATE UNIQUE CLUSTERED INDEX IX_RevenueByProduce
    ON Mkt.RevenueByProduce (ProduceID);
GO
```

Option c:

```sql
CREATE VIEW Mkt.RevenueByProduce
WITH SCHEMABINDING
AS
SELECT ProduceID,
       SUM(Kg * PricePerKg) AS Revenue,
       COUNT_BIG(*)         AS LineCount
FROM Mkt.OrderLine
GROUP BY ProduceID;
GO
CREATE UNIQUE CLUSTERED INDEX IX_RevenueByProduce
    ON Mkt.RevenueByProduce (ProduceID);
GO
```

Option d:

```sql
CREATE VIEW Mkt.RevenueByCategory
WITH SCHEMABINDING
AS
SELECT p.Category,
       SUM(ol.Kg * ol.PricePerKg) AS Revenue,
       COUNT_BIG(*)               AS LineCount
FROM Mkt.Produce AS p
LEFT JOIN Mkt.OrderLine AS ol
       ON ol.ProduceID = p.ProduceID
GROUP BY p.Category;
GO
CREATE UNIQUE CLUSTERED INDEX IX_RevenueByCategory
    ON Mkt.RevenueByCategory (Category);
GO
```

## 4. The question (ask exactly this)

"Each option is executed in isolation in a fresh copy of the database, from a session with QUOTED_IDENTIFIER and ANSI_NULLS set to ON. For exactly one option, both the CREATE VIEW and the CREATE UNIQUE CLUSTERED INDEX complete successfully, producing a working indexed view. Which one? The options are a, b, c and d, as I described them."

If the learner wants them again: option a, no SCHEMABINDING, COUNT_BIG. Option b, SCHEMABINDING, plain COUNT. Option c, SCHEMABINDING, COUNT_BIG, single table. Option d, SCHEMABINDING, COUNT_BIG, LEFT JOIN of two tables.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: c.**

| Option | Outcome | Why |
|---|---|---|
| a | CREATE VIEW succeeds; CREATE INDEX fails, error 1939 | The view is not schema bound. "Cannot create index on view 'RevenueByProduce' because the view is not schema bound." |
| b | CREATE VIEW succeeds; CREATE INDEX fails, error 10136 | Uses COUNT instead of COUNT_BIG. "Cannot create index on view ... because it uses the aggregate COUNT. Use COUNT_BIG instead." |
| c | Both succeed | SCHEMABINDING, two-part names, GROUP BY with COUNT_BIG(*), deterministic SUM over NOT NULL decimals, single table, first index is unique clustered on the GROUP BY key |
| d | CREATE VIEW succeeds; CREATE INDEX fails, error 10113 | Uses a LEFT JOIN. "Cannot create index on view ... because it uses a LEFT, RIGHT, or FULL OUTER join, and no OUTER joins are allowed in indexed views." |

Note for the assistant: in all three wrong options the view itself is created without error. The failure is always at the CREATE UNIQUE CLUSTERED INDEX step.

## 6. Hint ladder (one hint per attempt, in order)

1. "Think of the requirements as a checklist. An indexed view is materialized and maintained on every insert, update and delete of the base table. Ask of each option: does anything stop the engine from doing that safely and incrementally?"
2. "One requirement concerns the CREATE VIEW line itself, before the SELECT. Which keyword ties the view to the table's schema so that the table cannot be altered under the materialized rows? Which option lacks it?"
3. "Option a can be eliminated. Now look at the aggregates. When a view has GROUP BY, the engine must be able to count rows per group so it can remove a group when it becomes empty. Which exact function must be in the select list for that?"
4. "COUNT of star and COUNT underscore BIG of star are not interchangeable here. One returns a four-byte integer, the other a bigint. Only one of them is accepted in an indexed view. Which option uses the wrong one?"
5. "Options a and b are out. Between c and d, one of them has a FROM clause with two tables joined in a particular way. Is every kind of join allowed in an indexed view?"
6. "Outer joins produce NULL-extended rows that cannot be maintained incrementally. Which of the remaining two options has an outer join?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, because COUNT_BIG is there and the view is simple" | Forgets SCHEMABINDING is mandatory | "Compare the CREATE VIEW line of a with the others. Is anything missing before the AS?" |
| "b, SCHEMABINDING is there, so it works" | Believes COUNT and COUNT_BIG are equivalent | "Look at the aggregate in the select list once more. Read it letter by letter if it helps." |
| "d, it is the most complete, two tables and a category" | Thinks joins are fine in indexed views | "What kind of join is it? Is every join type allowed when the view must be maintained incrementally?" |
| "b and c both work" | Does not know COUNT_BIG is enforced with its own error | "Only one option works. What is the single token that differs between b and c, and does the engine care?" |
| "None of them, the view needs a primary key first" | Confuses indexed views with tables | "A view has no primary key. What is the first index that must be created on an indexed view?" |
| "The CREATE VIEW fails in option a" | Misplaces the failure | "Would a plain view without SCHEMABINDING fail to be created? Or does the problem appear later?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the idea first: an indexed view stores its result rows physically. Every insert, update or delete on the base table must update those stored rows incrementally. Every requirement on the checklist exists so that incremental maintenance is possible and safe.

The checklist, each item with its own error:

- **WITH SCHEMABINDING** is mandatory. Without it the base table could be altered or dropped under the materialized data. Missing it gives error 1939 at CREATE INDEX time. That is option a.
- **Two-part names inside the view**, such as Mkt dot OrderLine. Inside a schemabound module a one-part name fails at CREATE VIEW time with error 4512.
- **GROUP BY requires COUNT_BIG(*)** in the select list. The engine uses it to decrement group row counts and delete a group's row when it becomes empty. Plain COUNT gives error 10136. That is option b.
- **No outer joins.** LEFT, RIGHT or FULL OUTER joins give error 10113. NULL-extended rows cannot be maintained incrementally in a sound way. Rewriting option d with INNER JOIN would make it indexable. That is option d.
- **Deterministic and precise expressions over NOT NULL columns.** SUM over a nullable expression would be another disqualifier. Here Kg and PricePerKg are both NOT NULL decimals, so SUM of Kg times PricePerKg is fine.
- **ANSI SET options ON** when the view is created and when the index is created. With QUOTED_IDENTIFIER OFF, error 1935.
- **The first index must be UNIQUE CLUSTERED**, and here it is keyed on the GROUP BY column, which is unique per group by construction. A nonclustered index before the clustered one gives error 1940.

Two neighbouring facts worth saying aloud:

- After the unique clustered index exists, the engine maintains the view automatically on every DML against the base table. That maintenance cost is the price you pay.
- Reading through the index is guaranteed with the table hint WITH open paren NOEXPAND close paren. Automatic matching of queries to the indexed view without the hint is an optimizer feature of Enterprise-class editions. On other editions, without NOEXPAND, the query expands the view and re-aggregates the base table.

Memory hook: "Schemabound, two-part names, COUNT_BIG, no outer joins, unique clustered first."

## 9. Follow-up oral questions (optional)

1. "If option d were rewritten with INNER JOIN instead of LEFT JOIN, keeping everything else, would the index be created?" (Yes. The outer join was the only disqualifier.)
2. "On a valid indexed view, what happens if you create a nonclustered index before any clustered index?" (Error 1940. The view does not have a unique clustered index yet; the unique clustered index must come first.)
3. "How do you make sure a query reads the materialized rows rather than re-aggregating the base table?" (Reference the view with the table hint WITH NOEXPAND.)

## 10. References

- Create indexed views, full requirements list: https://learn.microsoft.com/en-us/sql/relational-databases/views/create-indexed-views
- CREATE VIEW, including SCHEMABINDING: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-view-transact-sql
- CREATE INDEX, indexed views section: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-index-transact-sql
- COUNT_BIG: https://learn.microsoft.com/en-us/sql/t-sql/functions/count-big-transact-sql
- Table hints, NOEXPAND: https://learn.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-table
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
