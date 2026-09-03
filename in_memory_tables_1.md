# SQL Server question — In-Memory Tables 1

## Statement

`FlashCart` is the checkout database of a flash-sale web shop. During a sale, tens of thousands of shopping baskets are created and updated per second, so the team decides to move the basket tables to **memory-optimized (In-Memory OLTP)** storage. The database has just been created:

```sql
CREATE DATABASE FlashCart;
GO
USE FlashCart;
GO
CREATE SCHEMA Cart;
GO
```

The following twelve statements are then executed **in order, each in its own batch**, in a single session:

```sql
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

For each statement S1–S12, state whether it **succeeds or raises an error** (and the value returned, where a result is produced). Then give the exact result of this final catalog query:

```sql
SELECT t.name AS TableName, t.durability_desc, i.name AS IndexName, i.type_desc, h.bucket_count
FROM sys.tables AS t
JOIN sys.indexes AS i ON i.object_id = t.object_id AND i.index_id > 0
LEFT JOIN sys.hash_indexes AS h ON h.object_id = i.object_id AND h.index_id = i.index_id
WHERE t.is_memory_optimized = 1
ORDER BY t.name, i.name;
```

## Correct Answer

Per-statement outcomes (all error numbers and messages are the engine's actual output):

| Stmt | Outcome | Detail |
|------|---------|--------|
| S1 | **Fails** | `Msg 41337` — `Cannot create memory optimized tables. To create memory optimized tables, the database must have a MEMORY_OPTIMIZED_FILEGROUP that is online and has at least one container.` |
| S2 | **Succeeds** | Filegroup `FlashCart_MO` (`CONTAINS MEMORY_OPTIMIZED_DATA`) and its container are added |
| S3 | **Succeeds** | `Cart.Basket` created; the hash PK gets **1024** buckets, not 1000 |
| S4 | **Fails** | `Msg 41321` — `The memory optimized table 'BasketLine' with DURABILITY=SCHEMA_AND_DATA must have a primary key.` followed by `Msg 1750` — `Could not create constraint or index. See previous errors.` |
| S5 | **Succeeds** | `SCHEMA_ONLY` table needs an index but not a primary key; hash index gets **512** buckets |
| S6 | **Fails** | `Msg 10794` — `The operation 'CREATE INDEX' is not supported with memory optimized tables.` |
| S7 | **Fails** | `Msg 10783` — `The body of a natively compiled module must be an ATOMIC block.` |
| S8 | **Succeeds** | Procedure created; both `EXEC` calls insert a row (2 rows in `Cart.Basket`) |
| S9 | **Fails** | `Msg 41368` — `Accessing memory optimized tables using the READ COMMITTED isolation level is supported only for autocommit transactions. It is not supported for explicit or implicit transactions. Provide a supported isolation level for the memory optimized table using a table hint, such as WITH (SNAPSHOT).` |
| S10 | **Succeeds** | Returns `OpenBaskets = 2` |
| S11 | **Fails** | `Msg 10794` — `The statement 'TRUNCATE TABLE' is not supported with memory optimized tables.` |
| S12 | **Fails** | `Msg 41332` — `Memory optimized tables and natively compiled modules cannot be accessed or created when the session TRANSACTION ISOLATION LEVEL is set to SNAPSHOT.` |

Final catalog query result:

| TableName | durability_desc | IndexName | type_desc | bucket_count |
|-----------|-----------------|-----------|-----------|--------------|
| Basket | SCHEMA_AND_DATA | IX_Basket_User | NONCLUSTERED | NULL |
| Basket | SCHEMA_AND_DATA | PK__Basket__8FDA77D4DF66B6F8 | NONCLUSTERED HASH | 1024 |
| ViewLog | SCHEMA_ONLY | IX_ViewLog_User | NONCLUSTERED HASH | 512 |

(The system-generated PK name differs per run; everything else is fixed.) `Cart.Basket` still holds its two rows: S11 failed, and S12 failed before reading anything.

## Explanation

This session walks through the prerequisites, index rules, natively compiled module rules and transaction rules of In-Memory OLTP. Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

### S1 and S2 — a memory-optimized filegroup with a container is mandatory

`MEMORY_OPTIMIZED = ON` is rejected (error 41337) until the database has a filegroup declared `CONTAINS MEMORY_OPTIMIZED_DATA` **and** at least one container (`ADD FILE ... TO FILEGROUP`). The "file" is actually a folder of checkpoint file pairs where durable rows are streamed. S2 adds both, so the identical statement succeeds as S3. (Azure SQL Database creates this filegroup for you; on SQL Server it is a manual, one-time step. Once in-memory objects exist, the filegroup can no longer be removed — verified: `ALTER DATABASE ... REMOVE FILE` raises error 41880.)

### S3 and S5 — BUCKET_COUNT is rounded up to a power of two

A hash index is an array of buckets; the requested count is always rounded **up to the next power of 2**: 1000 → 1024 and 500 → 512 (`sys.hash_indexes.bucket_count`). Guidance is 1–2× the number of distinct key values: too few buckets means long chains, too many wastes memory and slows scans. Hash indexes only serve **equality** predicates; a range predicate (`BasketID > 1`) or an `ORDER BY` needs the memory-optimized `NONCLUSTERED` (Bw-tree) index such as `IX_Basket_User`.

### S4 versus S5 — the durability option decides whether a PRIMARY KEY is required

- `DURABILITY = SCHEMA_AND_DATA` (the default) logs and checkpoints the rows; the engine **requires a primary key** (error 41321) because it identifies rows during recovery.
- `DURABILITY = SCHEMA_ONLY` persists only the table definition: rows are gone after a restart or failover. It is meant for staging/session-state data, and it only requires **at least one index** (error 41327 if none), not a primary key. S5 therefore succeeds with a lone hash index.

### S6 — indexes are part of the table definition

`CREATE INDEX` does not exist for memory-optimized tables (error 10794); indexes are declared inline in `CREATE TABLE` or added with `ALTER TABLE ... ADD INDEX` (verified to succeed). There is no clustered index at all: rows live in memory and every index (hash or nonclustered) points to them directly.

### S7 and S8 — natively compiled procedures have a fixed shape

A natively compiled module must be declared `WITH NATIVE_COMPILATION, SCHEMABINDING` and its body must be a single `BEGIN ATOMIC WITH (TRANSACTION ISOLATION LEVEL = ..., LANGUAGE = ...)` block (error 10783 without `ATOMIC`; omitting `SCHEMABINDING` raises error 10796). The atomic block **is** the transaction: it either commits or rolls back as a unit. Only `SNAPSHOT`, `REPEATABLE READ` and `SERIALIZABLE` are valid there — `READ COMMITTED` is rejected with error 10794 — and the module may reference only memory-optimized tables (a disk-based table raises error 10775). Interpreted T-SQL, by contrast, can freely join disk-based and memory-optimized tables.

### S9, S10 and S12 — cross-container transactions need an explicit isolation level

An autocommit `SELECT` (no `BEGIN TRANSACTION`) on a memory-optimized table runs happily under the session default `READ COMMITTED`. Inside an **explicit** transaction that same access is a *cross-container transaction*, and `READ COMMITTED` is not a valid isolation level for the memory-optimized side (error 41368). The fix is a table hint — `WITH (SNAPSHOT)` in S10 — or the database option `MEMORY_OPTIMIZED_ELEVATE_TO_SNAPSHOT = ON`, which applies `SNAPSHOT` to memory-optimized tables implicitly.

S12 is the subtle one: the hint is present, yet the statement fails with error 41332. Setting the **session** isolation level to `SNAPSHOT` (`SET TRANSACTION ISOLATION LEVEL SNAPSHOT`) is incompatible with memory-optimized tables altogether — the disk-based side would use versioned snapshot reads while the in-memory side uses its own row-versioning, and the engine refuses the combination regardless of hints. Session `SNAPSHOT` and table-hint `SNAPSHOT` are not interchangeable.

### S11 — TRUNCATE TABLE is unsupported

`TRUNCATE TABLE` is one of the operations memory-optimized tables do not support (error 10794); use `DELETE`. The two rows survive.

## DP-800 Exam Rule to Remember

```text
Prerequisite : filegroup CONTAINS MEMORY_OPTIMIZED_DATA + one container  (else 41337)
Durability   : SCHEMA_AND_DATA -> PRIMARY KEY required (41321)
               SCHEMA_ONLY     -> any one index suffices; rows lost on restart
Indexes      : declared inline / ALTER TABLE ADD INDEX, never CREATE INDEX (10794)
               HASH  -> equality only, BUCKET_COUNT rounded up to 2^n
               NONCLUSTERED (Bw-tree) -> ranges and ORDER BY
Native procs : NATIVE_COMPILATION + SCHEMABINDING + BEGIN ATOMIC WITH (...)
               isolation SNAPSHOT / REPEATABLE READ / SERIALIZABLE only
               memory-optimized tables only (10775)
Transactions : explicit tran + READ COMMITTED -> 41368, add WITH (SNAPSHOT)
               or MEMORY_OPTIMIZED_ELEVATE_TO_SNAPSHOT = ON
               session-level SNAPSHOT isolation -> 41332, always
Not supported: TRUNCATE TABLE, clustered indexes, IDENTITY other than (1,1)
```

If the scenario says "must survive a restart" pick `SCHEMA_AND_DATA` (and give it a primary key); if it says "staging data, maximum speed, loss acceptable" pick `SCHEMA_ONLY`. If it says "point lookups by exact key" pick a hash index with a bucket count near the distinct-key count; if it mentions ranges or sorting, pick a memory-optimized nonclustered index.
