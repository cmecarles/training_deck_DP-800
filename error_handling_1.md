# SQL Server question — Error Handling 1

## Statement

A parcel-shipping company validates shipping labels in the database `ParcelFlow` before they are handed to the carrier. The validation pipeline runs as a sequence of T-SQL batches; some of them are expected to hit errors (bad weights, duplicate label IDs), and the team wants to be certain about exactly what each error-handling construct does.

The following script is executed top to bottom, in one session, in SQL Server Management Studio. (In sqlcmd the output is identical except that each `Msg` line also carries a `Server` field.)

```sql
CREATE DATABASE ParcelFlow;
GO
USE ParcelFlow;
GO
SET NOCOUNT ON;
GO
CREATE SCHEMA Ship;
GO
CREATE TABLE Ship.Labels
(
    LabelID  int          NOT NULL CONSTRAINT PK_Labels PRIMARY KEY,
    Barcode  char(10)     NOT NULL,
    WeightKg decimal(6,2) NOT NULL CONSTRAINT CK_Labels_WeightKg CHECK (WeightKg > 0)
);
GO
-- Batch 1
SET XACT_ABORT OFF;
BEGIN TRY
    BEGIN TRANSACTION;
    INSERT INTO Ship.Labels VALUES (1, 'PKG0000001', 2.50);
    INSERT INTO Ship.Labels VALUES (2, 'PKG0000002', -1.00);
    INSERT INTO Ship.Labels VALUES (3, 'PKG0000003', 5.00);
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF XACT_STATE() = 1
    BEGIN
        PRINT 'Batch 1: committable, committing';
        COMMIT TRANSACTION;
    END
    IF XACT_STATE() = -1
    BEGIN
        PRINT 'Batch 1: doomed, rolling back';
        ROLLBACK TRANSACTION;
    END
END CATCH;
GO
-- Batch 2
SET XACT_ABORT ON;
BEGIN TRY
    BEGIN TRANSACTION;
    INSERT INTO Ship.Labels VALUES (4, 'PKG0000004', 1.25);
    INSERT INTO Ship.Labels VALUES (5, 'PKG0000005', -2.00);
    INSERT INTO Ship.Labels VALUES (6, 'PKG0000006', 3.75);
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF XACT_STATE() = 1
    BEGIN
        PRINT 'Batch 2: committable, committing';
        COMMIT TRANSACTION;
    END
    IF XACT_STATE() = -1
    BEGIN
        PRINT 'Batch 2: doomed, rolling back';
        ROLLBACK TRANSACTION;
    END
    IF XACT_STATE() = 0
        PRINT 'Batch 2: no open transaction remains';
END CATCH;
GO
-- Batch 3
SET XACT_ABORT OFF;
BEGIN TRY
    RAISERROR('Routine label audit notice.', 10, 1);
    PRINT 'Batch 3: after RAISERROR';
    THROW 50001, 'Label audit failed.', 1;
    PRINT 'Batch 3: after THROW';
END TRY
BEGIN CATCH
    PRINT CONCAT('Batch 3: caught error ', ERROR_NUMBER(), ', severity ', ERROR_SEVERITY());
END CATCH;
GO
-- Batch 4
BEGIN TRY
    INSERT INTO Ship.Labels VALUES (7, 'PKG0000007', 4.10);
    INSERT INTO Ship.Labels VALUES (7, 'PKG0000008', 6.00);
END TRY
BEGIN CATCH
    PRINT CONCAT('Batch 4: caught error ', ERROR_NUMBER());
    THROW;
END CATCH;
PRINT 'Batch 4: completed';
GO
-- Batch 5
SELECT LabelID, Barcode, WeightKg
FROM Ship.Labels
ORDER BY LabelID;
GO
```

Predict the complete output of batches 1 through 5: every `PRINT` line, every message, and the exact result set of the final `SELECT` — including which rows physically survive in `Ship.Labels`.

## Correct Answer

Messages produced (in this exact order):

```text
Batch 1: committable, committing
Batch 2: doomed, rolling back
Batch 2: no open transaction remains
Routine label audit notice.
Batch 3: after RAISERROR
Batch 3: caught error 50001, severity 16
Batch 4: caught error 2627
Msg 2627, Level 14, State 1, Line 4
Violation of PRIMARY KEY constraint 'PK_Labels'. Cannot insert duplicate key in object 'Ship.Labels'. The duplicate key value is (7).
```

`Batch 3: after THROW` and `Batch 4: completed` are **never** printed.

Result set of Batch 5 (exactly two rows survive):

```text
LabelID     Barcode    WeightKg
----------- ---------- --------
          1 PKG0000001     2.50
          7 PKG0000007     4.10
```

(Both the messages and the final result set above are the verbatim output of running the script on SQL Server 2025; the run is repeatable statement for statement.)

## Explanation

### Batch 1 — `XACT_ABORT OFF`: a constraint violation only kills the statement

Label 1 inserts. Label 2 violates `CK_Labels_WeightKg` (error 547, severity 16). With `SET XACT_ABORT OFF`, a constraint violation is a *statement-terminating* error: only the failing `INSERT` is rolled back, the surrounding transaction stays alive and committable. Control jumps to the CATCH block immediately, which means label 3's `INSERT` — and the `COMMIT` inside the TRY — **never execute**. In the CATCH, `XACT_STATE()` returns `1` ("active, committable user transaction"), so the first `IF` fires, prints `Batch 1: committable, committing`, and commits what is left of the transaction: **label 1 alone is persisted**. Two classic traps in one batch: a caught error still skips the rest of the TRY block, and committing in the CATCH keeps the pre-error work.

### Batch 2 — `XACT_ABORT ON`: the same error dooms the whole transaction

Same shape, but now `SET XACT_ABORT ON`. Per the TRY...CATCH docs, an error that would ordinarily just terminate the statement instead makes the transaction **uncommittable (doomed)** when it happens inside a TRY block with `XACT_ABORT ON`. In the CATCH, `XACT_STATE()` returns `-1`: the transaction still exists (locks held) but can only perform reads or `ROLLBACK TRANSACTION`; any write or `COMMIT` would fail with error 3930. So the `-1` branch prints `Batch 2: doomed, rolling back` and rolls back — label 4's successful insert is undone, and **no row from batch 2 survives**. After the `ROLLBACK`, `XACT_STATE()` is `0` (no active user transaction), so the third `IF` also fires and prints `Batch 2: no open transaction remains`. That is all three `XACT_STATE()` values — `1`, `-1`, `0` — observed in two batches.

### Batch 3 — `RAISERROR` severity 10 vs `THROW`

- `RAISERROR('Routine label audit notice.', 10, 1)` has severity 10. TRY...CATCH *"catches all execution errors that have a severity higher than 10"* — severity 10 and below are informational messages, not errors. So the text `Routine label audit notice.` is simply printed to the messages stream, the CATCH is **not** entered, and execution continues to print `Batch 3: after RAISERROR`.
- `THROW 50001, 'Label audit failed.', 1;` raises a real exception. When `THROW` *initiates* an error, the severity is always **16** (there is no severity parameter). It is caught, and the CATCH prints `Batch 3: caught error 50001, severity 16`. `Batch 3: after THROW` is unreachable.

Note the required syntax detail: the statement *before* a `THROW` must be terminated with a semicolon. Here the preceding `PRINT 'Batch 3: after RAISERROR';` is properly terminated; without the semicolon, `THROW` can be misparsed (for example as a column alias after a `SELECT`), which is the reason the docs mandate the terminator.

### Batch 4 — autocommit, plus a parameterless `THROW` re-throw

There is no `BEGIN TRANSACTION`, so each `INSERT` is its own autocommit transaction. Label 7's first insert commits immediately — that row is durable *before* anything fails. The second insert violates `PK_Labels` (error 2627, severity 14) and is caught: the CATCH prints `Batch 4: caught error 2627`, then executes `THROW;` with no arguments — legal only inside a CATCH block — which re-raises the *original* error to the client with its original number, severity, and state (2627 / Level 14 / State 1 — not 16, because a re-throw preserves the caught error's severity, unlike a new `THROW`). Re-raising an error from a `THROW` statement terminates the batch, so `PRINT 'Batch 4: completed'` never runs. The reported `Line 4` is the line of the failing `INSERT` within the batch (the batch starts at the `-- Batch 4` comment line, since a comment after `GO` belongs to the next batch).

### Batch 5 — who survived

- Batch 1: committed label 1 from the CATCH block (labels 2 and 3 never made it).
- Batch 2: everything rolled back (labels 4, 5, 6 all gone — including the valid ones).
- Batch 4: label 7's first insert autocommitted before the duplicate failed.

Hence exactly `(1, PKG0000001, 2.50)` and `(7, PKG0000007, 4.10)`, in `ORDER BY LabelID`.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

## DP-800 Exam Rule to Remember

```text
XACT_ABORT OFF + constraint error in TRY → statement dies, transaction
                                           committable: XACT_STATE() = 1
XACT_ABORT ON  + any error in TRY        → transaction DOOMED:
                                           XACT_STATE() = -1
                                           (reads + ROLLBACK only;
                                            writes/COMMIT raise 3930)
after ROLLBACK / no transaction          → XACT_STATE() = 0
```

And the `THROW` vs `RAISERROR` scorecard:

| | `RAISERROR` | `THROW` |
|---|---|---|
| Severity | caller chooses; **severity ≤ 10 is informational and is NOT caught by TRY...CATCH** | always 16 when initiating; a parameterless re-throw keeps the original severity |
| Re-throw the caught error | impossible (must rebuild it from `ERROR_*()`) | `THROW;` — allowed **only inside a CATCH block** |
| Statement terminator | none required | the *previous* statement must end with `;` |
| Honors `SET XACT_ABORT` | no | yes |
| After it fires in a CATCH | execution can continue | the batch is terminated |

Finally, remember that in autocommit mode every statement commits on its own: rows inserted *before* a failing statement in the same batch — but not in the same explicit transaction — are already durable, no matter what the CATCH block does afterward.
