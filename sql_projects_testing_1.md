# SQL Server question — SQL Projects Testing 1

## Statement

A gig-economy platform pays contractors weekly. The payout rules live in an Azure SQL Database whose schema is maintained as an SDK-style SQL Database Project named `PayoutEngine`, stored in GitHub and deployed with GitHub Actions.

The project contains, among other objects:

```sql
CREATE TABLE Pay.Contractors
(
    ContractorId INT           NOT NULL PRIMARY KEY,
    Country      CHAR(2)       NOT NULL,
    TaxRate      DECIMAL(5,4)  NOT NULL
);
GO
CREATE TABLE Pay.Jobs
(
    JobId        INT           NOT NULL PRIMARY KEY,
    ContractorId INT           NOT NULL REFERENCES Pay.Contractors (ContractorId),
    CompletedOn  DATE          NOT NULL,
    GrossAmount  DECIMAL(10,2) NOT NULL
);
GO
CREATE PROCEDURE Pay.usp_ComputeWeeklyPayout
    @ContractorId INT,
    @WeekStart    DATE
AS
SELECT c.ContractorId,
       SUM(j.GrossAmount)                     AS Gross,
       SUM(j.GrossAmount) * (1 - c.TaxRate)   AS Net
FROM Pay.Contractors AS c
JOIN Pay.Jobs        AS j ON j.ContractorId = c.ContractorId
WHERE c.ContractorId = @ContractorId
  AND j.CompletedOn >= @WeekStart
  AND j.CompletedOn <  DATEADD(DAY, 7, @WeekStart)
GROUP BY c.ContractorId, c.TaxRate;
GO
```

Last month a change to `usp_ComputeWeeklyPayout` was merged and deployed to production; it silently applied the tax rate twice and 1,200 contractors were underpaid. The engineering lead now requires a **testing strategy** that satisfies all of the following:

1. Every pull request must run **unit tests of the T-SQL logic** (for example: "a contractor at 20% tax with two jobs of 100.00 in the week gets `Net = 160.00`"), and each test must be **isolated** — it must not depend on rows left by another test or on whatever data happens to exist in a shared database.
2. Every pull request must also run **integration tests** that deploy the real build artifact of the project and exercise the payout REST API end to end against it.
3. No test may ever run against, or read data from, the **production** database.
4. The tests must run inside the GitHub Actions workflow on hosted runners, and the test database must not outlive the workflow run.

Which approach should you implement?

### a.

Add a `tests` folder to the repository containing tSQLt test classes, for example:

```sql
EXEC tSQLt.NewTestClass 'PayoutTests';
GO
CREATE PROCEDURE PayoutTests.[test net payout applies tax once]
AS
BEGIN
    EXEC tSQLt.FakeTable 'Pay.Contractors';
    EXEC tSQLt.FakeTable 'Pay.Jobs';
    INSERT INTO Pay.Contractors (ContractorId, Country, TaxRate) VALUES (1, 'ES', 0.2000);
    INSERT INTO Pay.Jobs (JobId, ContractorId, CompletedOn, GrossAmount)
    VALUES (10, 1, '2026-03-02', 100.00), (11, 1, '2026-03-05', 100.00);

    CREATE TABLE #Expected (ContractorId INT, Gross DECIMAL(10,2), Net DECIMAL(10,2));
    INSERT INTO #Expected VALUES (1, 200.00, 160.00);
    CREATE TABLE #Actual (ContractorId INT, Gross DECIMAL(10,2), Net DECIMAL(10,2));
    INSERT INTO #Actual EXEC Pay.usp_ComputeWeeklyPayout @ContractorId = 1, @WeekStart = '2026-03-02';

    EXEC tSQLt.AssertEqualsTable '#Expected', '#Actual';
END;
GO
```

In the workflow, start a throw-away SQL Server as a **service container**, build the project, publish the `.dacpac` to it, install tSQLt and the test classes, run `EXEC tSQLt.RunAll`, then run the API integration tests against the same container:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      sql:
        image: mcr.microsoft.com/mssql/server:2025-latest
        env:
          ACCEPT_EULA: "Y"
          MSSQL_SA_PASSWORD: ${{ secrets.CI_SA_PASSWORD }}
        ports: ["1433:1433"]
    steps:
      - uses: actions/checkout@v4
      - run: dotnet build PayoutEngine.sqlproj -c Release
      - run: dotnet tool install -g microsoft.sqlpackage
      - run: >
          sqlpackage /Action:Publish /SourceFile:bin/Release/PayoutEngine.dacpac
          /TargetConnectionString:"Server=localhost;Database=PayoutEngine;User ID=sa;Password=${{ secrets.CI_SA_PASSWORD }};TrustServerCertificate=True"
      - run: sqlcmd -S localhost -U sa -P "${{ secrets.CI_SA_PASSWORD }}" -d PayoutEngine -C -i tests/tSQLt.class.sql -i tests/PayoutTests.sql
      - run: sqlcmd -S localhost -U sa -P "${{ secrets.CI_SA_PASSWORD }}" -d PayoutEngine -C -Q "EXEC tSQLt.RunAll" -b
      - run: dotnet test tests/PayoutApi.IntegrationTests
```

### b.

Add the same tSQLt test classes to the `PayoutEngine` project so they are deployed everywhere, and have the workflow run `EXEC tSQLt.RunAll` against the **production** database right after each deployment. Because tSQLt wraps every test in a transaction that is rolled back, the tests leave no trace in production, and running against real data gives the most realistic coverage.

### c.

Write the unit tests as xUnit tests in C# that open a connection to a long-lived shared `PayoutEngine_QA` Azure SQL Database, insert the contractors and jobs they need with the IDs `1`, `10` and `11`, call `Pay.usp_ComputeWeeklyPayout`, and assert on the result. Run the same test project as the integration suite by pointing the API at `PayoutEngine_QA`.

### d.

Rely on the build pipeline as the test suite: `dotnet build` validates every object reference and the T-SQL syntax, and add a `sqlpackage /Action:DeployReport` step against a staging database so reviewers can see the changes before merge. Because the payout bug was a logic error inside a procedure, add a required pull-request reviewer from the finance team instead of automated tests.

## Correct Answer

**a**

## Explanation

The question separates two layers of testing that the DP-800 outline names explicitly — **unit tests** (T-SQL logic, isolated, fast) and **integration tests** (the real deployable artifact plus the application) — and then asks where each belongs in a GitHub Actions workflow. Only option a puts each layer in the right place and keeps production out of the loop.

### Why option a is correct

- **Unit tests of T-SQL logic, isolated.** tSQLt is the de-facto unit-testing framework for SQL Server T-SQL. `tSQLt.FakeTable` replaces `Pay.Contractors` and `Pay.Jobs` with empty, constraint-free copies for the duration of the test, so the test controls exactly which rows exist; `tSQLt.AssertEqualsTable` compares the expected and actual result sets; and tSQLt runs **every test inside a transaction that it rolls back**, so tests cannot leak rows into each other (requirement 1). Test procedures live in a test class created with `tSQLt.NewTestClass` and are discovered by the `test` prefix of their name; `tSQLt.RunAll` executes them all and fails the `sqlcmd -b` step (non-zero exit) when any assertion fails, which fails the pull-request check.
- **Integration tests against the real artifact.** The workflow builds the `.dacpac` with `dotnet build`, publishes it with `sqlpackage /Action:Publish` — the same artifact and the same deployment mechanism used for real environments — and then runs the API test project against that database (requirement 2).
- **Never production, never long-lived.** The database is a GitHub Actions **service container** started from the `mcr.microsoft.com/mssql/server` image; it is created for the job and destroyed when the job ends, so nothing outlives the run and nothing touches production (requirements 3 and 4). The only secret involved is a throw-away `sa` password for the container, stored as a GitHub secret rather than in the YAML.
- **Test data seeding** is explicit and local to each test (the `INSERT`s after `FakeTable`) for unit tests, while integration tests seed through the API or a post-deployment script — in both cases deterministic and reproducible.

### Why option b is wrong

Option b is the subtle distractor, because the claim about rollback is true — tSQLt does roll each test back — yet the approach violates requirement 3 outright and is unsafe in practice:

- tSQLt requires **CLR enabled** on the instance and either `TRUSTWORTHY ON` on the database or a signed assembly, and it installs its own schema and hundreds of objects. Putting that into a production database widens the attack surface and, in Azure SQL Database, CLR is not even available.
- `tSQLt.FakeTable` **renames the real table** and creates a fake in its place for the duration of the test. Even inside a transaction, that takes schema-modification locks on `Pay.Contractors` and `Pay.Jobs`, blocking the live payout workload while tests run.
- Tests that read "real data" are not deterministic: production rows change daily, so the assertion `Net = 160.00` only holds until someone adds a job. Unit tests must control their inputs, which is exactly what `FakeTable` plus explicit inserts do — and that works just as well in a container.

Testing against production is wrong regardless of how carefully the tests clean up; the requirement exists to make the failure mode impossible, not unlikely.

### Why option c is wrong

A single long-lived shared `PayoutEngine_QA` database breaks isolation (requirement 1): two pull requests running at the same time both insert `ContractorId = 1` and `JobId = 10`, so one of them hits a primary-key violation or reads the other's rows; a test that forgets to delete its rows poisons every later run; and schema drift accumulates in the QA database because it is never rebuilt from the project. It also blurs the two layers — the "unit" tests exercise a network round trip and whatever data is present rather than the procedure's logic in isolation — and requirement 4 (the test database must not outlive the run) is violated by design.

### Why option d is wrong

`dotnet build` validates that every object reference resolves and that the syntax is valid for the target platform; `/Action:DeployReport` lists the schema operations a publish would perform. Neither executes `usp_ComputeWeeklyPayout`, so neither can detect that `TaxRate` is applied twice — the build of last month's faulty procedure was green. A human reviewer is a control, not a test: it does not run on every pull request deterministically and cannot be re-run when a regression is suspected. Static validation is necessary but is not a testing strategy.

Conceptual question (Azure / tooling); not executed against an engine.

## DP-800 Exam Rule to Remember

```text
Unit test (T-SQL logic)      -> tSQLt: NewTestClass, FakeTable, AssertEquals/AssertEqualsTable,
                                ExpectException, SpyProcedure; each test rolled back; RunAll in CI.
Integration test (artifact)  -> build dacpac -> sqlpackage publish to a throw-away container/LocalDB
                                -> run the app's tests against it.
Where they run                -> a GitHub Actions job with a SQL Server *service container*
                                (mcr.microsoft.com/mssql/server), destroyed when the job ends.
Never                         -> production as a test target, or a shared long-lived "QA" DB
                                whose data drifts between runs.
```

`dotnet build` and `DeployReport` catch broken references and risky schema operations; only executing tests catches wrong results. Seed test data explicitly per test (fake tables plus inserts) so every run is deterministic.
