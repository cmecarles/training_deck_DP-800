# Instructor-Examiner guide — Nonclustered Indexes 1

Companion to [indexes_1.md](indexes_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Read the three requirements and all four options before taking an answer. The options use the same four columns and differ only in where each column sits, so say "key" and "include" very clearly each time.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Design and implement database objects.
- Task bullet: Design and implement tables, data types, columns and indexes.
- What is tested: composite nonclustered index design, the difference between key columns and INCLUDE columns, and how column placement decides seek predicates, covering and sort order.

## 2. Scenario to read aloud

**Piece 1, the story.** "PageTurner is the database behind an online second-hand bookstore. Its search page is powered by one table, which currently has no index other than the clustered primary key."

**Piece 2, the table.** "The table is Shop dot Listings. Eight columns. ListingID, an integer identity, the clustered primary key, named PK underscore Listings. ISBN13, a fixed thirteen-character string. Title, text up to one hundred twenty characters. CategoryID, an integer. Condition, text up to twelve characters. Price, a decimal with four digits and two decimals. ListedOn, a date. And IsSold, a bit with a default of zero. All columns are NOT NULL."

**Piece 3, the data.** "A generator script inserts two hundred thousand listings. They are spread over forty categories, using n modulo forty plus one. Prices run from one euro to ninety point ninety-nine. Conditions cycle through LikeNew, Good, Fair and Worn. ListedOn is spread over nine hundred days back from the first of August 2026. One listing in five is marked sold."

**Piece 4, the hot query.** "The single hottest query is the bargains-in-a-category search. It selects Title, Price and ListedOn from Shop dot Listings, where CategoryID equals seven and Price is less than fifteen, ordered by Price. Category seven matches eight hundred five of the two hundred thousand rows."

**Piece 5, the requirements.** "You may create exactly one nonclustered index. It must let the engine answer this exact query with a single covering index seek, meaning three things, all verifiable in the actual execution plan. One: one Index Seek on the new index in which both WHERE conditions appear as seek predicates, so the seek range contains only qualifying rows, with no residual WHERE predicate discarding rows after they are read. Two: no access to the base table, so no Key Lookup into PK underscore Listings. Three: no Sort operator; the seek must return the rows already ordered by Price."

**Piece 6, the options.** "Every option is CREATE NONCLUSTERED INDEX IX underscore Listings underscore Search on Shop dot Listings. They differ only in the key and the include list.

- Option a. Key: Price, then CategoryID. Include: Title and ListedOn.
- Option b. Key: CategoryID, then Price. Include: Title and ListedOn.
- Option c. Key: CategoryID only. Include: Price, Title and ListedOn.
- Option d. Key: CategoryID, then Price. No include list at all."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE PageTurner;
GO
USE PageTurner;
GO
CREATE SCHEMA Shop;
GO
CREATE TABLE Shop.Listings
(
    ListingID  INT IDENTITY(1,1) NOT NULL CONSTRAINT PK_Listings PRIMARY KEY,
    ISBN13     CHAR(13)      NOT NULL,
    Title      NVARCHAR(120) NOT NULL,
    CategoryID INT           NOT NULL,
    Condition  NVARCHAR(12)  NOT NULL,
    Price      DECIMAL(4,2)  NOT NULL,
    ListedOn   DATE          NOT NULL,
    IsSold     BIT           NOT NULL CONSTRAINT DF_Listings_IsSold DEFAULT (0)
);
GO
-- 200,000 listings spread over 40 categories; prices run from 1.00 to 90.99
;WITH N AS
(
    SELECT TOP (200000) ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n
    FROM sys.all_columns AS a CROSS JOIN sys.all_columns AS b
)
INSERT Shop.Listings (ISBN13, Title, CategoryID, Condition, Price, ListedOn, IsSold)
SELECT
    RIGHT('9780000000000' + CAST(n AS varchar(13)), 13),
    N'Used book #' + CAST(n AS nvarchar(10)),
    (n % 40) + 1,
    CASE n % 4 WHEN 0 THEN N'LikeNew' WHEN 1 THEN N'Good' WHEN 2 THEN N'Fair' ELSE N'Worn' END,
    CAST(1 + (n % 9000) / 100.0 AS decimal(4,2)),
    DATEADD(DAY, -(n % 900), '2026-08-01'),
    CASE WHEN n % 5 = 0 THEN 1 ELSE 0 END
FROM N;
GO
```

The hot query:

```sql
SELECT Title, Price, ListedOn
FROM Shop.Listings
WHERE CategoryID = 7
  AND Price < 15.00
ORDER BY Price;
```

The options:

```sql
-- a.
CREATE NONCLUSTERED INDEX IX_Listings_Search
    ON Shop.Listings (Price, CategoryID)
    INCLUDE (Title, ListedOn);

-- b.
CREATE NONCLUSTERED INDEX IX_Listings_Search
    ON Shop.Listings (CategoryID, Price)
    INCLUDE (Title, ListedOn);

-- c.
CREATE NONCLUSTERED INDEX IX_Listings_Search
    ON Shop.Listings (CategoryID)
    INCLUDE (Price, Title, ListedOn);

-- d.
CREATE NONCLUSTERED INDEX IX_Listings_Search
    ON Shop.Listings (CategoryID, Price);
```

## 4. The question (ask exactly this)

"Which index definition meets all three requirements: both WHERE conditions as seek predicates with no residual predicate, no Key Lookup into the base table, and no Sort operator? Option a, key Price then CategoryID, include Title and ListedOn. Option b, key CategoryID then Price, include Title and ListedOn. Option c, key CategoryID only, include Price, Title and ListedOn. Option d, key CategoryID then Price, with no include."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: b.**

| Option | Outcome | Why |
|---|---|---|
| a | Wrong, fails requirement 1 | Range column Price leads the key. Seek is only on Price < 15.00 across all forty categories, 32,199 rows; CategoryID = 7 becomes a residual WHERE predicate. It does cover and it is ordered, but the seek range is wrong. 238 logical reads |
| b | Correct | Plan: one Index Seek with SEEK on CategoryID = 7 AND Price < 15.00, ORDERED FORWARD. No lookup, no sort. 10 logical reads |
| c | Wrong, fails requirements 1 and 3 | Price is an INCLUDE column, so it cannot be a seek predicate and carries no order. Seek on CategoryID only, residual WHERE on Price over 5,000 rows, plus an explicit Sort |
| d | Wrong, fails requirement 2 | Perfect key but no INCLUDE, so Title and ListedOn need Key Lookups into PK_Listings. Left alone the optimizer even prefers a Clustered Index Scan plus Sort. Forced, it is 805 lookups and 2,480 logical reads |

Plan text for the correct option, if the learner asks for it:

```text
Index Seek(OBJECT:(IX_Listings_Search),
           SEEK:(CategoryID=(7) AND Price < (15.00)) ORDERED FORWARD)
```

## 6. Hint ladder (one hint per attempt, in order)

1. "Think of three tiers inside a nonclustered index: leading key columns, trailing key columns, and INCLUDE columns. Each tier can do strictly less than the one before it. Sort the two WHERE columns and the two SELECT-only columns into those tiers."
2. "Requirement two, no Key Lookup, is about whether every column the query touches lives in the index. Which option leaves Title and ListedOn out of the index entirely?"
3. "Option d is out. Now requirement three, no Sort. Only key columns give order. Which option puts Price where it cannot provide order?"
4. "Option c is out. Two options remain, both with the same four columns and the same include list. They differ only in the order of the two key columns. Which condition is an equality and which is a range?"
5. "A seek can consume the leading equality columns and then one trailing range condition. If the range column comes first, the seek chain ends immediately and the equality becomes a residual predicate. Which order puts equality first?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, because Price leads the key and the query orders by Price" | Optimizes for ORDER BY only, ignores seek predicates | "The output is ordered, yes. But look at the seek range. Would the seek on Price alone stay inside category seven, or read every category?" |
| "c, one key column is simpler and everything is included" | Thinks INCLUDE columns can be seeked or ordered | "Can an INCLUDE column ever appear as a seek predicate or supply an order? Which requirement does that break?" |
| "d, the key is right and includes are optional" | Forgets the covering requirement | "Where do Title and ListedOn come from if they are not in the index? What operator does that add?" |
| "a and b are equivalent, key order does not matter" | Does not know the seek chain rule | "In a composite key, does the engine seek on both columns regardless of order? Think about equality first, then range." |
| "b, but the Sort is still needed" | Does not see that trailing key column order applies within the equality prefix | "Within CategoryID equals seven, in what order are the rows stored in that index?" |
| "You need a filtered index on IsSold" | Adds a feature the query does not use | "Does the hot query mention IsSold? An index filter must be matched by the query predicate." |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the three-tier hierarchy:

- **Key, leading equality columns.** Seekable and ordered. CategoryID equals seven belongs here.
- **Key, one trailing range column.** Still seekable, and still ordered within the equality prefix, but it ends the seek chain. Price less than fifteen belongs here. Because rows within CategoryID seven are stored in Price order, the ORDER BY Price is free.
- **INCLUDE.** Leaf-level payload only. Makes the index covering, can be checked by a residual predicate, but can never define a seek range and never contributes ordering. Title and ListedOn belong here.

Why each wrong option fails, with the requirement number:

- Option a reverses the key. A range on the first key column ends the seek chain at once, so the seek is Price less than fifteen across all forty categories, 32,199 rows, and CategoryID equals seven is a residual WHERE that throws away 31,394 rows after reading them. Requirement one fails. It still covers and still avoids the sort, which makes it the subtle distractor. 238 logical reads against 10.
- Option c puts Price in INCLUDE. No seek on Price, so a residual predicate over all 5,000 category-seven rows, and an explicit Sort operator. Requirements one and three fail.
- Option d has the right key but no INCLUDE. Title and ListedOn are only in the base table, so 805 Key Lookups, or, left to itself, a Clustered Index Scan plus Sort. Requirement two fails. 2,480 logical reads when forced.

Verify with the plan, not intuition: seek predicates appear under SEEK, residual filtering under WHERE, and a leftover Sort or Key Lookup names the tier you got wrong.

Two adjacent facts the exam likes:

- A filtered index, CREATE INDEX with WHERE IsSold equals zero, could shrink this index because the shop only searches unsold books. But the query must contain the filter predicate for the optimizer to match it, and parameterized predicates on the filtered column may prevent matching.
- A UNIQUE nonclustered index is both a constraint and an index. It would be wrong here because CategoryID and Price is not unique. INCLUDE columns never participate in the uniqueness check; only key columns do.

Memory hook: "Equality first, one range second, everything else INCLUDE. Include means cover, not seek, not sort."

## 9. Follow-up oral questions (optional)

1. "Would the index with the INCLUDE list written as ListedOn then Title behave differently?" (No. Included columns are unordered leaf payload; their order is irrelevant.)
2. "If the query added AND IsSold equals zero and the index were filtered with WHERE IsSold equals zero, what must be true for the optimizer to use the filtered index?" (The query predicate must match the filter; a parameter on IsSold can prevent matching.)
3. "Why is the all-key variant CategoryID, Price, Title, ListedOn considered inferior even though it also gives a covering seek?" (Title and ListedOn would bloat the non-leaf levels and be subject to key-size limits for no benefit; INCLUDE exists to avoid that.)

## 10. References

- CREATE INDEX, key columns and INCLUDE: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-index-transact-sql
- Create indexes with included columns: https://learn.microsoft.com/en-us/sql/relational-databases/indexes/create-indexes-with-included-columns
- Create filtered indexes: https://learn.microsoft.com/en-us/sql/relational-databases/indexes/create-filtered-indexes
- SQL Server and Azure SQL index architecture and design guide: https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-index-design-guide
- SET STATISTICS PROFILE: https://learn.microsoft.com/en-us/sql/t-sql/statements/set-statistics-profile-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
