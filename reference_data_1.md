# SQL Server question — Reference Data 1

## Statement

A consumer-lending startup keeps the schema of its SQL Server database `LoanBook` in an SDK-style SQL Database Project (`LoanBook.sqlproj`) and deploys it with `sqlpackage /Action:Publish` to development, test and production. The project defines a lookup table and a table that references it:

```sql
-- Ref/Tables/LoanStatus.sql
CREATE TABLE Ref.LoanStatus
(
    StatusId   INT          IDENTITY(1,1) NOT NULL CONSTRAINT PK_LoanStatus PRIMARY KEY,
    Code       VARCHAR(12)  NOT NULL CONSTRAINT UQ_LoanStatus_Code UNIQUE,
    StatusName NVARCHAR(40) NOT NULL,
    IsTerminal BIT          NOT NULL
);
GO
-- Ref/Tables/Loans.sql
CREATE TABLE Ref.Loans
(
    LoanId   INT NOT NULL PRIMARY KEY,
    StatusId INT NOT NULL CONSTRAINT FK_Loans_Status FOREIGN KEY REFERENCES Ref.LoanStatus (StatusId)
);
GO
```

The application code refers to statuses by their numeric `StatusId` (for example `const int Approved = 2;`), so the **same `StatusId` values must exist in every environment**. Production currently contains three rows — `(1, 'APPLIED', N'Applied', 0)`, `(2, 'APPROVED', N'Approved (old)', 0)` and `(5, 'LEGACY', N'Legacy status', 1)` — the second of which was renamed by hand by an operator, and the third of which was retired and is referenced by no loan.

The next release must ship this **reference data as part of the project**, so that every deployment leaves the table in exactly this state:

| StatusId | Code | StatusName | IsTerminal |
|---|---|---|---|
| 1 | APPLIED | Applied | 0 |
| 2 | APPROVED | Approved | 0 |
| 3 | DISBURSED | Disbursed | 0 |
| 4 | CLOSED | Closed | 1 |

Requirements:

1. The first deployment to an empty database and the hundredth deployment to production must both succeed and both produce the table above (rename row 2, add rows 3 and 4, remove row 5).
2. The `StatusId` values must be the ones listed, never whatever `IDENTITY` would generate next.
3. The data must be versioned in git next to the schema and reviewed through the normal pull-request process.

Which implementation meets all three requirements?

### a.

Append the seed rows to the table's own definition file, `Ref/Tables/LoanStatus.sql`, after the `CREATE TABLE` statement:

```sql
SET IDENTITY_INSERT Ref.LoanStatus ON;
INSERT INTO Ref.LoanStatus (StatusId, Code, StatusName, IsTerminal) VALUES
 (1, 'APPLIED', N'Applied', 0), (2, 'APPROVED', N'Approved', 0),
 (3, 'DISBURSED', N'Disbursed', 0), (4, 'CLOSED', N'Closed', 1);
SET IDENTITY_INSERT Ref.LoanStatus OFF;
```

### b.

Add a post-deployment script (`<PostDeploy Include="Scripts\PostDeploy.sql" />`) containing the same `SET IDENTITY_INSERT ... ON` / `INSERT ... VALUES` / `SET IDENTITY_INSERT ... OFF` block as option a.

### c.

Add a pre-deployment script (`<PreDeploy Include="Scripts\PreDeploy.sql" />`) that empties and refills the table before the schema is compared, so the result is always exactly the listed rows:

```sql
TRUNCATE TABLE Ref.LoanStatus;
SET IDENTITY_INSERT Ref.LoanStatus ON;
INSERT INTO Ref.LoanStatus (StatusId, Code, StatusName, IsTerminal) VALUES
 (1, 'APPLIED', N'Applied', 0), (2, 'APPROVED', N'Approved', 0),
 (3, 'DISBURSED', N'Disbursed', 0), (4, 'CLOSED', N'Closed', 1);
SET IDENTITY_INSERT Ref.LoanStatus OFF;
```

### d.

Add a post-deployment script (`<PostDeploy Include="Scripts\PostDeploy.sql" />`) that includes one file per lookup table with `:r`, each file containing an idempotent `MERGE`:

```sql
-- Scripts\PostDeploy.sql
:r .\ReferenceData\LoanStatus.sql
```

```sql
-- Scripts\ReferenceData\LoanStatus.sql   (Build action: None / <Build Remove=...>)
SET IDENTITY_INSERT Ref.LoanStatus ON;
MERGE Ref.LoanStatus AS tgt
USING (VALUES
        (1, 'APPLIED',   N'Applied',   0),
        (2, 'APPROVED',  N'Approved',  0),
        (3, 'DISBURSED', N'Disbursed', 0),
        (4, 'CLOSED',    N'Closed',    1)
      ) AS src (StatusId, Code, StatusName, IsTerminal)
   ON tgt.StatusId = src.StatusId
WHEN MATCHED AND (tgt.Code <> src.Code OR tgt.StatusName <> src.StatusName OR tgt.IsTerminal <> src.IsTerminal)
    THEN UPDATE SET Code = src.Code, StatusName = src.StatusName, IsTerminal = src.IsTerminal
WHEN NOT MATCHED BY TARGET
    THEN INSERT (StatusId, Code, StatusName, IsTerminal) VALUES (src.StatusId, src.Code, src.StatusName, src.IsTerminal)
WHEN NOT MATCHED BY SOURCE
    THEN DELETE;
SET IDENTITY_INSERT Ref.LoanStatus OFF;
```

## Correct Answer

**d**

## Explanation

A `.dacpac` describes **schema**, not data. The build compiles every file with build action `Build` into a database model; anything that is not a declarative object definition does not belong there. Data that must exist in every environment — lookup/reference/static data — is delivered by the project's **post-deployment script**, which SqlPackage runs after the schema deployment plan completes, **every time** the project is deployed. That last word is why the script must be **idempotent**.

### Why option d is correct

- **Right vehicle.** The project may have exactly one `PreDeploy` and one `PostDeploy` item; the script is stored inside the `.dacpac` but is not validated against the model. Splitting per-table files and pulling them in with the SQLCMD `:r` include keeps each lookup table reviewable in its own file, while `Microsoft.Build.Sql` concatenates the included files into the single post-deployment script at build time. The included files must be excluded from the model build (`<Build Remove="Scripts\ReferenceData\LoanStatus.sql" />`, optionally re-added as `<None Include=...>` so they stay visible).
- **Idempotent by construction.** The `MERGE` compares the versioned rows with the table on the fixed key `StatusId`: renames become `UPDATE`s, new codes become `INSERT`s, retired codes become `DELETE`s, and rows that already match are left alone. Executed against the production state described above and then executed a second time:

  ```text
  --- run 1 ---            --- run 2 ---
  merge_action ins del     (no rows)
  UPDATE       2   2       rows affected by run 2: 0
  INSERT       3   NULL
  INSERT       4   NULL
  DELETE       NULL 5
  ```

  and the table afterwards is exactly the four required rows. Running it 100 times yields the same state — requirement 1.
- **Fixed identity values.** `SET IDENTITY_INSERT Ref.LoanStatus ON` is required to write explicit `StatusId` values into an `IDENTITY` column — with it off, both a plain `INSERT` and the `MERGE`'s insert branch fail with `Msg 544: Cannot insert explicit value for identity column in table 'LoanStatus' when IDENTITY_INSERT is set to OFF.` (verified for both). Only one table per session can have `IDENTITY_INSERT` on, so each per-table file turns it on and off around its own `MERGE` — requirement 2.
- The files live in the repository next to the schema and go through pull requests — requirement 3.

One caveat worth knowing: `WHEN NOT MATCHED BY SOURCE THEN DELETE` removes row 5 here because no loan references it; if a referenced row were removed from the source list, the `DELETE` would fail with `Msg 547: The DELETE statement conflicted with the REFERENCE constraint "FK_Loans_Status"` — which is the correct outcome (the deployment stops rather than orphaning data), and is why retiring a status is a two-step change: migrate the loans, then remove the row.

### Why option a is wrong

Files with build action `Build` are compiled into the schema model, and the model has no notion of rows. `SET IDENTITY_INSERT` and `INSERT` are not declarative object definitions, so the build fails with the DacFx error `SQL70001: This statement is not recognized in this context.` The project never produces a `.dacpac`, so nothing is deployed. Data does not go in object files.

### Why option b is wrong

A post-deployment script is the right place, but a plain `INSERT` is not idempotent. It works on the first deployment to an empty database and then breaks every later deployment, because the rows already exist. Verified by re-running the insert of row 3 on a table that already has it:

```text
Msg 2627, Level 14, State 1
Violation of PRIMARY KEY constraint 'PK_LoanStatus'. Cannot insert duplicate key in object 'Ref.LoanStatus'.
The duplicate key value is (3).
```

Requirement 1 ("the hundredth deployment") fails. It would also never rename row 2 or delete row 5, because an `INSERT` cannot express "make the table look like this".

### Why option c is wrong

This is the subtle distractor: "wipe and reload" *sounds* idempotent, and a pre-deployment script *is* a legitimate project feature. Three things break:

- **Ordering.** The pre-deployment script runs **before** the schema deployment (the deployment plan is computed first, then the pre-deployment script executes, then the plan). On the first deployment to an empty database `Ref.LoanStatus` does not exist yet, so the script fails immediately — half of requirement 1.
- **Foreign keys.** In production, `Ref.Loans` references the table, so `TRUNCATE` is rejected: `Msg 4712: Cannot truncate table 'Ref.LoanStatus' because it is being referenced by a FOREIGN KEY constraint.` A `DELETE` instead of `TRUNCATE` fails on the referenced rows with `Msg 547` (verified), so the script can never run on a database that actually has loans.
- **Semantics.** Even where it could run, deleting and re-inserting every lookup row on every deployment is a data-loss risk under load, resets the identity seed (`TRUNCATE`) and rewrites rows that never changed. `MERGE` touches only the differences.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

## DP-800 Exam Rule to Remember

```text
dacpac = schema.  Reference/static data = post-deployment script, run on EVERY deploy -> must be idempotent.

.sqlproj:   <PostDeploy Include="Scripts\PostDeploy.sql" />   (one PreDeploy + one PostDeploy per project)
            PostDeploy.sql:  :r .\ReferenceData\Table1.sql   (SQLCMD include; included files: Build Remove / None)
Pattern:    SET IDENTITY_INSERT t ON;  MERGE t USING (VALUES ...) ON key
              WHEN MATCHED AND differs THEN UPDATE
              WHEN NOT MATCHED BY TARGET THEN INSERT
              WHEN NOT MATCHED BY SOURCE THEN DELETE;   SET IDENTITY_INSERT t OFF;
Anti-patterns: INSERT in a Build file (SQL70001) | plain INSERT in PostDeploy (Msg 2627 on 2nd deploy)
               TRUNCATE/DELETE-and-reload in PreDeploy (runs before schema exists; Msg 4712 / 547 with FKs)
Pre-deploy = data prep that must happen BEFORE the plan runs (e.g. save data of a column about to be dropped).
```
