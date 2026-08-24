# SQL Server question — Isolation Levels 1

## Statement

A concert hall runs its ticket booking system on **Azure SQL Database**, in a database named `Ticketing`.

Seat inventory is stored in the following table:

```sql
CREATE TABLE Box.Seats
(
    SeatId    int NOT NULL PRIMARY KEY,
    Status    nvarchar(20) NOT NULL,   -- N'Available' or N'Booked'
    BookedBy  nvarchar(50) NULL
);
GO

INSERT INTO Box.Seats (SeatId, Status, BookedBy)
VALUES (101, N'Available', NULL);
GO
```

The database uses the **Azure SQL Database default settings** for row versioning, which you have verified:

```sql
SELECT is_read_committed_snapshot_on,   -- returns 1 (ON: Azure SQL Database default)
       snapshot_isolation_state_desc    -- returns 'ON' (Azure SQL Database default)
FROM sys.databases
WHERE name = N'Ticketing';
```

That is, `READ_COMMITTED_SNAPSHOT` is `ON` and `ALLOW_SNAPSHOT_ISOLATION` is `ON`.

Two customers try to book seat `101` at almost the same time, from two separate sessions. The statements execute in exactly the following order. Every step completes (or fails) before the next time step starts, except where the question itself is what happens.

**Session 1** (customer Alex):

```sql
-- T1
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
BEGIN TRANSACTION;

-- T2
SELECT Status, BookedBy
FROM Box.Seats
WHERE SeatId = 101;
-- Returns: Status = N'Available', BookedBy = NULL

-- T5
UPDATE Box.Seats
SET Status = N'Booked', BookedBy = N'Alex'
WHERE SeatId = 101;

-- T6
COMMIT TRANSACTION;
```

**Session 2** (customer Dana), running at the default `READ COMMITTED` isolation level:

```sql
-- T3
BEGIN TRANSACTION;

UPDATE Box.Seats
SET Status = N'Booked', BookedBy = N'Dana'
WHERE SeatId = 101;
-- Completes immediately at T3

-- T4
COMMIT TRANSACTION;
-- Completes at T4
```

Interleaved timeline:

| Time | Session 1 (SNAPSHOT)                              | Session 2 (READ COMMITTED)                  |
|------|---------------------------------------------------|---------------------------------------------|
| T1   | `SET ... SNAPSHOT;` `BEGIN TRANSACTION;`          |                                             |
| T2   | `SELECT` seat 101 → `Available`, `NULL`           |                                             |
| T3   |                                                   | `BEGIN TRANSACTION;` `UPDATE` seat 101 → Dana |
| T4   |                                                   | `COMMIT TRANSACTION;`                       |
| T5   | `UPDATE` seat 101 → Alex                          |                                             |
| T6   | `COMMIT TRANSACTION;`                             |                                             |

No other sessions touch `Box.Seats`, no table hints are used, and no lock timeout is configured.

What happens in Session 1 at T5 and T6, and what is the final committed state of seat `101`?

### a.

At T5 the `UPDATE` succeeds immediately, because under `SNAPSHOT` isolation Session 1 operates on its own row version of seat `101`. At T6 the `COMMIT` succeeds and the last write wins.

Final row: `Status = N'Booked'`, `BookedBy = N'Alex'`. Dana's booking is silently overwritten.

### b.

At T5 the `UPDATE` succeeds and writes a new row version, but the conflict with Session 2's change is detected when Session 1 tries to commit. The `COMMIT` at T6 fails with an update-conflict error and the transaction is rolled back.

Final row: `Status = N'Booked'`, `BookedBy = N'Dana'`.

### c.

At T5 the `UPDATE` fails immediately with error **Msg 3960** ("Snapshot isolation transaction aborted due to update conflict"), and Session 1's transaction is automatically rolled back. T6 no longer has an open transaction to commit.

Final row: `Status = N'Booked'`, `BookedBy = N'Dana'`. Alex's booking attempt must be retried in a new transaction.

### d.

At T5 the two sessions form a deadlock: Session 1 waits for the row modified by Session 2, while Session 2 waits for the row version that Session 1 read at T2. The Database Engine chooses Session 1 as the deadlock victim and terminates it with error 1205.

Final row: `Status = N'Booked'`, `BookedBy = N'Dana'`.

## Correct Answer

**c**

## Explanation

The correct answer is **c**.

This question tests the one behavior that separates `SNAPSHOT` isolation from every locking isolation level: **writes are validated with optimistic conflict detection**. A snapshot transaction that tries to modify a row that another transaction modified *and committed* after the snapshot transaction began does not block, does not silently overwrite, and does not wait until `COMMIT` — the conflicting DML statement itself fails with:

```text
Msg 3960, Level 16, State 2
Snapshot isolation transaction aborted due to update conflict.
```

and the snapshot transaction is **rolled back automatically**.

### Step-by-step trace of the row versions

- **T1** — `SET TRANSACTION ISOLATION LEVEL SNAPSHOT` is legal because `ALLOW_SNAPSHOT_ISOLATION` is `ON` (the Azure SQL Database default; on SQL Server and Azure SQL Managed Instance it defaults to `OFF`). `BEGIN TRANSACTION` alone does not yet fix the snapshot: a snapshot transaction logically starts **at its first data access**, not at `BEGIN TRANSACTION`.

- **T2** — The `SELECT` is Session 1's first data access. The transaction receives its transaction sequence number and from now on sees the database as it existed at this moment: seat `101` is `Available`. Crucially, under `SNAPSHOT` isolation this read acquires **no shared locks** on the row.

- **T3** — Session 2's `UPDATE` requests an exclusive (X) lock on seat `101`. Because Session 1 holds no locks on the row (versioned read), Session 2 is **not blocked** and modifies the row at once. The pre-update image (`Available`, `NULL`) is preserved in the version store, which is exactly what keeps Session 1's snapshot consistent.

- **T4** — Session 2 commits. Its X lock is released. The current committed row is now `Booked` / `Dana`.

- **T5** — Session 1 issues its `UPDATE`. A write under `SNAPSHOT` isolation cannot write to a stale version: the Database Engine checks the **latest committed version** of the row and finds it was modified by a transaction (Session 2) that committed *after* Session 1's snapshot transaction began. This is an update conflict. There is nothing to wait for — Session 2's lock is already gone — so the statement fails **immediately** with Msg 3960, and the Database Engine **terminates and rolls back Session 1's entire transaction**. The rollback is automatic; it does not wait for the application to issue `ROLLBACK`.

- **T6** — There is no longer an open transaction in Session 1, so the `COMMIT` has nothing to commit (attempting it raises the "no corresponding BEGIN TRANSACTION" error, Msg 3902).

Final committed state: `SeatId = 101`, `Status = N'Booked'`, `BookedBy = N'Dana'`. The seat correctly remains Dana's; Alex's application must catch error 3960 and **retry in a new transaction** — the standard retry-logic pattern for optimistic concurrency.

### Why option a is wrong

Option a describes a **lost update**, which is precisely what snapshot conflict detection exists to prevent. Row versioning applies to *reads*; it never lets a transaction *write* over a newer committed version.

For option a to happen, Session 1 would have to run under `READ COMMITTED` (with `READ_COMMITTED_SNAPSHOT ON`, the Azure SQL Database default) instead of `SNAPSHOT`. Read-committed snapshot provides only **statement-level** consistency and performs **no update-conflict detection**: at T5 the `UPDATE` would simply lock and modify the *current* committed row, and the commit at T6 would succeed, silently replacing `Dana` with `Alex`. That is the classic read-then-overwrite lost update — and the reason a booking system on RCSI alone needs additional protection (`UPDLOCK`, rowversion checks, or `SNAPSHOT` isolation).

### Why option b is wrong

Option b transplants **commit-time validation** from other optimistic systems into the wrong place. For disk-based tables, SQL Server detects a snapshot update conflict **when the conflicting `UPDATE`/`DELETE` statement executes**, not when the transaction commits. At T5 the latest committed version is already newer than Session 1's snapshot, so the statement itself is what fails; execution never reaches a `COMMIT` that could fail.

Commit-time (or lock-release-time) conflict resolution does exist elsewhere — for example, memory-optimized tables validate at commit — but not for a disk-based `UPDATE` under `SNAPSHOT` isolation. The only variation on timing in this pattern: if Session 2 had still been **uncommitted** at T5, Session 1's `UPDATE` would first block on Session 2's X lock and then fail with Msg 3960 the moment Session 2 committed — still at the statement, never at Session 1's `COMMIT`.

### Why option d is wrong

A deadlock requires a **cycle of mutual waiting**: each session must hold a lock that the other needs. That is impossible in this timeline for two reasons:

1. Session 1's `SELECT` at T2 acquired no locks at all — snapshot reads are versioned, not locked — so Session 2 never waits on Session 1.
2. Session 2's transaction has already committed at T4, before Session 1 ever requests a lock. At T5 there is only one active transaction; nobody is waiting on anybody.

For a deadlock (error 1205) to be possible, both sessions would have to hold locks and then request each other's resources — for example, both running under `REPEATABLE READ`, each doing `SELECT` (shared lock held to end of transaction) on the same seat and then both attempting the `UPDATE` (lock conversion to exclusive): each would wait for the other's shared lock to be released, and the Database Engine would kill one victim. One of `SNAPSHOT` isolation's selling points is exactly that its lock-free reads make this reader-writer deadlock pattern far less likely.

## DP-800 Exam Rule to Remember

`SNAPSHOT` isolation is **optimistic**: readers never block and are never blocked, but every write is validated against the latest committed version of the row.

```text
Row changed and committed by someone else
after my snapshot transaction began
              +
I try to UPDATE / DELETE that row
              =
Msg 3960 at the STATEMENT (not at COMMIT)
+ automatic rollback of my whole transaction
```

Decision points to memorize:

- **Azure SQL Database defaults**: `READ_COMMITTED_SNAPSHOT ON` **and** `ALLOW_SNAPSHOT_ISOLATION ON` (SQL Server on-premises defaults both to `OFF`).
- The snapshot is established at the transaction's **first data access**, not at `BEGIN TRANSACTION`.
- Conflicting writer still active → snapshot updater **blocks**, then gets 3960 when the writer commits (or succeeds if the writer rolls back). Conflicting writer already committed → **immediate** 3960.
- RCSI (`READ COMMITTED` + row versioning) gives versioned reads but **no conflict detection** → lost updates are possible.
- Mitigations: retry logic for error 3960, or take the lock early (`SELECT ... WITH (UPDLOCK)`) to trade optimism for blocking.
