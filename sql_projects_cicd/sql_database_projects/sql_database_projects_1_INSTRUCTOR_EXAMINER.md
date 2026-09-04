# Instructor-Examiner guide — SQL Database Projects 1

Companion to [sql_database_projects_1.md](sql_database_projects_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a, b, c and d. Read the project file, the workflow, the incident, the two requirements and all four options before taking an answer. Each option is a sqlpackage command; name the Action, the source and target parameters, and any slash p property precisely in words. This is a conceptual tooling question; nothing was executed against an engine.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Implement CI/CD by using SQL Database Projects.
- Task bullet: Deploy a SQL project with SqlPackage; detect and manage schema drift.
- What is tested: choosing the right SqlPackage action, DeployReport versus DriftReport versus Publish versus Extract, and knowing that out-of-band changes must be merged back into the project source because the project is the source of truth.

## 2. Scenario to read aloud

**Piece 1, the story.** "A supermarket chain's development team maintains the schema of the Azure SQL Database StoreCatalog as an SDK-style SQL Database Project stored in GitHub. The project file starts with a Project element whose Sdk attribute is Microsoft dot Build dot Sql slash one point zero point zero. Its PropertyGroup has Name StoreCatalog and DSP set to Microsoft dot Data dot Tools dot Schema dot Sql dot SqlAzureV12DatabaseSchemaProvider, the Azure SQL Database platform."

**Piece 2, the workflow.** "A GitHub Actions workflow runs on every merge to main. Step one, Build database project: dotnet build StoreCatalog dot sqlproj, configuration Release. Step two, Deploy to production: sqlpackage slash Action colon Publish, with SourceFile bin slash Release slash StoreCatalog dot dacpac and TargetConnectionString from the secret PROD underscore SQL underscore CONNECTION. The publish step runs with default SqlPackage publish properties. And one important fact: the StoreCatalog database has never been registered as a data-tier application."

**Piece 3, the incident.** "During a weekend incident, a DBA connected directly to production and hot-fixed the database. The DBA created a nonclustered index named IX underscore Product underscore SupplierId on dbo dot Product, and rewrote the body of the stored procedure dbo dot usp underscore GetProductsBySupplier. Neither change was made in the SQL Database Project. Several unrelated feature branches are about to be merged to main."

**Piece 4, the two requirements.** "Before the next deployment runs, the team lead requires two things. One: produce a reviewable report of the exact actions the next deployment would perform against the current state of production, without modifying production. Two: ensure the hot-fix is preserved by making it part of the project source through the normal pull-request process, so that future deployments do not revert it."

**Piece 5, option a.** "Option a. A step named Deploy with safety check: sqlpackage Action Publish, SourceFile bin slash Release slash StoreCatalog dot dacpac, TargetConnectionString from the production secret, and the property slash p colon BlockOnPossibleDataLoss equals True. Rely on BlockOnPossibleDataLoss to terminate the publish because the target schema no longer matches the dacpac. Review the error output to identify the hot-fixed objects, then add them to the project in a pull request."

**Piece 6, option b.** "Option b. A step named Detect drift: sqlpackage Action DriftReport, TargetConnectionString from the production secret, OutputPath drift dash report dot xml. No SourceFile. Run DriftReport against production to generate an XML report of the changes the DBA made, review the report, then add the hot-fixed objects to the project in a pull request."

**Piece 7, option c.** "Option c. Two steps. First, Build database project: dotnet build StoreCatalog dot sqlproj, configuration Release. Second, Report pending deployment actions: sqlpackage Action DeployReport, SourceFile bin slash Release slash StoreCatalog dot dacpac, TargetConnectionString from the production secret, OutputPath deploy dash report dot xml. Build the dacpac from main, run DeployReport with the dacpac as source and production as target, and review the XML report of the actions a publish would take. The report reveals that publish would drop IX underscore Product underscore SupplierId and alter usp underscore GetProductsBySupplier back to the old body. Add the index definition and the new procedure body to the project source in a pull request before the next deployment."

**Piece 8, option d.** "Option d. A step named Capture production baseline: sqlpackage Action Extract, SourceConnectionString from the production secret, TargetFile bin slash Release slash StoreCatalog dot dacpac. Run Extract against production and overwrite the pipeline's build artifact with the extracted dacpac, so that the next publish step deploys a dacpac that already contains the hot-fix and nothing is reverted."

## 3. Setup script (reference only; do not read verbatim unless asked)

Project file:

```xml
<Project Sdk="Microsoft.Build.Sql/1.0.0">
  <PropertyGroup>
    <Name>StoreCatalog</Name>
    <DSP>Microsoft.Data.Tools.Schema.Sql.SqlAzureV12DatabaseSchemaProvider</DSP>
  </PropertyGroup>
</Project>
```

Existing workflow:

```yaml
- name: Build database project
  run: dotnet build StoreCatalog.sqlproj --configuration Release

- name: Deploy to production
  run: >
    sqlpackage /Action:Publish
    /SourceFile:"bin/Release/StoreCatalog.dacpac"
    /TargetConnectionString:"${{ secrets.PROD_SQL_CONNECTION }}"
```

Option a:

```yaml
- name: Deploy with safety check
  run: >
    sqlpackage /Action:Publish
    /SourceFile:"bin/Release/StoreCatalog.dacpac"
    /TargetConnectionString:"${{ secrets.PROD_SQL_CONNECTION }}"
    /p:BlockOnPossibleDataLoss=True
```

Option b:

```yaml
- name: Detect drift
  run: >
    sqlpackage /Action:DriftReport
    /TargetConnectionString:"${{ secrets.PROD_SQL_CONNECTION }}"
    /OutputPath:"drift-report.xml"
```

Option c:

```yaml
- name: Build database project
  run: dotnet build StoreCatalog.sqlproj --configuration Release

- name: Report pending deployment actions
  run: >
    sqlpackage /Action:DeployReport
    /SourceFile:"bin/Release/StoreCatalog.dacpac"
    /TargetConnectionString:"${{ secrets.PROD_SQL_CONNECTION }}"
    /OutputPath:"deploy-report.xml"
```

Option d:

```yaml
- name: Capture production baseline
  run: >
    sqlpackage /Action:Extract
    /SourceConnectionString:"${{ secrets.PROD_SQL_CONNECTION }}"
    /TargetFile:"bin/Release/StoreCatalog.dacpac"
```

## 4. The question (ask exactly this)

"Which approach should you use? Option a, option b, option c, or option d?"

Options in full:

- **a.** Publish with `/p:BlockOnPossibleDataLoss=True`, expecting it to terminate because the target differs; read the error output; then add the objects to the project.
- **b.** `/Action:DriftReport` against production with an OutputPath, no SourceFile; review; then add the objects to the project.
- **c.** `dotnet build`, then `/Action:DeployReport` with the built dacpac as SourceFile and production as target, OutputPath deploy-report.xml; review the report that shows the index drop and the procedure revert; add the index and the procedure body to the project in a pull request.
- **d.** `/Action:Extract` from production into bin/Release/StoreCatalog.dacpac, overwriting the pipeline artifact so the next publish deploys the hot-fix.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: c.**

- DeployReport takes a source dacpac and a target connection and writes the would-be deployment plan as XML to OutputPath. It is read-only; production is not modified. Requirement one.
- Because the dacpac from main lacks the hot-fix, the report shows exactly the collision: IX_Product_SupplierId exists only in the target, and the publish default DropIndexesNotInSource=True means it would be dropped; usp_GetProductsBySupplier differs, so an ALTER PROCEDURE would revert the rewrite.
- Adding the index and the new procedure body to the project's declarative source through a pull request makes them part of the model, so future builds and publishes no longer revert them. Requirement two.

Why the others are wrong, one line each:

- **a.** BlockOnPossibleDataLoss=True is already the default and blocks only changes that could lose data, such as a narrowing type change; dropping a nonclustered index and altering a procedure lose no data, so the publish proceeds, modifies production and destroys the hot-fix.
- **b.** DriftReport reports changes to a registered database since it was last registered, target-only, no SourceFile; StoreCatalog was never registered, so there is no baseline, and even if it were, it would compare against the registration snapshot, not against the dacpac from main.
- **d.** Extract runs in the wrong direction, from the live database into a dacpac; swapping the artifact produces no report and never updates the .sqlproj source, so the very next merge rebuilds a dacpac without the hot-fix and the following publish reverts it after all.

## 6. Hint ladder (one hint per attempt, in order)

1. "Requirement one asks what the next deployment would do, without modifying production. Which SqlPackage actions modify the target, and which only compare?"
2. "Think about what each action compares. One compares a source dacpac against a target database. Another compares a database against a snapshot taken when it was registered. Which comparison answers what will the next publish change?"
3. "Reread the statement: the database has never been registered as a data-tier application. Which action depends on that registration?"
4. "Option a runs a real Publish and hopes a property stops it. Does dropping an index or altering a procedure lose any data? That eliminates a."
5. "Option d extracts a dacpac from production and swaps it into the pipeline. Where is the report? And what happens at the very next dotnet build on main? That eliminates d."
6. "You are down to b and c. One has a SourceFile and one does not. Which one compares the dacpac built from main against production, and which one needs a registration baseline that does not exist?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "b, the DBA's change is drift, so DriftReport" | Matches the action by name | "What baseline does DriftReport compare against, and does StoreCatalog have one? And which parameter is missing from that command compared to the others?" |
| "a, BlockOnPossibleDataLoss stops any mismatch" | Confuses data loss with schema difference | "Every incremental publish has a mismatch. What exactly does the property block? Does dropping a nonclustered index lose data?" |
| "d, extracting production guarantees nothing is reverted" | Treats the dacpac as the source of truth | "After the swap, what is in the sqlproj in GitHub? What does the next merge to main build from?" |
| "c, but Publish with the report is safer" | Wants to combine report and deploy | "Requirement one says without modifying production. Which single action produces the report and touches nothing?" |
| "c, but the index would not be dropped" | Does not know DropIndexesNotInSource defaults to true | "What is the default value of DropIndexesNotInSource on publish?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the three facts the question tests:

- An SDK-style Microsoft.Build.Sql project is built with dotnet build, and the artifact is a dacpac representing the intended state of the schema.
- SqlPackage Action DeployReport creates an XML report of the changes a publish would make, comparing a source dacpac against a target database, without executing anything against the target.
- Out-of-band changes are only safe once merged back into the project source, because the project, not the live database, is the source of truth for every future build and publish.

Then match each action to its question:

- What would a publish change on this target? DeployReport: source dacpac plus target database, XML report, read-only.
- What changed since this database was registered? DriftReport: registered data-tier application only, target-only parameters, no SourceFile. Useless here because StoreCatalog was never registered.
- Apply the source to the target now. Publish: modifies the target.
- Turn a live database into a dacpac. Extract: capture, not compare.

Then the defaults that make undetected drift dangerous: BlockOnPossibleDataLoss=True blocks data loss only, never schema drift in general; DropIndexesNotInSource=True means a hot-fix index added directly in production is silently dropped by the next publish.

Recovery when production has been changed out-of-band is always the same: detect the difference with a read-only report, then merge the change into the SQL Database Project source through a pull request.

Memory hook: "DeployReport asks what would happen. DriftReport needs a registration. Publish does it. Extract captures it. The project is the truth."

## 9. Follow-up oral questions (optional)

1. "Which publish property would stop the index from being dropped without merging it into the project, and why is that only a stopgap?" (DropIndexesNotInSource=False; it hides one symptom while the project still lacks the index, so the source of truth stays wrong.)
2. "What would DriftReport need in order to be useful on StoreCatalog?" (The database must first be registered as a data-tier application, which stores a snapshot baseline.)
3. "Name a schema change that BlockOnPossibleDataLoss would actually block." (A column narrowing or type change that requires a cast and could lose data, or dropping a column with data.)

## 10. References

- SqlPackage DeployReport and DriftReport: https://learn.microsoft.com/en-us/sql/tools/sqlpackage/sqlpackage-deploy-drift-report
- SqlPackage Publish parameters and properties, including BlockOnPossibleDataLoss and DropIndexesNotInSource: https://learn.microsoft.com/en-us/sql/tools/sqlpackage/sqlpackage-publish
- SqlPackage Extract: https://learn.microsoft.com/en-us/sql/tools/sqlpackage/sqlpackage-extract
- Register a database as a data-tier application: https://learn.microsoft.com/en-us/sql/relational-databases/data-tier-applications/register-a-database-as-a-dac
- SQL Database Projects overview: https://learn.microsoft.com/en-us/sql/tools/sql-database-projects/sql-database-projects
- Microsoft.Build.Sql on GitHub: https://github.com/microsoft/DacFx
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
