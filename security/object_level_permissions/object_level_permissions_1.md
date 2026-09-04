# SQL Server question — Object-Level Permissions 1

## Statement

An agricultural cooperative keeps its finance and operations data in a SQL Server 2025 database named `HarvestCoop`. Access is granted to two database users **without logins**, so that the whole scenario can be tested with `EXECUTE AS USER`.

The complete setup, executed by the database owner (`dbo`), is:

```sql
CREATE DATABASE HarvestCoop;
GO
USE HarvestCoop;
GO
CREATE SCHEMA Fin;
GO
CREATE SCHEMA Ops;
GO
CREATE USER Analyst  WITHOUT LOGIN;
CREATE USER Auditor  WITHOUT LOGIN;
CREATE USER OpsOwner WITHOUT LOGIN;
CREATE ROLE Reporting;
ALTER ROLE Reporting ADD MEMBER Analyst;
ALTER ROLE Reporting ADD MEMBER Auditor;
GO
CREATE TABLE Fin.Invoices
(
    InvoiceID INT           NOT NULL PRIMARY KEY,
    Member    VARCHAR(40)   NOT NULL,
    Amount    DECIMAL(10,2) NOT NULL,
    Margin    DECIMAL(10,2) NOT NULL
);
INSERT INTO Fin.Invoices VALUES
  (1, 'Mas Oliva',  1200.00, 180.00),
  (2, 'Can Roig',    850.00,  95.50),
  (3, 'Vall Verda', 2100.00, 310.00);
CREATE TABLE Ops.Deliveries
(
    DeliveryID INT          NOT NULL PRIMARY KEY,
    InvoiceID  INT          NOT NULL,
    Tons       DECIMAL(6,1) NOT NULL
);
INSERT INTO Ops.Deliveries VALUES (10, 1, 12.5), (11, 2, 8.0), (12, 3, 21.0);
GO
ALTER AUTHORIZATION ON SCHEMA::Ops TO OpsOwner;      -- Ops.Deliveries is now owned by OpsOwner
GO
CREATE VIEW Fin.vInvoiceMargin AS
SELECT InvoiceID, Member, Margin FROM Fin.Invoices;
GO
CREATE VIEW Fin.vDeliveries AS
SELECT d.DeliveryID, d.InvoiceID, d.Tons FROM Ops.Deliveries AS d;
GO
CREATE PROCEDURE Fin.GetMarginStatic @InvoiceID INT AS
    SELECT Margin FROM Fin.Invoices WHERE InvoiceID = @InvoiceID;
GO
CREATE PROCEDURE Fin.GetMarginDynamic @InvoiceID INT AS
    EXEC sp_executesql N'SELECT Margin FROM Fin.Invoices WHERE InvoiceID = @id',
                       N'@id INT', @InvoiceID;
GO
GRANT SELECT  ON SCHEMA::Fin TO Reporting;
GRANT EXECUTE ON SCHEMA::Fin TO Reporting;
DENY  SELECT  ON Fin.Invoices TO Analyst;
GRANT SELECT  ON Fin.Invoices (InvoiceID, Amount) TO Analyst;
GRANT SELECT  ON Ops.Deliveries TO Auditor;
DENY  SELECT  ON SCHEMA::Ops TO Reporting;
GO
```

Schema `Fin` (and therefore every object in it) is owned by `dbo`; schema `Ops` and `Ops.Deliveries` are owned by `OpsOwner`. The following statements are then run, each in its own batch, with the impersonation shown:

```sql
-- S1  (as Analyst)
SELECT InvoiceID, Amount FROM Fin.Invoices ORDER BY InvoiceID;

-- S2  (as Analyst)
SELECT InvoiceID, Margin FROM Fin.Invoices ORDER BY InvoiceID;

-- S3  (as Analyst)
SELECT InvoiceID, Member, Margin FROM Fin.vInvoiceMargin ORDER BY InvoiceID;

-- S4  (as Analyst)
EXEC Fin.GetMarginStatic @InvoiceID = 3;

-- S5  (as Analyst)
EXEC Fin.GetMarginDynamic @InvoiceID = 3;

-- S6  (as Analyst)
SELECT DeliveryID, Tons FROM Fin.vDeliveries ORDER BY DeliveryID;

-- S7  (as Auditor)
SELECT DeliveryID, Tons FROM Ops.Deliveries ORDER BY DeliveryID;

-- S8  (as dbo, then as Analyst)
REVOKE SELECT ON Fin.Invoices FROM Analyst;
EXECUTE AS USER = 'Analyst';
SELECT InvoiceID, Margin FROM Fin.Invoices ORDER BY InvoiceID;
REVERT;
```

For each statement S1–S8 state whether it **succeeds or fails** and, if it fails, with which error.

## Correct Answer

| Stmt | Outcome | Detail (engine's literal output) |
|------|---------|----------------------------------|
| S1 | **Succeeds** | 3 rows: `(1, 1200.00)`, `(2, 850.00)`, `(3, 2100.00)` — column-level `GRANT` wins over the table-level `DENY` |
| S2 | **Fails** | `Msg 230` — `The SELECT permission was denied on the column 'Margin' of the object 'Invoices', database 'HarvestCoop', schema 'Fin'.` |
| S3 | **Succeeds** | 3 rows including `Margin` (180.00, 95.50, 310.00) — unbroken ownership chain `dbo` view → `dbo` table; the `DENY` on the table is never checked |
| S4 | **Succeeds** | 1 row: `310.00` — ownership chain through the procedure |
| S5 | **Fails** | `Msg 230` — same message as S2: dynamic SQL breaks the chain, so `Analyst`'s own (denied) permission on `Margin` is checked |
| S6 | **Fails** | `Msg 229` — `The SELECT permission was denied on the object 'Deliveries', database 'HarvestCoop', schema 'Ops'.` — chain broken by the owner change (`dbo` view over an `OpsOwner` table) |
| S7 | **Fails** | `Msg 229` — same message: the `DENY` inherited through role `Reporting` overrides the direct `GRANT` |
| S8 | **Succeeds** | 3 rows including `Margin` — `REVOKE` removed the table-level `DENY` (and the column grants); the schema-level `GRANT` through `Reporting` now applies |

`HAS_PERMS_BY_NAME`, evaluated as `Analyst` before S8, agrees with the table: `Fin.Invoices`/`SELECT` = **0**, column `Amount` = **1**, column `Margin` = **0**, `Fin.vInvoiceMargin`/`SELECT` = **1**, `Ops.Deliveries`/`SELECT` = **0**.

## Explanation

Four separate permission rules are being exercised. Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

### Rule 1 — DENY beats GRANT, except that a column-level GRANT beats a table-level DENY (S1, S2, S7)

The general precedence is simple: if any applicable `DENY` exists — granted directly or through **any role** the user belongs to — access is refused, regardless of how many `GRANT`s the user also has. S7 is the textbook case: `Auditor` holds a direct `GRANT SELECT ON Ops.Deliveries`, but is a member of `Reporting`, which has `DENY SELECT ON SCHEMA::Ops`; the schema-scoped deny covers the table and wins (`Msg 229`).

The one documented exception is the column-level quirk: *"a table-level DENY does not take precedence over a column-level GRANT"*. `Analyst` has `DENY SELECT ON Fin.Invoices` but also `GRANT SELECT ON Fin.Invoices (InvoiceID, Amount)`. Selecting only those two columns therefore **succeeds** (S1); referencing `Margin`, which has no column grant, hits the deny (S2, `Msg 230` — note the column-specific error number, different from the object-level `Msg 229`). The documentation calls this behaviour legacy and says it may change, but it is what the engine does today.

### Rule 2 — ownership chaining skips permission checks on referenced objects (S3, S4)

When a view or procedure references an object **owned by the same principal**, SQL Server checks permissions only on the *entry* object; the referenced objects are not checked at all — not even for `DENY`s. `Fin.vInvoiceMargin` and `Fin.Invoices` are both owned by `dbo` (they inherit the owner of schema `Fin`). `Analyst` has `SELECT` on the view (via `GRANT SELECT ON SCHEMA::Fin TO Reporting`), so S3 returns every column, **including the denied `Margin`**. This is the subtle point: a `DENY` on a base table is *not* a guarantee that the data is unreachable — any same-owner view or procedure that exposes the column is a legal path. Ownership chaining is the reason the "grant EXECUTE on procedures, nothing on tables" pattern works, and it is also why you must audit which views and modules expose sensitive columns.

S4 is the same mechanism through a stored procedure: `EXECUTE` comes from the schema grant, and the static `SELECT` inside the module is covered by the `dbo`→`dbo` chain.

### Rule 3 — the chain breaks with dynamic SQL or a different owner (S5, S6)

- **Dynamic SQL** (`sp_executesql`, `EXEC (@sql)`) is compiled and executed as a **new batch in the caller's security context**; it is not part of the module's chain. In S5 the inner `SELECT Margin` is checked against `Analyst` directly, and the column deny fires (`Msg 230`). To let a procedure with dynamic SQL work without granting table permissions, use `EXECUTE AS OWNER` on the module or sign it with a certificate.
- **Different owners**: `Fin.vDeliveries` is owned by `dbo`, `Ops.Deliveries` by `OpsOwner`. When the owner changes along the path, the chain breaks at that point and the engine checks the caller's permission on `Ops.Deliveries`. `Analyst` inherits `DENY SELECT ON SCHEMA::Ops` through `Reporting`, so S6 fails with `Msg 229`. Even with no deny, the analyst would still need an explicit `GRANT` on the table. Cross-database chaining is off by default for the same reason.

### Rule 4 — REVOKE removes a DENY too (S8)

`REVOKE` is not "the opposite of GRANT"; it removes **whatever explicit entry** exists for that permission on that securable — a `GRANT` *or* a `DENY`. After `REVOKE SELECT ON Fin.Invoices FROM Analyst`, `sys.database_permissions` lists only `CONNECT` for `Analyst` (the column-level grants were also removed, because they hang off the same table-level permission). Nothing now blocks the schema-scoped `GRANT` that `Analyst` inherits from `Reporting`, so S8 reads `Margin` successfully. The lesson: to *remove* access, do not revoke — `DENY`; to *restore* the default, `REVOKE`.

### Testing without logins

`CREATE USER ... WITHOUT LOGIN` plus `EXECUTE AS USER = '...'` / `REVERT` is the standard way to test a permission model on a single connection; `HAS_PERMS_BY_NAME(securable, securable_class, permission [, sub_securable, sub_securable_class])` evaluates the *effective* permission of the current context, including role membership and denies, and `fn_my_permissions` lists everything the current context holds on a securable.

## DP-800 Exam Rule to Remember

```text
Effective permission = ANY DENY (direct or via any role, at any covering scope) ? no : ANY GRANT ? yes : no
  ... except: column-level GRANT overrides table-level DENY (Msg 230 for other columns).
  Scope nesting: server -> database -> schema -> object -> column ; a schema GRANT/DENY
  covers every object in it.
REVOKE = delete the explicit GRANT *or* DENY.  It is not a denial.

Ownership chaining: same owner all the way down -> only the entry object is checked
  (DENYs on the base table are bypassed by a same-owner view/procedure).
Chain breaks when: the owner changes (ALTER AUTHORIZATION, different schema owner),
  dynamic SQL runs (sp_executesql / EXEC(@s) -> caller's own permissions),
  or the reference crosses databases (unless cross-db ownership chaining is enabled).
Fix for broken chains without granting table rights: EXECUTE AS OWNER / module signing.

Errors: 229 = object-level permission denied, 230 = column-level permission denied.
Test with: CREATE USER x WITHOUT LOGIN; EXECUTE AS USER = 'x'; ... REVERT;
           HAS_PERMS_BY_NAME('Fin.Invoices','OBJECT','SELECT','Margin','COLUMN')
```
