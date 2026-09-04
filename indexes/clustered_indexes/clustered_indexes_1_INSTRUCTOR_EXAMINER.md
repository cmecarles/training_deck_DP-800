# Instructor-Examiner guide — Clustered Indexes 1

Companion to [clustered_indexes_1.md](clustered_indexes_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Three of the options are small result tables that differ in only one or two cells, so read each option slowly, row by row, and offer to repeat any row. Take one letter as the answer.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Design and implement database objects.
- Task bullet: Design and implement clustered and nonclustered indexes.
- What is tested: that a primary key is clustered only by default, that a clustered index need not be unique, that index_id 1 is reserved for the clustered index regardless of creation order, and that a heap and a clustered index never coexist.

## 2. Scenario to read aloud

**Piece 1, the story.** "A database called AeroPark tracks long-stay parking sessions at an airport. A DBA runs one batch, top to bottom, on a fresh SQL Server instance. The order of events matters, so listen carefully. The primary key is declared NONCLUSTERED. Then a nonclustered index is added while the table is still a heap. Then rows are inserted, and three of them share the same EntryTime. Only after that is a clustered index created on the EntryTime column, which is not unique."

**Piece 2, the table.** "The table is Park dot Sessions, in a schema called Park. Five columns. SessionID, an integer identity, not null. PlateNo, a char of eight, not null. TerminalCode, a char of two, not null. EntryTime, a DATETIME2 with zero fractional seconds, not null. And ExitTime, the same type, which allows null. The constraint is named PK underscore Sessions, PRIMARY KEY NONCLUSTERED on SessionID."

**Piece 3, the first index.** "Next, while the table is still a heap, the DBA creates a nonclustered index named IX underscore Sessions underscore PlateNo, on the PlateNo column."

**Piece 4, the data.** "Four rows are inserted, with PlateNo, TerminalCode, EntryTime and ExitTime. Row one: plate KL dash 402 dash B, terminal T1, entered first of August 2026 at six fifteen in the morning, exited ninth of August. Row two: plate MN dash 118 dash C, terminal T2, entered first of August at six fifteen, no exit yet. Row three: plate KL dash 402 dash B again, terminal T1, entered fourteenth of August at five past ten at night, no exit. Row four: plate PQ dash 773 dash A, terminal T2, entered first of August at six fifteen, exited twentieth of August. So three of the four rows have exactly the same EntryTime."

**Piece 5, the clustered index.** "Then the DBA creates a clustered index named CIX underscore Sessions underscore EntryTime on Park dot Sessions, on the EntryTime column. No UNIQUE keyword."

**Piece 6, the final query.** "Finally, a SELECT against sys dot indexes. It returns index underscore id, name, type underscore desc, is underscore unique and is underscore primary underscore key, for the object Park dot Sessions, ordered by index underscore id. The question is what that SELECT returns. Four options."

**Piece 7, option a.** "Option a. The batch never reaches the SELECT. The CREATE CLUSTERED INDEX statement fails, because PK underscore Sessions already defines the table's clustered index and a table cannot have more than one."

**Piece 8, option b.** "Option b. Three rows. Index id 1, CIX underscore Sessions underscore EntryTime, CLUSTERED, is unique zero, is primary key zero. Index id 2, PK underscore Sessions, NONCLUSTERED, is unique one, is primary key one. Index id 3, IX underscore Sessions underscore PlateNo, NONCLUSTERED, is unique zero, is primary key zero."

**Piece 9, option c.** "Option c. Also three rows, but with different ids. Index id 2, PK underscore Sessions, NONCLUSTERED, unique one, primary key one. Index id 3, IX underscore Sessions underscore PlateNo, NONCLUSTERED, zero and zero. Index id 4, CIX underscore Sessions underscore EntryTime, CLUSTERED, unique zero, primary key zero."

**Piece 10, option d.** "Option d. Same three rows as option b, same ids 1, 2 and 3, with one difference: the clustered index CIX underscore Sessions underscore EntryTime shows is unique equals one."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE AeroPark;
GO
USE AeroPark;
GO
CREATE SCHEMA Park;
GO
CREATE TABLE Park.Sessions
(
    SessionID    INT IDENTITY(1,1) NOT NULL,
    PlateNo      CHAR(8)      NOT NULL,
    TerminalCode CHAR(2)      NOT NULL,
    EntryTime    DATETIME2(0) NOT NULL,
    ExitTime     DATETIME2(0) NULL,
    CONSTRAINT PK_Sessions PRIMARY KEY NONCLUSTERED (SessionID)
);
GO
CREATE NONCLUSTERED INDEX IX_Sessions_PlateNo
    ON Park.Sessions (PlateNo);
GO
INSERT Park.Sessions (PlateNo, TerminalCode, EntryTime, ExitTime) VALUES
  ('KL-402-B', 'T1', '2026-08-01 06:15:00', '2026-08-09 11:40:00'),
  ('MN-118-C', 'T2', '2026-08-01 06:15:00', NULL),
  ('KL-402-B', 'T1', '2026-08-14 22:05:00', NULL),
  ('PQ-773-A', 'T2', '2026-08-01 06:15:00', '2026-08-20 07:55:00');
GO
CREATE CLUSTERED INDEX CIX_Sessions_EntryTime
    ON Park.Sessions (EntryTime);
GO
SELECT index_id, name, type_desc, is_unique, is_primary_key
FROM sys.indexes
WHERE object_id = OBJECT_ID(N'Park.Sessions')
ORDER BY index_id;
GO
```

Option b:

```text
index_id  name                     type_desc     is_unique  is_primary_key
1         CIX_Sessions_EntryTime   CLUSTERED     0          0
2         PK_Sessions              NONCLUSTERED  1          1
3         IX_Sessions_PlateNo      NONCLUSTERED  0          0
```

Option c:

```text
index_id  name                     type_desc     is_unique  is_primary_key
2         PK_Sessions              NONCLUSTERED  1          1
3         IX_Sessions_PlateNo      NONCLUSTERED  0          0
4         CIX_Sessions_EntryTime   CLUSTERED     0          0
```

Option d:

```text
index_id  name                     type_desc     is_unique  is_primary_key
1         CIX_Sessions_EntryTime   CLUSTERED     1          0
2         PK_Sessions              NONCLUSTERED  1          1
3         IX_Sessions_PlateNo      NONCLUSTERED  0          0
```

## 4. The question (ask exactly this)

"What is the outcome of the final SELECT? Option a, the batch fails at CREATE CLUSTERED INDEX because the primary key is already the clustered index. Option b, three rows: clustered index at id 1, not unique; PK at id 2; PlateNo index at id 3. Option c, three rows: PK at id 2, PlateNo index at id 3, clustered index at id 4, not unique. Option d, same as b but the clustered index shows is unique equals one. Give me one letter."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct: b.** The batch runs without error. The clustered index takes index_id 1 even though it was created last, with is_unique 0 and is_primary_key 0. PK_Sessions stays at id 2, IX_Sessions_PlateNo at id 3. The heap row, id 0, disappears.

- **a is wrong.** A PRIMARY KEY is enforced by a unique index that is clustered only by default. Here it was declared NONCLUSTERED, so the table was a heap and the clustered slot was free.
- **c is wrong.** index_id is a role, not a birth order. index_id 1 is reserved for the clustered index whenever it is created. A clustered index with id 4 is impossible.
- **d is wrong.** The clustered index was created without UNIQUE and three rows share the same EntryTime. The engine adds a hidden uniqueifier to duplicates, but is_unique stays 0.

## 6. Hint ladder (one hint per attempt, in order)

1. "Look at the primary key declaration again. It has one extra keyword after PRIMARY KEY. What does that keyword do to the table's physical structure right after CREATE TABLE?"
2. "After CREATE TABLE the table is a heap. So is the one clustered-index slot taken or free when CIX underscore Sessions underscore EntryTime is created? That decides option a."
3. "Now the ids. Which index underscore id value does sys dot indexes reserve for the clustered index? Is that number ever handed to a nonclustered index, and is it ever skipped for a clustered one?"
4. "Two options left, and they differ in a single cell: is underscore unique on the clustered index. Was the word UNIQUE in the CREATE CLUSTERED INDEX statement? And do three rows share the same EntryTime?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, the PK is always the clustered index" | Confuses default with rule | "Re-read the constraint. Which keyword follows PRIMARY KEY, and what does it override?" |
| "a, EntryTime has duplicates so the clustered index cannot be built" | Thinks clustered implies unique | "Does the statement say UNIQUE? What does the engine do internally with duplicate clustered keys?" |
| "c, the clustered index was created last so it gets the next id" | Treats index_id as sequential | "What is index underscore id 1 reserved for? Where did the heap's id 0 go?" |
| "d, clustered indexes are unique because of the uniqueifier" | Confuses internal bookkeeping with the is_unique flag | "The uniqueifier is hidden. Does it change what sys dot indexes reports, or whether a duplicate insert succeeds?" |
| "The heap row with id 0 is still there as well" | Thinks heap and clustered index coexist | "Can a table be a heap and have a clustered index at the same time?" |

## 8. Teaching notes (after the answer is complete or revealed)

Four separable facts, each wrong option is one of them misremembered:

- **PRIMARY KEY is not the clustered index.** A PRIMARY KEY constraint is enforced by a unique index, and that index is clustered only by default. PRIMARY KEY NONCLUSTERED leaves the table as a heap with a unique nonclustered index. The clustered slot is free, and CREATE CLUSTERED INDEX on EntryTime claims it legally, on a column that is neither the key nor unique. Row identity and physical organisation are independent roles. The opposite trap: a second CREATE CLUSTERED INDEX after this one would fail with "Cannot create more than one clustered index on table".
- **index_id is a role, not a birth order.** id 0 is the heap, only while there is no clustered index. id 1 is always and only the clustered index. id 2 and up are nonclustered. Before the clustered index was created, the metadata query returned id 0 HEAP, id 2 PK, id 3 PlateNo. Creating the clustered index converts the heap: row 0 disappears and the new index appears at id 1 even though it was created last. Heap and clustered index are the same slot in two states; they never coexist.
- **A clustered index need not be unique.** Three rows share EntryTime and the CREATE succeeds without UNIQUE. The engine appends a hidden 4-byte uniqueifier to rows that duplicate an earlier key value; the first occurrence carries none. Verified with sys.dm_db_index_physical_stats in DETAILED mode: leaf records of 33 bytes for non-duplicated rows and 41 bytes for duplicated ones. But is_unique stays 0, duplicates keep inserting, and the optimizer gets none of the plan benefits of a declared unique index.
- **What happened invisibly to the nonclustered indexes.** Every nonclustered index row carries a row locator. On a heap it is the RID, file colon page colon slot. On a clustered table it is the clustering key plus the uniqueifier when present. PK_Sessions and IX_Sessions_PlateNo were built on a heap and stored RIDs, so CREATE CLUSTERED INDEX silently rebuilt both to store EntryTime plus uniqueifier instead. That is why creating or dropping a clustered index on a large table is expensive, and why a wide clustering key inflates every nonclustered index.

One more myth: a clustered index defines the logical order of the data. It is not a promise of physical contiguity, and it is not a promise that SELECT without ORDER BY returns rows in EntryTime order. Only ORDER BY guarantees order.

Memory hook: "PK is clustered only by default. The clustered index is always id 1, never needs to be unique, and its key becomes every nonclustered index's row locator."

## 9. Follow-up oral questions (optional)

1. "If the DBA now runs CREATE CLUSTERED INDEX on PlateNo as well, what happens?" (Error: cannot create more than one clustered index on the table.)
2. "What did the row locator of IX_Sessions_PlateNo contain before the clustered index existed, and what does it contain now?" (Before: the RID, file, page and slot. Now: the EntryTime value plus the uniqueifier where present.)
3. "Does the uniqueifier prevent a fifth row with EntryTime first of August six fifteen from being inserted?" (No. It is internal bookkeeping; duplicates keep inserting and is_unique stays 0.)

## 10. References

- Clustered and nonclustered indexes described: https://learn.microsoft.com/en-us/sql/relational-databases/indexes/clustered-and-nonclustered-indexes-described
- CREATE INDEX: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-index-transact-sql
- sys.indexes, index_id meaning: https://learn.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-indexes-transact-sql
- Heaps, tables without clustered indexes: https://learn.microsoft.com/en-us/sql/relational-databases/indexes/heaps-tables-without-clustered-indexes
- Primary and foreign key constraints: https://learn.microsoft.com/en-us/sql/relational-databases/tables/primary-and-foreign-key-constraints
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
