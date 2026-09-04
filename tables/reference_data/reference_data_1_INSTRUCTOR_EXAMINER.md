# Instructor-Examiner guide — Reference Data 1

Companion to [reference_data_1.md](reference_data_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Read all four options before taking an answer. The three requirements in piece 5 are the judging criteria; if the learner asks, repeat them. This is a tooling question about SQL Database Projects, so there is nothing to "run"; the learner reasons from how the build and the deployment pipeline behave.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Implement CI/CD by using SQL Database Projects.
- Task bullet: Manage reference data in a SQL Database Project.
- What is tested: that a dacpac carries schema and not data, that reference data goes in the post-deployment script, that the script runs on every deployment and so must be idempotent, and why an INSERT in an object file, a plain INSERT in post-deploy, and a truncate-and-reload in pre-deploy all fail.

## 2. Scenario to read aloud

**Piece 1, the story.** "A consumer-lending startup keeps the schema of its SQL Server database, called LoanBook, in an SDK-style SQL Database Project. The project file is LoanBook dot sqlproj. They deploy it with sqlpackage, action Publish, to development, to test, and to production. The project defines a lookup table and a table that references it."

**Piece 2, the lookup table.** "The first file is Ref, slash, Tables, slash, LoanStatus dot sql. It creates the table Ref dot LoanStatus with four columns. StatusId, an integer, IDENTITY starting at one, step one, NOT NULL, the primary key, constraint name PK underscore LoanStatus. Code, VARCHAR twelve, NOT NULL, with a UNIQUE constraint. StatusName, NVARCHAR forty, NOT NULL. And IsTerminal, a BIT, NOT NULL."

**Piece 3, the referencing table.** "The second file is Ref, slash, Tables, slash, Loans dot sql. It creates Ref dot Loans with two columns. LoanId, an integer, the primary key. And StatusId, an integer, NOT NULL, with a foreign key named FK underscore Loans underscore Status that references Ref dot LoanStatus on StatusId."

**Piece 4, what production holds today.** "The application code refers to statuses by their numeric StatusId. For example, a constant Approved equals two. So the same StatusId values must exist in every environment. Production currently has three rows. Row 1, code APPLIED, name Applied, not terminal. Row 2, code APPROVED, but its name is Approved, open paren, old, close paren, because an operator renamed it by hand. And row 5, code LEGACY, name Legacy status, terminal. Row 5 was retired, and no loan references it."

**Piece 5, the target and the requirements.** "The next release must ship the reference data as part of the project, so every deployment leaves the table with exactly four rows. One, APPLIED, Applied, zero. Two, APPROVED, Approved, zero. Three, DISBURSED, Disbursed, zero. Four, CLOSED, Closed, one. So on production that means: rename row 2, add rows 3 and 4, remove row 5. Three requirements. One: the first deployment to an empty database and the hundredth deployment to production must both succeed and both produce that table. Two: the StatusId values must be exactly those listed, never whatever IDENTITY would generate next. Three: the data must be versioned in git next to the schema and reviewed through the normal pull-request process."

**Piece 6, option a.** "Option a appends the seed rows to the table's own definition file, LoanStatus dot sql, right after the CREATE TABLE statement. The appended block is: SET IDENTITY underscore INSERT Ref dot LoanStatus ON. Then one INSERT with the four rows and explicit StatusId values one to four. Then SET IDENTITY underscore INSERT OFF."

**Piece 7, option b.** "Option b adds a post-deployment script to the project. In the project file that is a PostDeploy item pointing at Scripts, backslash, PostDeploy dot sql. The script contains the very same block as option a: IDENTITY underscore INSERT ON, the plain INSERT of the four rows, IDENTITY underscore INSERT OFF."

**Piece 8, option c.** "Option c adds a pre-deployment script instead, a PreDeploy item pointing at Scripts, backslash, PreDeploy dot sql. The idea is to empty and refill the table before the schema is compared. The script is: TRUNCATE TABLE Ref dot LoanStatus. Then IDENTITY underscore INSERT ON. Then the same INSERT of the four rows. Then IDENTITY underscore INSERT OFF."

**Piece 9, option d.** "Option d adds a post-deployment script, PostDeploy dot sql, whose only content is a colon r include, pointing at dot, backslash, ReferenceData, backslash, LoanStatus dot sql. That included file has build action None, or is removed from Build in the project file. It contains: IDENTITY underscore INSERT ON. Then a MERGE into Ref dot LoanStatus as target, using a VALUES list of the four rows as source, matched on StatusId. When matched and any of Code, StatusName or IsTerminal differs, update those three columns. When not matched by target, insert the row with its explicit StatusId. When not matched by source, delete. Then IDENTITY underscore INSERT OFF."

## 3. Setup script (reference only; do not read verbatim unless asked)

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

Production rows today: `(1, 'APPLIED', N'Applied', 0)`, `(2, 'APPROVED', N'Approved (old)', 0)`, `(5, 'LEGACY', N'Legacy status', 1)`.

Target rows after every deployment:

| StatusId | Code | StatusName | IsTerminal |
|---|---|---|---|
| 1 | APPLIED | Applied | 0 |
| 2 | APPROVED | Approved | 0 |
| 3 | DISBURSED | Disbursed | 0 |
| 4 | CLOSED | Closed | 1 |

Option a (appended to `Ref/Tables/LoanStatus.sql`) and option b (in `Scripts\PostDeploy.sql`):

```sql
SET IDENTITY_INSERT Ref.LoanStatus ON;
INSERT INTO Ref.LoanStatus (StatusId, Code, StatusName, IsTerminal) VALUES
 (1, 'APPLIED', N'Applied', 0), (2, 'APPROVED', N'Approved', 0),
 (3, 'DISBURSED', N'Disbursed', 0), (4, 'CLOSED', N'Closed', 1);
SET IDENTITY_INSERT Ref.LoanStatus OFF;
```

Option c (`Scripts\PreDeploy.sql`):

```sql
TRUNCATE TABLE Ref.LoanStatus;
SET IDENTITY_INSERT Ref.LoanStatus ON;
INSERT INTO Ref.LoanStatus (StatusId, Code, StatusName, IsTerminal) VALUES
 (1, 'APPLIED', N'Applied', 0), (2, 'APPROVED', N'Approved', 0),
 (3, 'DISBURSED', N'Disbursed', 0), (4, 'CLOSED', N'Closed', 1);
SET IDENTITY_INSERT Ref.LoanStatus OFF;
```

Option d:

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

## 4. The question (ask exactly this)

"Which implementation meets all three requirements? Option a: put the IDENTITY underscore INSERT and INSERT block in the table's own definition file, after CREATE TABLE. Option b: put that same plain INSERT block in a post-deployment script. Option c: put a TRUNCATE followed by the INSERT block in a pre-deployment script. Option d: a post-deployment script that includes, with colon r, one file per lookup table, each file holding an idempotent MERGE wrapped in IDENTITY underscore INSERT ON and OFF. Which one, a, b, c or d?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: d.**

A dacpac describes schema, not data. Reference data is delivered by the post-deployment script, which SqlPackage runs after the schema deployment plan completes, on every deployment. So the script must be idempotent. The MERGE on the fixed key StatusId turns the rename into an UPDATE, the new codes into INSERTs, the retired row into a DELETE, and leaves matching rows alone. Run 1 against production: UPDATE row 2, INSERT rows 3 and 4, DELETE row 5. Run 2: zero rows affected. `SET IDENTITY_INSERT ON` is required to write explicit StatusId values; without it both a plain INSERT and the MERGE insert branch fail with Msg 544. The `:r` include keeps each lookup table in its own reviewable file; `Microsoft.Build.Sql` concatenates the included files into the single post-deployment script at build time. Included files must be excluded from the model build with `<Build Remove=...>` or build action None.

Why each wrong option is wrong:

- **a** — Files with build action Build are compiled into the schema model, which has no notion of rows. `SET IDENTITY_INSERT` and `INSERT` are not declarative object definitions, so the build fails with DacFx error `SQL70001: This statement is not recognized in this context.` No dacpac is produced, nothing deploys.
- **b** — Right vehicle, wrong statement. A plain INSERT works once on an empty database and breaks on every later deployment: `Msg 2627, Violation of PRIMARY KEY constraint 'PK_LoanStatus'. Cannot insert duplicate key ... The duplicate key value is (3).` It also never renames row 2 or deletes row 5. Requirement 1 fails.
- **c** — The pre-deployment script runs before the schema deployment plan, so on the first deployment to an empty database the table does not exist yet and the script fails. In production, Ref dot Loans references the table, so `TRUNCATE` fails with `Msg 4712: Cannot truncate table 'Ref.LoanStatus' because it is being referenced by a FOREIGN KEY constraint.` A DELETE instead would fail on referenced rows with Msg 547. And wipe-and-reload rewrites unchanged rows and resets the identity seed. Requirement 1 fails on both ends.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start from what a dacpac is. Does it carry rows, or only object definitions? Where does data that must exist in every environment go?"
2. "One requirement says the hundredth deployment must succeed. Post-deployment scripts run on every deployment. What property must such a script have?"
3. "Take option a. The table file is compiled into the schema model. Is an INSERT statement a declarative object definition? What does the build do with a statement it does not recognize?"
4. "Take option c. In what order do the pre-deployment script and the schema plan run? On a brand-new empty database, does the table exist when the pre-deployment script runs? And in production, what does a foreign key do to TRUNCATE?"
5. "You are down to b and d. Both are post-deployment scripts. Run each one twice in your head against production. Which one survives the second run, and which one can rename row 2 and remove row 5?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, because the data belongs with the table definition" | Thinks object files can contain any T-SQL | "The build compiles that file into a schema model. Which statements does the model understand? Is INSERT one of them?" |
| "b, post-deployment is the documented place for reference data" | Right location, does not test the second deployment | "The location is right. Now run that INSERT a second time against a table that already has rows one to four. What happens?" |
| "c, because truncate and reload is always the same result" | Ignores execution order and foreign keys | "Two things. When does a pre-deployment script run relative to the schema plan, and what does Ref dot Loans do to a TRUNCATE?" |
| "c, but with DELETE instead of TRUNCATE" | Thinks DELETE avoids the foreign key issue | "If a loan references row 2, what does DELETE do to row 2? And the ordering problem on an empty database remains." |
| "d, but IDENTITY underscore INSERT is unnecessary because MERGE handles keys" | Does not know explicit identity values need the setting | "Try inserting an explicit StatusId into an IDENTITY column with the setting off. What error do you get?" |
| "d cannot work because colon r is not T-SQL" | Unaware that post-deployment scripts are SQLCMD mode | "Post-deployment scripts support SQLCMD commands. What does the build do with an included file?" |
| "d will fail because it deletes row 5" | Worried about foreign keys on the delete branch | "Check the story. Does any loan reference row 5?" |

## 8. Teaching notes (after the answer is complete or revealed)

Start with the principle: **a dacpac is schema, reference data is a post-deployment script, and that script runs on every deployment, so it must be idempotent.**

Then walk the four options:

- **Option a, wrong file.** Anything with build action Build is compiled into the model. The model only knows declarative object definitions. INSERT and SET IDENTITY underscore INSERT are not, so DacFx raises SQL70001, the build fails, and no dacpac exists.
- **Option b, wrong statement.** Post-deployment is right. But INSERT cannot say "make the table look like this". It succeeds once and then hits Msg 2627 on the primary key. It also never renames row 2 or deletes row 5.
- **Option c, wrong moment and wrong tool.** Pre-deployment runs before the schema plan. The plan is computed first, then the pre-deployment script executes, then the plan applies. On an empty database the table does not exist yet. In production, the foreign key from Ref dot Loans blocks TRUNCATE with Msg 4712, and DELETE would hit Msg 547 on referenced rows. Even where it could run, wiping and reloading every deployment rewrites unchanged rows and resets the identity seed. Pre-deployment is for data preparation that must happen before the plan, for example saving the data of a column about to be dropped.
- **Option d, the pattern.** One PostDeploy item per project. It includes per-table files with colon r; Microsoft dot Build dot Sql concatenates them into the single post-deployment script at build time. The included files are removed from Build, optionally re-added as None so they stay visible. Each file wraps a MERGE in IDENTITY underscore INSERT ON and OFF: only one table per session can have it on. The MERGE on the fixed key does UPDATE when matched and different, INSERT when not matched by target, DELETE when not matched by source. Run once on production: one update, two inserts, one delete. Run again: zero rows. Run a hundred times: same table.

One caveat to mention: WHEN NOT MATCHED BY SOURCE THEN DELETE works here because no loan references row 5. If a referenced row were dropped from the source list, the DELETE would fail with Msg 547, which is the correct outcome. The deployment stops rather than orphaning data. Retiring a status is a two-step change: migrate the loans, then remove the row.

Memory hook: "Dacpac is schema. Data goes post-deploy. Post-deploy runs every time, so MERGE, not INSERT."

## 9. Follow-up oral questions (optional)

1. "How many pre-deployment and post-deployment scripts can a SQL Database Project have?" (One of each.)
2. "What happens if the MERGE in option d runs with IDENTITY underscore INSERT left OFF?" (The insert branch fails with Msg 544, cannot insert explicit value for identity column.)
3. "Give one legitimate use of a pre-deployment script." (Data preparation that must happen before the schema plan, for example copying the values of a column that the release is about to drop.)

## 10. References

- SQL Database Projects, pre-deployment and post-deployment scripts: https://learn.microsoft.com/en-us/sql/tools/sql-database-projects/concepts/pre-post-deployment-scripts
- SQL Database Projects overview and SDK-style projects: https://learn.microsoft.com/en-us/sql/tools/sql-database-projects/sql-database-projects
- SqlPackage Publish: https://learn.microsoft.com/en-us/sql/tools/sqlpackage/sqlpackage-publish
- MERGE: https://learn.microsoft.com/en-us/sql/t-sql/statements/merge-transact-sql
- SET IDENTITY_INSERT: https://learn.microsoft.com/en-us/sql/t-sql/statements/set-identity-insert-transact-sql
- Microsoft.Build.Sql on GitHub: https://github.com/microsoft/DacFx
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
