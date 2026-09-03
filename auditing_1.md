# SQL Server question — Auditing 1

## Statement

A port authority runs a SQL Server 2025 instance hosting the database `DockYard`. Customs regulations require an audit trail of **every read of the cargo manifests** and of **every change to database principals**, written to a binary file that auditors read back with T-SQL.

The security administrator (a `sysadmin`) runs the following script. The audit target folder `C:\dp800audit\` exists and the SQL Server service account can write to it.

```sql
USE master;
GO
CREATE SERVER AUDIT DockYard_Audit
TO FILE (FILEPATH = 'C:\dp800audit\', MAXSIZE = 10 MB, MAX_ROLLOVER_FILES = 2)
WITH (QUEUE_DELAY = 0, ON_FAILURE = CONTINUE);
GO
CREATE SERVER AUDIT SPECIFICATION DockYard_ServerSpec
FOR SERVER AUDIT DockYard_Audit
ADD (FAILED_LOGIN_GROUP),
ADD (SERVER_PRINCIPAL_CHANGE_GROUP)
WITH (STATE = ON);
GO
CREATE DATABASE DockYard;
GO
USE DockYard;
GO
CREATE SCHEMA Cargo;
GO
CREATE TABLE Cargo.Manifests
(
    ManifestID    INT           NOT NULL PRIMARY KEY,
    Vessel        VARCHAR(40)   NOT NULL,
    Shipper       VARCHAR(60)   NOT NULL,
    DeclaredValue DECIMAL(12,2) NOT NULL
);
INSERT INTO Cargo.Manifests VALUES
  (1, 'MV Tramontana', 'Nordic Steel AS',  250000.00),
  (2, 'MV Garbi',      'Iberia Citrus SL',  48000.00);
CREATE TABLE Cargo.Berths (BerthID INT NOT NULL PRIMARY KEY, Name VARCHAR(20) NOT NULL);
INSERT INTO Cargo.Berths VALUES (1, 'North-1'), (2, 'South-3');
GO
CREATE VIEW Cargo.vManifestSummary AS SELECT ManifestID, Vessel FROM Cargo.Manifests;
GO
CREATE USER Customs WITHOUT LOGIN;
GRANT SELECT, UPDATE ON SCHEMA::Cargo TO Customs;
GO
CREATE DATABASE AUDIT SPECIFICATION DockYard_DbSpec
FOR SERVER AUDIT DockYard_Audit
ADD (SELECT ON OBJECT::Cargo.Manifests BY public),
ADD (DATABASE_PRINCIPAL_CHANGE_GROUP)
WITH (STATE = ON);
GO
```

Then these actions are executed in order (all statements succeed unless stated otherwise):

```sql
-- A1  (as Customs)                          -- the server audit has NOT been enabled yet
SELECT ManifestID FROM Cargo.Manifests WHERE ManifestID = 1;

-- A2  (as the administrator, in master)
ALTER SERVER AUDIT DockYard_Audit WITH (STATE = ON);

-- A3  (as Customs)
SELECT Vessel, DeclaredValue FROM Cargo.Manifests WHERE ManifestID = 1;

-- A4  (as Customs)
SELECT Name FROM Cargo.Berths;

-- A5  (as Customs)
UPDATE Cargo.Manifests SET DeclaredValue = 49000.00 WHERE ManifestID = 2;

-- A6  (as Customs)
SELECT Vessel FROM Cargo.vManifestSummary;

-- A7  (as the administrator)
CREATE USER Inspector WITHOUT LOGIN;

-- A8  (as the administrator)
GRANT SELECT ON Cargo.Berths TO Inspector;

-- A9  (as the administrator, i.e. dbo)
SELECT COUNT(*) FROM Cargo.Manifests;
```

Finally the auditor runs:

```sql
SELECT action_id, database_principal_name, object_name, statement
FROM sys.fn_get_audit_file('C:\dp800audit\DockYard_Audit_*.sqlaudit', DEFAULT, DEFAULT)
ORDER BY event_time, sequence_number;
```

Which rows does the query return (one row per audited event, in order)?

### a.

| action_id | database_principal_name | object_name | statement |
|---|---|---|---|
| SL | Customs | Manifests | A1 `SELECT ManifestID ...` |
| AUSC | | | (audit state change) |
| SL | Customs | Manifests | A3 |
| SL | Customs | Berths | A4 |
| SL | Customs | Manifests | A6 |
| CR | dbo | Inspector | A7 |
| SL | dbo | Manifests | A9 |

### b.

| action_id | database_principal_name | object_name | statement |
|---|---|---|---|
| AUSC | | | (audit state change) |
| SL | Customs | Manifests | A3 |
| SL | Customs | vManifestSummary | A6 |
| CR | dbo | Inspector | A7 |
| G | dbo | Berths | A8 |

### c.

| action_id | database_principal_name | object_name | statement |
|---|---|---|---|
| AUSC | | | (audit state change) |
| SL | Customs | Manifests | A3 |
| SL | Customs | Manifests | A5 `UPDATE Cargo.Manifests SET ...` |
| SL | Customs | Manifests | A6 |
| CR | dbo | Inspector | A7 |
| SL | dbo | Manifests | A9 |

### d.

| action_id | database_principal_name | object_name | statement |
|---|---|---|---|
| AUSC | | | (audit state change) |
| SL | Customs | Manifests | A3 |
| UP | Customs | Manifests | A5 |
| SL | Customs | Manifests | A6 |
| CR | dbo | Inspector | A7 |

## Correct Answer

**c**

## Explanation

The correct answer is **c**. The whole script was executed and the audit file read back; the returned rows were, verbatim (statement column truncated):

```text
action_id class_type succeeded db_principal object_name statement
AUSC      A          1                                  (ALTER SERVER AUDIT ... STATE = ON)
SL        U          1         Customs      Manifests   SELECT Vessel, DeclaredValue FROM Cargo.Manifests WHERE ManifestID = 1
SL        U          1         Customs      Manifests   UPDATE Cargo.Manifests SET DeclaredValue = 49000.00 WHERE ManifestID = 2
SL        U          1         Customs      Manifests   SELECT Vessel FROM Cargo.vManifestSummary
CR        SU         1         dbo          Inspector   CREATE USER Inspector WITHOUT LOGIN;
SL        U          1         dbo          Manifests   SELECT COUNT(*) AS N FROM Cargo.Manifests
```

### Why option c is correct

Walk through the audit model:

- **A server audit is created disabled** ("You create a server audit in a disabled state"). `DockYard_DbSpec` was created `WITH (STATE = ON)`, but a specification only feeds an audit; until `ALTER SERVER AUDIT ... WITH (STATE = ON)` the target receives nothing. **A1 is not recorded.** Note also that `ALTER SERVER AUDIT` must be issued from `master`: run from `DockYard` it fails with `Msg 33074 — Cannot alter a server audit from a user database. This operation must be performed in the master database.`
- **Enabling the audit is itself audited** (`AUSC`, class `A`): "Server Audit state change (setting state to ON or OFF)" is intrinsically audited, with no action group required. That is the first row.
- **A3** is a `SELECT` on `Cargo.Manifests` by a member of `public` → `SL` on `Manifests`.
- **A4** reads `Cargo.Berths`, which is not in the specification → not recorded. The database audit action `SELECT ON OBJECT::Cargo.Manifests` is object-specific; auditing the whole schema would need `ON SCHEMA::Cargo`, and auditing every object access in the database would need `SCHEMA_OBJECT_ACCESS_GROUP`.
- **A5** is the subtle row. Only `SELECT` was added to the specification, yet the `UPDATE` appears — as action `SL`, not `UP`. An `UPDATE ... WHERE ManifestID = 2` must *read* the table to evaluate its predicate, so the engine checks the `SELECT` permission on `Cargo.Manifests` and the audit action for `SELECT` fires ("This event is raised whenever a `SELECT` is issued" is really "whenever the SELECT permission on the object is used"). The same happens for an `UPDATE Cargo.Berths ... WHERE BerthID = (SELECT MIN(ManifestID) FROM Cargo.Manifests)`: it is logged as `SL` on `Manifests`. An `UP` row would require `ADD (UPDATE ON OBJECT::Cargo.Manifests BY public)`.
- **A6** goes through the view, but the audit record names the **base table** `Manifests`: object-level audit actions are recorded for the objects whose permission is actually used, and with an unbroken ownership chain the access still counts as access to `Manifests`.
- **A7** `CREATE USER` is captured by `DATABASE_PRINCIPAL_CHANGE_GROUP` → action `CR` (create), class `SU` (user), `object_name = Inspector`.
- **A8** `GRANT` is a *permission* change, covered by `DATABASE_OBJECT_PERMISSION_CHANGE_GROUP` / `SCHEMA_OBJECT_PERMISSION_CHANGE_GROUP`, which were not added → not recorded.
- **A9** — `BY public` includes `dbo`; administrators are not exempt. "To audit actions of the administrators, audit the actions of the dbo user."

`QUEUE_DELAY = 0` means synchronous delivery, so the rows are in the file immediately (the default, 1000 ms, could leave the last events in memory for up to a second). `ON_FAILURE = CONTINUE` keeps the instance running if the file cannot be written (events are lost); `SHUTDOWN` stops the instance and requires the creating login to hold the `SHUTDOWN` permission; `FAIL_OPERATION` fails only the audited actions. Reading the file with `sys.fn_get_audit_file` requires `VIEW SERVER SECURITY AUDIT` on SQL Server 2022+ (`CONTROL SERVER` before).

### Why option a is wrong

It records A1, made *before* the audit was enabled, and A4 on `Berths`, which is not an audited object. Both mistakes come from reading the database audit specification's `STATE = ON` as "auditing is on" — it is not; only the server audit's state matters — and from assuming an object-level action covers the whole schema. It also omits A5.

### Why option b is wrong

Option b attributes A6 to the view name and records the `GRANT` (A8), but omits A5 and A9. The audit names the base object whose permission was used (`Manifests`, not `vManifestSummary`); `GRANT` is not a principal change; and `dbo` is a member of `public`, so administrator reads are captured — that is the point of auditing "BY public".

### Why option d is wrong

Option d gets every row right except two: it labels A5 as `UP`, and it drops A9. A5 is recorded, but as `SL`, because the specification only contains the `SELECT` action — the update itself was **not** audited; only its read of the table was. And `dbo` is not exempt from a `BY public` action. This option is the closest distractor: it tests whether you know that *what gets audited is the permission used*, not the statement type.

### Related facts verified on the same run

- A database-level action cannot go in a **server** audit specification: `CREATE SERVER AUDIT SPECIFICATION ... ADD (SELECT ON OBJECT::Cargo.Manifests BY public)` fails with `Msg 156 — Incorrect syntax near the keyword 'ON'.` Server specifications take server-level action groups (`FAILED_LOGIN_GROUP`, `SERVER_PRINCIPAL_CHANGE_GROUP`, `AUDIT_CHANGE_GROUP`, ...), and some groups exist at both levels (`SCHEMA_OBJECT_ACCESS_GROUP`, `DATABASE_PRINCIPAL_CHANGE_GROUP`, `BATCH_COMPLETED_GROUP`) — at server level they apply to every database.
- A server-only group cannot go in a **database** specification: `ADD (FAILED_LOGIN_GROUP)` in `CREATE DATABASE AUDIT SPECIFICATION` fails with `Msg 102 — Incorrect syntax near 'FAILED_LOGIN_GROUP'.`
- One server audit specification per audit; one database audit specification per database per audit; multiple audits per instance.

### Azure SQL Database

Azure SQL Database has no `CREATE SERVER AUDIT`. Auditing is configured on the logical server (applies to all databases) or per database, and the destination is an Azure Storage account (append blobs, `.xel`), a Log Analytics workspace, or Event Hubs; the default action groups are `BATCH_COMPLETED_GROUP`, `SUCCESSFUL_DATABASE_AUTHENTICATION_GROUP` and `FAILED_DATABASE_AUTHENTICATION_GROUP`, and storage logs are read with `sys.fn_get_audit_file` over the blob URL. Azure SQL Managed Instance keeps the `CREATE SERVER AUDIT` syntax with `TO URL` / `TO EXTERNAL_MONITOR` targets.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

## DP-800 Exam Rule to Remember

```text
SERVER AUDIT (master; the container + target: FILE | APPLICATION_LOG | SECURITY_LOG; MI: URL)
   |- created DISABLED  -> ALTER SERVER AUDIT ... WITH (STATE = ON), from master (Msg 33074)
   |- QUEUE_DELAY 0 = synchronous; ON_FAILURE = CONTINUE | SHUTDOWN (needs SHUTDOWN perm) | FAIL_OPERATION
   |- SERVER AUDIT SPECIFICATION   (1 per audit)    -> server-level groups (logins, principals...)
   '- DATABASE AUDIT SPECIFICATION (1 per DB/audit) -> db-level groups + actions:
        ADD (SELECT|INSERT|UPDATE|DELETE|EXECUTE ON OBJECT::s.t | SCHEMA::s | DATABASE::d BY principal)
Audit records the PERMISSION USED on the BASE object: an UPDATE ... WHERE shows up as SL;
   a read through a view shows the table; BY public includes dbo. Enabling the audit logs AUSC.
Read: sys.fn_get_audit_file('path\Name_*.sqlaudit', DEFAULT, DEFAULT)  (VIEW SERVER SECURITY AUDIT)
Auditing = who did what (reads included)  |  Ledger = tamper-evident proof of data history
Temporal = queryable data history (no security guarantee)  |  Azure SQL DB: storage / Log Analytics / Event Hubs
```
