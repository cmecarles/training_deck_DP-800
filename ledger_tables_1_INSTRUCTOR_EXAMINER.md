# Instructor-Examiner guide — Ledger Tables 1

Companion to [ledger_tables_1.md](ledger_tables_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is an eight-statement lab question on ledger tables followed by three final queries. For S7 and S8 "succeeds" is only half the answer: ask what physically happens to the column and the table. For Q2, the ledger view, take the rows one transaction at a time; accept "transaction 1, 2, 3" for TxnRank. Do not require the GUID suffixes in Q3. If the learner already ran the script, ask what they observed before judging.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Design and implement database objects.
- Task bullet: Design and implement specialized tables, including ledger tables.
- What is tested: the difference between updatable and append-only ledger tables, the hidden GENERATED ALWAYS columns, how an UPDATE appears in the ledger view as DELETE plus INSERT, which operations are blocked to protect evidence, and that dropping a ledger column or table only renames it.

## 2. Scenario to read aloud

**Piece 1, the story.** "GrantLedger is the database of a charitable foundation that must prove to its auditors that grant records have never been silently altered. The team creates two ledger tables. Awards can legitimately be corrected, so Award is an updatable ledger table. Individual payouts must be immutable once written, so Payout is an append-only ledger table."

**Piece 2, the Award table.** "There is a schema called Fund. Fund dot Award has AwardID, an integer primary key; Charity, an NVARCHAR sixty; and Amount, a DECIMAL ten comma two. Its WITH clause says SYSTEM underscore VERSIONING equals ON with HISTORY underscore TABLE equals Fund dot Award underscore History, and LEDGER equals ON. So it is an updatable ledger table with a history table."

**Piece 3, the Payout table.** "Fund dot Payout has PayoutID, an integer primary key; AwardID, an integer; and Amount, a DECIMAL ten comma two. Its WITH clause says LEDGER equals ON, open paren APPEND underscore ONLY equals ON close paren. No system versioning, no history table."

**Piece 4, the data.** "Two awards are inserted in one statement: award 1, River Trust, five thousand; award 2, Book Aid, twelve hundred. Then one payout is inserted: payout 10, for award 1, twenty-five hundred."

**Piece 5, S1 to S4.** "Eight statements then run in order, each in its own batch, in one session. S1 updates Award, setting Amount to fifty-five hundred where AwardID is 1. S2 updates Payout, setting Amount to twenty-six hundred where PayoutID is 10. S3 deletes from Payout where PayoutID is 10. S4 deletes from Award where AwardID is 2."

**Piece 6, S5 to S8.** "S5 is TRUNCATE TABLE Fund dot Award. S6 is ALTER TABLE Fund dot Award SET open paren SYSTEM underscore VERSIONING equals OFF close paren. S7 is ALTER TABLE Fund dot Payout DROP COLUMN Amount. S8 is DROP TABLE Fund dot Payout."

**Piece 7, the final queries.** "Three final queries. Q1 is SELECT star from Fund dot Award. Q2 selects from the ledger view Fund dot Award underscore Ledger: AwardID, Charity, Amount, then DENSE underscore RANK over order by ledger underscore transaction underscore id, aliased TxnRank, then ledger underscore sequence underscore number aliased Seq, and ledger underscore operation underscore type underscore desc aliased Op, ordered by transaction id then sequence number. Transaction ids are replaced by their rank so the output is reproducible. Q3 selects name, ledger underscore type underscore desc and is underscore dropped underscore ledger underscore table from sys dot tables, ordered by name."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE GrantLedger;
GO
USE GrantLedger;
GO
CREATE SCHEMA Fund;
GO
CREATE TABLE Fund.Award
(
    AwardID INT           NOT NULL PRIMARY KEY,
    Charity NVARCHAR(60)  NOT NULL,
    Amount  DECIMAL(10,2) NOT NULL
)
WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = Fund.Award_History), LEDGER = ON);
GO
CREATE TABLE Fund.Payout
(
    PayoutID INT           NOT NULL PRIMARY KEY,
    AwardID  INT           NOT NULL,
    Amount   DECIMAL(10,2) NOT NULL
)
WITH (LEDGER = ON (APPEND_ONLY = ON));
GO
INSERT INTO Fund.Award (AwardID, Charity, Amount)
VALUES (1, N'River Trust', 5000.00), (2, N'Book Aid', 1200.00);
INSERT INTO Fund.Payout (PayoutID, AwardID, Amount)
VALUES (10, 1, 2500.00);
GO
-- S1
UPDATE Fund.Award SET Amount = 5500.00 WHERE AwardID = 1;

-- S2
UPDATE Fund.Payout SET Amount = 2600.00 WHERE PayoutID = 10;

-- S3
DELETE FROM Fund.Payout WHERE PayoutID = 10;

-- S4
DELETE FROM Fund.Award WHERE AwardID = 2;

-- S5
TRUNCATE TABLE Fund.Award;

-- S6
ALTER TABLE Fund.Award SET (SYSTEM_VERSIONING = OFF);

-- S7
ALTER TABLE Fund.Payout DROP COLUMN Amount;

-- S8
DROP TABLE Fund.Payout;
```

Final queries:

```sql
-- Q1
SELECT * FROM Fund.Award;

-- Q2  (transaction ids are replaced by their rank so the output is reproducible)
SELECT AwardID, Charity, Amount,
       DENSE_RANK() OVER (ORDER BY ledger_transaction_id) AS TxnRank,
       ledger_sequence_number      AS Seq,
       ledger_operation_type_desc  AS Op
FROM Fund.Award_Ledger
ORDER BY ledger_transaction_id, ledger_sequence_number;

-- Q3
SELECT name, ledger_type_desc, is_dropped_ledger_table FROM sys.tables ORDER BY name;
```

## 4. The question (ask exactly this)

"For each statement, S1 to S8, tell me whether it succeeds or raises an error. Let's go one at a time, starting with S1."

After all eight: "Now Q1: what does SELECT star from Fund dot Award return, columns and rows?" Then: "Q2: the ledger view. Tell me the rows, transaction by transaction, with the operation of each." Then: "Q3: what tables does sys dot tables list, with their ledger type and the dropped flag?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

| Stmt | Outcome | Detail |
|---|---|---|
| S1 | Succeeds, 1 row | The old version moves to Fund.Award_History |
| S2 | Fails, error 37359 | Updates are not allowed for the append only ledger table Fund.Payout |
| S3 | Fails, error 37359, state 3 | Same message, even for a DELETE |
| S4 | Succeeds, 1 row | The deleted version moves to Fund.Award_History |
| S5 | Fails, error 13545 | Truncate is not a supported operation on system-versioned tables |
| S6 | Fails, error 37356 | System versioning cannot be altered for ledger tables |
| S7 | Succeeds | The column is not removed; it is renamed MSSQL_DroppedLedgerColumn_Amount_GUID and stays in the table |
| S8 | Succeeds | The table is not removed; it is renamed MSSQL_DroppedLedgerTable_Payout_GUID, and its ledger view MSSQL_DroppedLedgerView_Payout_Ledger_GUID |

Q1, only the three user columns; the four GENERATED ALWAYS ledger columns are hidden:

| AwardID | Charity | Amount |
|---|---|---|
| 1 | River Trust | 5500.00 |

Q2, the ledger view, three committed transactions:

| AwardID | Charity | Amount | TxnRank | Seq | Op |
|---|---|---|---|---|---|
| 1 | River Trust | 5000.00 | 1 | 0 | INSERT |
| 2 | Book Aid | 1200.00 | 1 | 1 | INSERT |
| 1 | River Trust | 5500.00 | 2 | 0 | INSERT |
| 1 | River Trust | 5000.00 | 2 | 1 | DELETE |
| 2 | Book Aid | 1200.00 | 3 | 0 | DELETE |

Q3:

| name | ledger_type_desc | is_dropped_ledger_table |
|---|---|---|
| Award | UPDATABLE_LEDGER_TABLE | 0 |
| Award_History | HISTORY_TABLE | 0 |
| MSSQL_DroppedLedgerTable_Payout_GUID | APPEND_ONLY_LEDGER_TABLE | 1 |

## 6. Hint ladder (one hint per attempt, in order)

**S1**
1. "Which kind of ledger table is Award? Does that kind allow corrections?"
2. "It allows the update. Where does the previous version of the row go?"

**S2**
1. "Which kind of ledger table is Payout? What is the only DML operation an append-only table accepts?"

**S3**
1. "Is a DELETE any different from an UPDATE, from the point of view of an append-only table? Both remove evidence."
2. "It fails with the same error number as S2. The message even uses the same wording."

**S4**
1. "Same table as S1. Is DELETE allowed on an updatable ledger table? Where does the deleted version go?"

**S5**
1. "Award is system-versioned. Is TRUNCATE ever allowed on a system-versioned table?"

**S6**
1. "On an ordinary temporal table you can switch system versioning off. What would that allow someone to do with the history of a ledger table?"
2. "Because that would let history be deleted, the engine refuses it for ledger tables."

**S7**
1. "Does the statement raise an error? It does not. But does the Amount column physically disappear?"
2. "Think about the auditor. Would the evidence in that column be lost? What does the engine do instead of deleting it?"

**S8**
1. "Same idea as S7, one level up. The statement succeeds. What happens to the table and its ledger view?"
2. "Look at Q3's third column, is underscore dropped underscore ledger underscore table. What does that tell you?"

**Q1**
1. "How many awards are left after S4? And did S5 empty the table?"
2. "Does SELECT star show the ledger columns, or are they hidden?"

**Q2**
1. "How many transactions touched Award? Count the seed insert, S1 and S4."
2. "The ledger view has no UPDATE operation type. How is an UPDATE represented?"
3. "Transaction 2 is S1. It has two rows: the new version and the old version. Which one is the INSERT and which the DELETE?"
4. "Transaction 3 is S4. One row. What operation, and what award?"

**Q3**
1. "Which tables exist: Award, its history table, and what about Payout, under what name?"
2. "What ledger type does each one report?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "S1 fails, ledger tables are immutable" | Thinks every ledger table is append-only | "There are two kinds. Which kind is Award?" |
| "S3 succeeds, only UPDATE is blocked" | Thinks append-only blocks updates but not deletes | "Would a DELETE erase evidence? Then is it allowed?" |
| "S5 succeeds, TRUNCATE is DDL" | Forgets TRUNCATE is refused on system-versioned tables | "Is TRUNCATE allowed on any system-versioned table?" |
| "S6 succeeds, like any temporal table" | Applies temporal rules to ledger | "What could someone do to the history table once versioning is off? Would the ledger tolerate that?" |
| "S7 fails, you cannot drop a ledger column" | Thinks the engine refuses rather than renames | "The engine does not refuse. What does it do so that the evidence stays?" |
| "S8 fails, you cannot drop a ledger table" | Same as above | "The statement succeeds. Look at Q3's dropped flag." |
| "S8 removes the table completely" | Does not know drop means rename | "Then why would sys dot tables have a column called is underscore dropped underscore ledger underscore table?" |
| "Q1 shows seven columns" | Thinks the ledger columns are visible | "The GENERATED ALWAYS columns are hidden. What does SELECT star show?" |
| "Q2 has a row with Op equal to UPDATE" | Does not know an UPDATE is DELETE plus INSERT | "Which two operation types exist in the ledger view?" |
| "Q2 has only three rows" | Counts transactions, not row versions | "Transaction 2 records both the new version and the old version." |

## 8. Teaching notes (after the answer is complete or revealed)

Explain what ledger tables are for:

- Ledger tables make the history of a table tamper-evident. Every transaction is hashed into a blockchain-like structure, visible in sys dot database underscore ledger underscore transactions and sys dot database underscore ledger underscore blocks, whose digests can be stored outside the database and verified later with sys dot sp underscore verify underscore database underscore ledger. To keep that guarantee, the engine blocks every operation that would erase evidence.

Then the two kinds:

- **Updatable ledger table**, SYSTEM_VERSIONING ON plus LEDGER ON. UPDATE and DELETE are allowed, but every previous row version is kept in the history table, and the table gets four hidden GENERATED ALWAYS columns: ledger start and end transaction id, ledger start and end sequence number. That is why S1 and S4 succeed and why Q1 shows only three columns.
- **Append-only ledger table**, LEDGER ON with APPEND_ONLY ON. Only INSERT is allowed; there is no history table and only the two start columns exist. S2 and S3 both fail with error 37359. The message says "Updates" even for the DELETE, which surfaces as state 3.
- Both kinds get an automatically created ledger view, Fund dot Award underscore Ledger and Fund dot Payout underscore Ledger, that unions current and history rows and exposes ledger transaction id, ledger sequence number, ledger operation type, 1 for INSERT and 2 for DELETE, and its description.

Then Q2, the subtle part:

- The ledger view never shows an UPDATE operation. S1 appears in transaction 2 as an INSERT of the new version, 5500, sequence 0, and a DELETE of the old version, 5000, sequence 1. Transaction 1 holds the two seed inserts, transaction 3 the delete of award 2. The history table therefore contains exactly the two superseded versions, while the base table keeps only the live row.

Then what is never allowed:

- TRUNCATE on any system-versioned table, error 13545, that is S5. SYSTEM_VERSIONING OFF on a ledger table, error 37356, that is S6, because it would let someone delete history rows. LEDGER OFF is not even valid syntax, error 102.

Then drop means rename:

- A dropped ledger column is renamed MSSQL underscore DroppedLedgerColumn underscore name underscore GUID and stays in the table, that is S7. A dropped ledger table is renamed MSSQL underscore DroppedLedgerTable underscore name underscore GUID with is_dropped_ledger_table equal to 1, and its ledger view is renamed MSSQL underscore DroppedLedgerView, that is S8 and Q3. The auditor can still verify the dropped data. The only way to physically remove a ledger table is to drop the entire database. Adding a column is allowed on both kinds; only removing evidence is blocked.

Exam heuristic: pick append-only for immutable facts such as payments, audit events and sensor readings; pick updatable when business corrections are legitimate but must remain provable. Neither kind lets anyone, including db_owner, destroy history without leaving a trace.

Memory hook: "Append-only: insert only. Updatable: history kept, update shows as delete plus insert. Never truncate, never versioning off. Drop means rename."

## 9. Follow-up oral questions (optional)

1. "How would you physically remove the renamed Payout table?" (You cannot; only dropping the whole database removes a ledger table.)
2. "Which two catalog views and which procedure support ledger verification?" (sys.database_ledger_transactions, sys.database_ledger_blocks, and sys.sp_verify_database_ledger with a stored digest.)
3. "Can you add a column to Fund dot Payout with ALTER TABLE ADD?" (Yes. Adding is allowed on both kinds; only removing evidence is blocked.)

## 10. References

- Ledger overview: https://learn.microsoft.com/en-us/sql/relational-databases/security/ledger/ledger-overview
- Updatable ledger tables: https://learn.microsoft.com/en-us/sql/relational-databases/security/ledger/ledger-updatable-ledger-tables
- Append-only ledger tables: https://learn.microsoft.com/en-us/sql/relational-databases/security/ledger/ledger-append-only-ledger-tables
- Ledger considerations and limitations, including dropped columns and tables: https://learn.microsoft.com/en-us/sql/relational-databases/security/ledger/ledger-limits
- Digest management and database verification: https://learn.microsoft.com/en-us/sql/relational-databases/security/ledger/ledger-digest-management
- sys.database_ledger_transactions: https://learn.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-database-ledger-transactions-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
