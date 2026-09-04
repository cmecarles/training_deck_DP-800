# SQL Server question — Triggers 1

## Statement

The city public library runs its lending system on SQL Server. The T-SQL below is the **complete** history of the database, executed top to bottom in a single session with no errors.

```sql
CREATE DATABASE CityLibrary;
GO
USE CityLibrary;
GO
CREATE SCHEMA Lending;
GO

CREATE TABLE Lending.Books
(
    BookId          int          NOT NULL PRIMARY KEY,
    Title           nvarchar(80) NOT NULL,
    CopiesAvailable int          NOT NULL
);
GO

CREATE TABLE Lending.Loans
(
    LoanId   int  IDENTITY(1,1) PRIMARY KEY,
    BookId   int  NOT NULL REFERENCES Lending.Books(BookId),
    MemberId int  NOT NULL,
    LoanDate date NOT NULL
);
GO

CREATE TABLE Lending.AuditTrail
(
    AuditId   int          IDENTITY(1,1) PRIMARY KEY,
    EventType nvarchar(20) NOT NULL,
    LoanCount int          NOT NULL
);
GO

CREATE TRIGGER Lending.trg_Loans_AfterInsert
ON Lending.Loans
AFTER INSERT
AS
BEGIN
    DECLARE @RowsInserted int = @@ROWCOUNT;

    SET NOCOUNT ON;

    IF @RowsInserted = 0
        RETURN;

    INSERT INTO Lending.AuditTrail (EventType, LoanCount)
    VALUES (N'LOAN_BATCH', @RowsInserted);

    UPDATE b
       SET b.CopiesAvailable = b.CopiesAvailable - 1
      FROM Lending.Books AS b
     INNER JOIN inserted AS i
             ON i.BookId = b.BookId;
END;
GO

INSERT INTO Lending.Books (BookId, Title, CopiesAvailable) VALUES
    (101, N'The Sea Atlas',     4),
    (202, N'Clockwork Cities',  2),
    (303, N'Paper Lanterns',    5);
GO

-- Statement S1: one INSERT statement, three rows
INSERT INTO Lending.Loans (BookId, MemberId, LoanDate) VALUES
    (101, 11, '2026-08-01'),
    (101, 12, '2026-08-01'),
    (202, 13, '2026-08-01');
GO

-- Statement S2: one INSERT statement, one row
INSERT INTO Lending.Loans (BookId, MemberId, LoanDate) VALUES
    (303, 14, '2026-08-02');
GO
```

After the last batch, the following two queries are run:

```sql
SELECT AuditId, EventType, LoanCount
FROM Lending.AuditTrail
ORDER BY AuditId;

SELECT BookId, CopiesAvailable
FROM Lending.Books
ORDER BY BookId;
```

Which option shows exactly the two result sets returned?

### a.

```text
AuditId  EventType   LoanCount
-------  ----------  ---------
1        LOAN_BATCH  1
2        LOAN_BATCH  1
3        LOAN_BATCH  1
4        LOAN_BATCH  1

BookId  CopiesAvailable
------  ---------------
101     2
202     1
303     4
```

### b.

```text
AuditId  EventType   LoanCount
-------  ----------  ---------
1        LOAN_BATCH  3
2        LOAN_BATCH  1

BookId  CopiesAvailable
------  ---------------
101     2
202     1
303     4
```

### c.

```text
AuditId  EventType   LoanCount
-------  ----------  ---------
1        LOAN_BATCH  3
2        LOAN_BATCH  1

BookId  CopiesAvailable
------  ---------------
101     3
202     1
303     4
```

### d.

```text
AuditId  EventType   LoanCount
-------  ----------  ---------
1        LOAN_BATCH  0
2        LOAN_BATCH  0

BookId  CopiesAvailable
------  ---------------
101     3
202     1
303     4
```

## Correct Answer

**c**

There is no other equivalent correct option: only **c** matches both result sets.

## Explanation

The correct answer is **c**.

Three documented facts decide this question:

1. **A DML `AFTER` trigger fires once per statement, not once per row.** A single `INSERT` that adds many rows causes a single trigger invocation, and the `inserted` logical table then holds *all* of the new rows at once.
2. **At the moment the trigger starts, `@@ROWCOUNT` holds the number of rows affected by the triggering statement.** It must be captured by the very first statement of the trigger body, because almost any subsequent statement — including `SET NOCOUNT ON`, since `SET` options reset `@@ROWCOUNT` to 0 — overwrites it.
3. **An `UPDATE ... FROM ... JOIN` updates each qualifying target row exactly once**, even when that target row matches several rows of the source. When the same `BookId` appears twice in `inserted`, `Lending.Books` still loses only one copy for that book — the classic "single-row assumption" trigger bug, which Microsoft's own multirow-trigger documentation demonstrates and warns against.

Note on determinism: when a target row matches multiple `inserted` rows, SQL Server does not define *which* matching source row feeds the `SET` clause — but here the `SET` clause (`b.CopiesAvailable - 1`) never references `i`, so the outcome is identical whichever match is used. The result is fully deterministic: one decrement per matched book.

### Statement-by-statement trace

**Seeding `Lending.Books`** (no trigger on `Books`):

| BookId | CopiesAvailable |
|---|---|
| 101 | 4 |
| 202 | 2 |
| 303 | 5 |

**Statement S1** inserts three rows into `Lending.Loans` (`LoanId` 1, 2, 3). The trigger fires **once**:

- `DECLARE @RowsInserted int = @@ROWCOUNT;` runs first, so it still sees the triggering statement's row count: **3**. Only afterwards does `SET NOCOUNT ON` reset `@@ROWCOUNT` to 0 — too late to matter.
- The guard `IF @RowsInserted = 0 RETURN;` does not fire (3 ≠ 0). (It exists because an `AFTER` trigger fires even for a statement that affects zero rows.)
- The audit `INSERT ... VALUES` writes exactly **one** row: `(AuditId 1, N'LOAN_BATCH', 3)`. A `VALUES` clause with one row writes one row, no matter how many rows `inserted` holds.
- The `UPDATE b ... INNER JOIN inserted` runs once. `inserted` holds `{101, 101, 202}`, so the join qualifies two target rows in `Books`: 101 and 202. Each qualifying target row is decremented **once**:
  - Book 101: 4 − 1 = **3** (not 4 − 2 = 2, despite being borrowed twice in this statement — this is the trigger's latent bug).
  - Book 202: 2 − 1 = **1**.
  - Book 303: untouched, still 5.

**Statement S2** inserts one row (`LoanId` 4). The trigger fires once more:

- `@RowsInserted` = **1**.
- Audit row `(AuditId 2, N'LOAN_BATCH', 1)` is written.
- `inserted` holds `{303}`; book 303: 5 − 1 = **4**.

Final state — exactly option **c**:

| AuditId | EventType | LoanCount |
|---|---|---|
| 1 | LOAN_BATCH | 3 |
| 2 | LOAN_BATCH | 1 |

| BookId | CopiesAvailable |
|---|---|
| 101 | 3 |
| 202 | 1 |
| 303 | 4 |

### Why option a is wrong

Option a is the **row-level-trigger myth**: four loan rows → four trigger firings → four audit rows with `LoanCount = 1` each, and book 101 decremented twice (4 → 2).

SQL Server DML triggers are **statement-level only** (there is no `FOR EACH ROW` as in Oracle or PostgreSQL). Two statements ran against `Lending.Loans`, so the trigger fired exactly twice and `Lending.AuditTrail` contains exactly two rows. `LoanCount = 3` for the first firing, because `@@ROWCOUNT` at trigger start reflects the whole triggering statement.

### Why option b is wrong

Option b gets the audit table right (per-statement firing, `LoanCount` 3 and 1) but assumes the `UPDATE ... JOIN inserted` decrements `CopiesAvailable` **once per matching `inserted` row**, giving book 101 the value 4 − 2 = 2.

That is not how `UPDATE ... FROM` works: each target row of `Lending.Books` is updated at most once per `UPDATE` statement, regardless of how many `inserted` rows join to it. Book 101 matches two `inserted` rows but is decremented only once: 4 − 1 = 3. (A trigger that *wanted* to subtract the true per-book count would need something set-based like `SET b.CopiesAvailable = b.CopiesAvailable - i.Cnt FROM ... JOIN (SELECT BookId, COUNT(*) AS Cnt FROM inserted GROUP BY BookId) AS i ...`.)

### Why option d is wrong

Option d gets `Lending.Books` right but claims `LoanCount = 0` in both audit rows, reasoning that `SET NOCOUNT ON` (or the trigger's startup itself) resets `@@ROWCOUNT` before it is read.

`SET` statements do reset `@@ROWCOUNT` to 0 — but order matters. In this trigger, `DECLARE @RowsInserted int = @@ROWCOUNT;` is the **first** statement of the body, so it captures 3 (statement S1) and 1 (statement S2) before `SET NOCOUNT ON` executes. Had the two lines been swapped, `@RowsInserted` would have read 0, the `IF @RowsInserted = 0 RETURN;` guard would have exited immediately, and **no audit rows at all** would have been written (not even option d's two zero rows) — that fragility is exactly why the capture-first pattern is the documented convention.

## DP-800 Exam Rule to Remember

In SQL Server, a DML trigger is a **per-statement** object, never a per-row one:

```text
1 statement  →  1 firing  →  inserted/deleted hold ALL affected rows
```

Whenever you read a trigger on the exam, run this checklist:

- **How many times does it fire?** Once per triggering statement — even for a statement that affects zero rows (guard with `IF @@ROWCOUNT = 0 RETURN;` or `IF (ROWCOUNT_BIG() = 0) RETURN;`).
- **What is in `inserted`/`deleted`?** Every row the statement touched, as a set. Any code that assumes one row (`SELECT @x = Col FROM inserted`, a scalar subquery, a single `VALUES` audit insert) silently misbehaves on multirow statements.
- **Capture `@@ROWCOUNT` in the trigger's first statement** — `SET` options and most other statements reset it.
- **`UPDATE ... FROM` touches each target row once**, no matter how many source rows join to it; to accumulate per-key effects, aggregate `inserted` (`GROUP BY` + `COUNT`/`SUM`) before joining.

Write trigger bodies as set-based logic over `inserted`/`deleted`, and the row count of the triggering statement stops mattering.
