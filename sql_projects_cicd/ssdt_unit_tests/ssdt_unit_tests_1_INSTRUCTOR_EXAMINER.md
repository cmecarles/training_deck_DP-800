# Instructor-Examiner guide — SSDT Unit Tests 1

Companion to [ssdt_unit_tests_1.md](ssdt_unit_tests_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d, and exactly one correct answer. Read all four options, pieces 8 to 14 of section 2, before taking an answer. When the learner picks a letter, ask them also to say, for each option they rejected, which of the five rules it breaks. That second part is where the hints go if needed.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Implement CI/CD by using SQL Database Projects.
- Task bullet: Implement unit tests for a SQL Server Database Project and run them from Azure Pipelines.
- What is tested: how the Visual Studio SQL Server unit test framework splits a test into pre-test, test and post-test scripts, which connection runs each script, which test condition checks what, how a negative test names a specific error, and how the build is deployed before the tests run.

## 2. Scenario to read aloud

**Piece 1, the story.** "RidgeWater Utilities bills its customers from meter readings stored in an Azure SQL Database. The schema is kept as a SQL Server Database Project called RidgeMeters, in Visual Studio with SQL Server Data Tools, and it is built and deployed from Azure Pipelines. Last quarter a release removed a safety check from a stored procedure by mistake, and three thousand negative consumptions were billed. So the team now wants an automated unit test suite."

**Piece 2, the tables.** "The project has a schema called Ops with two tables. Ops dot Meter has three columns: MeterId, an integer, the primary key. Serial, a ten-character code, not null and unique. And InstalledOn, a date. Ops dot Reading has four columns: ReadingId, an identity integer, the primary key. MeterId, an integer that references Ops dot Meter. ReadAt, a datetime2 with zero decimals. And CubicM, a decimal ten comma one, the cubic metres read. There is a unique constraint named UQ underscore Reading on the pair MeterId and ReadAt."

**Piece 3, the procedure.** "There is one stored procedure, Ops dot usp underscore RecordReading. It takes three parameters: at MeterId, at ReadAt, and at CubicM. It sets NOCOUNT on. It looks up the most recent reading of that meter, ordered by ReadAt descending, into a variable called at Last. Then the guard: if at Last is not null and the new CubicM is lower than at Last, it runs THROW with error number fifty thousand ten, the message Reading is lower than the previous reading for this meter, and state one. Otherwise it inserts the new reading into Ops dot Reading, and finally it selects the row it just inserted: ReadingId, MeterId, ReadAt and CubicM, using SCOPE underscore IDENTITY. So on success the procedure returns exactly one row with four columns, and CubicM is the fourth column."

**Piece 4, the security.** "The project also creates a database user called meter underscore app for a login of the same name, and grants EXECUTE on the procedure to that user. That is the only permission meter underscore app has. The billing application connects as meter underscore app. It cannot insert or delete on the tables directly."

**Piece 5, rules one and two.** "The team wrote five rules for the test suite. Rule one, the positive test: with meter seven seeded with a reading of one thousand two hundred point zero, calling the procedure for meter seven, on the first of September twenty twenty-six, with one thousand two hundred fifty point zero, must return exactly one row whose CubicM is one thousand two hundred fifty point zero. Rule two, the negative test: calling the procedure with one thousand one hundred point zero for the same meter must be rejected with error fifty thousand ten, severity sixteen, state one. The test must pass only when that specific error is raised, and must fail if no error, or a different error, occurs."

**Piece 6, rule three.** "Rule three is about accounts. The test scripts that call the procedure must run as meter underscore app, so that a missing grant is caught. But the seeding and cleanup statements must run under a separate, privileged account. The meter underscore app account must not be granted INSERT or DELETE on the tables."

**Piece 7, rules four and five.** "Rule four: every test run must first deploy the current build of the project to the test database, so the tests always exercise the schema in the pull request. Rule five: the tests must run in Azure Pipelines on a hosted Windows agent and publish results, using the Visual Studio SQL Server unit test framework, the one in the namespace Microsoft dot Data dot Tools dot Schema dot Sql dot UnitTesting. No third-party framework, and no CLR on the test server."

**Piece 8, option a, the setup.** "Option a. In SQL Server Object Explorer, right-click the procedure and choose Create Unit Tests. That creates a new C sharp test project called RidgeMeters dot Tests, and the generated class derives from SqlDatabaseTestClass. In the SQL Server Test Configuration dialog, the execution context is set to a connection that logs in as meter underscore app. The privileged context is set to ridge underscore deployer, which is a member of db underscore owner. The box Automatically deploy the database project before unit tests are run is ticked, and RidgeMeters dot sqlproj with configuration Release is selected."

**Piece 9, option a, the two tests.** "Then, in the SQL Server Unit Test Designer, two tests. The first is usp underscore RecordReading underscore Increasing. Its Pre-Test script is an idempotent seed: it puts meter seven in place with the twelve hundred reading and deletes any later readings. Its Test script is one line: EXEC the procedure with MeterId seven, ReadAt the first of September twenty twenty-six at midnight, CubicM one thousand two hundred fifty point zero. The default Inconclusive condition is deleted, and two conditions are added: Row Count equal to one, and Scalar Value on result set one, row one, column four, with expected value one thousand two hundred fifty point zero. The Post-Test script deletes the row the test inserted. The second test is usp underscore RecordReading underscore LowerFails. Same Pre-Test. Its Test script executes the procedure for meter seven, on the second of September, with one thousand one hundred point zero. The Inconclusive condition is deleted. And in the C sharp file the method is marked with two attributes: TestMethod, and ExpectedSqlException with MessageNumber fifty thousand ten, Severity sixteen, State one."

**Piece 10, option a, the pipeline.** "The pipeline builds the solution on windows-latest with the VSBuild task, solution RidgeMeters dot sln, configuration Release. Then it runs the tests with the Visual Studio Test task, VSTest at version three, selecting test assemblies matching RidgeMeters dot Tests dot dll under the default working directory. The two connection strings are substituted into app dot config from pipeline secrets."

**Piece 11, option b.** "Option b. Same test project, same configuration dialog and same pipeline as option a, with two simplifications. First, the seeding statements are moved to the top of each Test script, so the whole test is one script. Second, the negative test is marked with MSTest's generic attribute ExpectedException of type SqlException, instead of ExpectedSqlException, so the test passes whenever the procedure throws."

**Piece 12, option c.** "Option c. Install tSQLt in the test database. That means running sp underscore configure clr enabled one, and ALTER DATABASE RidgeMeters SET TRUSTWORTHY ON. Then write EXEC tSQLt dot NewTestClass with the name ReadingTests, and a test procedure that calls tSQLt dot ExpectException with ExpectedErrorNumber fifty thousand ten before executing the procedure. The pipeline publishes the dacpac and then runs sqlcmd with the query EXEC tSQLt dot RunAll and the dash b flag."

**Piece 13, option d.** "Option d. Same test project as option a, but the negative Test script has two lines: first EXEC the procedure for meter seven, second of September, one thousand one hundred point zero; then SELECT at at ERROR AS Err. A Scalar Value condition expects fifty thousand ten. Both the execution context and the privileged context point at ridge underscore deployer, to avoid permission noise. The box Automatically deploy the database project is unticked, and the tests run against the shared RidgeMeters underscore Dev database that already exists."

**Piece 14, recap.** "So, four options. A: two contexts, seed in Pre-Test, ExpectedSqlException with number, severity and state, auto-deploy ticked, VSTest. B: same but seed inside the Test script and generic ExpectedException. C: tSQLt with CLR and sqlcmd. D: SELECT at at ERROR after the EXEC, one privileged account for everything, no auto-deploy. Tell me when you are ready for the question."

## 3. Setup script (reference only; do not read verbatim unless asked)

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

Option a, Test scripts:

```sql
-- usp_RecordReading_Increasing, Test script
EXEC Ops.usp_RecordReading @MeterId = 7, @ReadAt = '2026-09-01T00:00:00', @CubicM = 1250.0;
-- usp_RecordReading_LowerFails, Test script
EXEC Ops.usp_RecordReading @MeterId = 7, @ReadAt = '2026-09-02T00:00:00', @CubicM = 1100.0;
```

Option a, attribute on the negative test method:

```csharp
[TestMethod(), ExpectedSqlException(MessageNumber = 50010, Severity = 16, State = 1)]
public void usp_RecordReading_LowerFails()
```

Option a, pipeline steps:

```yaml
- task: VSBuild@1
  inputs: { solution: 'RidgeMeters.sln', configuration: 'Release' }
- task: VSTest@3
  inputs:
    testSelector: testAssemblies
    testAssemblyVer2: '**\RidgeMeters.Tests.dll'
    searchFolder: '$(System.DefaultWorkingDirectory)'
```

Option b, attribute on the negative test method:

```csharp
[ExpectedException(typeof(SqlException))]
```

Option c, tSQLt setup and call:

```sql
EXEC sp_configure 'clr enabled', 1;
ALTER DATABASE RidgeMeters SET TRUSTWORTHY ON;
EXEC tSQLt.NewTestClass 'ReadingTests';
EXEC tSQLt.ExpectException @ExpectedErrorNumber = 50010;
-- pipeline: sqlcmd -Q "EXEC tSQLt.RunAll" -b
```

Option d, negative Test script:

```sql
EXEC Ops.usp_RecordReading @MeterId = 7, @ReadAt = '2026-09-02T00:00:00', @CubicM = 1100.0;
SELECT @@ERROR AS Err;
```

## 4. The question (ask exactly this)

"Which implementation meets every rule? Choose one letter."

Options, read in full if the learner asks:

- **a.** In SQL Server Object Explorer, right-click `Ops.usp_RecordReading` and choose Create Unit Tests, creating a new C# test project `RidgeMeters.Tests` whose generated class derives from `SqlDatabaseTestClass`. In the SQL Server Test Configuration dialog set the execution context to a connection that logs in as `meter_app`, the privileged context to `ridge_deployer` (a member of `db_owner`), tick "Automatically deploy the database project before unit tests are run", and select `RidgeMeters.sqlproj`, configuration Release. Then, in the SQL Server Unit Test Designer: `usp_RecordReading_Increasing` with a Pre-Test script that idempotently seeds meter 7 with the 1200.0 reading and deletes any later readings, a Test script `EXEC Ops.usp_RecordReading @MeterId = 7, @ReadAt = '2026-09-01T00:00:00', @CubicM = 1250.0;`, the default Inconclusive condition deleted, Row Count = 1 and Scalar Value (ResultSet 1, Row 1, Column 4, expected 1250.0) added, and a Post-Test script that deletes the inserted row. And `usp_RecordReading_LowerFails` with the same Pre-Test, Test script `EXEC Ops.usp_RecordReading @MeterId = 7, @ReadAt = '2026-09-02T00:00:00', @CubicM = 1100.0;`, Inconclusive deleted, and the method marked `[TestMethod(), ExpectedSqlException(MessageNumber = 50010, Severity = 16, State = 1)]`. The pipeline builds the solution on windows-latest with VSBuild@1 and runs the tests with VSTest@3, substituting the two connection strings into app.config from pipeline secrets.
- **b.** Same test project, configuration dialog and pipeline as option a, with two simplifications: put the seeding statements at the top of each Test script "so the whole test is one script", and mark the negative test with MSTest's generic attribute `[ExpectedException(typeof(SqlException))]` instead of `ExpectedSqlException`, so the test passes whenever the procedure throws.
- **c.** Install tSQLt in the test database (`EXEC sp_configure 'clr enabled', 1; ALTER DATABASE RidgeMeters SET TRUSTWORTHY ON;`), write `EXEC tSQLt.NewTestClass 'ReadingTests';` and a test procedure that calls `EXEC tSQLt.ExpectException @ExpectedErrorNumber = 50010;` before executing `Ops.usp_RecordReading`, then have the pipeline run `sqlcmd -Q "EXEC tSQLt.RunAll" -b` after publishing the `.dacpac`.
- **d.** Same test project as option a, but write the negative Test script as the `EXEC` followed by `SELECT @@ERROR AS Err;` with a Scalar Value condition whose expected value is 50010; point both the execution context and the privileged context at `ridge_deployer` "to avoid permission noise"; untick "Automatically deploy the database project" and run the tests against the shared `RidgeMeters_Dev` database that already exists.

After the letter: "Now, for each option you rejected, tell me which rule it breaks."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

Why a is correct, rule by rule:

- Rule 3: the Pre-Test and Post-Test scripts run under the privileged context, `ridge_deployer`; the Test script runs under the execution context, `meter_app`. Seed and cleanup are privileged, the `EXEC` is low-privilege. A missing `GRANT EXECUTE` would make Test 1 fail with error 229.
- Rule 1: the Inconclusive condition is deleted; Row Count = 1 checks exactly one row; Scalar Value on ResultSet 1, Row 1, Column 4 checks `CubicM` = 1250.0 (column 4 of the procedure's SELECT). Expected row: ReadingId 2, MeterId 7, ReadAt 2026-09-01 00:00:00, CubicM 1250.0.
- Rule 2: `ExpectedSqlException(MessageNumber = 50010, Severity = 16, State = 1)` passes only for that error. The engine output is `Msg 50010, Level 16, State 1, Procedure Ops.usp_RecordReading, Line 12` followed by the message text.
- Rule 4: "Automatically deploy the database project before unit tests are run" builds and publishes the sqlproj at the start of the run, under the privileged connection.
- Rule 5: the test assembly is MSTest, so VSTest@3 runs it on the hosted Windows agent and publishes results. No CLR, no third-party framework.

Why each wrong option is wrong:

- **b** — the seed moves into the Test script, which runs as `meter_app`; the INSERT fails with error 229, "The INSERT permission was denied on the object 'Meter', database 'RidgeMeters', schema 'Ops'." (Level 14, State 5). That breaks Test 1 and rule 3. And the generic `[ExpectedException(typeof(SqlException))]` passes for any SqlException, 229, 2627, a typo, so Test 2 proves nothing; breaks rule 2.
- **c** — tSQLt is a CLR assembly needing `clr enabled` and `TRUSTWORTHY ON`; it is a third-party framework, not the Visual Studio one; VSTest discovers nothing and no results are published; and it has no execution/privileged split. Breaks rules 5 and 3.
- **d** — `THROW` aborts the batch, so `SELECT @@ERROR` never runs and the Scalar Value condition is never evaluated; Test 2 fails whenever the procedure is correct (breaks rule 2). Both contexts on `ridge_deployer` means a missing grant is never caught (breaks rule 3). No auto-deploy and a shared dev database means the pull request's schema is not what is tested (breaks rule 4).

## 6. Hint ladder (one hint per attempt, in order)

**Choosing the letter**
1. "Start with rule five. It names one framework and forbids CLR. Does any option install something that needs CLR enabled?"
2. "Now rule three. It says which account runs which script. In the Visual Studio framework there are two connections. Which scripts run under the privileged one, and which under the execution one?"
3. "In one of the remaining options the seed is inside the Test script. Under which connection does the Test script run? Does that account have INSERT?"
4. "Rule two asks for the specific error, number fifty thousand ten, severity sixteen, state one. Which attribute lets you name number, severity and state? And what does the generic MSTest attribute accept?"
5. "One option reads at at ERROR on the line after the EXEC. Think about what THROW does to the batch. Does the next line ever run?"
6. "You are down to two options. One of them ticks the automatic deploy box and keeps two different accounts. The other unticks it and uses one account for everything. Which one satisfies rules three and four?"

**Explaining why b is wrong**
1. "Which connection runs the Test script, and which runs the Pre-Test script?"
2. "If the seed INSERT runs as meter underscore app, what error do you get? Think of the permission denied message."
3. "And the generic ExpectedException: if the seed itself throws error 229, does the negative test pass or fail?"

**Explaining why c is wrong**
1. "What does tSQLt need on the server before you can install it?"
2. "Rule five names a namespace, Microsoft dot Data dot Tools dot Schema dot Sql dot UnitTesting. Is tSQLt that framework?"
3. "How would the Visual Studio Test task find tSQLt tests, and where would the results be published?"

**Explaining why d is wrong**
1. "There are three separate problems in d. Start with the SELECT at at ERROR. Does the batch continue after THROW?"
2. "Second: if both contexts are ridge underscore deployer, which rule can no longer be checked?"
3. "Third: if the project is not deployed first and the tests run against RidgeMeters underscore Dev, whose schema are you testing?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "b, it is the same as a but simpler" | Does not know that the Test script runs under the execution context | "Simpler how? Ask yourself which connection executes the Test script in this framework." |
| "b, any exception means the guard works" | Thinks a generic expected exception satisfies rule two | "Rule two says the test must fail if a different error occurs. Does a generic SqlException attribute distinguish errors?" |
| "c, tSQLt is the standard for T-SQL unit tests" | Ignores rule five | "Read rule five again. Which framework is named, and what is forbidden on the server?" |
| "d, at at ERROR captures the number" | Does not know THROW terminates the batch | "After a THROW inside the procedure, what happens to the rest of the calling batch?" |
| "d, one privileged account avoids noise" | Misses that rule three is about catching missing grants | "What is rule three trying to catch? Can a db owner ever hit a missing grant?" |
| "a, but the column index should be one" | Miscounts the procedure's SELECT list | "Read the SELECT at the end of the procedure. Count the columns. Which position is CubicM?" |
| "a is wrong because the Post-Test also runs as meter underscore app" | Confuses which scripts are privileged | "Which scripts run under the privileged context? Pre-Test, Post-Test, or Test?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the framework as three scripts and two connections:

- **Three scripts per test.** Pre-Test arranges the data, Test executes the code under test, Post-Test cleans up. Test conditions are evaluated only on the result of the Test script. TestInitialize and TestCleanup are common scripts for the whole class. A new test starts with the Inconclusive condition, which must be deleted once real conditions are added.
- **Two connections in app.config.** The execution context runs the Test script and should carry the same credentials your users have. The privileged context runs Pre-Test, Post-Test, TestInitialize, TestCleanup, and also deploys the project. The two only differ when SQL authentication is used, which is why `meter_app` is a SQL login. That is why option a puts the seed in the Pre-Test and why option b's seed fails with error 229.
- **Conditions.** Row Count fails if the result set does not have the expected number of rows. Scalar Value is addressed by ResultSet, Row and Column; column 4 of the procedure's output is `CubicM`. Others: Empty ResultSet, Not Empty ResultSet, Expected Schema, Data Checksum, Execution Time (default thirty seconds, Test script only).
- **Negative tests.** `ExpectedSqlException(MessageNumber, Severity, State, MatchFirstError)`; unspecified parts are ignored, and the values are the ones you pass to THROW. The generic MSTest `ExpectedException(typeof(SqlException))` matches any SqlException, so it cannot enforce rule two. Without any attribute the run reports "Test method threw exception: SqlException: Reading is lower than the previous reading for this meter."
- **THROW aborts the batch.** `SELECT @@ERROR` after a failing EXEC is dead code. The client gets the exception, the test method throws, and no condition is evaluated. That kills option d's negative test.
- **Deploy first.** The "Automatically deploy the database project before unit tests are run" box builds and publishes the selected sqlproj at the start of the run under the privileged connection. Unticking it and testing a shared dev database means you test whatever is there, not the pull request.
- **CI.** The SSDT test assembly is MSTest, so VSBuild builds it and VSTest@3 runs it with vstest.console.exe on a Windows agent with Visual Studio, publishing results. VSTest@2 is the older version being deprecated. tSQLt is a CLR assembly needing `clr enabled` and TRUSTWORTHY or a signed assembly; wrong framework for rule five, and invisible to VSTest.

Memory hook: "Pre and Post are privileged. Test is the user. Name the error number, or the green light means nothing. THROW kills the batch. Deploy before you test."

## 9. Follow-up oral questions (optional)

1. "If a future change forgot the GRANT EXECUTE to meter underscore app, which test in option a would fail, and with which error number?" (Test 1, the positive test; error 229, EXECUTE permission denied.)
2. "In option a, the negative test's Test script runs as meter underscore app. Would ExpectedSqlException still pass if the seed in the Pre-Test failed with error 229?" (No. The Pre-Test error is not the expected 50010 with severity 16 and state 1, so the test fails, which is the desired behaviour.)
3. "Which SSDT test condition would you add to make sure the procedure's result set has the four expected columns with the right types?" (Expected Schema.)

## 10. References

- SQL Server unit tests in SSDT, overview: https://learn.microsoft.com/en-us/sql/ssdt/verifying-database-code-by-using-sql-server-unit-tests
- Creating and defining SQL Server unit tests, including scripts and conditions: https://learn.microsoft.com/en-us/sql/ssdt/creating-and-defining-sql-server-unit-tests
- Configure SQL Server unit test execution, execution and privileged contexts, automatic deployment: https://learn.microsoft.com/en-us/sql/ssdt/how-to-configure-sql-server-unit-test-execution
- Test conditions, Inconclusive, Row Count, Scalar Value and the others: https://learn.microsoft.com/en-us/sql/ssdt/using-test-conditions-in-sql-server-unit-tests
- Visual Studio Test task, VSTest@3: https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/vstest-v3
- THROW, batch termination: https://learn.microsoft.com/en-us/sql/t-sql/language-elements/throw-transact-sql
- tSQLt source, for context on option c: https://github.com/tSQLt-org/tSQLt
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
