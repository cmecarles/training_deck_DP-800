# Instructor-Examiner guide — Ledger Verification 1

Companion to [ledger_verification_1.md](ledger_verification_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a long multi-part question: ten statements, then four final queries. Take the statements one at a time; for S2 to S6 also ask what the statement returns. Then take the four queries one at a time. Exact error numbers are a bonus, not a requirement; accept the right outcome with the right reason. Hash values and object id suffixes differ per run, so never ask for them; ask for the shape of the output instead, for example "block zero, a hash, a commit time".

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked. Say "sys dot s p underscore verify underscore database underscore ledger" slowly the first time, then "the verify procedure" afterwards.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Implement data integrity and security features.
- Task bullet: Implement ledger tables and ledger databases; generate and verify ledger digests.
- What is tested: what a ledger database forces on every table, how digests and blocks work, what verification needs and what it does and does not detect, and which DDL a ledger table refuses.

## 2. Scenario to read aloud

**Piece 1, the story.** "A bond registrar keeps municipal bonds and their coupon payments in a SQL Server database called BondRegistry. Regulators must be able to prove that nothing in the whole database has been altered outside the normal application path. Not just a few tables, everything. So the team creates a ledger database rather than individual ledger tables: CREATE DATABASE BondRegistry WITH LEDGER equals ON. Then a schema called Reg is created."

**Piece 2, the first table, Coupon.** "Reg dot Coupon has three columns: CouponID, an integer, the primary key; BondID, an integer; and Amount, a DECIMAL ten comma two. All not null. It is created WITH LEDGER equals ON, open paren, APPEND underscore ONLY equals ON, close paren. So it is explicitly an append-only ledger table."

**Piece 3, the second table, Bond.** "Reg dot Bond has three columns: BondID, an integer, the primary key; Issuer, an NVARCHAR of sixty; and FaceValue, a DECIMAL twelve comma two. All not null. Note this carefully: Bond is created with no WITH clause at all. No ledger option is mentioned."

**Piece 4, the seed data.** "Three seed statements run. First, two bonds are inserted: BondID 1, issuer City of Lakeside, face value one thousand; and BondID 2, issuer Port Authority, face value five thousand. Second, one coupon is inserted: CouponID 10, BondID 1, amount twenty five. Third, Bond 1 is updated: FaceValue set to one thousand one hundred."

**Piece 5, S1 and S2.** "Ten statements then run in order, each in its own batch, in one session. For the exercise the digest is kept in a table inside the same database; in production it must be stored outside.

- S1 creates a table Reg dot Scratch, with ScratchID, an integer primary key, and Note, a nullable NVARCHAR of one hundred, and ends WITH LEDGER equals OFF.
- S2 creates a table Reg dot Digest with one column, DigestJson, NVARCHAR MAX, not null. Then it runs INSERT INTO Reg dot Digest EXEC sys dot sp underscore generate underscore database underscore ledger underscore digest, and then selects DigestJson from the table."

**Piece 6, S3 and S4.** "S3 declares a variable at d as NVARCHAR MAX, loads it with the DigestJson from Reg dot Digest, and runs EXEC sys dot sp underscore verify underscore database underscore ledger with that variable. S4 first runs ALTER DATABASE BondRegistry SET ALLOW underscore SNAPSHOT underscore ISOLATION ON, then a GO, then repeats exactly the same verify call as S3, with the stored digest."

**Piece 7, S5 and S6.** "S5 inserts two new coupons: CouponID 11, BondID 1, amount twenty five; and CouponID 12, BondID 2, amount one hundred twenty five. Then, in the same batch, it verifies again with the old stored digest. S6 loads the stored digest, extracts the hash property with JSON underscore VALUE, changes one hexadecimal digit of that hash, the third character, from A to B or to A if it was not A, puts the changed hash back into the JSON with REPLACE, and runs the verify procedure with that altered digest."

**Piece 8, S7 to S10.** "S7 runs ALTER DATABASE BondRegistry SET LEDGER equals OFF. S8 looks up, in sys dot tables, the name of the history table that the engine created for Reg dot Bond, builds a DROP TABLE statement for it, and executes it dynamically. S9 alters table Reg dot Bond, ALTER COLUMN FaceValue to DECIMAL fourteen comma two, not null. S10 runs SELECT BondID, Issuer INTO Reg dot BondCopy FROM Reg dot Bond."

**Piece 9, the final queries.** "After the ten statements, four queries run.

- Q1 selects block underscore id from sys dot database underscore ledger underscore blocks, ordered by block id.
- Q2 executes sys dot sp underscore generate underscore database underscore ledger underscore digest again.
- Q3 joins sys dot database underscore ledger underscore blocks to sys dot database underscore ledger underscore transactions on block id, and returns, per block, the block id, a count of transactions, and the word NULL if the previous block hash is null, otherwise the word hash. Ordered by block id.
- Q4 lists every table that is not a history table, from sys dot tables, with its ledger type description, the name of its history table if any, and the name of a ledger view named table name plus underscore Ledger if one exists. Ordered by table name."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE BondRegistry WITH LEDGER = ON;
GO
USE BondRegistry;
GO
CREATE SCHEMA Reg;
GO
CREATE TABLE Reg.Coupon
(
    CouponID INT           NOT NULL PRIMARY KEY,
    BondID   INT           NOT NULL,
    Amount   DECIMAL(10,2) NOT NULL
)
WITH (LEDGER = ON (APPEND_ONLY = ON));
GO
-- Note: no WITH clause at all
CREATE TABLE Reg.Bond
(
    BondID    INT           NOT NULL PRIMARY KEY,
    Issuer    NVARCHAR(60)  NOT NULL,
    FaceValue DECIMAL(12,2) NOT NULL
);
GO
INSERT INTO Reg.Bond (BondID, Issuer, FaceValue)
VALUES (1, N'City of Lakeside', 1000.00), (2, N'Port Authority', 5000.00);
INSERT INTO Reg.Coupon (CouponID, BondID, Amount) VALUES (10, 1, 25.00);
UPDATE Reg.Bond SET FaceValue = 1100.00 WHERE BondID = 1;
GO
-- S1
CREATE TABLE Reg.Scratch
(
    ScratchID INT           NOT NULL PRIMARY KEY,
    Note      NVARCHAR(100) NULL
)
WITH (LEDGER = OFF);
-- S2
CREATE TABLE Reg.Digest (DigestJson NVARCHAR(MAX) NOT NULL);
INSERT INTO Reg.Digest EXEC sys.sp_generate_database_ledger_digest;
SELECT DigestJson FROM Reg.Digest;
-- S3
DECLARE @d NVARCHAR(MAX) = (SELECT DigestJson FROM Reg.Digest);
EXEC sys.sp_verify_database_ledger @d;
-- S4
ALTER DATABASE BondRegistry SET ALLOW_SNAPSHOT_ISOLATION ON;
GO
DECLARE @d NVARCHAR(MAX) = (SELECT DigestJson FROM Reg.Digest);
EXEC sys.sp_verify_database_ledger @d;
-- S5  (new transactions AFTER the digest, then verify with the OLD digest)
INSERT INTO Reg.Coupon (CouponID, BondID, Amount) VALUES (11, 1, 25.00), (12, 2, 125.00);
DECLARE @d NVARCHAR(MAX) = (SELECT DigestJson FROM Reg.Digest);
EXEC sys.sp_verify_database_ledger @d;
-- S6  (one hex digit of the stored hash is changed before verifying)
DECLARE @d NVARCHAR(MAX) = (SELECT DigestJson FROM Reg.Digest);
DECLARE @h NVARCHAR(100) = JSON_VALUE(@d, '$.hash');
SET @d = REPLACE(@d, @h, STUFF(@h, 3, 1, CASE WHEN SUBSTRING(@h, 3, 1) = 'A' THEN 'B' ELSE 'A' END));
EXEC sys.sp_verify_database_ledger @d;
-- S7
ALTER DATABASE BondRegistry SET LEDGER = OFF;
-- S8  (drop the history table that the engine created for Reg.Bond)
DECLARE @hist SYSNAME = (SELECT h.name FROM sys.tables AS t
                         JOIN sys.tables AS h ON h.object_id = t.history_table_id
                         WHERE t.name = 'Bond');
DECLARE @sql NVARCHAR(300) = N'DROP TABLE Reg.' + QUOTENAME(@hist);
EXEC (@sql);
-- S9
ALTER TABLE Reg.Bond ALTER COLUMN FaceValue DECIMAL(14,2) NOT NULL;
-- S10
SELECT BondID, Issuer INTO Reg.BondCopy FROM Reg.Bond;
-- Q1
SELECT block_id FROM sys.database_ledger_blocks ORDER BY block_id;
-- Q2
EXEC sys.sp_generate_database_ledger_digest;
-- Q3
SELECT b.block_id, COUNT(t.transaction_id) AS transactions,
       CASE WHEN b.previous_block_hash IS NULL THEN 'NULL' ELSE 'hash' END AS previous_block_hash
FROM sys.database_ledger_blocks AS b
JOIN sys.database_ledger_transactions AS t ON t.block_id = b.block_id
GROUP BY b.block_id, b.previous_block_hash
ORDER BY b.block_id;
-- Q4
SELECT t.name, t.ledger_type_desc, h.name AS history_table, v.name AS ledger_view
FROM sys.tables AS t
LEFT JOIN sys.tables AS h ON h.object_id = t.history_table_id
LEFT JOIN sys.views  AS v ON v.ledger_view_type = 1 AND v.name = t.name + '_Ledger'
WHERE t.ledger_type_desc <> 'HISTORY_TABLE'
ORDER BY t.name;
```

## 4. The question (ask exactly this)

"For each of the ten statements, S1 to S10, tell me whether it succeeds or raises an error. For S2 to S6, also tell me what the statement returns. Let's go one at a time, starting with S1."

After all ten: "Now the four final queries, one at a time. Q1: which block ids are listed? Q2: what does the digest report now, and what side effect does running it have? Q3: for each block, how many transactions and is the previous block hash null? Q4: for each of the tables, what is its ledger type, does it have a history table, and does it have a ledger view?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

| Stmt | Outcome | Detail |
|---|---|---|
| S1 | Fails, error 37420 | LEDGER = OFF cannot be specified for tables in databases that were created with LEDGER = ON |
| S2 | Succeeds | Returns one JSON row with database_name BondRegistry, block_id 0, a hash, last_transaction_commit_time and digest_time. Reg.Digest itself becomes an updatable ledger table |
| S3 | Fails, error 37498 | Snapshot isolation must be enabled on the database when sp_verify_database_ledger is executed |
| S4 | Succeeds | Message "Ledger verification successfully verified up to block 0", plus a one-row result set last_verified_block_id = 0, return code 0 |
| S5 | Succeeds | Same as S4: verified up to block 0, last_verified_block_id = 0. The two new coupons are not covered by the old digest but do not invalidate it |
| S6 | Fails, errors 37368 and 37392 | The hash of block 0 in the database ledger does not match the hash provided in the digest; Ledger verification failed; return code 1 |
| S7 | Fails, error 102 | Incorrect syntax near LEDGER. There is no such database option; a ledger database can never be converted back |
| S8 | Fails, error 37386 | Cannot drop object Reg.MSSQL_LedgerHistoryFor_<object_id> because it is a ledger history table or a ledger view |
| S9 | Fails, error 37391 | ALTER TABLE ALTER COLUMN failed for table Bond because it is a ledger table and the operation would need to modify existing immutable data |
| S10 | Succeeds | Reg.BondCopy is created as an updatable ledger table with its own history table and a Reg.BondCopy_Ledger view |

Q1: one row, block_id 0. Everything after the S2 digest is still in the open block.

Q2: one JSON row that now reports block_id 1, with a new hash, last_transaction_commit_time and digest_time. Generating the digest closes the open block.

Q3, after Q2:

| block_id | transactions | previous_block_hash |
|---|---|---|
| 0 | 6 | NULL |
| 1 | 4 | hash |

Block 0 holds: CREATE TABLE Coupon, CREATE TABLE Bond, the Bond insert, the Coupon insert, the Bond update, CREATE TABLE Digest. Block 1 holds: the insert of the digest row, the S5 coupon insert, and the CREATE plus INSERT of SELECT INTO.

Q4:

| name | ledger_type_desc | history_table | ledger_view |
|---|---|---|---|
| Bond | UPDATABLE_LEDGER_TABLE | MSSQL_LedgerHistoryFor_<object_id> | Bond_Ledger |
| BondCopy | UPDATABLE_LEDGER_TABLE | MSSQL_LedgerHistoryFor_<object_id> | BondCopy_Ledger |
| Coupon | APPEND_ONLY_LEDGER_TABLE | NULL | Coupon_Ledger |
| Digest | UPDATABLE_LEDGER_TABLE | MSSQL_LedgerHistoryFor_<object_id> | Digest_Ledger |

Scratch does not appear: S1 failed. The numeric suffixes are object ids and differ per run.

## 6. Hint ladder (one hint per attempt, in order)

**S1**
1. "How was the database created? What does that option promise about every table inside it?"
2. "If every table must be a ledger table, can a CREATE TABLE opt out with LEDGER equals OFF?"

**S2**
1. "Nothing here is forbidden. What does the generate digest procedure return, and how many rows?"
2. "It returns one JSON document. Name the properties: the database name, a block id, a hash, and two times. Which block id, if no digest has been generated before?"
3. "Bonus: the Digest table has no WITH clause. What kind of table does it become in this database?"

**S3**
1. "The verify procedure needs to read a consistent snapshot of the whole database while others keep working. Which database option gives it that?"
2. "Was that option turned on before S3? Look at what S4 does first."

**S4**
1. "Now the option is on. The digest is genuine and nothing has changed since it was taken. What does a successful verification look like?"
2. "It prints a message about verifying up to a block, and returns a result set with one column. Which block?"

**S5**
1. "Two new rows were committed after the digest. Does the old digest become wrong, or just incomplete?"
2. "Verification checks that what the digest covers is unchanged. Block 0 is unchanged. So what is the outcome, and up to which block?"

**S6**
1. "One hex digit of the hash was changed. The engine recomputes the hash of block 0 from the data. Does it match the digest you gave it?"
2. "It does not match. Is a mismatch reported as a warning or as errors? How many messages?"

**S7**
1. "Think about whether the LEDGER property of a database is reversible at all."
2. "There is no such SET option, so the parser never gets past the word LEDGER. What kind of error is that?"

**S8**
1. "The history table was created by the engine for Bond. Who is allowed to drop the history table of a ledger table?"
2. "Nobody. What would deleting history do to the proof? So the engine refuses."

**S9**
1. "Changing a column's data type rewrites every existing value. What does a ledger table promise about existing data?"
2. "Compare with widening NVARCHAR sixty to eighty, which does not rewrite the stored bytes. Is a DECIMAL twelve to fourteen change the same kind of thing?"

**S10**
1. "SELECT INTO creates a new table. In this database, what does every new table become?"
2. "Is there anything in a SELECT INTO that could conflict with ledger? No. So does it succeed, and what extras does the new table get?"

**Q1**
1. "When is a block closed: on every transaction, or when a digest is generated?"
2. "How many digests have been generated so far? Q2 has not run yet."

**Q2**
1. "Generating a digest closes the block that is currently open. What is the id of the block that was just closed?"

**Q3**
1. "Count the transactions that touched ledger tables before the S2 digest: two CREATE TABLEs, three seed statements, and the CREATE of the Digest table. Remember each seed statement is its own transaction."
2. "Now count after the digest: the digest row insert, the S5 insert, and SELECT INTO which counts as a CREATE plus an INSERT. Failed statements do not count."
3. "The first block has nothing before it. What is its previous block hash?"

**Q4**
1. "Which tables actually exist? Remember which CREATEs failed."
2. "Coupon asked for append-only. What does an append-only ledger table not need, that an updatable one has?"
3. "Bond, Digest and BondCopy never asked for anything. What does the ledger database give them by default?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "S1 succeeds, you can always opt a table out" | Thinks ledger database is just a default | "Re-read the CREATE DATABASE option. Is it a default or a guarantee?" |
| "S3 succeeds, the digest is fresh" | Forgets the snapshot isolation prerequisite | "Before checking the digest, what does the verify procedure need from the database itself?" |
| "S5 fails, the data changed after the digest" | Thinks a digest becomes invalid when new transactions arrive | "Was anything the digest covers changed? Or were new things simply added afterwards?" |
| "S6 succeeds with a warning" | Thinks a mismatch is soft | "A digest mismatch is exactly what ledger exists to catch. Would the engine report that softly?" |
| "S7 succeeds, ledger is switched off" | Believes the property is reversible | "Look for that option in the ALTER DATABASE syntax. Does it exist?" |
| "S8 succeeds because the drop is dynamic SQL" | Thinks dynamic SQL bypasses protection | "Dynamic SQL is just SQL. What protects the history table?" |
| "S9 succeeds, it is only widening" | Confuses a width change with a type change | "Widening NVARCHAR leaves the stored bytes alone. Does changing DECIMAL precision?" |
| "S10 creates a plain table" | Thinks implicit table creation escapes the ledger rule | "SELECT INTO still creates a table. What rule applies to every new table here?" |
| "Q1 lists blocks 0 and 1" | Thinks blocks close with each transaction | "What closes a block? How many times has that happened so far?" |
| "Q4 includes Scratch" | Forgot S1 failed | "Did S1 succeed?" |
| "Coupon has a history table" | Confuses append-only with updatable | "Which ledger type keeps old row versions? Does an append-only table ever have old versions?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain that a ledger database is the "protect everything" option, and walk through the four ideas:

- **In a ledger database every table is a ledger table.** CREATE DATABASE WITH LEDGER = ON makes every new table a ledger table, updatable by default, with an engine-named history table MSSQL_LedgerHistoryFor_<object_id> and a ledger view named table name underscore Ledger. You may still ask for APPEND_ONLY = ON, as Coupon does, and then there is no history table. The one thing you cannot ask for is LEDGER = OFF, error 37420. The rule also catches implicit creation: SELECT INTO and the little Digest helper table both become full updatable ledger tables. Temporary tables and table variables live in tempdb and are unaffected. The property can never be removed; there is no SET LEDGER = OFF, which is why S7 is a syntax error. That is S1, S7, S10 and Q4.
- **The digest is the hash of the latest block.** sp_generate_database_ledger_digest takes no parameters, returns one JSON row with database_name, block_id, hash, last_transaction_commit_time and digest_time, and closes the current block. Every transaction that touched a ledger table, DDL included, is a row in sys.database_ledger_transactions with its block id and per-table hashes; each closed block is a row in sys.database_ledger_blocks with a transactions root hash and the previous block's hash, which chains the blocks. Transactions accumulate in an open block until a digest is generated, or, with automatic digest storage, about every 30 seconds, or at 100 thousand transactions. That is S2, Q1, Q2 and Q3.
- **Store the digest where a DBA cannot rewrite it.** Keeping it in a table in the same database proves nothing: an attacker who can rewrite data can regenerate a matching digest. The supported destinations are Azure Blob Storage with a locked immutability policy, in a container named sqldbledgerdigests, and Azure Confidential Ledger, or a WORM device for manual digests. In Azure it is switched on from the portal under Security, Ledger, Enable automatic digest storage, or with az sql db ledger-digest-uploads enable, or Enable-AzSqlDatabaseLedgerDigestUpload; the server's managed identity needs Storage Blob Data Contributor on the storage account or Contributor on the Confidential Ledger. On SQL Server it is ALTER DATABASE SCOPED CONFIGURATION SET LEDGER_DIGEST_STORAGE_ENDPOINT with a matching CREATE CREDENTIAL, and only Azure Storage is supported there. When automatic storage is on, a digest is uploaded every 30 seconds if there were transactions, and the locations appear in sys.database_ledger_digest_locations. Generating a digest needs the GENERATE LEDGER DIGEST permission; viewing or verifying the ledger needs VIEW LEDGER CONTENT.
- **Verification recomputes every hash.** sp_verify_database_ledger takes a JSON document with one or more digests, rehashes every ledger and history row, rebuilds each block and the chain, and compares. It needs ALLOW_SNAPSHOT_ISOLATION ON, error 37498 otherwise; success is a message plus a result set with last_verified_block_id, return code 0. Because it rehashes everything it is resource-intensive, so schedule it off-peak with SQL Agent or Elastic Jobs. With automatic storage use sp_verify_database_ledger_from_digest_storage with the locations JSON; passing NULL fails with 37365. A mismatch, whether from a changed digest or from tampered data, gives 37368 plus 37392 and return code 1. The engine does not check the database_name field in the digest, and it checks nothing outside ledger and history tables. That is S3, S4 and S6.
- **A digest attests what it covers, not what comes later.** After S5's two inserts the old digest still verifies up to block 0. Later transactions are simply not attested yet; the next digest covers them. That is why digests must be produced regularly and verification scheduled, not run once.
- **What tampering is and is not detected.** Detected: any change to ledger rows, history rows or append-only rows that bypasses the engine, including a restored older copy presented as current or a deleted history row; dropped tables and columns are only renamed, so their data still takes part. Not prevented, only detected: ledger is tamper-evident, not tamper-proof; recovery means restoring a backup and re-verifying. Not detected: tampering with digests kept in an unprotected place, transactions after the last stored digest, and confidentiality breaches, since ledger encrypts nothing.
- **What a ledger table refuses.** Dropping or modifying history tables and ledger views, errors 37386, 37361 and 37395. Changing a column's data type, 37391, while widening NVARCHAR, changing nullability or collation is fine because existing bytes do not change. Adding a column is allowed only if it is nullable with no default, otherwise 37387. Also unsupported: TRUNCATE TABLE, partition SWITCH, DBCC CLONEDATABASE, in-memory tables, full-text, graph and FileTables, transactional replication, and the xml, sql_variant, vector, user-defined and FILESTREAM types. sp_rename is allowed and is recorded in sys.ledger_table_history. That is S8 and S9.

Memory hook: "Ledger tables answer, was this table tampered with. A ledger database answers, was anything tampered with. Both are only as trustworthy as where you keep the digests, and both only detect tampering after the fact."

## 9. Follow-up oral questions (optional)

1. "S9 fails. Would ALTER COLUMN Issuer NVARCHAR eighty NOT NULL also fail?" (No. Widening NVARCHAR does not rewrite existing data, so it succeeds.)
2. "Where must ledger digests be stored in production, and why not in the database?" (Azure Blob Storage with a locked immutability policy, Azure Confidential Ledger, or a WORM device; anyone who can rewrite the data could otherwise regenerate a matching digest.)
3. "Coupon has no history table. Why?" (It is append-only; rows are never updated or deleted, so there are no old versions to keep. It still has the Coupon_Ledger view.)

## 10. References

- Ledger overview: https://learn.microsoft.com/en-us/sql/relational-databases/security/ledger/ledger-overview
- Ledger database: https://learn.microsoft.com/en-us/sql/relational-databases/security/ledger/ledger-database
- Updatable ledger tables: https://learn.microsoft.com/en-us/sql/relational-databases/security/ledger/ledger-updatable-ledger-tables
- Append-only ledger tables: https://learn.microsoft.com/en-us/sql/relational-databases/security/ledger/ledger-append-only-ledger-tables
- Digest management and automatic digest storage: https://learn.microsoft.com/en-us/sql/relational-databases/security/ledger/ledger-digest-management
- Verify a ledger database: https://learn.microsoft.com/en-us/sql/relational-databases/security/ledger/ledger-verify-database
- sys.sp_generate_database_ledger_digest: https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sys-sp-generate-database-ledger-digest-transact-sql
- sys.sp_verify_database_ledger: https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sys-sp-verify-database-ledger-transact-sql
- sys.database_ledger_blocks and sys.database_ledger_transactions: https://learn.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-database-ledger-blocks-transact-sql and https://learn.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-database-ledger-transactions-transact-sql
- Ledger limitations: https://learn.microsoft.com/en-us/sql/relational-databases/security/ledger/ledger-limits
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
