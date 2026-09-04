# SQL Server question — Clustered Indexes 1

## Statement

`AeroPark` tracks long-stay parking sessions at an airport. A DBA runs the following batch, top to bottom, on a fresh SQL Server instance. Note the order of events carefully: the primary key is declared **NONCLUSTERED**, a nonclustered index is added while the table is still a heap, rows are inserted (three of them share the same `EntryTime`), and only then is a clustered index created on the non-unique `EntryTime` column.

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

What is the outcome of the final `SELECT`?

### a.

The batch never reaches the `SELECT`: `CREATE CLUSTERED INDEX CIX_Sessions_EntryTime` fails, because `PK_Sessions` already defines the table's clustered index and a table cannot have more than one.

### b.

```text
index_id  name                     type_desc     is_unique  is_primary_key
--------  -----------------------  ------------  ---------  --------------
1         CIX_Sessions_EntryTime   CLUSTERED     0          0
2         PK_Sessions              NONCLUSTERED  1          1
3         IX_Sessions_PlateNo      NONCLUSTERED  0          0
```

### c.

```text
index_id  name                     type_desc     is_unique  is_primary_key
--------  -----------------------  ------------  ---------  --------------
2         PK_Sessions              NONCLUSTERED  1          1
3         IX_Sessions_PlateNo      NONCLUSTERED  0          0
4         CIX_Sessions_EntryTime   CLUSTERED     0          0
```

### d.

```text
index_id  name                     type_desc     is_unique  is_primary_key
--------  -----------------------  ------------  ---------  --------------
1         CIX_Sessions_EntryTime   CLUSTERED     1          0
2         PK_Sessions              NONCLUSTERED  1          1
3         IX_Sessions_PlateNo      NONCLUSTERED  0          0
```

## Correct Answer

**b**

## Explanation

The whole batch runs without error, and the final `SELECT` returns exactly the result in option b (verified by executing the script as-is on SQL Server 2025).

The question packs four separable clustered-index facts; each wrong option is one of them mis-remembered.

### Why option a is wrong — PRIMARY KEY ≠ clustered index

"One clustered index per table" is true, but `PK_Sessions` is not it. A `PRIMARY KEY` constraint is enforced by a **unique index**, and that index is clustered only **by default**. Here the constraint was declared `PRIMARY KEY NONCLUSTERED`, so after `CREATE TABLE` the table is a **heap** with a unique nonclustered index enforcing the key. The one clustered-index slot is still free, and `CREATE CLUSTERED INDEX ... ON (EntryTime)` legally claims it — on a column that is neither the key nor unique. The two roles — "row identity" (PK) and "physical organization of the table" (CI) — are independent, and separating them like this is a real design pattern (cluster on the insert/range column, keep the surrogate key nonclustered).

Also note the opposite trap: had the batch instead tried to create a *second* clustered index after this one, **that** would fail with *"Cannot create more than one clustered index on table"*.

### Why option b is correct — index_id is a role, not a birth order

`sys.indexes` uses fixed slots:

- `index_id = 0` — the heap (`type_desc = 'HEAP'`), only while the table has no clustered index;
- `index_id = 1` — always and only the clustered index;
- `index_id >= 2` — nonclustered indexes.

Running the metadata query **before** `CREATE CLUSTERED INDEX` returns (verified):

```text
index_id  name                 type_desc     is_unique  is_primary_key
--------  -------------------  ------------  ---------  --------------
0         NULL                 HEAP          0          0
2         PK_Sessions          NONCLUSTERED  1          1
3         IX_Sessions_PlateNo  NONCLUSTERED  0          0
```

Creating the clustered index converts the heap: row 0 disappears, `CIX_Sessions_EntryTime` appears as `index_id = 1` even though it was created *last*, and the two nonclustered indexes keep ids 2 and 3. `is_unique = 0` for the clustered index (see option d) and `is_primary_key = 0` (see option a).

### Why option c is wrong — the creation-order myth

Option c is the subtle distractor: identical rows, but it assumes `index_id` values are handed out sequentially in creation order (heap 0, PK 2, PlateNo 3, so the new index gets 4). `index_id = 1` is *reserved* for the clustered index, whenever it is created; a heap-to-clustered conversion moves the table from slot 0 to slot 1. An `index_id` of 4 for a clustered index is impossible, as is the survival of a `HEAP` row alongside a clustered index — the heap and the clustered index are the *same* slot in two states, never coexisting.

### Why option d is wrong — the uniqueifier does not make the index unique

Three inserted rows share `EntryTime = '2026-08-01 06:15:00'`, and the `CREATE CLUSTERED INDEX` (no `UNIQUE` keyword) still succeeds — a clustered index does **not** have to be unique. Internally the engine must still tell duplicate key rows apart, so it appends a hidden 4-byte **uniqueifier** to the key of every row that duplicates an earlier key value (the first occurrence carries none — it is a variable-length column added only when needed). Verified on this data via `sys.dm_db_index_physical_stats(... 'DETAILED')`: leaf records of `CIX_Sessions_EntryTime` are 33 bytes for the non-duplicated rows and 41 bytes for the duplicated ones.

But the uniqueifier is internal bookkeeping, not a constraint: `sys.indexes.is_unique` stays `0`, duplicate `EntryTime` values keep inserting successfully (verified), and the optimizer does not get the plan-simplification benefits of a declared unique index.

### What also happened, invisibly, to the nonclustered indexes

Every nonclustered index row carries a **row locator** pointing at its base-table row:

- on a **heap**: the RID (file:page:slot);
- on a **clustered table**: the **clustering key** (plus the uniqueifier when present).

`PK_Sessions` and `IX_Sessions_PlateNo` were built while the table was a heap, so they stored RIDs. The `CREATE CLUSTERED INDEX` statement therefore silently **rebuilt both nonclustered indexes** to replace RIDs with `EntryTime` (+ uniqueifier) locators — which is why creating (or dropping) a clustered index on a large table is far more expensive than the clustered index alone, and why a wide clustered key inflates *every* nonclustered index. Dropping `CIX_Sessions_EntryTime` would rebuild them back to RIDs.

One last myth for completeness: a clustered index defines the *logical* order of the data and is not a promise that pages sit physically contiguous on disk, nor that `SELECT ... FROM Park.Sessions` without `ORDER BY` will return rows in `EntryTime` order. Order of results is guaranteed only by `ORDER BY`.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

## DP-800 Exam Rule to Remember

```text
one table = at most ONE clustered index (index_id 1) — or a heap (index_id 0), never both
PRIMARY KEY        → unique index, clustered only BY DEFAULT (override: NONCLUSTERED)
CLUSTERED INDEX    → need not be the PK, need not be unique
non-unique CI      → hidden 4-byte uniqueifier on duplicates; is_unique stays 0
nonclustered index → row locator = RID on a heap, clustering key (+uniqueifier) on a clustered table
create/drop the CI → every nonclustered index is rebuilt to swap its locators
```

When a question shows `PRIMARY KEY NONCLUSTERED` followed by `CREATE CLUSTERED INDEX` on another column, read it as: heap → clustered conversion, clustered index takes `index_id = 1` regardless of creation order, and the wide/narrow-clustering-key trade-off now applies to every secondary index on the table.
