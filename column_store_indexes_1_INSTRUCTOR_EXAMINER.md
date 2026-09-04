# Instructor-Examiner guide — Columnstore Indexes 1

Companion to [column_store_indexes_1.md](column_store_indexes_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d, each a short script. Read all three requirements and all four options before taking an answer. Each option touches both tables, so when the learner answers, ask them to say what the option does to SaleLines and what it does to Payments.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Design and implement indexes.
- Task bullet: Design and implement columnstore indexes.
- What is tested: the division of labour between clustered and nonclustered columnstore, that a CCI table can carry unique rowstore B-tree indexes, that an NCCI is the HTAP pattern over an untouched rowstore table, and that a columnstore index can never be unique.

## 2. Scenario to read aloud

**Piece 1, the story.** "MartMetrics is the analytics database of a retail chain. It receives point-of-sale data into two tables that have just been created in a schema called Pos. One is a huge fact table of sale line items. The other is a live operational table of payments from the registers."

**Piece 2, the fact table.** "The first table is Pos dot SaleLines. It is append-mostly, about six hundred million rows in production, and right now it is a heap: no indexes at all. Six columns. SaleLineID, a bigint, not null. StoreID, an integer. SaleDate, a date. ProductID, an integer. Quantity, a smallint. And NetAmount, a decimal ten comma two. All not null. No primary key yet."

**Piece 3, the operational table.** "The second table is Pos dot Payments. It gets high-frequency singleton inserts and updates through its clustered primary key. Five columns. PaymentID, a bigint identity, not null, with the constraint PK underscore Payments, PRIMARY KEY CLUSTERED. RegisterID, an integer. PaidAt, a DATETIME2 zero. Method, a varchar of ten. And Amount, a decimal ten comma two."

**Piece 4, requirement 1.** "Three requirements. Requirement 1. SaleLines is queried almost only by large aggregations, revenue by store, by day, by product, that scan most of the table. Its only full copy of the data must be stored in columnstore format, for maximum compression, about ten times, and batch-mode scans. The six-hundred-million-row table must not additionally keep a complete rowstore copy of itself."

**Piece 5, requirement 2.** "Requirement 2. The engine, not the loading application, must enforce uniqueness of SaleLineID. And occasional single-line corrections must be able to fetch one row by SaleLineID efficiently."

**Piece 6, requirement 3.** "Requirement 3. Payments must keep its rowstore base storage and its clustered primary key exactly as created, because the register workload depends on cheap singleton seeks and updates through PK underscore Payments. And analysts must additionally get real-time aggregations over the live payment rows. No ETL copy. No change to the table's base storage."

**Piece 7, option a.** "Option a. Three statements. First, ALTER TABLE SaleLines ADD CONSTRAINT PK underscore SaleLines PRIMARY KEY CLUSTERED on SaleLineID. Second, CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI underscore SaleLines on SaleLines, over StoreID, SaleDate, ProductID, Quantity and NetAmount. Third, CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI underscore Payments on Payments, over RegisterID, PaidAt, Method and Amount."

**Piece 8, option b.** "Option b. Three statements. First, CREATE CLUSTERED COLUMNSTORE INDEX CCI underscore SaleLines on SaleLines. Second, CREATE UNIQUE NONCLUSTERED INDEX UX underscore SaleLines underscore ID on SaleLines, on SaleLineID. Third, the same nonclustered columnstore index on Payments as in option a: NCCI underscore Payments over RegisterID, PaidAt, Method and Amount."

**Piece 9, option c.** "Option c. Five statements. The first two are the same as option b: a clustered columnstore index on SaleLines, and a unique nonclustered index on SaleLineID. Then, for Payments: ALTER TABLE DROP CONSTRAINT PK underscore Payments. Then CREATE CLUSTERED COLUMNSTORE INDEX CCI underscore Payments on Payments. Then ALTER TABLE ADD CONSTRAINT PK underscore Payments PRIMARY KEY NONCLUSTERED on PaymentID."

**Piece 10, option d.** "Option d. Two statements. First, CREATE UNIQUE CLUSTERED COLUMNSTORE INDEX CCI underscore SaleLines on SaleLines, ORDER, open paren, SaleLineID, close paren. Second, the same nonclustered columnstore index on Payments as in options a and b."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE MartMetrics;
GO
USE MartMetrics;
GO
CREATE SCHEMA Pos;
GO
-- Fact table: sale line items from every store. Append-mostly,
-- ~600 million rows in production, currently a heap.
CREATE TABLE Pos.SaleLines
(
    SaleLineID  BIGINT        NOT NULL,
    StoreID     INT           NOT NULL,
    SaleDate    DATE          NOT NULL,
    ProductID   INT           NOT NULL,
    Quantity    SMALLINT      NOT NULL,
    NetAmount   DECIMAL(10,2) NOT NULL
);
GO
-- Operational table: live card/cash payments from the registers.
-- High-frequency singleton INSERT/UPDATE through the clustered PK.
CREATE TABLE Pos.Payments
(
    PaymentID   BIGINT IDENTITY(1,1) NOT NULL CONSTRAINT PK_Payments PRIMARY KEY CLUSTERED,
    RegisterID  INT           NOT NULL,
    PaidAt      DATETIME2(0)  NOT NULL,
    Method      VARCHAR(10)   NOT NULL,
    Amount      DECIMAL(10,2) NOT NULL
);
GO
```

Option a:

```sql
ALTER TABLE Pos.SaleLines
    ADD CONSTRAINT PK_SaleLines PRIMARY KEY CLUSTERED (SaleLineID);
GO
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_SaleLines
    ON Pos.SaleLines (StoreID, SaleDate, ProductID, Quantity, NetAmount);
GO
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_Payments
    ON Pos.Payments (RegisterID, PaidAt, Method, Amount);
GO
```

Option b:

```sql
CREATE CLUSTERED COLUMNSTORE INDEX CCI_SaleLines ON Pos.SaleLines;
GO
CREATE UNIQUE NONCLUSTERED INDEX UX_SaleLines_ID
    ON Pos.SaleLines (SaleLineID);
GO
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_Payments
    ON Pos.Payments (RegisterID, PaidAt, Method, Amount);
GO
```

Option c:

```sql
CREATE CLUSTERED COLUMNSTORE INDEX CCI_SaleLines ON Pos.SaleLines;
GO
CREATE UNIQUE NONCLUSTERED INDEX UX_SaleLines_ID
    ON Pos.SaleLines (SaleLineID);
GO
ALTER TABLE Pos.Payments DROP CONSTRAINT PK_Payments;
GO
CREATE CLUSTERED COLUMNSTORE INDEX CCI_Payments ON Pos.Payments;
GO
ALTER TABLE Pos.Payments
    ADD CONSTRAINT PK_Payments PRIMARY KEY NONCLUSTERED (PaymentID);
GO
```

Option d:

```sql
CREATE UNIQUE CLUSTERED COLUMNSTORE INDEX CCI_SaleLines
    ON Pos.SaleLines ORDER (SaleLineID);
GO
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_Payments
    ON Pos.Payments (RegisterID, PaidAt, Method, Amount);
GO
```

## 4. The question (ask exactly this)

"Which indexing strategy meets all three requirements? Option a, clustered primary key on SaleLines plus nonclustered columnstore on both tables. Option b, clustered columnstore on SaleLines, a unique nonclustered B-tree on SaleLineID, and nonclustered columnstore on Payments. Option c, the same for SaleLines, but Payments gets its primary key dropped, a clustered columnstore, and the primary key re-added as nonclustered. Option d, a unique clustered columnstore on SaleLines ordered by SaleLineID, and nonclustered columnstore on Payments. Give me one letter, and tell me what it does to each table."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct: b.** The clustered columnstore index becomes the table itself for SaleLines, so the only full copy is columnstore (requirement 1). A CCI table can carry rowstore B-tree nonclustered indexes since SQL Server 2016, including unique ones, so UX_SaleLines_ID gives engine-enforced uniqueness and a seek path (requirement 2); a duplicate insert fails with error 2601. NCCI_Payments adds an updatable columnstore copy while PK_Payments stays the rowstore clustered index (requirement 3).

- **a is wrong.** Everything runs, and the Payments half is right, which is the bait. But PRIMARY KEY CLUSTERED makes a rowstore B-tree the primary storage of SaleLines, and NCCI_SaleLines is a second, additional copy. Six hundred million rows stored in rowstore plus a columnstore duplicate; the opposite of requirement 1.
- **c is wrong.** The SaleLines half matches b, but the Payments half converts the operational table's base storage to columnstore and demotes the PK to nonclustered. The script is legal and runs, but it violates requirement 3 outright, and it is the wrong tool for singleton seeks and updates.
- **d is wrong.** It fails at the first statement with error 35301: a columnstore index cannot be unique. Columnstore has no key columns. ORDER (SaleLineID) alone is legal, but it only sorts data for segment elimination; it is performance, not a constraint. Even without UNIQUE, requirement 2 would be unmet.

## 6. Hint ladder (one hint per attempt, in order)

1. "There are two flavours of columnstore. One of them is the table itself. The other is an extra copy on top of a rowstore table. Which is which, and which one does requirement 1 call for on SaleLines?"
2. "Now read requirement 3 again. Payments must keep its rowstore base and its clustered PK exactly as created. Which option changes the base storage of Payments? Cross it out."
3. "One option puts the word UNIQUE on a columnstore index. Does a columnstore index have key columns? Can it enforce uniqueness at all?"
4. "Two options left. In one of them, SaleLines ends up with a rowstore clustered B-tree holding every row, plus a columnstore copy of five columns. Count the full copies of the fact table. Does that satisfy requirement 1?"
5. "Since SQL Server 2016, can a table whose base storage is a clustered columnstore also have a unique rowstore B-tree index on top? If yes, which option uses exactly that?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, it gives uniqueness, seeks and batch mode on both tables" | Misses that a clustered PK is a full rowstore copy of SaleLines | "How many complete copies of the six-hundred-million-row table does option a store, and in which formats?" |
| "b cannot work, you cannot add a B-tree index on a columnstore table" | Pre-2016 rule | "That was true once. Since which version can a CCI table carry nonclustered B-tree indexes?" |
| "c, columnstore everywhere is best for analytics" | Ignores requirement 3 and the OLTP workload | "What does requirement 3 say must stay exactly as created on Payments?" |
| "d, the ordered unique CCI gives compression and uniqueness in one" | Believes a columnstore index can be unique | "What does the engine say when you write UNIQUE before COLUMNSTORE? What does ORDER actually do?" |
| "An NCCI makes the Payments table read-only" | Pre-2016 rule again | "Is a nonclustered columnstore index updatable in current versions?" |

## 8. Teaching notes (after the answer is complete or revealed)

The division of labour:

- A **clustered columnstore index** is the table. It replaces the heap or B-tree as the primary storage of every row and every column. That is what delivers the roughly ten times compression on a fact table, because there is no second full copy anywhere. After option b, sys.indexes shows CCI_SaleLines at index_id 1 with type CLUSTERED COLUMNSTORE. A bulk load of 110,000 rows lands directly as one COMPRESSED rowgroup because bulk inserts of at least 102,400 rows bypass the delta store, and an aggregation over it runs in batch mode on both the scan and the hash aggregate.
- A **nonclustered columnstore index** is a secondary, column-compressed copy of chosen columns on top of a rowstore table. It exists for real-time operational analytics, HTAP: OLTP keeps using the rowstore B-tree, analytics scan the columnstore copy of the same live data, no ETL. Since SQL Server 2016 an NCCI is updatable; singleton inserts, updates and deletes work, and the columnstore copy is maintained through the delta store and tuple mover.

Why b works: CCI for SaleLines, then a unique rowstore B-tree on top for the key. Since 2016 a CCI table can carry B-tree nonclustered indexes, including unique ones. Inserting a duplicate SaleLineID fails with error 2601, "Cannot insert duplicate key row ... with unique index UX_SaleLines_ID". An equivalent alternative is ADD CONSTRAINT PRIMARY KEY NONCLUSTERED; what it must not be is PRIMARY KEY CLUSTERED, which would claim the clustered slot the CCI occupies. NCCI_Payments sits beside PK_Payments, which stays CLUSTERED.

Why the others fail:

- **a.** PRIMARY KEY CLUSTERED makes rowstore the primary storage; NCCI_SaleLines is a duplicate. Full rowstore fact table plus a columnstore copy is the opposite of "the only full copy is columnstore". NCCI over rowstore is the pattern for operational tables, not warehouse facts.
- **c.** Legal and runs, but converts Payments to columnstore base storage. Every point update now goes through delete bitmap plus delta store, and every PK lookup seeks a B-tree only to fetch the row from columnstore segments. Rowstore for OLTP plus NCCI for analytics is the supported pattern.
- **d.** Error 35301: a columnstore index cannot be unique. A columnstore index has no key columns; all columns are included. ORDER on a columnstore index is legal, clustered since 2022 and nonclustered since 2025, and sorts data for rowgroup and segment elimination. Performance, never uniqueness.

Extra limits: rowgroups compress up to 1,048,576 rows; loads of at least 102,400 rows go straight to compressed rowgroups; smaller trickle inserts wait in the delta store. Check ActualExecutionMode equals Batch in the actual plan.

Memory hook: "Fact table, scans, compression: clustered columnstore is the table. OLTP table that needs live analytics: nonclustered columnstore on top. Columnstore is never unique; put the key in a B-tree."

## 9. Follow-up oral questions (optional)

1. "How else could requirement 2 have been met on top of the clustered columnstore, without a plain unique index?" (ADD CONSTRAINT PRIMARY KEY NONCLUSTERED on SaleLineID. Not CLUSTERED, because the CCI holds that slot.)
2. "What does ORDER on a columnstore index buy you?" (Sorted data for better rowgroup and segment elimination. It is a performance feature, not a constraint.)
3. "A load of 50,000 rows into the CCI table: where does it land first?" (In the delta store, because it is below the 102,400-row threshold; the tuple mover compresses it later.)

## 10. References

- Columnstore indexes overview: https://learn.microsoft.com/en-us/sql/relational-databases/indexes/columnstore-indexes-overview
- Columnstore indexes, design guidance: https://learn.microsoft.com/en-us/sql/relational-databases/indexes/columnstore-indexes-design-guidance
- Get started with columnstore for real-time operational analytics: https://learn.microsoft.com/en-us/sql/relational-databases/indexes/get-started-with-columnstore-for-real-time-operational-analytics
- CREATE COLUMNSTORE INDEX: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-columnstore-index-transact-sql
- Columnstore indexes, data loading guidance: https://learn.microsoft.com/en-us/sql/relational-databases/indexes/columnstore-indexes-data-loading-guidance
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
