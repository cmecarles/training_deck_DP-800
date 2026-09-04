# SQL Server question — SSDT Unit Tests 1

## Statement

RidgeWater Utilities bills customers from meter readings stored in an Azure SQL Database. The schema is maintained as a **SQL Server Database Project** named `RidgeMeters` in Visual Studio (SQL Server Data Tools), built and deployed from Azure Pipelines. The project contains, among other objects:

```sql
CREATE SCHEMA Ops;
GO
CREATE TABLE Ops.Meter
(
    MeterId     INT           NOT NULL PRIMARY KEY,
    Serial      CHAR(10)      NOT NULL UNIQUE,
    InstalledOn DATE          NOT NULL
);
GO
CREATE TABLE Ops.Reading
(
    ReadingId  INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    MeterId    INT           NOT NULL REFERENCES Ops.Meter (MeterId),
    ReadAt     DATETIME2(0)  NOT NULL,
    CubicM     DECIMAL(10,1) NOT NULL,
    CONSTRAINT UQ_Reading UNIQUE (MeterId, ReadAt)
);
GO
CREATE PROCEDURE Ops.usp_RecordReading
    @MeterId INT,
    @ReadAt  DATETIME2(0),
    @CubicM  DECIMAL(10,1)
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @Last DECIMAL(10,1) =
        (SELECT TOP (1) CubicM FROM Ops.Reading
         WHERE MeterId = @MeterId ORDER BY ReadAt DESC);
    IF @Last IS NOT NULL AND @CubicM < @Last
        THROW 50010, N'Reading is lower than the previous reading for this meter.', 1;

    INSERT INTO Ops.Reading (MeterId, ReadAt, CubicM)
    VALUES (@MeterId, @ReadAt, @CubicM);

    SELECT ReadingId, MeterId, ReadAt, CubicM
    FROM Ops.Reading
    WHERE ReadingId = SCOPE_IDENTITY();
END;
GO
CREATE USER meter_app FOR LOGIN meter_app;
GRANT EXECUTE ON Ops.usp_RecordReading TO meter_app;
GO
```

The billing application connects as `meter_app`, which has **only** `EXECUTE` on the procedure. A release last quarter removed the `IF @Last ... THROW` guard by mistake and 3,000 negative consumptions were billed. The team now requires an automated **SQL Server unit test** suite with these rules:

1. **Test 1 (positive).** With meter 7 seeded with a reading of `1200.0`, calling `Ops.usp_RecordReading 7, '2026-09-01', 1250.0` must return **exactly one row** whose `CubicM` is **1250.0**.
2. **Test 2 (negative).** Calling the procedure with `1100.0` for the same meter must be rejected with **error 50010, severity 16, state 1**. The test must pass **only** when that specific error is raised, and must fail if no error or a *different* error occurs.
3. The test scripts that call the procedure must run **as `meter_app`** (so that missing grants are caught), while the seeding and cleanup statements run under a **separate, privileged** account — the `meter_app` account must not be granted `INSERT`/`DELETE` on the tables.
4. Every test run must first **deploy the current build of the project** to the test database, so the tests always exercise the schema in the pull request.
5. The tests must run in **Azure Pipelines** on a hosted Windows agent and publish results, using the Visual Studio **SQL Server unit test** framework (`Microsoft.Data.Tools.Schema.Sql.UnitTesting`) — no third-party framework and **no CLR** on the test server.

Which implementation meets every rule?

### a.

In **SQL Server Object Explorer**, right-click `Ops.usp_RecordReading` > **Create Unit Tests**, creating a new C# test project `RidgeMeters.Tests` (the generated class derives from `SqlDatabaseTestClass`). In the **SQL Server Test Configuration** dialog set the **execution context** to a connection that logs in as `meter_app`, the **privileged context** to `ridge_deployer` (a member of `db_owner`), tick **Automatically deploy the database project before unit tests are run**, and select `RidgeMeters.sqlproj` / configuration `Release`. Then, in the SQL Server Unit Test Designer:

- `usp_RecordReading_Increasing` — **Pre-Test** script (idempotent seed of meter 7 with the `1200.0` reading and deletion of any later readings); **Test** script `EXEC Ops.usp_RecordReading @MeterId = 7, @ReadAt = '2026-09-01T00:00:00', @CubicM = 1250.0;`; delete the default **Inconclusive** condition; add **Row Count** = `1` and **Scalar Value** (ResultSet 1, Row 1, Column 4, Expected value `1250.0`); **Post-Test** script that deletes the row the test inserted.
- `usp_RecordReading_LowerFails` — same Pre-Test; **Test** script `EXEC Ops.usp_RecordReading @MeterId = 7, @ReadAt = '2026-09-02T00:00:00', @CubicM = 1100.0;`; delete the Inconclusive condition; and in the `.cs` file mark the method

  ```csharp
  [TestMethod(), ExpectedSqlException(MessageNumber = 50010, Severity = 16, State = 1)]
  public void usp_RecordReading_LowerFails()
  ```

The pipeline builds the solution on `windows-latest` and runs the tests with the **Visual Studio Test** task, substituting the two connection strings into `app.config` from pipeline secrets:

```yaml
- task: VSBuild@1
  inputs: { solution: 'RidgeMeters.sln', configuration: 'Release' }
- task: VSTest@3
  inputs:
    testSelector: testAssemblies
    testAssemblyVer2: '**\RidgeMeters.Tests.dll'
    searchFolder: '$(System.DefaultWorkingDirectory)'
```

### b.

Same test project, configuration dialog and pipeline as option a, with two simplifications: put the seeding statements at the top of each **Test** script "so the whole test is one script", and mark the negative test with MSTest's generic attribute `[ExpectedException(typeof(SqlException))]` instead of `ExpectedSqlException`, so the test passes whenever the procedure throws.

### c.

Install **tSQLt** in the test database (`EXEC sp_configure 'clr enabled', 1; ALTER DATABASE RidgeMeters SET TRUSTWORTHY ON;`), write `EXEC tSQLt.NewTestClass 'ReadingTests';` and a test procedure that calls `EXEC tSQLt.ExpectException @ExpectedErrorNumber = 50010;` before executing `Ops.usp_RecordReading`, then have the pipeline run `sqlcmd -Q "EXEC tSQLt.RunAll" -b` after publishing the `.dacpac`.

### d.

Same test project as option a, but write the negative **Test** script as

```sql
EXEC Ops.usp_RecordReading @MeterId = 7, @ReadAt = '2026-09-02T00:00:00', @CubicM = 1100.0;
SELECT @@ERROR AS Err;
```

with a **Scalar Value** condition whose Expected value is `50010`; point both the execution context and the privileged context at `ridge_deployer` "to avoid permission noise"; untick **Automatically deploy the database project** and run the tests against the shared `RidgeMeters_Dev` database that already exists.

## Correct Answer

**a**

## Explanation

SQL Server unit tests in SSDT are MSTest classes generated by Visual Studio; each test is a **pre-test / test / post-test** triple of T-SQL scripts plus **test conditions** evaluated on the result of the *test* script, and each script runs under one of two connections declared in the test project's `app.config`. Option a is the only one that uses those pieces the way the documentation defines them.

### Why option a is correct

- **Two connection contexts (rule 3).** The SQL Server Test Configuration dialog stores an **Execution Context** — "a user account for running the test script. This connection string should have the same credentials that you expect your users to have" — and a **Privileged Context** — "an account on the same database that has higher permissions for running the pre-test action, post-test action, TestInitialize, and TestCleanup scripts... This connection string is also used to deploy database changes". The two only differ when SQL authentication is used, which is why `meter_app` is a SQL login. The seed `INSERT` lives in the Pre-Test script and therefore runs as `ridge_deployer`; the `EXEC` lives in the Test script and runs as `meter_app`. If a future change forgot the `GRANT EXECUTE`, Test 1 would fail with error 229 — exactly what rule 3 wants caught.
- **Test conditions (rule 1).** A new test carries the **Inconclusive** condition, which "always produces a test with a result of Inconclusive... Delete this test condition from your test after you add other test conditions." **Row Count** "fails if the result set doesn't contain the expected number of rows"; **Scalar Value** "fails if a particular value in the result set doesn't equal the specified value" and is addressed by ResultSet / Row / Column — column 4 of the row the procedure returns is `CubicM`. On SQL Server 2025 the call returns one row: `ReadingId 2, MeterId 7, ReadAt 2026-09-01 00:00:00, CubicM 1250.0`. A wrong value produces the documented failure text pattern `ScalarValueCondition Condition (...) Failed: ResultSet 1 Row 1 Column 4: values do not match, actual '...' expected '1250.0'.`
- **Negative test (rule 2).** The framework's attribute is `[ExpectedSqlException(MessageNumber = nnnnn, Severity = x, MatchFirstError = false, State = y)]`; "any unspecified parameters are ignored" and the values are the ones "you pass ... to the THROW statement in your database code". With `MessageNumber = 50010, Severity = 16, State = 1` the test passes only for this error. Without the attribute the run reports `Test method ... threw exception: System.Data.SqlClient.SqlException: Reading is lower than the previous reading for this meter.` The procedure's actual output on SQL Server 2025 is:

  ```text
  Msg 50010, Level 16, State 1, Procedure Ops.usp_RecordReading, Line 12
  Reading is lower than the previous reading for this meter.
  ```

- **Deploy before test (rule 4).** "Automatically deploy the database project before unit tests are run" builds and publishes the selected `.sqlproj` at the start of the run, "under the privileged context connection string", so the tests always see the pull request's schema.
- **CI (rule 5).** The SSDT test assembly is an MSTest assembly, so the **Visual Studio Test** task (`VSTest@3`; `VSTest@2` is the older version being deprecated) runs it with `vstest.console.exe` and publishes the results. The task's `vstest` demand is satisfied because Visual Studio is installed on the hosted Windows agent.

### Why option b is wrong

This is the subtle distractor because it is *almost* option a. Moving the seed into the Test script changes which connection executes it: the Test script runs under the **execution context**, `meter_app`, which has no `INSERT` permission. On SQL Server 2025 the seed then fails with the engine's literal message:

```text
Msg 229, Level 14, State 5
The INSERT permission was denied on the object 'Meter', database 'RidgeMeters', schema 'Ops'.
```

That alone breaks Test 1. Worse, the generic MSTest `[ExpectedException(typeof(SqlException))]` treats **any** `SqlException` as success — error 229 from the seed, 2627 from a duplicate `(MeterId, ReadAt)`, or a typo in the script — so Test 2 turns green without ever proving that error 50010 is raised, violating rule 2. `ExpectedSqlException` exists precisely to match number, severity and state.

### Why option c is wrong

tSQLt is a capable T-SQL unit-testing framework, and `tSQLt.ExpectException @ExpectedErrorNumber = 50010` is the right call *in that framework* — but rule 5 mandates the Visual Studio SQL Server unit test framework and forbids CLR: tSQLt is a CLR assembly that needs `clr enabled` and either `TRUSTWORTHY ON` or a signed assembly on the test server. It is also not a Visual Studio test project, so `VSTest` has nothing to discover and no test results are published; and it has no notion of the execution/privileged split of rule 3 (everything runs as the `sqlcmd` login). Wrong framework for the stated constraints.

### Why option d is wrong

Three independent failures:

1. `THROW` **terminates the batch**: when the procedure throws 50010 the `SELECT @@ERROR` on the next line never executes (verified on SQL Server 2025: the batch prints the Msg 50010 text and returns no result set). The client receives a `SqlException`, the test method throws, and the Scalar Value condition is never evaluated — Test 2 fails every time the procedure behaves correctly.
2. Pointing both contexts at `ridge_deployer` runs the `EXEC` as `db_owner`, so a missing `GRANT EXECUTE` is never detected (rule 3).
3. Unticking automatic deployment and reusing `RidgeMeters_Dev` tests whatever schema happens to be there, not the pull request's build (rule 4).

Conceptual question (Visual Studio tooling / Azure DevOps); not executed against an engine — except the T-SQL of `Ops.usp_RecordReading` and the permission check, whose messages quoted above are the literal output of SQL Server 2025 (RTM 17.0.1000.7).

## DP-800 Exam Rule to Remember

```text
SSDT SQL Server unit test = Visual Studio test project, class : SqlDatabaseTestClass
  Scripts per test : Pre-Test  -> privileged context   (seed, arrange)
                     Test      -> EXECUTION context    (the only required script; conditions apply here)
                     Post-Test -> privileged context   (cleanup / extra validation)
                     TestInitialize / TestCleanup      -> common scripts, privileged, once per test class
  app.config       : ExecutionContext + PrivilegedContext connection strings (differ only with SQL auth)
  Conditions       : Inconclusive (default, delete it) | Row Count | Scalar Value (ResultSet/Row/Column)
                     | Empty ResultSet | Not Empty ResultSet | Expected Schema | Data Checksum
                     | Execution Time (test script only, default 30 s)
  Negative test    : [ExpectedSqlException(MessageNumber=, Severity=, State=, MatchFirstError=)]
                     - unspecified parts are ignored; generic [ExpectedException] matches ANY SqlException
  Deploy first     : "Automatically deploy the database project before unit tests are run" (privileged ctx)
  CI               : build with VSBuild -> VSTest@3 / vstest.console.exe on a Windows agent with VS
```

Read the option for *which connection runs which script*: anything that seeds data inside the Test script runs as the low-privilege user, and any "expected exception" that does not name the error number proves nothing. `THROW` aborts the batch, so `SELECT @@ERROR` after a failed `EXEC` is dead code.
