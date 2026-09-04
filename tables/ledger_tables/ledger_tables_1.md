# SQL Server question — Ledger Tables 1

## Statement

`GrantLedger` is the database of a charitable foundation that must prove to its auditors that grant records have never been silently altered. The team creates two **ledger tables**: awards can legitimately be corrected (an *updatable* ledger table), while individual payouts must be immutable once written (an *append-only* ledger table).

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
```

The following eight statements are then executed **in order, each in its own batch**, in a single session:

```sql
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

For each statement S1–S8, state whether it **succeeds or raises an error**. Then give the exact result of these three final queries:

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

## Correct Answer

Per-statement outcomes (all error numbers and messages are the engine's actual output):

| Stmt | Outcome | Detail |
|------|---------|--------|
| S1 | **Succeeds** | 1 row updated; the old version moves to `Fund.Award_History` |
| S2 | **Fails** | `Msg 37359` — `Updates are not allowed for the append only Ledger table 'Fund.Payout'.` |
| S3 | **Fails** | `Msg 37359` (State 3) — same text: `Updates are not allowed for the append only Ledger table 'Fund.Payout'.` |
| S4 | **Succeeds** | 1 row deleted; the deleted version moves to `Fund.Award_History` |
| S5 | **Fails** | `Msg 13545` — `Truncate failed on table 'GrantLedger.Fund.Award' because it is not a supported operation on system-versioned tables.` |
| S6 | **Fails** | `Msg 37356` — `System Versioning cannot be altered for Ledger Tables.` |
| S7 | **Succeeds** | The column is not physically removed: it is renamed `MSSQL_DroppedLedgerColumn_Amount_<GUID>` and stays in the table |
| S8 | **Succeeds** | The table is not physically removed: it is renamed `MSSQL_DroppedLedgerTable_Payout_<GUID>` (and its ledger view `MSSQL_DroppedLedgerView_Payout_Ledger_<GUID>`) |

Q1 — `SELECT *` shows only the user columns; the four `GENERATED ALWAYS` ledger columns are hidden:

| AwardID | Charity | Amount |
|---------|---------|--------|
| 1 | River Trust | 5500.00 |

Q2 — the ledger view (three committed transactions):

| AwardID | Charity | Amount | TxnRank | Seq | Op |
|---------|---------|--------|---------|-----|--------|
| 1 | River Trust | 5000.00 | 1 | 0 | INSERT |
| 2 | Book Aid | 1200.00 | 1 | 1 | INSERT |
| 1 | River Trust | 5500.00 | 2 | 0 | INSERT |
| 1 | River Trust | 5000.00 | 2 | 1 | DELETE |
| 2 | Book Aid | 1200.00 | 3 | 0 | DELETE |

Q3 — catalog:

| name | ledger_type_desc | is_dropped_ledger_table |
|------|------------------|-------------------------|
| Award | UPDATABLE_LEDGER_TABLE | 0 |
| Award_History | HISTORY_TABLE | 0 |
| MSSQL_DroppedLedgerTable_Payout_4380ACC1FB6040F586F1B8C412AE856B | APPEND_ONLY_LEDGER_TABLE | 1 |

(The GUID suffix differs per run.)

## Explanation

Ledger tables make the history of a table **tamper-evident**: every transaction is hashed into a blockchain-like structure (`sys.database_ledger_transactions`, `sys.database_ledger_blocks`) whose digests can be stored outside the database and later verified with `sys.sp_verify_database_ledger`. To keep that guarantee, the engine blocks every operation that would erase evidence. Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

### Two kinds of ledger table

- **Updatable ledger table** (`SYSTEM_VERSIONING = ON ... LEDGER = ON`): `UPDATE` and `DELETE` are allowed, but every previous row version is kept in the **history table** and the table gets four hidden `GENERATED ALWAYS` columns — `ledger_start_transaction_id`, `ledger_end_transaction_id`, `ledger_start_sequence_number`, `ledger_end_sequence_number` (`sys.columns.is_hidden = 1`, `generated_always_type_desc = AS_TRANSACTION_ID_START` etc.). That is why S1 and S4 succeed and why Q1 shows only three columns.
- **Append-only ledger table** (`LEDGER = ON (APPEND_ONLY = ON)`): only `INSERT` is allowed; there is no history table and only the two `_start_` columns exist. S2 and S3 both fail with error 37359 (the message says "Updates" even for the `DELETE`, which surfaces as State 3).

Both kinds get an automatically created **ledger view** (`Fund.Award_Ledger`, `Fund.Payout_Ledger`, `sys.views.ledger_view_type_desc = LEDGER_VIEW`) that unions the current and history rows and exposes `ledger_transaction_id`, `ledger_sequence_number`, `ledger_operation_type` (1 = INSERT, 2 = DELETE) and `ledger_operation_type_desc`.

### Q2 — an UPDATE is recorded as DELETE + INSERT

This is the subtle part. The ledger view never shows an "UPDATE" operation: S1 appears in transaction 2 as an `INSERT` of the new version (5500.00, sequence 0) **and** a `DELETE` of the old version (5000.00, sequence 1). Transaction 1 holds the two seed inserts, transaction 3 the delete of award 2. The history table therefore contains exactly the two superseded versions (award 1 at 5000.00 and award 2 at 1200.00), while the base table keeps only the live row. `sys.database_ledger_transactions` lists every one of these transactions with `commit_time`, `principal_name` and a `table_hashes` value.

### S5 and S6 — nothing may erase history

`TRUNCATE TABLE` is refused on any system-versioned table (error 13545), and, unlike an ordinary temporal table, an updatable ledger table can never have its system versioning switched off (error 37356) — that would let someone delete history rows. `LEDGER = ON` itself cannot be turned off either.

### S7 and S8 — "drop" means "rename and keep"

Both statements succeed, but neither destroys anything. A dropped ledger column is renamed `MSSQL_DroppedLedgerColumn_<name>_<GUID>` and stays in the table; a dropped ledger table is renamed `MSSQL_DroppedLedgerTable_<name>_<GUID>` with `sys.tables.is_dropped_ledger_table = 1`, and its ledger view is renamed `MSSQL_DroppedLedgerView_...`. The auditor can still verify the dropped data. The only way to physically remove a ledger table is to drop the entire database.

### Equivalent alternatives

- `ALTER TABLE Fund.Award SET (LEDGER = OFF)` is not valid syntax at all (error 102).
- Adding a column with `ALTER TABLE ... ADD` is allowed on both kinds of ledger table; only *removing* evidence is blocked.

## DP-800 Exam Rule to Remember

```text
LEDGER = ON (APPEND_ONLY = ON)        -> INSERT only (UPDATE/DELETE: Msg 37359)
                                         2 hidden columns, no history table
SYSTEM_VERSIONING = ON, LEDGER = ON   -> updatable; 4 hidden GENERATED ALWAYS columns
                                         history table keeps every old version
                                         UPDATE shows in the ledger view as DELETE + INSERT
Never allowed : TRUNCATE (13545), SYSTEM_VERSIONING = OFF (37356), LEDGER = OFF
"Drop" = rename : MSSQL_DroppedLedgerColumn_* / MSSQL_DroppedLedgerTable_* /
                  MSSQL_DroppedLedgerView_*  (sys.tables.is_dropped_ledger_table = 1)
Verification  : sys.database_ledger_transactions, sys.database_ledger_blocks,
                digest storage + sys.sp_verify_database_ledger
```

Pick **append-only** for immutable facts (payments, audit events, sensor readings); pick **updatable** when business corrections are legitimate but must remain provable. Neither kind lets anyone — including `db_owner` — destroy history without leaving a trace.
