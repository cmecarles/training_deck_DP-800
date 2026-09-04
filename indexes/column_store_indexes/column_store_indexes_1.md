# SQL Server question — Columnstore Indexes 1

## Statement

`MartMetrics` is the analytics database of a retail chain. It receives point-of-sale data into two tables that have just been created:

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

Requirements:

1. `Pos.SaleLines` is queried almost exclusively by large aggregations (revenue by store, by day, by product) that scan most of the table. Its **only full copy of the data must be stored in columnstore format**, to get maximum (~10x) compression and batch-mode scans; the 600-million-row table must not additionally keep a complete rowstore copy of itself.
2. The engine (not the loading application) must **enforce uniqueness of `SaleLineID`**, and occasional single-line corrections must be able to fetch one row by `SaleLineID` efficiently.
3. `Pos.Payments` must keep its **rowstore base storage and clustered primary key exactly as created** (the register workload depends on cheap singleton seeks and updates through `PK_Payments`), and analysts must additionally get **real-time aggregations over the live payment rows** — no ETL copy, no change to the table's base storage.

Which indexing strategy meets all three requirements?

### a.

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

### b.

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

### c.

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

### d.

```sql
CREATE UNIQUE CLUSTERED COLUMNSTORE INDEX CCI_SaleLines
    ON Pos.SaleLines ORDER (SaleLineID);
GO
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_Payments
    ON Pos.Payments (RegisterID, PaidAt, Method, Amount);
GO
```

## Correct Answer

**b**

## Explanation

The question hinges on the exact division of labor between the two columnstore flavors:

- A **clustered columnstore index (CCI)** *is* the table: it replaces the heap/B-tree as the primary storage of every row and every column. That is what delivers the ~10x compression on the fact table — there is no second full copy anywhere.
- A **nonclustered columnstore index (NCCI)** is a *secondary, column-compressed copy* of chosen columns **on top of** a rowstore table. It exists precisely for real-time operational analytics (HTAP): OLTP keeps using the rowstore B-tree, analytics scan the columnstore copy of the same live data, no ETL.

Every claim below was verified by executing the scripts on SQL Server 2025.

### Why option b is correct

- `CREATE CLUSTERED COLUMNSTORE INDEX CCI_SaleLines` converts the heap into pure columnstore storage (requirement 1). `sys.indexes` afterwards shows `CCI_SaleLines` as `index_id = 1`, `type_desc = 'CLUSTERED COLUMNSTORE'` — the base storage itself. A bulk load of 110,000 rows lands directly as a `COMPRESSED` rowgroup (`sys.dm_db_column_store_row_group_physical_stats`: one rowgroup, `state_desc = COMPRESSED`, `total_rows = 110000`), because bulk inserts of at least 102,400 rows bypass the delta store; an aggregation over it runs with `ActualExecutionMode="Batch"` on both the columnstore scan and the hash aggregate (captured from the actual plan XML).
- Since SQL Server 2016, a table whose primary storage is a CCI **can also carry rowstore B-tree nonclustered indexes** — including unique ones. `CREATE UNIQUE NONCLUSTERED INDEX UX_SaleLines_ID` succeeds, gives the engine-enforced uniqueness plus an efficient B-tree seek path for the single-row corrections (requirement 2). Verified: inserting a second row with `SaleLineID = 42` fails with

  ```text
  Msg 2601: Cannot insert duplicate key row in object 'Pos.SaleLines'
  with unique index 'UX_SaleLines_ID'. The duplicate key value is (42).
  ```

  (Equivalent alternative: `ALTER TABLE Pos.SaleLines ADD CONSTRAINT PK_SaleLines PRIMARY KEY NONCLUSTERED (SaleLineID)` — a PK constraint backed by a *nonclustered* B-tree on top of the CCI enforces the same thing; what it must not be is `PRIMARY KEY CLUSTERED`, which would claim the clustered slot the CCI occupies.)
- `NCCI_Payments` adds the analytics copy on `Pos.Payments` while `PK_Payments` remains the rowstore clustered index (requirement 3). `sys.indexes` shows `PK_Payments  CLUSTERED` and `NCCI_Payments  NONCLUSTERED COLUMNSTORE` side by side. And since SQL Server 2016 an NCCI is **updatable**: singleton `INSERT`/`UPDATE`/`DELETE` against the table all succeeded with the NCCI in place, the columnstore copy being maintained automatically (delta store + tuple mover behind the scenes).

### Why option a is wrong

This is the subtle distractor, because everything in it *runs* and even *performs*: the PK gives uniqueness and seeks, both NCCIs give batch-mode analytics. The design it builds for `Pos.Payments` is exactly right — and that is the bait. For `Pos.SaleLines` it violates requirement 1: `PRIMARY KEY CLUSTERED` makes a rowstore B-tree the table's primary storage, and `NCCI_SaleLines` is then a *second*, additional copy of the analytic columns. The 600-million-row fact table stores its full data uncompressed-rowstore *plus* a columnstore duplicate — the opposite of "the only full copy is columnstore". A fact table that is scanned, not sought, wants the CCI as the table itself; NCCI-on-rowstore is the pattern for operational tables, not warehouse facts.

### Why option c is wrong

The `SaleLines` half is identical to option b, but the `Payments` half converts the operational table's base storage to a clustered columnstore (the script is legal and ran successfully: `PK_Payments` re-created as `NONCLUSTERED` on top of `CCI_Payments`). That directly violates requirement 3, which froze the rowstore base and clustered PK. Beyond the letter of the requirement, it is the wrong tool: columnstore base storage is optimized for scans, while the register workload is singleton seeks and updates — every point update now goes through delete-bitmap + delta-store mechanics, and every PK lookup seeks a B-tree only to fetch the row from columnstore segments. Rowstore-for-OLTP + NCCI-for-analytics is the supported HTAP pattern; CCI-for-everything is not.

### Why option d is wrong

It fails at the first statement. Verified verbatim:

```text
Msg 35301: The statement failed because a columnstore index cannot be unique.
Create the columnstore index without the UNIQUE keyword or create a unique
index without the COLUMNSTORE keyword.
```

A columnstore index has **no key columns at all** — all columns are metadata-included — so it cannot enforce uniqueness. The `ORDER (SaleLineID)` clause is legal by itself (ordered *clustered* columnstore since SQL Server 2022, ordered *nonclustered* columnstore new in SQL Server 2025 — verified: `CREATE CLUSTERED COLUMNSTORE INDEX ... ORDER (col)` succeeds and `sys.index_columns.column_store_order_ordinal = 1` for the ordered column), but ordering only sorts data to improve **rowgroup/segment elimination**; it is a performance feature, not a constraint. Even with `UNIQUE` removed, option d would leave requirement 2 unmet — uniqueness on a CCI table needs a separate unique B-tree, which is exactly what option b adds.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

## DP-800 Exam Rule to Remember

```text
CLUSTERED columnstore    = the table itself (one full copy, max compression)
                           → data-warehouse fact tables and big dimensions
NONCLUSTERED columnstore = extra columnstore copy over a rowstore table
                           → real-time operational analytics (HTAP), updatable since 2016
```

Bolt-ons and limits that pair with the storage choice:

- CCI table + rowstore B-tree NCIs (unique, PK/FK enforcement, point seeks): **allowed since SQL Server 2016** — the classic combo is CCI for scans plus a unique B-tree for the key.
- A columnstore index itself can never be `UNIQUE` and has no key columns (Msg 35301).
- Rowgroups compress up to 1,048,576 rows; bulk loads of ≥ 102,400 rows go straight to `COMPRESSED` rowgroups, smaller trickle inserts wait in the delta store for the tuple mover.
- Batch-mode execution processes rows in vectors and is what makes columnstore aggregations fast — check `ActualExecutionMode="Batch"` in the actual plan.
- `ORDER (col)` on a columnstore index (clustered: SQL 2022+; nonclustered: SQL 2025) sorts data for segment elimination — performance, never uniqueness.

If the scenario says "fact table, scans, compression" → clustered columnstore. If it says "OLTP table must stay OLTP but needs live analytics" → nonclustered columnstore on top. Any option that makes the OLTP table's base storage columnstore, or keeps a full rowstore fact table under an NCCI, is trading away exactly what the scenario asked to keep.
