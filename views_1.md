# SQL Server question — Views 1

## Statement

A university stores its course-registration catalog in a SQL Server database named `CampusReg`. The registrar's office exposes the Computer Science part of the catalog to advisors through a view, and advisors modify data **only through that view**.

The complete setup is:

```sql
CREATE DATABASE CampusReg;
GO
USE CampusReg;
GO
CREATE SCHEMA Acad;
GO
CREATE TABLE Acad.Course
(
    CourseID INT          NOT NULL PRIMARY KEY,
    Dept     CHAR(4)      NOT NULL,
    Title    NVARCHAR(60) NOT NULL,
    Credits  TINYINT      NOT NULL,
    Seats    SMALLINT     NOT NULL CONSTRAINT DF_Course_Seats DEFAULT (30)
);
GO
INSERT INTO Acad.Course (CourseID, Dept, Title, Credits, Seats) VALUES
  (101, 'CS',   N'Programming I',  3, 120),
  (102, 'CS',   N'Databases',      4,  80),
  (205, 'MATH', N'Linear Algebra', 3,  60),
  (301, 'CS',   N'Compilers',      6,  40);
GO
CREATE VIEW Acad.CoreCatalog
WITH SCHEMABINDING
AS
SELECT CourseID, Dept, Title, Credits
FROM Acad.Course
WHERE Dept = 'CS' AND Credits <= 4
WITH CHECK OPTION;
GO
CREATE VIEW Acad.DeptStats
AS
SELECT Dept, COUNT(*) AS CourseCount
FROM Acad.Course
GROUP BY Dept;
GO
```

The following eight statements are then executed **in order, each in its own batch**, in a single session:

```sql
-- S1
INSERT INTO Acad.CoreCatalog (CourseID, Dept, Title, Credits)
VALUES (110, 'CS', N'Operating Systems', 4);

-- S2
INSERT INTO Acad.CoreCatalog (CourseID, Dept, Title, Credits)
VALUES (210, 'MATH', N'Calculus II', 4);

-- S3
UPDATE Acad.CoreCatalog SET Credits = 5 WHERE CourseID = 102;

-- S4
UPDATE Acad.CoreCatalog SET Title = N'Advanced Compilers' WHERE CourseID = 301;

-- S5
UPDATE Acad.CoreCatalog SET Credits = 2 WHERE CourseID = 101;

-- S6
ALTER TABLE Acad.Course ALTER COLUMN Title NVARCHAR(120) NOT NULL;

-- S7
CREATE VIEW Acad.RankedCatalog
AS
SELECT CourseID, Title, Credits
FROM Acad.Course
ORDER BY Credits DESC;

-- S8
DELETE FROM Acad.DeptStats WHERE Dept = 'MATH';
```

For each statement S1–S8, state whether it **succeeds or raises an error** (and, for successes, how many rows are affected). Then give the exact result of this final query:

```sql
SELECT CourseID, Dept, Title, Credits, Seats
FROM Acad.Course
ORDER BY CourseID;
```

## Correct Answer

Per-statement outcomes (all error numbers and messages are the engine's actual output):

| Stmt | Outcome | Detail |
|------|---------|--------|
| S1 | **Succeeds** | `(1 rows affected)` — row (110) inserted; `Seats` filled by its default (30) |
| S2 | **Fails** | `Msg 550` — `The attempted insert or update failed because the target view either specifies WITH CHECK OPTION or spans a view that specifies WITH CHECK OPTION and one or more rows resulting from the operation did not qualify under the CHECK OPTION constraint.` |
| S3 | **Fails** | `Msg 550` — same message: the updated row (`Credits = 5`) would leave the view |
| S4 | **Succeeds** | `(0 rows affected)` — course 301 is not visible through the view (6 credits), so nothing matches |
| S5 | **Succeeds** | `(1 rows affected)` — `Credits = 2` still satisfies the view's predicate |
| S6 | **Fails** | `Msg 5074` — `The object 'CoreCatalog' is dependent on column 'Title'.` followed by `Msg 4922` — `ALTER TABLE ALTER COLUMN Title failed because one or more objects access this column.` |
| S7 | **Fails** | `Msg 1033` — `The ORDER BY clause is invalid in views, inline functions, derived tables, subqueries, and common table expressions, unless TOP, OFFSET or FOR XML is also specified.` |
| S8 | **Fails** | `Msg 4403` — `Cannot update the view or function 'Acad.DeptStats' because it contains aggregates, or a DISTINCT or GROUP BY clause, or PIVOT or UNPIVOT operator.` |

Final contents of `Acad.Course`:

| CourseID | Dept | Title | Credits | Seats |
|----------|------|-------------------|---------|-------|
| 101 | CS | Programming I | 2 | 120 |
| 102 | CS | Databases | 4 | 80 |
| 110 | CS | Operating Systems | 4 | 30 |
| 205 | MATH | Linear Algebra | 3 | 60 |
| 301 | CS | Compilers | 6 | 40 |

Only two DML statements changed data: S1 added course 110, and S5 changed course 101 from 3 to 2 credits. Everything else either failed or matched zero rows.

## Explanation

This session exercises four distinct view rules. Verified against SQL Server 2025 (RTM); every message above is the engine's literal output.

### S1 — an INSERT through a view can succeed even though the view hides a column

`Acad.CoreCatalog` does not expose `Seats`, yet `Seats` is `NOT NULL`. The insert still succeeds because the column has a `DEFAULT` constraint, so the engine fills in 30. Had `Seats` been `NOT NULL` with no default, S1 would have failed with error 515 (cannot insert NULL). The inserted row (`'CS'`, 4 credits) satisfies the view's `WHERE` clause, so `WITH CHECK OPTION` is happy.

### S2 and S3 — WITH CHECK OPTION polices both INSERT and UPDATE

`WITH CHECK OPTION` forces every modification made **through the view** to produce rows that remain visible through the view:

- S2 inserts a `'MATH'` row. The row would be invisible through `CoreCatalog` (`Dept = 'CS'` fails), so the insert is rejected with error 550. Without `WITH CHECK OPTION`, this insert would have **succeeded** and silently landed in the base table, invisible to the view — that is the exact behavior the option exists to prevent.
- S3 updates a visible row (`CourseID` 102) so that `Credits = 5`, pushing it outside the `Credits <= 4` predicate. An update that makes a row "escape" the view is likewise rejected with error 550.

Note that `CHECK OPTION` applies only to modifications made through the view; a direct `UPDATE Acad.Course SET Credits = 5 WHERE CourseID = 102;` against the base table would succeed.

### S4 — rows the view cannot see cannot be modified through it

Course 301 exists in the base table but has 6 credits, so the view's `WHERE` filters it out. An `UPDATE` against the view can only target rows the view returns, so S4 matches nothing: it **succeeds** with `(0 rows affected)`. This is the subtle one — it is not an error, and the base row keeps its original title `Compilers`.

### S5 — a modification that keeps the row inside the view passes the check

`Credits = 2` still satisfies `Dept = 'CS' AND Credits <= 4`, so the row remains visible after the update and `WITH CHECK OPTION` allows it.

### S6 — SCHEMABINDING blocks ALTER TABLE on referenced columns

`CoreCatalog` was created `WITH SCHEMABINDING`, which binds the view to the schema of `Acad.Course`. Any `ALTER TABLE` that affects a column the view references fails — even a widening, apparently harmless change such as `NVARCHAR(60)` to `NVARCHAR(120)`. The engine raises errors 5074 and 4922. To make the change you must first `ALTER`/`DROP` the view to remove the binding. (SCHEMABINDING also prevents dropping the table outright.)

### S7 — ORDER BY is not allowed in a view without TOP/OFFSET/FOR XML

A bare `ORDER BY` in a view definition is a compile-time error (1033). The classic workaround `SELECT TOP (100) PERCENT ... ORDER BY ...` does parse, but it is a trap: per the Microsoft documentation for `CREATE VIEW`, the `ORDER BY` inside a view is used **only** to determine which rows `TOP`/`OFFSET` returns — it does **not** guarantee any ordering when the view is queried. Only an `ORDER BY` on the outer query guarantees result order.

### S8 — a view with GROUP BY / aggregates is never updatable

`Acad.DeptStats` contains `COUNT(*)` and `GROUP BY`, so no `INSERT`, `UPDATE`, or `DELETE` can be traced back unambiguously to base-table rows; the `DELETE` fails with error 4403. The updatability rules require that a modification through a view reference columns from **exactly one base table**, that the modified columns directly reference base columns (no aggregates, no computed expressions), and that they not be affected by `GROUP BY`, `HAVING`, or `DISTINCT`. (The only way to make such a view writable is an `INSTEAD OF` trigger.)

### Equivalent alternatives

- In S2/S3 the error is identical whether the violating column is `Dept` or `Credits`; any modification whose resulting row fails the view predicate raises the same error 550.
- S6 fails identically for `Dept`, `Credits`, or `CourseID`; altering `Seats` (not referenced by any schemabound view) **would succeed**.

## DP-800 Exam Rule to Remember

Modifications through a view follow a two-gate model:

```text
Gate 1 — Is the view updatable for this statement?
         one base table affected, modified columns map directly
         to base columns, no aggregate/GROUP BY/DISTINCT
         (else: error 4403 / 4405)

Gate 2 — WITH CHECK OPTION: does every resulting row
         still satisfy the view's WHERE clause?
         (else: error 550)
```

And the two side rules that pair with it:

- **A view only "sees" its own rows**: DML through the view silently affects 0 rows for base rows the predicate filters out — no error is raised.
- **SCHEMABINDING** freezes the referenced columns of the base table (`ALTER TABLE ... ALTER COLUMN` → errors 5074 + 4922), and a view's inner `ORDER BY` (requires `TOP`/`OFFSET`/`FOR XML` to even compile) never guarantees the order of results returned from the view.
