# SQL Server question — Triggers 2

## Statement

A fish farm tracks its ponds and the batches of fish stocked in them in a SQL Server database named `AquaFarm`. Operators enter stock through a grouped view, and the DBA protects the schema with a DDL trigger. The T-SQL below is the **complete** history of the database, executed top to bottom in SQL Server Management Studio (default session options) with no errors. The instance runs with the default `nested triggers` server option (`1`).

```sql
CREATE DATABASE AquaFarm;
GO
USE AquaFarm;
GO
CREATE SCHEMA Farm;
GO
CREATE TABLE Farm.Tanks
(
    TankId   INT          NOT NULL PRIMARY KEY,
    TankName NVARCHAR(30) NOT NULL
);
CREATE TABLE Farm.Batches
(
    BatchId   INT IDENTITY(1,1) PRIMARY KEY,
    TankId    INT         NOT NULL REFERENCES Farm.Tanks(TankId),
    Species   VARCHAR(20) NOT NULL,
    FishCount INT         NOT NULL
);
CREATE TABLE Farm.EventLog
(
    LogId  INT IDENTITY(1,1) PRIMARY KEY,
    Source VARCHAR(40)   NOT NULL,
    Detail NVARCHAR(200) NULL
);
GO
INSERT INTO Farm.Tanks (TankId, TankName) VALUES (1, N'East Pond'), (2, N'West Pond');
INSERT INTO Farm.Batches (TankId, Species, FishCount) VALUES (1, 'Salmon', 400), (2, 'Trout', 250);
GO
CREATE VIEW Farm.TankStock
AS
SELECT t.TankId, t.TankName, b.Species, SUM(b.FishCount) AS TotalFish
FROM Farm.Tanks AS t
JOIN Farm.Batches AS b ON b.TankId = t.TankId
GROUP BY t.TankId, t.TankName, b.Species;
GO
CREATE TRIGGER Farm.trg_TankStock_Insert
ON Farm.TankStock
INSTEAD OF INSERT
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO Farm.EventLog (Source, Detail)
    SELECT 'INSTEAD OF TankStock', CONCAT(COUNT(*), N' view row(s)') FROM inserted;

    INSERT INTO Farm.Tanks (TankId, TankName)
    SELECT DISTINCT i.TankId, i.TankName
    FROM inserted AS i
    WHERE NOT EXISTS (SELECT 1 FROM Farm.Tanks AS t WHERE t.TankId = i.TankId);

    INSERT INTO Farm.Batches (TankId, Species, FishCount)
    SELECT i.TankId, i.Species, i.TotalFish FROM inserted AS i;
END;
GO
CREATE TRIGGER Farm.trg_Batches_AuditA
ON Farm.Batches
AFTER INSERT
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO Farm.EventLog (Source, Detail)
    SELECT 'AFTER Batches A', CONCAT(COUNT(*), N' batch row(s)') FROM inserted;
END;
GO
CREATE TRIGGER Farm.trg_Batches_AuditB
ON Farm.Batches
AFTER INSERT
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO Farm.EventLog (Source, Detail)
    SELECT 'AFTER Batches B', CONCAT(COUNT(*), N' batch row(s)') FROM inserted;
END;
GO
CREATE TRIGGER Farm.trg_Batches_Cull
ON Farm.Batches
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    IF NOT EXISTS (SELECT 1 FROM inserted WHERE FishCount > 1000) RETURN;

    INSERT INTO Farm.EventLog (Source, Detail)
    SELECT 'AFTER Batches Cull', CONCAT(N'batch ', BatchId, N' at ', FishCount)
    FROM inserted WHERE FishCount > 1000;

    UPDATE b SET b.FishCount = b.FishCount / 2
    FROM Farm.Batches AS b
    JOIN inserted AS i ON i.BatchId = b.BatchId
    WHERE i.FishCount > 1000;
END;
GO
CREATE TRIGGER trg_ProtectSchema
ON DATABASE
FOR DROP_TABLE, ALTER_TABLE
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @e XML = EVENTDATA();
    DECLARE @what NVARCHAR(200) =
        @e.value('(/EVENT_INSTANCE/EventType)[1]',  'NVARCHAR(50)') + N' '
      + @e.value('(/EVENT_INSTANCE/SchemaName)[1]', 'NVARCHAR(50)') + N'.'
      + @e.value('(/EVENT_INSTANCE/ObjectName)[1]', 'NVARCHAR(50)');

    INSERT INTO Farm.EventLog (Source, Detail) VALUES ('DDL attempt', @what);
    ROLLBACK;
    INSERT INTO Farm.EventLog (Source, Detail) VALUES ('DDL blocked', @what);
END;
GO
```

The following eleven statements are then executed **in order, each in its own batch**, in the same session:

```sql
-- S1
EXEC sp_settriggerorder @triggername = 'Farm.trg_Batches_AuditB', @order = 'First', @stmttype = 'INSERT';

-- S2
EXEC sp_settriggerorder @triggername = 'Farm.trg_TankStock_Insert', @order = 'First', @stmttype = 'INSERT';

-- S3
INSERT INTO Farm.TankStock (TankId, TankName, Species, TotalFish)
VALUES (3, N'North Pond', 'Trout', 300),
       (1, N'East Pond',  'Trout', 120);

-- S4
UPDATE Farm.Batches SET FishCount = 5000 WHERE BatchId = 1;

-- S5
ALTER DATABASE AquaFarm SET RECURSIVE_TRIGGERS ON;

-- S6
UPDATE Farm.Batches SET FishCount = 5000 WHERE BatchId = 2;

-- S7
DROP TABLE Farm.Batches;

-- S8
DISABLE TRIGGER trg_ProtectSchema ON DATABASE;

-- S9
ALTER TABLE Farm.Batches ADD Notes NVARCHAR(50) NULL;

-- S10
DISABLE TRIGGER Farm.trg_Batches_AuditA ON Farm.Batches;

-- S11
INSERT INTO Farm.Batches (TankId, Species, FishCount) VALUES (2, 'Carp', 90);
```

For each statement S1–S11, state whether it **succeeds or raises an error** (and, for DML successes, the rows-affected count reported to the client). Then give the exact result of these two final queries:

```sql
SELECT LogId, Source, Detail FROM Farm.EventLog ORDER BY LogId;
SELECT BatchId, TankId, Species, FishCount FROM Farm.Batches ORDER BY BatchId;
```

## Correct Answer

Per-statement outcomes (all error numbers and messages are the engine's actual output):

| Stmt | Outcome | Detail |
|------|---------|--------|
| S1 | **Succeeds** | `trg_Batches_AuditB` is now the first `AFTER INSERT` trigger on `Farm.Batches` |
| S2 | **Fails** | `Msg 15133` — `INSTEAD OF trigger 'Farm.trg_TankStock_Insert' cannot be associated with an order.` |
| S3 | **Succeeds** | `(2 rows affected)` — the `INSTEAD OF` trigger creates tank 3 and batches 3 and 4; the two `AFTER INSERT` triggers on `Farm.Batches` fire once each (B before A) |
| S4 | **Succeeds** | `(1 rows affected)` — `trg_Batches_Cull` fires once and halves batch 1 to 2500; its own `UPDATE` does **not** re-fire it (`RECURSIVE_TRIGGERS` is `OFF`) |
| S5 | **Succeeds** | direct recursion is now enabled for the database |
| S6 | **Succeeds** | `(1 rows affected)` — `trg_Batches_Cull` fires three times: 5000 → 2500 → 1250 → 625, then the `IF NOT EXISTS ... RETURN` guard stops the recursion |
| S7 | **Fails** | `Msg 3609` — `The transaction ended in the trigger. The batch has been aborted.` — the table is not dropped; the `'DDL attempt'` row is rolled back, the `'DDL blocked'` row (written after `ROLLBACK`) persists |
| S8 | **Succeeds** | the DDL trigger is disabled (`sys.triggers.is_disabled = 1`), not dropped |
| S9 | **Succeeds** | column `Notes` added; nothing is logged |
| S10 | **Succeeds** | `trg_Batches_AuditA` disabled |
| S11 | **Succeeds** | `(1 rows affected)` — only `trg_Batches_AuditB` fires |

Final contents of `Farm.EventLog` (note the gap at `LogId 8`):

| LogId | Source | Detail |
|---|---|---|
| 1 | INSTEAD OF TankStock | 2 view row(s) |
| 2 | AFTER Batches B | 2 batch row(s) |
| 3 | AFTER Batches A | 2 batch row(s) |
| 4 | AFTER Batches Cull | batch 1 at 5000 |
| 5 | AFTER Batches Cull | batch 2 at 5000 |
| 6 | AFTER Batches Cull | batch 2 at 2500 |
| 7 | AFTER Batches Cull | batch 2 at 1250 |
| 9 | DDL blocked | DROP_TABLE Farm.Batches |
| 10 | AFTER Batches B | 1 batch row(s) |

Final contents of `Farm.Batches`:

| BatchId | TankId | Species | FishCount |
|---|---|---|---|
| 1 | 1 | Salmon | 2500 |
| 2 | 2 | Trout | 625 |
| 3 | 3 | Trout | 300 |
| 4 | 1 | Trout | 120 |
| 5 | 2 | Carp | 90 |

## Explanation

### S1 and S2 — `sp_settriggerorder` orders `AFTER` triggers only

When a table has several `AFTER` triggers for the same event, SQL Server does not define their firing order; `sp_settriggerorder` lets you pin exactly one trigger as `First` and one as `Last` per event (`@stmttype = 'INSERT' | 'UPDATE' | 'DELETE'`, or a DDL event name together with `@namespace = 'DATABASE' | 'SERVER'` — verified: omitting `@namespace` for a DDL trigger raises `Msg 15600 — An invalid parameter or option was specified for procedure 'sys.sp_settriggerorder'.`). Any trigger not pinned fires somewhere in between. S1 therefore makes B fire before A, and `sys.trigger_events` shows `is_first = 1` for `trg_Batches_AuditB`.

There can be only **one** `INSTEAD OF` trigger per event per table or view, so an order makes no sense for it: S2 fails with error 15133. This is the first subtle distractor — the procedure call is syntactically perfect.

### S3 — an `INSTEAD OF` trigger makes a grouped, two-table view writable

Without the trigger, an `INSERT` into `Farm.TankStock` would be rejected (the view has `GROUP BY`, an aggregate and two base tables). An `INSTEAD OF INSERT` trigger replaces the statement entirely: the engine never tries to insert into the view, it just fires the trigger with `inserted` populated from the `VALUES` list — including a value for the aggregate column `TotalFish`, which the trigger is free to interpret (here, as the batch size). The trigger writes log row 1, adds tank 3 (tank 1 already exists, so `NOT EXISTS` skips it) and inserts two rows into `Farm.Batches`.

That nested `INSERT` fires the base table's `AFTER INSERT` triggers because the server option `nested triggers` is `1` (its default): rows 2 and 3, B first (S1), each seeing both batch rows at once. So the ordering rule is: **`INSTEAD OF` runs first and in place of the statement; `AFTER` triggers run after the DML the `INSTEAD OF` body actually performs; `sp_settriggerorder` decides among the `AFTER` ones**. The client still receives `(2 rows affected)` — the count of the original statement, not of the trigger's inner inserts.

### S4, S5 and S6 — direct recursion needs `RECURSIVE_TRIGGERS ON`

`trg_Batches_Cull` updates the very table it is defined on. Whether that inner `UPDATE` fires the trigger again is governed by the **database** option `RECURSIVE_TRIGGERS` (direct recursion), which is `OFF` by default (`sys.databases.is_recursive_triggers_on = 0`). So S4 logs one row and halves batch 1 exactly once: 5000 → 2500.

After S5 turns the option on, S6 recurses: 5000 → log row 5, halve to 2500 → trigger fires again on its own update → log row 6, halve to 1250 → fires again → log row 7, halve to 625 → fires a fourth time, but `inserted` now holds 625, the guard returns, recursion stops. Only the guard makes this safe: a trigger that always updates its own table recurses until `Msg 217 — Maximum stored procedure, function, trigger, or view nesting level exceeded (limit 32).`, and the whole statement is rolled back (verified on this build with a guard-less trigger).

Two different switches, easily confused on the exam: `nested triggers` (server-level, `sp_configure`, default 1) controls whether a trigger's DML fires triggers on **other** tables (S3 relied on it), and also gates *indirect* recursion; `RECURSIVE_TRIGGERS` (database-level `ALTER DATABASE`, default OFF) controls **direct** recursion — a trigger firing itself.

### S7 — `EVENTDATA()` and `ROLLBACK` in a DDL trigger

A DDL trigger `ON DATABASE FOR DROP_TABLE, ALTER_TABLE` fires after the DDL statement has run, inside its transaction. `EVENTDATA()` returns an `xml` document whose `/EVENT_INSTANCE` element carries `EventType` (`DROP_TABLE`), `SchemaName`, `ObjectName`, `LoginName`, `TSQLCommand/CommandText`, and so on; the trigger extracts three of them into `@what`. Because the trigger uses XML methods, it must be created with `QUOTED_IDENTIFIER ON` (SSMS's default; a session with it `OFF` would compile the trigger but get `Msg 1934` at fire time).

`ROLLBACK` undoes the `DROP TABLE` **and** everything the trigger did before it — so the `'DDL attempt'` row vanishes. Statements after the `ROLLBACK` still execute, in their own auto-commit transactions, which is why `'DDL blocked'` survives as `LogId 9`. The gap at 8 is the identity value consumed by the rolled-back row (identity values are never reused after a rollback). The client sees error 3609 and the batch is aborted; `Farm.Batches` still exists. (Logging *before* the `ROLLBACK` and expecting the row to persist is the second subtle distractor; `RAISERROR`/`PRINT` is the other common way to report from a DDL trigger, since messages are not transactional.)

### S8 to S11 — `DISABLE TRIGGER` keeps the object, stops the firing

`DISABLE TRIGGER trg_ProtectSchema ON DATABASE` (or `DISABLE TRIGGER ALL ON DATABASE`) flips `is_disabled` to 1 without dropping the trigger; S9's `ALTER TABLE` then runs unlogged and unblocked, and `ENABLE TRIGGER` would restore protection. Likewise, after S10 disables `trg_Batches_AuditA` on the table, the direct insert in S11 fires only `trg_Batches_AuditB` (log row 10). The `ALTER TABLE ... ADD Notes` in S9 does not invalidate the triggers on `Farm.Batches`; they keep working in S11.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

## DP-800 Exam Rule to Remember

```text
INSTEAD OF  -> replaces the statement; the only way to INSERT/UPDATE/DELETE through a
               GROUP BY / multi-table view; one per event per object; no ordering (Msg 15133)
AFTER       -> fires after the DML that actually happened (including DML issued by an
               INSTEAD OF body); several per event, order undefined unless sp_settriggerorder
               @order = 'First' | 'Last' | 'None'  (+ @namespace for DDL triggers)
Nesting     -> 'nested triggers' server option (default 1): a trigger's DML fires other tables' triggers
Recursion   -> ALTER DATABASE ... SET RECURSIVE_TRIGGERS ON (default OFF): a trigger can fire itself;
               always guard the recursion or hit the 32-level limit (Msg 217) and lose the whole statement
DDL trigger -> ON DATABASE | ALL SERVER FOR <event list>; EVENTDATA() is XML (EventType, ObjectName,
               TSQLCommand); ROLLBACK cancels the DDL AND the trigger's earlier writes (Msg 3609 to the client);
               code after ROLLBACK still runs and commits
DISABLE TRIGGER x ON table | ON DATABASE | ON ALL SERVER  -> is_disabled = 1, object kept; ENABLE TRIGGER undoes it
```

When you trace a trigger scenario, ask three questions per statement: *which* triggers can fire (`INSTEAD OF` first, then `AFTER`, disabled ones never), *how many times* (once per statement, plus recursion only if `RECURSIVE_TRIGGERS ON` and the body touches its own table), and *what survives a `ROLLBACK`* (nothing written before it — including identity values, which are burned, not reused).
