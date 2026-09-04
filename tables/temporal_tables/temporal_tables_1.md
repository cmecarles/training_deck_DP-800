# SQL Server question — Temporal Tables 1

## Statement

The revenue team of a seaside hotel tracks nightly room rates in the database `HarborHotel`. Rates change during the season, and the team must be able to audit every price a room has ever had, so the rate table is a system-versioned temporal table.

The script below is the complete history of the database, executed top to bottom in a single session. Every batch succeeds.

```sql
CREATE DATABASE HarborHotel;
GO
USE HarborHotel;
GO
CREATE SCHEMA Rooms;
GO
CREATE TABLE Rooms.RoomRate
(
    RoomNumber  int           NOT NULL PRIMARY KEY,
    RoomType    nvarchar(30)  NOT NULL,
    NightlyRate decimal(8,2)  NOT NULL,
    ValidFrom   datetime2 GENERATED ALWAYS AS ROW START HIDDEN NOT NULL,
    ValidTo     datetime2 GENERATED ALWAYS AS ROW END   HIDDEN NOT NULL,
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
)
WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = Rooms.RoomRateHistory));
GO
-- Season opening: initial rate card
INSERT INTO Rooms.RoomRate (RoomNumber, RoomType, NightlyRate) VALUES
  (101, N'Standard', 120.00),
  (102, N'Standard', 120.00),
  (201, N'Sea View', 180.00),
  (301, N'Suite',    320.00);
GO
-- Rate maintenance during the season
UPDATE Rooms.RoomRate SET NightlyRate = 135.00 WHERE RoomNumber = 101;
GO
UPDATE Rooms.RoomRate SET NightlyRate = 120.00 WHERE RoomNumber = 102;
GO
BEGIN TRANSACTION;
UPDATE Rooms.RoomRate SET NightlyRate = 199.00 WHERE RoomNumber = 201;
UPDATE Rooms.RoomRate SET NightlyRate = 210.00 WHERE RoomNumber = 201;
COMMIT;
GO
-- The suite is taken out of inventory for renovation
DELETE Rooms.RoomRate WHERE RoomNumber = 301;
GO
```

After the script completes, an auditor runs these two queries:

```sql
-- Query A
SELECT RoomNumber, NightlyRate
FROM Rooms.RoomRate FOR SYSTEM_TIME ALL
ORDER BY RoomNumber, NightlyRate;

-- Query B
SELECT RoomNumber, NightlyRate
FROM Rooms.RoomRateHistory
ORDER BY RoomNumber, NightlyRate;
```

What do Query A and Query B return?

### a.

```text
Query A (8 rows)                Query B (5 rows)
RoomNumber  NightlyRate         RoomNumber  NightlyRate
----------  -----------         ----------  -----------
       101       120.00                101       120.00
       101       135.00                102       120.00
       102       120.00                201       180.00
       102       120.00                201       199.00
       201       180.00                301       320.00
       201       199.00
       201       210.00
       301       320.00
```

### b.

```text
Query A (7 rows)                Query B (4 rows)
RoomNumber  NightlyRate         RoomNumber  NightlyRate
----------  -----------         ----------  -----------
       101       120.00                101       120.00
       101       135.00                102       120.00
       102       120.00                201       180.00
       102       120.00                301       320.00
       201       180.00
       201       210.00
       301       320.00
```

### c.

```text
Query A (7 rows)                Query B (5 rows)
RoomNumber  NightlyRate         RoomNumber  NightlyRate
----------  -----------         ----------  -----------
       101       120.00                101       120.00
       101       135.00                102       120.00
       102       120.00                201       180.00
       102       120.00                201       199.00
       201       180.00                301       320.00
       201       210.00
       301       320.00
```

### d.

```text
Query A (6 rows)                Query B (4 rows)
RoomNumber  NightlyRate         RoomNumber  NightlyRate
----------  -----------         ----------  -----------
       101       120.00                101       120.00
       101       135.00                201       180.00
       102       120.00                201       199.00
       201       180.00                301       320.00
       201       210.00
       301       320.00
```

## Correct Answer

**c**

(Verified against SQL Server 2025: Query A returns exactly the 7 rows and Query B exactly the 5 rows shown in option c.)

## Explanation

The question can be answered without knowing a single clock value, because it only asks *which row versions* exist — never *when* they were valid. That is the only way a temporal-table question can be deterministic: the period columns are filled in by the engine from the transaction start time and are therefore nondeterministic, but the *set* of versions produced by a given DML sequence is fully determined.

### Step 1 — build the version ledger

Track what each DML statement does to the current table and to `Rooms.RoomRateHistory`. Use symbolic times `T0 < T1 < T2 < T3 < T4` for the five data-modifying transactions (the engine stamps every row touched by a transaction with the transaction's start time, in UTC):

| Statement | Current table afterward | Row(s) written to history |
|---|---|---|
| `INSERT` 4 rows (at `T0`) | 101→120, 102→120, 201→180, 301→320 | **none** — an INSERT only opens a row (`ValidFrom = T0`, `ValidTo = 9999-12-31`); it never writes history |
| `UPDATE` 101 → 135.00 (at `T1`) | 101→135 | (101, 120.00) closed with period `[T0, T1)` |
| `UPDATE` 102 → 120.00 (at `T2`) | 102→120 (unchanged value) | (102, 120.00) closed with period `[T0, T2)` — **a history row is written even though no value changed** |
| `UPDATE` 201 → 199.00 then → 210.00, both inside one transaction (at `T3`) | 201→210 | (201, 180.00) with period `[T0, T3)` **and** (201, 199.00) with period `[T3, T3)` — the intermediate version gets `ValidFrom = ValidTo = T3` because both updates carry the same transaction start time (a *zero-duration* row) |
| `DELETE` 301 (at `T4`) | row removed | (301, 320.00) closed with period `[T0, T4)` |

Two Microsoft Learn rules drive the two subtle entries:

- *"When you run any data modification queries on a temporal table, the Database Engine adds a row to the history table, even if no column values change."* — that is room 102.
- *"The times recorded in the system datetime2 columns are based on the begin time of the transaction itself."* — that is why the two same-transaction updates to room 201 produce a history row whose `ValidFrom` equals its `ValidTo`.

### Step 2 — Query B: the history table is just a table

`Rooms.RoomRateHistory` queried directly returns every version the ledger placed there, zero-duration or not:

`(101, 120.00), (102, 120.00), (201, 180.00), (201, 199.00), (301, 320.00)` — **5 rows**.

### Step 3 — Query A: `FOR SYSTEM_TIME ALL` is *not* "current UNION ALL history"

`FOR SYSTEM_TIME ALL` returns the union of the current table and the history table, **but the docs add**: *"`FOR SYSTEM_TIME` filters out rows that have a period of validity with zero duration (`ValidFrom = ValidTo`). The Database Engine generates those rows if you perform multiple updates on the same primary key within the same transaction."* This filter applies to all five sub-clauses (`AS OF`, `FROM ... TO`, `BETWEEN`, `CONTAINED IN`, `ALL`).

So Query A = current rows + history rows − zero-duration rows:

- current: (101, 135.00), (102, 120.00), (201, 210.00)
- history: the 5 rows from Step 2, minus (201, 199.00) whose period is `[T3, T3)`

Result, ordered by `RoomNumber, NightlyRate`: `(101, 120.00), (101, 135.00), (102, 120.00), (102, 120.00), (201, 180.00), (201, 210.00), (301, 320.00)` — **7 rows**. Note that room 102 legitimately appears twice with the same rate: once as the current row and once as the history row written by the no-op update.

A side note on the `HIDDEN` keyword: because `ValidFrom`/`ValidTo` are declared `HIDDEN`, a `SELECT *` against `Rooms.RoomRate` returns only `RoomNumber, RoomType, NightlyRate`; the period columns appear only when named explicitly. That is also why the `INSERT` statements can omit a column list for them entirely. It has no effect on which rows the queries above return.

### Why option a is wrong

Option a treats `FOR SYSTEM_TIME ALL` as a plain `UNION ALL` of current and history, returning 8 rows including (201, 199.00). It is exactly one row off: every `FOR SYSTEM_TIME` sub-clause, including `ALL`, silently discards versions whose `ValidFrom = ValidTo`. The intermediate rate 199.00 exists *only* when the history table is queried directly. This is the subtle distractor — its Query B is correct.

### Why option b is wrong

Option b assumes the engine "collapses" multiple updates made to the same row inside one transaction, keeping only the pre-transaction version (201, 180.00) in history. It does not: each `UPDATE` statement moves the row version it replaces into the history table, so the intermediate (201, 199.00) is physically stored — it merely gets a zero-duration period. Its Query A happens to match the correct one (because `ALL` hides that row), which is what makes the pair internally consistent yet wrong: Query B against the raw history table returns 5 rows, not 4.

### Why option d is wrong

Option d assumes that updating room 102 to the value it already has (120.00 → 120.00) writes nothing to history. System-versioning does not compare old and new values: *any* `UPDATE` versionizes the row, even if no column value changes. Consequently (102, 120.00) is missing from its Query B, and the duplicate (102, 120.00) pair is missing from its Query A.

## DP-800 Exam Rule to Remember

For a system-versioned temporal table:

```text
INSERT  → current table only; history untouched
UPDATE  → old version moved to history (even if values are identical)
DELETE  → row leaves current table; closed version moved to history
```

Timestamps are the **transaction start time (UTC)** — so N updates to the same row in one transaction leave N−1 zero-duration versions (`ValidFrom = ValidTo`) in the history table.

And the query-side rule that decides this question:

```text
FOR SYSTEM_TIME (any sub-clause, including ALL)
        → filters out zero-duration versions
SELECT ... FROM <history table>
        → returns every stored version, zero-duration included
```

Finally: `HIDDEN` period columns disappear from `SELECT *` and can be omitted from `INSERT` column lists, but they are still there — name them explicitly to see them. Never design (or trust) a temporal question whose answer depends on the actual `datetime2` values: they are system-generated and nondeterministic; only the *set* of versions is predictable.
