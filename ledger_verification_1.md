# SQL Server question — Ledger Verification 1

## Statement

`BondRegistry` records municipal bonds and their coupon payments for a registrar that must prove to regulators that **nothing in the database** — not just a few tables — has been altered outside the normal application path. The team therefore creates a **ledger database** instead of individual ledger tables:

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
```

The following ten statements are then executed **in order, each in its own batch**, in a single session. (For the exercise the digest is kept in a table inside the same database; in production it must be stored outside the database — see the explanation.)

```sql
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
```

For each statement S1–S10, state whether it **succeeds or raises an error** (and what S2–S6 return). Then give the exact result of these four final queries:

```sql
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

## Correct Answer

Per-statement outcomes (all error numbers and messages are the engine's actual output):

| Stmt | Outcome | Detail |
|------|---------|--------|
| S1 | **Fails** | `Msg 37420` — `LEDGER = OFF cannot be specified for tables in databases that were created with LEDGER = ON.` |
| S2 | **Succeeds** | Returns one JSON row, e.g. `{"database_name":"BondRegistry","block_id":0,"hash":"0x581FF247...D03B","last_transaction_commit_time":"2026-09-04T10:00:15.2600000","digest_time":"2026-09-04T08:00:15.7510170"}` (hash and times differ per run). `Reg.Digest` itself is created as an updatable ledger table. |
| S3 | **Fails** | `Msg 37498` — `Snapshot isolation must be enabled on the database when sp_verify_database_ledger is executed.` |
| S4 | **Succeeds** | Message `Ledger verification successfully verified up to block 0.` plus a one-row result set `last_verified_block_id = 0` (return code 0) |
| S5 | **Succeeds** | Same output as S4: `... verified up to block 0.`, `last_verified_block_id = 0` — the two new coupon rows are not covered by the old digest, but they do not invalidate it |
| S6 | **Fails** | `Msg 37368` — `The hash of block 0 in the database ledger does not match the hash provided in the digest for this block.` followed by `Msg 37392` — `Ledger verification failed.` (return code 1) |
| S7 | **Fails** | `Msg 102` — `Incorrect syntax near 'LEDGER'.` — there is no such database option; a ledger database can never be converted back |
| S8 | **Fails** | `Msg 37386` — `Cannot drop object 'Reg.MSSQL_LedgerHistoryFor_1269579561' because it is a ledger history table or a ledger view.` (the numeric suffix is the object_id of `Reg.Bond`) |
| S9 | **Fails** | `Msg 37391` — `ALTER TABLE ALTER COLUMN failed for table 'Bond' because it is a ledger table and the operation would need to modify existing data that is immutable.` |
| S10 | **Succeeds** | `Reg.BondCopy` is created — as an **updatable ledger table** with its own history table and `Reg.BondCopy_Ledger` view |

Q1 — only the block closed by the S2 digest exists; everything after it is still in the open block:

| block_id |
|----------|
| 0 |

Q2 — generating a digest closes the open block, so the JSON now reports `"block_id":1` (with a new `hash`, `last_transaction_commit_time` and `digest_time`).

Q3 — after Q2 there are two blocks:

| block_id | transactions | previous_block_hash |
|----------|--------------|---------------------|
| 0 | 6 | NULL |
| 1 | 4 | hash |

Q4 — every table in the database is a ledger table:

| name | ledger_type_desc | history_table | ledger_view |
|------|------------------|---------------|-------------|
| Bond | UPDATABLE_LEDGER_TABLE | MSSQL_LedgerHistoryFor_1269579561 | Bond_Ledger |
| BondCopy | UPDATABLE_LEDGER_TABLE | MSSQL_LedgerHistoryFor_1381579960 | BondCopy_Ledger |
| Coupon | APPEND_ONLY_LEDGER_TABLE | NULL | Coupon_Ledger |
| Digest | UPDATABLE_LEDGER_TABLE | MSSQL_LedgerHistoryFor_1333579789 | Digest_Ledger |

(The numeric suffixes are object_ids and differ per run.)

## Explanation

A ledger **database** is the "protect everything" option: `CREATE DATABASE ... WITH LEDGER = ON` makes every table in it a ledger table, and the property can never be removed. Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

### S1, S10, Q4 — in a ledger database every table is a ledger table

`Reg.Bond` was created with no `WITH` clause at all, yet Q4 shows it as `UPDATABLE_LEDGER_TABLE` with an engine-named history table `MSSQL_LedgerHistoryFor_<object_id>` and a ledger view `Reg.Bond_Ledger`. That is the documented default: "all new tables created by default (without specifying the `APPEND_ONLY = ON` clause) in the database are updatable ledger tables". You may still ask for `LEDGER = ON (APPEND_ONLY = ON)` (as `Reg.Coupon` does), but the one thing you cannot ask for is `LEDGER = OFF` — S1 fails with error 37420. The rule also catches statements that create tables implicitly: `SELECT ... INTO Reg.BondCopy` (S10) succeeds and produces a full updatable ledger table with history table and ledger view, and the little `Reg.Digest` helper table of S2 becomes one too. (Temporary tables and table variables live in `tempdb` and are unaffected.)

### S2 — the digest is the hash of the latest block

`sys.sp_generate_database_ledger_digest` takes no parameters and returns one JSON row with `database_name`, `block_id`, `hash`, `last_transaction_commit_time` and `digest_time`; running it **closes the current block**. Every transaction that touched a ledger table — DDL included — is a row in `sys.database_ledger_transactions` with its `block_id`, `transaction_ordinal`, `commit_time`, `principal_name` and per-table Merkle-tree root in `table_hashes`; each closed block is a row in `sys.database_ledger_blocks` with `transactions_root_hash` and `previous_block_hash`, which chains the blocks. That is why Q3 shows block 0 with six transactions (`CREATE TABLE Reg.Coupon`, `CREATE TABLE Reg.Bond`, the two seed `INSERT`s, the `UPDATE`, and `CREATE TABLE Reg.Digest`), and block 1 — closed only by Q2 — with four (the `INSERT` of the digest row, the S5 coupon insert, and the `CREATE`+`INSERT` of `SELECT INTO`). Block 0 has `previous_block_hash = NULL`; block 1 carries the hash of block 0. Before Q2 runs, Q1 lists block 0 only: transactions accumulate in an open block until a digest is generated (or, with automatic digest storage, about every 30 seconds, or when the block reaches 100 K transactions).

Because the digest represents the state of the database, **it must be stored where a database administrator cannot rewrite it** — that is the whole trust model. Storing it in `Reg.Digest` inside the same database (as the exercise does) proves nothing to an auditor: an attacker who can rewrite the data can regenerate a matching digest and overwrite the table. The supported destinations are **Azure Blob Storage with a locked immutability policy** (container `sqldbledgerdigests`, path `ServerName/DatabaseName/CreationTime`) and **Azure Confidential Ledger**, or a WORM device of your choice for manual digests. Automatic generation and upload is switched on in the Azure portal (**Security > Ledger > Enable automatic digest storage**), `az sql db ledger-digest-uploads enable --endpoint ...` or `Enable-AzSqlDatabaseLedgerDigestUpload`, with the server's system-assigned identity needing **Storage Blob Data Contributor** (blob) or **Contributor** (Confidential Ledger). On SQL Server the same is `ALTER DATABASE SCOPED CONFIGURATION SET LEDGER_DIGEST_STORAGE_ENDPOINT = 'https://<account>.blob.core.windows.net'` with a matching `CREATE CREDENTIAL` (shared access signature or, from SQL Server 2022 CU17 / SQL Server 2025 on Azure VMs and Arc, managed identity); only Azure Storage is supported there, not Confidential Ledger. When automatic storage is on, a digest is uploaded every 30 seconds *if* transactions occurred, and the configured locations appear in `sys.database_ledger_digest_locations`. Generating a digest requires the `GENERATE LEDGER DIGEST` permission; viewing or verifying the ledger requires `VIEW LEDGER CONTENT`.

### S3–S6 — what verification checks

`sys.sp_verify_database_ledger @digests [, @table_name]` takes a JSON document with one or more digests, recomputes the SHA-256 hashes of every ledger and history table row, rebuilds each block hash and the chain, and compares the block named in each digest with the hash you supplied. It needs **`ALLOW_SNAPSHOT_ISOLATION ON`** so that it can read a consistent snapshot while users keep working — S3 fails with error 37498 until S4 enables it. Success is reported as a message plus a result set with `last_verified_block_id`, and return code 0.

S5 is the subtle one: after two more inserts, the **old** digest still verifies. Verification proves that the data covered by the digest (block 0 and everything hashed into it) is unchanged; transactions committed after the digest are simply not yet attested — they will be covered by the next digest. That is why digests must be produced regularly (automatic storage every 30 seconds) and why the recommendation is to schedule verification, for example with SQL Server Agent or Elastic Jobs, off-peak: the recomputation is resource-intensive because it rehashes everything. With automatic storage you verify with `sys.sp_verify_database_ledger_from_digest_storage @locations`, passing `SELECT * FROM sys.database_ledger_digest_locations FOR JSON AUTO, INCLUDE_NULL_VALUES`; passing `NULL` (no storage configured) fails with error 37365, `The '@locations' parameter provided for ledger verification cannot be null.`

S6 shows a mismatch: a single changed hex digit yields 37368 + 37392 and return code 1. The same pair appears when the *data* has been tampered with — the recomputed hash then differs from the genuine digest. Two things the engine observably does **not** check: the `database_name` field (a digest with a wrong database name still verifies, so filenames and digest locations must be managed carefully) and anything outside ledger and history tables.

### What tampering is — and is not — detected

- **Detected**: any change to the rows of a ledger table, its history table or an append-only table that bypasses the engine (editing the data files, a restored older copy presented as current, a deleted history row) — the recomputed hashes no longer match the stored digest. Dropped tables and columns are only renamed, so their data still takes part in verification.
- **Not prevented, only detected**: ledger is *tamper-evident*, not tamper-proof; an administrator with file access can still alter data, and recovery means restoring a backup and re-verifying.
- **Not detected**: tampering with digests that are stored in an unprotected place (the attacker regenerates them), transactions committed after the last stored digest (covered only by the next one), and confidentiality breaches — ledger does not encrypt anything.

### S7, S8, S9 — statements a ledger database refuses

- There is no `ALTER DATABASE ... SET LEDGER = OFF`; S7 is a syntax error (102). "A ledger database ... can't be converted to a regular database."
- History tables and ledger views cannot be dropped (37386) or modified (`DELETE` from a history table: 37361; `UPDATE` through the ledger view: 37395, `View 'BondRegistry.Reg.Bond_Ledger' is not updatable because it is a ledger view.`).
- Schema changes that would rewrite existing data are blocked: changing a column's data type fails with 37391, while widening `NVARCHAR(60)` to `NVARCHAR(80)`, changing nullability or collation succeeds because the hashes of existing rows do not change. Adding columns is allowed only when they are nullable and have no default: `ADD Rating CHAR(3) NOT NULL DEFAULT 'AAA'` fails with 37387, `Only nullable columns without a default value WITH VALUES can be added to ledger tables.`
- Also unsupported on ledger tables: `TRUNCATE TABLE`, `SWITCH` partitions, `DBCC CLONEDATABASE`, in-memory tables, full-text indexes, graph and FileTables, transactional replication, and the `xml`, `sql_variant`, `vector`, user-defined and `FILESTREAM` types. `sp_rename` on a ledger table *is* allowed and is recorded as a `RENAME` row in `sys.ledger_table_history` (its `_Ledger` view keeps the old name).

## DP-800 Exam Rule to Remember

```text
CREATE DATABASE db WITH LEDGER = ON   -> every table is a ledger table (default: updatable,
                                         history MSSQL_LedgerHistoryFor_<object_id>, view <t>_Ledger)
                                         LEDGER = OFF on a table: Msg 37420; no way back (no SET LEDGER = OFF)
Digest  = hash of the latest block    -> sys.sp_generate_database_ledger_digest (closes the block)
          store OUTSIDE the database  -> Azure Blob immutable (locked policy) / Azure Confidential Ledger /
                                         WORM; automatic upload every 30 s if there were transactions;
                                         sys.database_ledger_digest_locations
Verify  = recompute every hash        -> sys.sp_verify_database_ledger @digests  (manual digests)
                                         sys.sp_verify_database_ledger_from_digest_storage @locations
                                         needs ALLOW_SNAPSHOT_ISOLATION ON (37498)
                                         OK: "verified up to block N" / last_verified_block_id, return 0
                                         tampered: 37368 + 37392, return 1
Chain   = sys.database_ledger_transactions (per txn, incl. DDL) + sys.database_ledger_blocks
Blocked = drop/modify history table or ledger view (37386/37361/37395), ALTER COLUMN type (37391),
          NOT NULL / DEFAULT column adds (37387), TRUNCATE, SWITCH, CLONEDATABASE
```

Ledger tables answer "was this table tampered with?"; a ledger database answers "was **anything** tampered with?". Both are only as trustworthy as the place you keep the digests — and both only *detect* tampering after the fact, so schedule digests and verification rather than running them once.
