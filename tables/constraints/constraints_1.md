# SQL Server question — Constraints 1

## Statement

`KennelBook` is the booking database of a dog-boarding kennel. Owners register dogs, dogs are booked for stays, and owners receive invoices. The schema relies entirely on declarative constraints:

```sql
CREATE DATABASE KennelBook;
GO
USE KennelBook;
GO
CREATE SCHEMA Board;
GO
CREATE TABLE Board.Owner
(
    OwnerID  INT          NOT NULL CONSTRAINT PK_Owner PRIMARY KEY,
    FullName NVARCHAR(60) NOT NULL,
    Email    VARCHAR(80)  NULL     CONSTRAINT UQ_Owner_Email UNIQUE
);
CREATE TABLE Board.Dog
(
    DogID      INT          NOT NULL CONSTRAINT PK_Dog PRIMARY KEY,
    OwnerID    INT          NULL,
    Name       NVARCHAR(40) NOT NULL,
    WeightKg   DECIMAL(5,1) NULL     CONSTRAINT CK_Dog_Weight CHECK (WeightKg > 0 AND WeightKg < 100),
    Vaccinated BIT          NOT NULL CONSTRAINT DF_Dog_Vaccinated DEFAULT (0),
    CONSTRAINT FK_Dog_Owner FOREIGN KEY (OwnerID) REFERENCES Board.Owner (OwnerID) ON DELETE SET NULL
);
CREATE UNIQUE NONCLUSTERED INDEX UX_Dog_Name ON Board.Dog (Name);
CREATE TABLE Board.Stay
(
    StayID   INT  NOT NULL CONSTRAINT PK_Stay PRIMARY KEY,
    DogID    INT  NOT NULL,
    CheckIn  DATE NOT NULL,
    CheckOut DATE NULL,
    CONSTRAINT FK_Stay_Dog FOREIGN KEY (DogID) REFERENCES Board.Dog (DogID) ON DELETE CASCADE,
    CONSTRAINT CK_Stay_Dates CHECK (CheckOut >= CheckIn)
);
CREATE TABLE Board.Invoice
(
    InvoiceID INT          NOT NULL CONSTRAINT PK_Invoice PRIMARY KEY,
    OwnerID   INT          NOT NULL CONSTRAINT FK_Invoice_Owner FOREIGN KEY REFERENCES Board.Owner (OwnerID),
    Total     DECIMAL(8,2) NOT NULL
);
GO
INSERT INTO Board.Owner (OwnerID, FullName, Email) VALUES
  (1, N'Ana Ruiz', 'ana@example.com'),
  (2, N'Ben Cole', NULL),
  (3, N'Cy Dorn',  'cy@example.com');
INSERT INTO Board.Dog (DogID, OwnerID, Name, WeightKg, Vaccinated) VALUES
  (10, 1, N'Rex', 32.5, 1),
  (11, 2, N'Mia', NULL, 1),
  (12, 3, N'Tor',  8.0, 0);
INSERT INTO Board.Stay (StayID, DogID, CheckIn, CheckOut) VALUES
  (100, 10, '2026-09-01', '2026-09-05'),
  (101, 11, '2026-09-02', NULL),
  (102, 12, '2026-09-03', '2026-09-04');
INSERT INTO Board.Invoice (InvoiceID, OwnerID, Total) VALUES (500, 1, 120.00);
GO
```

The following fourteen statements are then executed **in order, each in its own batch**, in a single session:

```sql
-- S1
INSERT INTO Board.Owner (OwnerID, FullName, Email) VALUES (4, N'Di Eng', NULL);

-- S2
INSERT INTO Board.Dog (DogID, OwnerID, Name, WeightKg) VALUES (13, 3, N'Rex', 12.0);

-- S3
INSERT INTO Board.Dog (DogID, OwnerID, Name) VALUES (13, 3, N'Zed');

-- S4
INSERT INTO Board.Dog (DogID, OwnerID, Name, WeightKg) VALUES (14, 3, N'Bruno', 150.0);

-- S5
INSERT INTO Board.Dog (DogID, OwnerID, WeightKg) VALUES (15, 3, 10.0);

-- S6
INSERT INTO Board.Dog (DogID, OwnerID, Name) VALUES (16, 99, N'Ghost');

-- S7
DELETE FROM Board.Owner WHERE OwnerID = 1;

-- S8
DELETE FROM Board.Owner WHERE OwnerID = 2;

-- S9
DELETE FROM Board.Dog WHERE DogID = 12;

-- S10
ALTER TABLE Board.Dog WITH NOCHECK ADD CONSTRAINT CK_Dog_Name CHECK (LEN(Name) >= 4);

-- S11
INSERT INTO Board.Dog (DogID, OwnerID, Name) VALUES (17, 3, N'Bo');

-- S12
ALTER TABLE Board.Dog WITH CHECK CHECK CONSTRAINT CK_Dog_Name;

-- S13
UPDATE Board.Stay SET CheckIn = '2026-09-10' WHERE StayID = 101;

-- S14
UPDATE Board.Stay SET CheckOut = '2026-08-30' WHERE StayID = 100;
```

For each statement S1–S14, state whether it **succeeds or raises an error** (and, for successes, how many rows the statement reports). Then give the exact final contents of `Board.Owner`, `Board.Dog` and `Board.Stay` (ordered by key), and the result of:

```sql
SELECT name, is_not_trusted FROM sys.check_constraints
WHERE parent_object_id = OBJECT_ID('Board.Dog') ORDER BY name;
```

## Correct Answer

Per-statement outcomes (all error numbers and messages are the engine's actual output):

| Stmt | Outcome | Detail |
|------|---------|--------|
| S1 | **Fails** | `Msg 2627` — `Violation of UNIQUE KEY constraint 'UQ_Owner_Email'. Cannot insert duplicate key in object 'Board.Owner'. The duplicate key value is (<NULL>).` |
| S2 | **Fails** | `Msg 2601` — `Cannot insert duplicate key row in object 'Board.Dog' with unique index 'UX_Dog_Name'. The duplicate key value is (Rex).` |
| S3 | **Succeeds** | `(1 rows affected)` — `WeightKg` NULL passes the CHECK, `Vaccinated` gets the default 0 |
| S4 | **Fails** | `Msg 547` — `The INSERT statement conflicted with the CHECK constraint "CK_Dog_Weight". The conflict occurred in database "KennelBook", table "Board.Dog", column 'WeightKg'.` |
| S5 | **Fails** | `Msg 515` — `Cannot insert the value NULL into column 'Name', table 'KennelBook.Board.Dog'; column does not allow nulls. INSERT fails.` |
| S6 | **Fails** | `Msg 547` — `The INSERT statement conflicted with the FOREIGN KEY constraint "FK_Dog_Owner". The conflict occurred in database "KennelBook", table "Board.Owner", column 'OwnerID'.` |
| S7 | **Fails** | `Msg 547` — `The DELETE statement conflicted with the REFERENCE constraint "FK_Invoice_Owner". The conflict occurred in database "KennelBook", table "Board.Invoice", column 'OwnerID'.` |
| S8 | **Succeeds** | `(1 rows affected)` — owner 2 deleted; Mia's `OwnerID` set to NULL |
| S9 | **Succeeds** | `(1 rows affected)` — dog 12 deleted; stay 102 deleted by cascade (not counted in the message) |
| S10 | **Succeeds** | Constraint added without checking existing rows; `is_not_trusted = 1` |
| S11 | **Fails** | `Msg 547` — `The INSERT statement conflicted with the CHECK constraint "CK_Dog_Name". The conflict occurred in database "KennelBook", table "Board.Dog", column 'Name'.` |
| S12 | **Fails** | `Msg 547` — `The ALTER TABLE statement conflicted with the CHECK constraint "CK_Dog_Name". The conflict occurred in database "KennelBook", table "Board.Dog", column 'Name'.` |
| S13 | **Succeeds** | `(1 rows affected)` — `CheckOut` is NULL, so `CheckOut >= CheckIn` is UNKNOWN and passes |
| S14 | **Fails** | `Msg 547` — `The UPDATE statement conflicted with the CHECK constraint "CK_Stay_Dates". The conflict occurred in database "KennelBook", table "Board.Stay".` |

Final state:

| OwnerID | FullName | Email |
|---------|----------|-------|
| 1 | Ana Ruiz | ana@example.com |
| 3 | Cy Dorn | cy@example.com |

| DogID | OwnerID | Name | WeightKg | Vaccinated |
|-------|---------|------|----------|------------|
| 10 | 1 | Rex | 32.5 | 1 |
| 11 | NULL | Mia | NULL | 1 |
| 13 | 3 | Zed | NULL | 0 |

| StayID | DogID | CheckIn | CheckOut |
|--------|-------|---------|----------|
| 100 | 10 | 2026-09-01 | 2026-09-05 |
| 101 | 11 | 2026-09-10 | NULL |

| name | is_not_trusted |
|------|----------------|
| CK_Dog_Name | 1 |
| CK_Dog_Weight | 0 |

## Explanation

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

### S1 — a UNIQUE constraint allows exactly one NULL

Unlike the ANSI standard (where NULLs are never equal), SQL Server treats NULL as a value for uniqueness purposes: `UQ_Owner_Email` already holds the NULL of owner 2, so a second NULL is a duplicate — error 2627, key value `(<NULL>)`. If several NULL e-mails must be allowed, use a **filtered unique index** (`CREATE UNIQUE INDEX ... WHERE Email IS NOT NULL`).

### S2 — 2601 versus 2627

Both errors mean "duplicate key", but the number tells you what enforced it: **2627** is a `PRIMARY KEY`/`UNIQUE` *constraint*, **2601** is a unique *index* created with `CREATE UNIQUE INDEX` (here `UX_Dog_Name`). Same protection, different metadata (`sys.key_constraints` versus only `sys.indexes`).

### S3, S4, S13, S14 — CHECK constraints reject only FALSE, never UNKNOWN

A CHECK passes whenever its predicate is TRUE **or UNKNOWN**. S3 inserts a NULL weight: `NULL > 0` is UNKNOWN, so the row is accepted; S13 sets a check-in later than a NULL check-out for the same reason. Only a definite FALSE (S4: 150 kg; S14: check-out before check-in) raises 547. Note the message for the multi-column `CK_Stay_Dates` does not name a column. `DEFAULT (0)` on `Vaccinated` fills the omitted column in S3; a `NOT NULL` column *without* a default that is omitted fails with error 515 (S5).

### S6, S7, S8, S9 — foreign keys and their referential actions

- **Insert side**: a child row whose parent does not exist violates the FK (S6, error 547 naming the *parent* table and column).
- **Delete side**: what happens depends on the FK's action. `FK_Invoice_Owner` has the default `NO ACTION`, so deleting owner 1, who has an invoice, is refused (S7) — even though the *other* FK on the same parent (`FK_Dog_Owner`) would have happily set Rex's owner to NULL; one blocking reference is enough. Deleting owner 2 (S8) hits only `ON DELETE SET NULL`, so it succeeds and Mia's `OwnerID` becomes NULL (the column must be nullable for this action). Deleting dog 12 (S9) triggers `ON DELETE CASCADE` on `FK_Stay_Dog`, silently removing stay 102; the `(1 rows affected)` message counts only the row targeted by the statement, never the rows changed by a cascade or a `SET NULL`.

### S10, S11, S12 — WITH NOCHECK and trust

`WITH NOCHECK` adds the constraint without validating existing rows: `Rex`, `Mia`, `Zed` (3 letters) violate it, yet S10 succeeds and marks the constraint `is_not_trusted = 1`. The constraint is nevertheless **enabled** for new data: S11's `Bo` fails immediately. Untrusted constraints are ignored by the optimizer (it cannot assume the data conforms, so it will not eliminate joins or simplify predicates based on them). To make it trusted you must run `ALTER TABLE ... WITH CHECK CHECK CONSTRAINT`, which re-validates every row — and fails here (S12) because the three short names are still present; `is_not_trusted` stays 1. The same rule applies to foreign keys added or re-enabled with `NOCHECK` (`sys.foreign_keys.is_not_trusted`).

### Equivalent alternatives

- S1 would succeed if `UQ_Owner_Email` were replaced by a filtered unique index on `Email IS NOT NULL`.
- S7 would succeed (deleting owner 1, nulling Rex's owner, and deleting invoice 500) only if `FK_Invoice_Owner` were declared `ON DELETE CASCADE`.

## DP-800 Exam Rule to Remember

```text
515  NULL into NOT NULL column (no DEFAULT)         2627 duplicate in PK / UNIQUE constraint
547  CHECK or FOREIGN KEY violated (INSERT/UPDATE/  2601 duplicate in a unique INDEX
     DELETE/ALTER TABLE WITH CHECK)

UNIQUE  : allows ONE NULL (NULL = NULL for uniqueness) -> filtered unique index for many NULLs
CHECK   : fails only on FALSE; NULL/UNKNOWN passes
DEFAULT : used when the column is omitted or the keyword DEFAULT is supplied
FK      : NO ACTION (default) blocks the parent delete; CASCADE deletes children;
          SET NULL / SET DEFAULT rewrite the child column (must allow it);
          any single blocking FK stops the whole delete
NOCHECK : skips existing rows, still enforced for new rows, is_not_trusted = 1,
          optimizer ignores it; WITH CHECK CHECK CONSTRAINT re-validates (547 if data is bad)
```

When a question asks "which rows exist afterwards", walk the statements in order, remember that a failed statement changes nothing, and count the cascaded rows separately from the "(n rows affected)" message.
