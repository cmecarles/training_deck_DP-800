# Instructor-Examiner guide — In-Memory Tables 1

Companion to [in_memory_tables_1.md](in_memory_tables_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a long twelve-statement lab question on In-Memory OLTP. Take the statements strictly one at a time and keep a running tally of the state: which tables exist, how many rows Basket holds. Accept "succeeds" or "fails" without the error number, but ask for the reason. If the learner already ran the script, ask what they observed at each step before judging. The final catalog query has three rows; take them one at a time and do not require the system-generated primary key name.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Design and implement database objects.
- Task bullet: Design and implement specialized tables, including in-memory tables.
- What is tested: the MEMORY_OPTIMIZED_DATA filegroup prerequisite, SCHEMA_AND_DATA versus SCHEMA_ONLY and the primary key rule, hash bucket count rounding, why CREATE INDEX and TRUNCATE are unsupported, the ATOMIC block shape of natively compiled procedures, and the cross-container isolation rules.

## 2. Scenario to read aloud

**Piece 1, the story.** "FlashCart is the checkout database of a flash-sale web shop. During a sale, tens of thousands of shopping baskets are created and updated per second, so the team decides to move the basket tables to memory-optimized storage, that is In-Memory OLTP. The database has just been created, with a schema called Cart, and nothing else. No special filegroup yet."

**Piece 2, S1 and S2.** "Twelve statements then run in order, each in its own batch, in one session. S1 creates a table Cart dot Basket with BasketID, an integer, primary key nonclustered hash with bucket count one thousand; UserID, an integer; Created, a DATETIME2 zero; and an inline nonclustered index on UserID called IX underscore Basket underscore User. The WITH clause says MEMORY underscore OPTIMIZED equals ON and DURABILITY equals SCHEMA underscore AND underscore DATA. S2 alters the database to add a filegroup called FlashCart underscore MO that CONTAINS MEMORY underscore OPTIMIZED underscore DATA, and then adds a file, a container, to that filegroup."

**Piece 3, S3 and S4.** "S3 is exactly the same CREATE TABLE as S1, character for character. S4 creates a table Cart dot BasketLine with BasketID, ProductID and Qty, all NOT NULL, and one inline nonclustered index on BasketID. No primary key. The WITH clause is MEMORY underscore OPTIMIZED ON and DURABILITY SCHEMA underscore AND underscore DATA."

**Piece 4, S5 and S6.** "S5 creates a table Cart dot ViewLog with UserID, ProductID and ViewedAt, all NOT NULL, and one inline nonclustered hash index on UserID with bucket count five hundred. No primary key. The WITH clause is MEMORY underscore OPTIMIZED ON and DURABILITY SCHEMA underscore ONLY. S6 runs CREATE CLUSTERED INDEX CIX underscore ViewLog on Cart dot ViewLog on the ViewedAt column."

**Piece 5, S7 and S8, the procedures.** "S7 creates a procedure Cart dot AddBasket with two integer parameters, at BasketID and at UserID. It is declared WITH NATIVE underscore COMPILATION and SCHEMABINDING. Its body is a plain BEGIN, an INSERT into Cart dot Basket of the two parameters and SYSDATETIME, and END. S8 creates the same procedure, but the body starts with BEGIN ATOMIC WITH open paren TRANSACTION ISOLATION LEVEL equals SNAPSHOT, LANGUAGE equals us underscore english close paren, then the same INSERT, then END. After S8, in the same batch group, two EXEC calls run: AddBasket 1, 100, and AddBasket 2, 200."

**Piece 6, S9 and S10, the reads.** "S9 is BEGIN TRANSACTION, then SELECT COUNT star as OpenBaskets from Cart dot Basket, with no hint, then COMMIT. S10 is the same, but the SELECT has the table hint WITH open paren SNAPSHOT close paren."

**Piece 7, S11 and S12.** "S11 is TRUNCATE TABLE Cart dot Basket. S12 first runs SET TRANSACTION ISOLATION LEVEL SNAPSHOT for the session, then BEGIN TRANSACTION, then the same SELECT COUNT star from Cart dot Basket WITH SNAPSHOT, then COMMIT."

**Piece 8, the final catalog query.** "Finally a catalog query joins sys dot tables to sys dot indexes on object id, keeping index id greater than zero, and left joins sys dot hash underscore indexes on object id and index id. It filters to tables where is underscore memory underscore optimized equals one and returns TableName, durability underscore desc, IndexName, type underscore desc and bucket underscore count, ordered by table name then index name."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE FlashCart;
GO
USE FlashCart;
GO
CREATE SCHEMA Cart;
GO
-- S1
CREATE TABLE Cart.Basket
(
    BasketID INT NOT NULL PRIMARY KEY NONCLUSTERED HASH WITH (BUCKET_COUNT = 1000),
    UserID   INT NOT NULL,
    Created  DATETIME2(0) NOT NULL,
    INDEX IX_Basket_User NONCLUSTERED (UserID)
) WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_AND_DATA);

-- S2
ALTER DATABASE FlashCart ADD FILEGROUP FlashCart_MO CONTAINS MEMORY_OPTIMIZED_DATA;
ALTER DATABASE FlashCart ADD FILE
    (NAME = 'FlashCart_MO',
     FILENAME = 'C:\Program Files\Microsoft SQL Server\MSSQL17.MSSQLSERVER\MSSQL\DATA\FlashCart_MO')
    TO FILEGROUP FlashCart_MO;

-- S3  (exactly the same CREATE TABLE as S1)
CREATE TABLE Cart.Basket
(
    BasketID INT NOT NULL PRIMARY KEY NONCLUSTERED HASH WITH (BUCKET_COUNT = 1000),
    UserID   INT NOT NULL,
    Created  DATETIME2(0) NOT NULL,
    INDEX IX_Basket_User NONCLUSTERED (UserID)
) WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_AND_DATA);

-- S4
CREATE TABLE Cart.BasketLine
(
    BasketID  INT      NOT NULL,
    ProductID INT      NOT NULL,
    Qty       SMALLINT NOT NULL,
    INDEX IX_BasketLine_Basket NONCLUSTERED (BasketID)
) WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_AND_DATA);

-- S5
CREATE TABLE Cart.ViewLog
(
    UserID    INT          NOT NULL,
    ProductID INT          NOT NULL,
    ViewedAt  DATETIME2(0) NOT NULL,
    INDEX IX_ViewLog_User NONCLUSTERED HASH (UserID) WITH (BUCKET_COUNT = 500)
) WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_ONLY);

-- S6
CREATE CLUSTERED INDEX CIX_ViewLog ON Cart.ViewLog (ViewedAt);

-- S7
CREATE PROCEDURE Cart.AddBasket @BasketID INT, @UserID INT
WITH NATIVE_COMPILATION, SCHEMABINDING
AS
BEGIN
    INSERT INTO Cart.Basket (BasketID, UserID, Created)
    VALUES (@BasketID, @UserID, SYSDATETIME());
END;

-- S8
CREATE PROCEDURE Cart.AddBasket @BasketID INT, @UserID INT
WITH NATIVE_COMPILATION, SCHEMABINDING
AS
BEGIN ATOMIC WITH (TRANSACTION ISOLATION LEVEL = SNAPSHOT, LANGUAGE = N'us_english')
    INSERT INTO Cart.Basket (BasketID, UserID, Created)
    VALUES (@BasketID, @UserID, SYSDATETIME());
END;
GO
EXEC Cart.AddBasket 1, 100;
EXEC Cart.AddBasket 2, 200;

-- S9
BEGIN TRANSACTION;
SELECT COUNT(*) AS OpenBaskets FROM Cart.Basket;
COMMIT;

-- S10
BEGIN TRANSACTION;
SELECT COUNT(*) AS OpenBaskets FROM Cart.Basket WITH (SNAPSHOT);
COMMIT;

-- S11
TRUNCATE TABLE Cart.Basket;

-- S12
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
BEGIN TRANSACTION;
SELECT COUNT(*) AS OpenBaskets FROM Cart.Basket WITH (SNAPSHOT);
COMMIT;
```

Final catalog query:

```sql
SELECT t.name AS TableName, t.durability_desc, i.name AS IndexName, i.type_desc, h.bucket_count
FROM sys.tables AS t
JOIN sys.indexes AS i ON i.object_id = t.object_id AND i.index_id > 0
LEFT JOIN sys.hash_indexes AS h ON h.object_id = i.object_id AND h.index_id = i.index_id
WHERE t.is_memory_optimized = 1
ORDER BY t.name, i.name;
```

## 4. The question (ask exactly this)

"For each statement, S1 to S12, tell me whether it succeeds or raises an error, and where a result is produced, the value returned. Let's go one at a time, starting with S1."

After all twelve: "Now give me the exact result of the final catalog query: for each row, the table name, the durability, the index name, the index type and the bucket count."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

| Stmt | Outcome | Detail |
|---|---|---|
| S1 | Fails, error 41337 | Cannot create memory optimized tables: the database needs a MEMORY_OPTIMIZED_DATA filegroup that is online with at least one container |
| S2 | Succeeds | Filegroup FlashCart_MO and its container are added |
| S3 | Succeeds | Cart.Basket created; the hash primary key gets 1024 buckets, not 1000 |
| S4 | Fails, errors 41321 and 1750 | A SCHEMA_AND_DATA memory-optimized table must have a primary key |
| S5 | Succeeds | SCHEMA_ONLY needs an index but not a primary key; the hash index gets 512 buckets |
| S6 | Fails, error 10794 | CREATE INDEX is not supported with memory-optimized tables |
| S7 | Fails, error 10783 | The body of a natively compiled module must be an ATOMIC block |
| S8 | Succeeds | Procedure created; both EXEC calls insert a row, Basket has 2 rows |
| S9 | Fails, error 41368 | READ COMMITTED on a memory-optimized table is allowed only for autocommit transactions; inside an explicit transaction it needs a hint such as WITH (SNAPSHOT) |
| S10 | Succeeds | Returns OpenBaskets equals 2 |
| S11 | Fails, error 10794 | TRUNCATE TABLE is not supported with memory-optimized tables |
| S12 | Fails, error 41332 | Memory-optimized tables cannot be accessed when the session isolation level is SNAPSHOT, hint or no hint |

Final catalog query, three rows:

| TableName | durability_desc | IndexName | type_desc | bucket_count |
|---|---|---|---|---|
| Basket | SCHEMA_AND_DATA | IX_Basket_User | NONCLUSTERED | NULL |
| Basket | SCHEMA_AND_DATA | PK__Basket__ (system-generated name) | NONCLUSTERED HASH | 1024 |
| ViewLog | SCHEMA_ONLY | IX_ViewLog_User | NONCLUSTERED HASH | 512 |

BasketLine does not exist. Cart.Basket still holds its two rows: S11 failed and S12 failed before reading anything.

## 6. Hint ladder (one hint per attempt, in order)

**S1**
1. "Before a memory-optimized table can exist, the database needs a special kind of filegroup. Has anything been added to FlashCart besides the schema?"
2. "Look at what S2 does right after. Does that suggest what was missing at S1?"

**S2**
1. "This adds a filegroup that CONTAINS MEMORY underscore OPTIMIZED underscore DATA and a container. Is that a valid, ordinary ALTER DATABASE?"

**S3**
1. "The prerequisite is now in place. The statement is identical to S1. Does anything else block it? Check that it has a primary key and its durability."
2. "It succeeds. One detail for later: the requested bucket count is one thousand. Does the engine store exactly one thousand?"

**S4**
1. "Compare BasketLine with Basket. What does Basket have that BasketLine lacks?"
2. "The durability is SCHEMA underscore AND underscore DATA. For that durability, is a primary key optional or required?"

**S5**
1. "ViewLog also has no primary key. But look at its durability. Is it the same as BasketLine's?"
2. "For SCHEMA underscore ONLY, what is the minimum the engine demands: a primary key, or just any one index?"
3. "It succeeds. And the bucket count of five hundred: what does it become?"

**S6**
1. "Is CREATE INDEX a statement that works on memory-optimized tables at all? How are their indexes normally declared?"
2. "Also: can a memory-optimized table have a clustered index of any kind?"

**S7**
1. "The procedure is natively compiled. What must its body be wrapped in? Look at how S8 differs from S7."

**S8**
1. "Native compilation, SCHEMABINDING, BEGIN ATOMIC with an isolation level and a language. Is anything missing? Is SNAPSHOT a valid level for an atomic block?"
2. "It succeeds. Then two EXEC calls run. How many rows does Basket hold now?"

**S9**
1. "The SELECT has no hint, and the session is at the default READ COMMITTED. Is it an autocommit statement, or inside an explicit transaction?"
2. "Inside an explicit transaction, is READ COMMITTED an acceptable isolation level for the memory-optimized side?"

**S10**
1. "Same as S9 but with a table hint. Which isolation level does the hint provide, and is that one acceptable?"
2. "It works. How many rows are in Basket?"

**S11**
1. "TRUNCATE is one of a short list of operations memory-optimized tables do not support. What would you use instead?"

**S12**
1. "The hint is there, so why might it still fail? Look at the SET statement that runs first."
2. "Is a session-level SNAPSHOT isolation the same thing as the WITH SNAPSHOT table hint, as far as memory-optimized tables are concerned?"
3. "The engine refuses memory-optimized access altogether when the session isolation level is SNAPSHOT, regardless of hints."

**Final catalog query**
1. "Which memory-optimized tables actually exist at the end? Count the CREATE TABLE statements that succeeded."
2. "Basket has two indexes: the hash primary key and the nonclustered index on UserID. Which of them has a bucket count, and what is it after rounding?"
3. "ViewLog has one hash index. What is its rounded bucket count?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "S1 succeeds, the syntax is fine" | Unaware of the filegroup prerequisite | "The syntax is fine. But where would durable in-memory rows be checkpointed? Has that place been set up?" |
| "S3 stores 1000 buckets" | Does not know bucket counts round up | "Hash buckets are an array. Does the engine keep arbitrary sizes, or a particular shape of number?" |
| "S4 succeeds, it has an index" | Confuses the SCHEMA_ONLY rule with the SCHEMA_AND_DATA rule | "An index is enough for one durability setting. Is it enough for this one?" |
| "S5 fails, no primary key" | Applies the SCHEMA_AND_DATA rule to SCHEMA_ONLY | "Rows in a SCHEMA_ONLY table are not recovered after a restart. Does the engine still need a key to identify them during recovery?" |
| "S6 succeeds" | Thinks CREATE INDEX works everywhere | "How are indexes added to a memory-optimized table: CREATE INDEX, or inline and ALTER TABLE ADD INDEX?" |
| "S7 succeeds, the INSERT is valid" | Does not know the ATOMIC requirement | "The INSERT is fine. Look at the very first keyword of the body." |
| "S9 succeeds with 2" | Thinks any SELECT works under READ COMMITTED | "It would, if it were an autocommit statement. Is it?" |
| "S11 succeeds and empties the table" | Assumes TRUNCATE is universal | "TRUNCATE is on the unsupported list. So what happens to the rows?" |
| "S12 succeeds with 2, the hint is there" | Treats session SNAPSHOT and hint SNAPSHOT as interchangeable | "The hint is there. But what did the SET statement do to the session before the transaction started?" |
| "The catalog query shows BasketLine" | Forgot S4 failed | "Did S4 succeed?" |
| "Bucket count 1000 and 500 in the catalog" | Forgot rounding | "Powers of two." |

## 8. Teaching notes (after the answer is complete or revealed)

Walk through the five rule groups of In-Memory OLTP:

- **Prerequisite.** MEMORY underscore OPTIMIZED equals ON is rejected with error 41337 until the database has a filegroup declared CONTAINS MEMORY_OPTIMIZED_DATA and at least one container. The "file" is really a folder of checkpoint file pairs where durable rows are streamed. That is S1 versus S3. Azure SQL Database creates the filegroup for you; on SQL Server it is a manual, one-time step, and once in-memory objects exist the filegroup cannot be removed, error 41880.
- **Durability.** SCHEMA_AND_DATA, the default, logs and checkpoints rows and requires a primary key, error 41321, because the engine identifies rows by it during recovery. That is S4. SCHEMA_ONLY persists only the definition; rows are gone after a restart or failover. It is meant for staging or session state and needs only one index, error 41327 if none, not a primary key. That is S5.
- **Indexes.** They are part of the table definition: inline in CREATE TABLE or with ALTER TABLE ADD INDEX, never CREATE INDEX, error 10794. There is no clustered index at all; rows live in memory and every index points to them directly. That is S6. A hash index is an array of buckets and the requested count is rounded up to the next power of two: 1000 becomes 1024, 500 becomes 512. Guidance is one to two times the number of distinct keys. Hash indexes serve equality only; ranges and ORDER BY need the memory-optimized NONCLUSTERED, Bw-tree, index like IX_Basket_User.
- **Natively compiled modules.** Declared WITH NATIVE_COMPILATION and SCHEMABINDING, body a single BEGIN ATOMIC WITH open paren TRANSACTION ISOLATION LEVEL, LANGUAGE close paren block. Without ATOMIC, error 10783; without SCHEMABINDING, error 10796. The atomic block is the transaction. Only SNAPSHOT, REPEATABLE READ and SERIALIZABLE are valid there; READ COMMITTED is rejected. Native modules may reference only memory-optimized tables, error 10775 otherwise, while interpreted T-SQL can join disk-based and memory-optimized tables freely. That is S7 and S8.
- **Transactions.** An autocommit SELECT on a memory-optimized table runs fine under READ COMMITTED. Inside an explicit transaction the same access is a cross-container transaction and READ COMMITTED is invalid for the memory-optimized side, error 41368. Fix with the table hint WITH SNAPSHOT, or the database option MEMORY_OPTIMIZED_ELEVATE_TO_SNAPSHOT equals ON. That is S9 and S10. But a session-level SET TRANSACTION ISOLATION LEVEL SNAPSHOT is incompatible with memory-optimized tables altogether, error 41332, even with the hint, because the disk-based side would use versioned snapshot reads while the in-memory side uses its own row versioning. That is S12. Session SNAPSHOT and hint SNAPSHOT are not interchangeable.
- **Unsupported operations.** TRUNCATE TABLE, error 10794, use DELETE. Also clustered indexes, and IDENTITY other than one comma one. That is S11.

Exam heuristics: "must survive a restart" means SCHEMA_AND_DATA with a primary key; "staging data, maximum speed, loss acceptable" means SCHEMA_ONLY. "Point lookups by exact key" means a hash index with a bucket count near the distinct-key count; "ranges or sorting" means a memory-optimized nonclustered index.

Memory hook: "Filegroup first. Durable needs a key. Buckets round to powers of two. Native means ATOMIC. Explicit transaction needs the SNAPSHOT hint, never the SNAPSHOT session."

## 9. Follow-up oral questions (optional)

1. "How would you add an index to Cart dot ViewLog after it is created?" (ALTER TABLE Cart.ViewLog ADD INDEX; CREATE INDEX is not supported.)
2. "Which database option makes S9 succeed without changing the query?" (MEMORY_OPTIMIZED_ELEVATE_TO_SNAPSHOT equals ON.)
3. "A basket lookup query filters with BasketID greater than one thousand. Can the hash primary key serve it?" (No. Hash indexes serve equality only; a range needs a memory-optimized nonclustered index.)

## 10. References

- In-Memory OLTP overview: https://learn.microsoft.com/en-us/sql/relational-databases/in-memory-oltp/overview-and-usage-scenarios
- Creating a memory-optimized filegroup and container: https://learn.microsoft.com/en-us/sql/relational-databases/in-memory-oltp/the-memory-optimized-filegroup
- Defining durability for memory-optimized objects: https://learn.microsoft.com/en-us/sql/relational-databases/in-memory-oltp/defining-durability-for-memory-optimized-objects
- Indexes for memory-optimized tables, hash and nonclustered: https://learn.microsoft.com/en-us/sql/relational-databases/in-memory-oltp/indexes-for-memory-optimized-tables
- Hash indexes and BUCKET_COUNT: https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-index-design-guide#hash_index
- Natively compiled stored procedures and ATOMIC blocks: https://learn.microsoft.com/en-us/sql/relational-databases/in-memory-oltp/creating-natively-compiled-stored-procedures
- Transactions with memory-optimized tables, cross-container isolation: https://learn.microsoft.com/en-us/sql/relational-databases/in-memory-oltp/transactions-with-memory-optimized-tables
- Unsupported features for In-Memory OLTP: https://learn.microsoft.com/en-us/sql/relational-databases/in-memory-oltp/transact-sql-constructs-not-supported-by-in-memory-oltp
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
