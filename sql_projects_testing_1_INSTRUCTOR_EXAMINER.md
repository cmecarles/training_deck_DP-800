# Instructor-Examiner guide — SQL Projects Testing 1

Companion to [sql_projects_testing_1.md](sql_projects_testing_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a, b, c and d. Read the schema, the procedure, the four requirements and all four options before taking an answer. Option a contains a tSQLt test and a workflow YAML that are longer than fifteen lines; summarise them faithfully as written in section 2 and say "I can read any line on request". This is a conceptual tooling question; nothing was executed against an engine.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Manage database solutions with CI/CD (20–25%).
- Skill: Build and manage SQL Database Projects.
- Task bullet: Implement unit tests and integration tests for SQL projects; manage test data.
- What is tested: where unit tests of T-SQL logic and integration tests of the built artifact belong in a GitHub Actions workflow, how tSQLt isolates a test, and why production or a shared long-lived QA database is never the test target.

## 2. Scenario to read aloud

**Piece 1, the story.** "A gig-economy platform pays contractors weekly. The payout rules live in an Azure SQL Database whose schema is maintained as an SDK-style SQL Database Project named PayoutEngine, stored in GitHub and deployed with GitHub Actions. Last month a change to the payout procedure was merged and deployed to production. It silently applied the tax rate twice, and one thousand two hundred contractors were underpaid. The engineering lead now requires a testing strategy."

**Piece 2, the tables.** "Two tables in the schema Pay. Contractors has ContractorId, an integer primary key; Country, char two; and TaxRate, decimal five comma four. Jobs has JobId, an integer primary key; ContractorId, an integer with a foreign key to Contractors; CompletedOn, a date; and GrossAmount, decimal ten comma two. All columns are not null."

**Piece 3, the procedure.** "The procedure Pay dot usp underscore ComputeWeeklyPayout takes two parameters, at ContractorId, an integer, and at WeekStart, a date. It joins Contractors to Jobs on ContractorId, filters to the given contractor and to jobs completed on or after WeekStart and before WeekStart plus seven days, and groups by ContractorId and TaxRate. It returns ContractorId, Gross, which is the SUM of GrossAmount, and Net, which is the SUM of GrossAmount times open paren one minus TaxRate close paren."

**Piece 4, the four requirements.** "Requirement one: every pull request must run unit tests of the T-SQL logic. For example: a contractor at twenty percent tax with two jobs of one hundred in the week gets Net equals one hundred sixty. Each test must be isolated. It must not depend on rows left by another test or on whatever data happens to exist in a shared database. Requirement two: every pull request must also run integration tests that deploy the real build artifact of the project and exercise the payout REST API end to end against it. Requirement three: no test may ever run against, or read data from, the production database. Requirement four: the tests must run inside the GitHub Actions workflow on hosted runners, and the test database must not outlive the workflow run."

**Piece 5, option a, the test.** "Option a. Add a tests folder to the repository containing tSQLt test classes. The example runs tSQLt dot NewTestClass with the name PayoutTests, then creates a procedure PayoutTests dot, in square brackets, test net payout applies tax once. Inside: it calls tSQLt dot FakeTable on Pay dot Contractors and on Pay dot Jobs. It inserts contractor one, country ES, tax rate zero point two. It inserts two jobs, ten and eleven, for contractor one, completed on the second and fifth of March 2026, each with gross one hundred. It creates a temp table Expected with one row: contractor one, gross two hundred, net one hundred sixty. It creates a temp table Actual and fills it with INSERT EXEC of the procedure for contractor one and week start the second of March 2026. Then it calls tSQLt dot AssertEqualsTable comparing Expected and Actual. I can read any line on request."

**Piece 6, option a, the workflow.** "Still option a. The workflow has one job named test, on ubuntu-latest, with a services block defining a service container named sql from the image mcr dot microsoft dot com slash mssql slash server, tag 2025 dash latest, with environment variables ACCEPT underscore EULA Y and MSSQL underscore SA underscore PASSWORD from the secret CI underscore SA underscore PASSWORD, and port fourteen thirty three mapped. The steps: checkout; dotnet build PayoutEngine dot sqlproj in Release; dotnet tool install microsoft dot sqlpackage; sqlpackage Action Publish with SourceFile bin slash Release slash PayoutEngine dot dacpac and a TargetConnectionString to localhost, database PayoutEngine, user sa, the secret password, TrustServerCertificate True; sqlcmd against localhost, database PayoutEngine, running two input files, tests slash tSQLt dot class dot sql and tests slash PayoutTests dot sql; sqlcmd with the query EXEC tSQLt dot RunAll and the dash b flag; and finally dotnet test on tests slash PayoutApi dot IntegrationTests. I can read any line on request."

**Piece 7, option b.** "Option b. Add the same tSQLt test classes to the PayoutEngine project so they are deployed everywhere, and have the workflow run EXEC tSQLt dot RunAll against the production database right after each deployment. The argument: because tSQLt wraps every test in a transaction that is rolled back, the tests leave no trace in production, and running against real data gives the most realistic coverage."

**Piece 8, option c.** "Option c. Write the unit tests as xUnit tests in C sharp that open a connection to a long-lived shared Azure SQL Database named PayoutEngine underscore QA, insert the contractors and jobs they need with the IDs one, ten and eleven, call the procedure, and assert on the result. Run the same test project as the integration suite by pointing the API at PayoutEngine underscore QA."

**Piece 9, option d.** "Option d. Rely on the build pipeline as the test suite. dotnet build validates every object reference and the T-SQL syntax. Add a sqlpackage Action DeployReport step against a staging database so reviewers can see the changes before merge. Because the payout bug was a logic error inside a procedure, add a required pull-request reviewer from the finance team instead of automated tests."

## 3. Setup script (reference only; do not read verbatim unless asked)

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

Option a, tSQLt test:

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

Option a, workflow:

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

## 4. The question (ask exactly this)

"Which approach should you implement? Option a, option b, option c, or option d?"

Options in full:

- **a.** tSQLt test classes in a tests folder, using FakeTable and AssertEqualsTable; a GitHub Actions job with a SQL Server service container; dotnet build, sqlpackage publish of the dacpac to the container, install tSQLt and the test classes, EXEC tSQLt.RunAll with sqlcmd dash b, then dotnet test of the API integration tests against the same container.
- **b.** The same tSQLt classes inside the PayoutEngine project, deployed everywhere, with RunAll executed against production after each deployment, relying on tSQLt's rollback.
- **c.** xUnit tests in C# against a long-lived shared PayoutEngine_QA database, inserting IDs 1, 10 and 11, with the same project doubling as the integration suite.
- **d.** dotnet build plus a sqlpackage DeployReport against staging, and a required finance reviewer instead of automated tests.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

- Unit tests, isolated: tSQLt.FakeTable replaces Contractors and Jobs with empty, constraint-free copies for the test, so the test controls exactly which rows exist; AssertEqualsTable compares expected and actual; every test runs inside a transaction that tSQLt rolls back. Tests live in a class from NewTestClass and are found by the test prefix; RunAll executes them, and sqlcmd dash b returns non-zero on failure, failing the pull-request check. Requirement one.
- Integration tests: dotnet build produces the dacpac, sqlpackage Action Publish deploys it with the real mechanism, and the API test project runs against that database. Requirement two.
- Never production, never long-lived: the service container from mcr.microsoft.com/mssql/server is created for the job and destroyed when it ends. The only secret is a throw-away sa password held as a GitHub secret. Requirements three and four.
- Test data is seeded explicitly per test, so every run is deterministic.

Why the others are wrong, one line each:

- **b.** Violates requirement three outright; tSQLt needs CLR enabled and TRUSTWORTHY or a signed assembly, installs hundreds of objects, and CLR is not available in Azure SQL Database; FakeTable renames the real table and takes schema-modification locks that block live payouts; and real data changes daily, so Net equals 160 is not deterministic.
- **c.** A shared long-lived database breaks isolation: two concurrent pull requests both insert ContractorId 1 and JobId 10, a forgotten cleanup poisons later runs, schema drifts because the database is never rebuilt from the project; it blurs unit and integration layers; and the database outlives the run, violating requirement four.
- **d.** dotnet build and DeployReport validate references, syntax and schema operations; neither executes the procedure, so neither can see the tax applied twice, and last month's faulty build was green. A human reviewer is a control, not a repeatable test.

## 6. Hint ladder (one hint per attempt, in order)

1. "Separate the two layers. A unit test checks the procedure's logic with inputs it controls. An integration test deploys the real artifact and drives the API. Which options actually execute the procedure at all?"
2. "Requirement three says never production, and requirement four says the test database must not outlive the run. Which option gives you a database that appears for the job and disappears afterwards?"
3. "Requirement one says isolated. If two pull requests run at the same time and both insert contractor one, what happens in a shared database?"
4. "Option d never runs the procedure. Could it have caught last month's double tax? That eliminates d."
5. "Option b runs FakeTable, which renames the real table, inside production, and asserts on data that changes daily. That eliminates b."
6. "You are down to a and c. One uses a throw-away service container and fake tables per test. The other uses one long-lived QA database shared by everyone. Which one satisfies isolation and does not outlive the run?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "b, tSQLt rolls back, so production is untouched" | Trusts rollback as a safety guarantee | "What does FakeTable do to the real table while the test runs, and what locks does that take? And what does requirement three say, regardless of cleanup?" |
| "b, real data is the most realistic" | Confuses realism with determinism | "If someone adds a job tomorrow, does Net still equal one hundred sixty? Should a unit test control its inputs?" |
| "c, a QA database is a standard practice" | Ignores concurrency and drift | "Two pull requests run at once and both insert JobId ten. What happens? And who rebuilds the QA schema from the project?" |
| "d, the build already validates the T-SQL" | Confuses static validation with testing | "Was last month's faulty procedure a valid build? What did the build check, and what did it not run?" |
| "d, a finance reviewer would have caught it" | Treats review as a test | "Does a reviewer run on every pull request deterministically? Can you re-run a reviewer when you suspect a regression?" |
| "a, but the sa password in the YAML is a problem" | Misreads the secret reference | "Where does the password come from in the workflow, a literal or a secret? And how long does that container live?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the two layers:

- Unit test of T-SQL logic: tSQLt. NewTestClass creates a schema for a class; test procedures are found by the test prefix. FakeTable swaps a table for an empty, constraint-free copy so the test seeds exactly the rows it needs. AssertEquals and AssertEqualsTable compare results; ExpectException and SpyProcedure cover errors and dependencies. Each test runs in a transaction that tSQLt rolls back. RunAll runs everything in CI, and sqlcmd dash b turns a failure into a non-zero exit code.
- Integration test of the artifact: build the dacpac, publish it with sqlpackage to a throw-away container or LocalDB, then run the application's tests against it. Same artifact, same deployment mechanism as real environments.

Then where they run: a GitHub Actions job with a SQL Server service container from mcr.microsoft.com/mssql/server, with ACCEPT underscore EULA and MSSQL underscore SA underscore PASSWORD from a secret, destroyed when the job ends. Nothing outlives the run and nothing touches production.

Then what is never right: production as a test target, even with rollback, because tSQLt needs CLR and TRUSTWORTHY, FakeTable takes schema locks on live tables, and real data is not deterministic; and a shared long-lived QA database whose data and schema drift between runs and whose rows collide across concurrent pull requests.

Then what static validation does and does not do: dotnet build catches broken references and platform syntax; DeployReport lists the schema operations a publish would perform. Only executing tests catches wrong results. Seed test data explicitly per test with fake tables plus inserts so every run is deterministic.

Memory hook: "Fake the tables, assert the result, roll it back. Publish the dacpac to a container that dies with the job. Never production, never a shared QA."

## 9. Follow-up oral questions (optional)

1. "Which tSQLt procedure would you use to check that the payout procedure raises an error for an unknown contractor?" (tSQLt.ExpectException.)
2. "Why is the dash b flag on the sqlcmd RunAll step important?" (It makes sqlcmd exit with a non-zero code on error, so a failing test fails the workflow step and the pull-request check.)
3. "What would the integration layer use to seed data, since FakeTable is not involved there?" (The API itself, or a post-deployment script, so the seed is deterministic and reproducible.)

## 10. References

- tSQLt on GitHub: https://github.com/tSQLt-org/tSQLt
- GitHub Actions service containers: https://docs.github.com/en/actions/use-cases-and-examples/using-containerized-services/about-service-containers
- SQL Server container image, environment variables: https://learn.microsoft.com/en-us/sql/linux/quickstart-install-connect-docker
- SqlPackage Publish: https://learn.microsoft.com/en-us/sql/tools/sqlpackage/sqlpackage-publish
- SqlPackage DeployReport: https://learn.microsoft.com/en-us/sql/tools/sqlpackage/sqlpackage-deploy-drift-report
- sqlcmd utility, including the dash b flag: https://learn.microsoft.com/en-us/sql/tools/sqlcmd/sqlcmd-utility
- SQL Database Projects, CI/CD tutorial: https://learn.microsoft.com/en-us/sql/tools/sql-database-projects/tutorials/create-deploy-sql-project
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
