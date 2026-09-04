# Instructor-Examiner guide — SQL Projects Build 1

Companion to [sql_projects_build_1.md](sql_projects_build_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a, b, c and d. Read the project file, the two objects, the build errors and all four options before taking an answer. The options are XML fragments of the project file; describe the element names and attribute values precisely in words. This is a conceptual tooling question; nothing was executed against an engine.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Implement CI/CD by using SQL Database Projects.
- Task bullet: Build a SQL project; manage database references; manage source control for SQL projects.
- What is tested: why SQL71501 unresolved reference is a build error, how package references fix system-database and cross-database references with a SQLCMD variable, and what belongs in git.

## 2. Scenario to read aloud

**Piece 1, the story.** "A charity manages grants in a SQL Server 2025 database. A developer has just created an SDK-style SQL Database Project named GrantTracker in Visual Studio Code, committed it to a new GitHub repository, and wired up a GitHub Actions workflow whose first step is dotnet build GrantTracker dot sqlproj."

**Piece 2, the project file.** "The project file is short. A Project element with DefaultTargets Build. Inside it an Sdk element with Name Microsoft dot Build dot Sql and Version one point zero point zero. Then a PropertyGroup with three properties: Name is GrantTracker; DSP, the target platform, is Microsoft dot Data dot Tools dot Schema dot Sql dot Sql one seventy DatabaseSchemaProvider, meaning SQL Server 2025; and ModelCollation is ten thirty three comma C I."

**Piece 3, the first object.** "Two objects matter. The first is in the file Ops slash Stored Procedures slash usp underscore NotifyGrantOfficer dot sql. It creates the procedure Ops dot usp underscore NotifyGrantOfficer with one parameter, at GrantId, an integer. It builds a message body with CONCAT and then executes msdb dot dbo dot sp underscore send underscore dbmail with profile name GrantMail, recipients officers at example dot org, a subject Grant review, and the body."

**Piece 4, the second object.** "The second is in Rep slash Views slash vw underscore DonorSummary dot sql. A comment says Crm dot Donors lives in the DonorHub database on the same server. It creates the view Rep dot vw underscore DonorSummary that selects GrantId, Amount and DonorName from Grants dot Grant aliased g, joined to a three-part name: square brackets dollar open paren DonorHub close paren, dot Crm dot Donors, aliased d, on d dot DonorId equals g dot DonorId. So the database part of the name is a SQLCMD variable called DonorHub."

**Piece 5, the build output.** "The build fails. There are no warnings, only errors, and they look like this. In the procedure file, line five: Build error SQL seven one five zero one, Procedure Ops dot usp underscore NotifyGrantOfficer has an unresolved reference to object msdb dot dbo dot sp underscore send underscore dbmail. In the view file, line six: Build error SQL seven one five zero one, View Rep dot vw underscore DonorSummary has an unresolved reference to object dollar DonorHub dot Crm dot Donors. Build FAILED."

**Piece 6, the DonorHub package and the goal.** "The DonorHub team publishes their own SQL project as the NuGet package Charity dot DonorHub, version three point two point zero, to the organization's Azure Artifacts feed, and the build agent already has that feed as a package source. You must make dotnet build succeed and produce a valid GrantTracker dot dacpac, keep the model validated with no suppressed errors, and keep the repository clean, with no build outputs under source control."

**Piece 7, option a.** "Option a. Add an ItemGroup with two PackageReference items. The first has Include Microsoft dot SqlServer dot Dacpacs dot Msdb and Version one seventy point zero point three. The second has Include Charity dot DonorHub, Version three point two point zero, and a child element DatabaseSqlCmdVariable with the value DonorHub. Then a second ItemGroup with a SqlCmdVariable item, Include DonorHub, with a DefaultValue of DonorHub and a Value of dollar SqlCmdVar underscore underscore one. And a dot gitignore file with two lines: bin slash and obj slash."

**Piece 8, option b.** "Option b. Build the DonorHub project locally and commit DonorHub dot dacpac, together with the msdb dot dacpac shipped with Visual Studio, into a lib folder of the GrantTracker repository. Reference them with an ItemGroup of two ArtifactReference items: Include lib backslash msdb dot dacpac, and Include lib backslash DonorHub dot dacpac with a child DatabaseSqlCmdVariable of DonorHub. Also commit bin slash Debug slash GrantTracker dot dacpac so the deployment job can pick it up without rebuilding."

**Piece 9, option c.** "Option c. Keep the project as is and stop the two diagnostics from failing the build. Add a PropertyGroup with SuppressTSqlWarnings set to seven one five zero one, and TreatTSqlWarningsAsErrors set to false."

**Piece 10, option d.** "Option d. Switch the target platform to Azure SQL Database, which has no msdb, so the mail reference is no longer validated. That means DSP becomes Microsoft dot Data dot Tools dot Schema dot Sql dot SqlAzureV12DatabaseSchemaProvider. And replace the SQLCMD variable in the view with the real database name, so the join reads DonorHub dot Crm dot Donors, so the view resolves on the server."

## 3. Setup script (reference only; do not read verbatim unless asked)

Project file:

```xml
<Project DefaultTargets="Build">
  <Sdk Name="Microsoft.Build.Sql" Version="1.0.0" />
  <PropertyGroup>
    <Name>GrantTracker</Name>
    <DSP>Microsoft.Data.Tools.Schema.Sql.Sql170DatabaseSchemaProvider</DSP>
    <ModelCollation>1033, CI</ModelCollation>
  </PropertyGroup>
</Project>
```

Objects:

```sql
-- Ops/Stored Procedures/usp_NotifyGrantOfficer.sql
CREATE PROCEDURE Ops.usp_NotifyGrantOfficer @GrantId INT
AS
DECLARE @body NVARCHAR(400) = CONCAT(N'Grant ', @GrantId, N' needs review.');
EXEC msdb.dbo.sp_send_dbmail @profile_name = 'GrantMail',
                             @recipients   = 'officers@example.org',
                             @subject      = N'Grant review',
                             @body         = @body;
GO
-- Rep/Views/vw_DonorSummary.sql   (Crm.Donors lives in the DonorHub database on the same server)
CREATE VIEW Rep.vw_DonorSummary
AS
SELECT g.GrantId, g.Amount, d.DonorName
FROM Grants.Grant AS g
JOIN [$(DonorHub)].Crm.Donors AS d ON d.DonorId = g.DonorId;
GO
```

Build output:

```text
Ops/Stored Procedures/usp_NotifyGrantOfficer.sql(5,6): Build error SQL71501:
   Procedure: [Ops].[usp_NotifyGrantOfficer] has an unresolved reference to object [msdb].[dbo].[sp_send_dbmail].
Rep/Views/vw_DonorSummary.sql(6,6): Build error SQL71501:
   View: [Rep].[vw_DonorSummary] has an unresolved reference to object [$(DonorHub)].[Crm].[Donors].
Build FAILED.
```

Option a:

```xml
  <ItemGroup>
    <PackageReference Include="Microsoft.SqlServer.Dacpacs.Msdb" Version="170.0.3" />
    <PackageReference Include="Charity.DonorHub" Version="3.2.0">
      <DatabaseSqlCmdVariable>DonorHub</DatabaseSqlCmdVariable>
    </PackageReference>
  </ItemGroup>
  <ItemGroup>
    <SqlCmdVariable Include="DonorHub">
      <DefaultValue>DonorHub</DefaultValue>
      <Value>$(SqlCmdVar__1)</Value>
    </SqlCmdVariable>
  </ItemGroup>
```

```gitignore
# .gitignore
bin/
obj/
```

Option b:

```xml
  <ItemGroup>
    <ArtifactReference Include="lib\msdb.dacpac" />
    <ArtifactReference Include="lib\DonorHub.dacpac">
      <DatabaseSqlCmdVariable>DonorHub</DatabaseSqlCmdVariable>
    </ArtifactReference>
  </ItemGroup>
```

Option c:

```xml
  <PropertyGroup>
    <SuppressTSqlWarnings>71501</SuppressTSqlWarnings>
    <TreatTSqlWarningsAsErrors>false</TreatTSqlWarningsAsErrors>
  </PropertyGroup>
```

Option d:

```xml
    <DSP>Microsoft.Data.Tools.Schema.Sql.SqlAzureV12DatabaseSchemaProvider</DSP>
```

```sql
JOIN DonorHub.Crm.Donors AS d ON d.DonorId = g.DonorId;
```

## 4. The question (ask exactly this)

"You must make dotnet build succeed and produce a valid GrantTracker dot dacpac, keep the model validated with no suppressed errors, and keep the repository clean with no build outputs under source control. Which set of changes should you make? Option a, option b, option c, or option d?"

Options in full:

- **a.** PackageReference to Microsoft.SqlServer.Dacpacs.Msdb 170.0.3; PackageReference to Charity.DonorHub 3.2.0 with DatabaseSqlCmdVariable DonorHub; a SqlCmdVariable item DonorHub with DefaultValue DonorHub; .gitignore with bin/ and obj/.
- **b.** Commit msdb.dacpac and DonorHub.dacpac under lib/, reference them with ArtifactReference items, and commit bin/Debug/GrantTracker.dacpac.
- **c.** SuppressTSqlWarnings 71501 and TreatTSqlWarningsAsErrors false.
- **d.** DSP changed to SqlAzureV12DatabaseSchemaProvider and the view rewritten with the literal DonorHub.Crm.Donors.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

- SQL71501 is raised when a reference cannot be resolved; the model does not know msdb or another database until told through database references.
- Microsoft publishes the system databases as NuGet packages: Microsoft.SqlServer.Dacpacs.Master, Microsoft.SqlServer.Dacpacs.Msdb, and Microsoft.SqlServer.Dacpacs.Azure.Master, versioned by SQL Server major version, 170.x for SQL Server 2025 matching Sql170. A PackageReference makes sp_send_dbmail part of the model.
- A PackageReference to Charity.DonorHub with DatabaseSqlCmdVariable DonorHub says the package's objects live in a different database on the same server, named by the variable. The SqlCmdVariable item declares it with a default; at deploy time sqlpackage /v:DonorHub=... or the publish profile sets it per environment. A different server would add ServerSqlCmdVariable; the same database uses neither.
- bin/ and obj/ are build outputs regenerated by CI, so they are ignored. The repository holds the sqlproj, the .sql files, which SDK-style globbing includes automatically, pre and post-deployment scripts, and publish profiles without credentials.

Why the others are wrong, one line each:

- **b.** Commits build artifacts, binary and stale, into git, including the project's own output; ArtifactReference is the legacy form not recommended for SDK-style projects; and it pins msdb to whatever Visual Studio shipped instead of the versioned package.
- **c.** SQL71501 is an error, not a warning, so neither property applies and the build still fails; and suppressing reference validation defeats the point of a project build. The requirement says no suppressed errors.
- **d.** Changing DSP does not remove the unresolved msdb reference, and Azure SQL Database has no Database Mail anyway; the literal three-part name is still unresolved without a reference and loses per-environment naming; and the dacpac now targets Azure, so deploying to SQL Server 2025 is refused as a platform mismatch unless AllowIncompatiblePlatform is passed.

## 6. Hint ladder (one hint per attempt, in order)

1. "Look at the word before the number in the build output. Does it say warning or error? What kind of project property could apply to it?"
2. "A project build validates every object reference against its model. What does the model need in order to know msdb dot dbo dot sp underscore send underscore dbmail exists?"
3. "The DonorHub team already publishes a NuGet package, and the agent has the feed. What kind of reference restores at build time without any file in the repository?"
4. "Option c tries to suppress a warning numbered seven one five zero one. Is SQL seven one five zero one a warning? That eliminates c."
5. "Option d changes the platform to Azure SQL Database. Does the procedure still call msdb? And can the Azure dacpac deploy to SQL Server 2025 without a mismatch? That eliminates d."
6. "You are down to a and b. Both add references and both use DatabaseSqlCmdVariable. One puts dacpac files, including the project's own output, into git. The other uses package references and a gitignore. Which one keeps the repository clean?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "c, the properties are real and they silence the diagnostic" | Confuses warnings with errors | "The properties are real. Read the build output again: is SQL seven one five zero one labelled warning or error?" |
| "d, Azure has no msdb so the reference is not validated" | Thinks the platform switch removes the reference | "The procedure text still calls msdb. Does the model now resolve it, or is it still unresolved? And what does the target SQL Server 2025 say to an Azure dacpac?" |
| "b, committing the dacpac makes deployment simpler" | Puts build outputs under source control | "What happens to a committed dacpac the moment someone edits a dot sql file? Is it reviewable? Is it in sync?" |
| "b, ArtifactReference is fine in SDK-style projects" | Legacy reference form | "What does the documentation say about ArtifactReference for new SDK-style development, and what must exist on every build agent for it to work?" |
| "a, but the view should use a literal database name" | Misses per-environment naming | "How would the same view point at DonorHub underscore Test in the test environment?" |
| "a, but msdb should be Master" | Wrong system database | "Where does sp underscore send underscore dbmail live?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain what a build does:

- An SDK-style project uses the Sdk element Microsoft.Build.Sql, or the Project Sdk attribute form. DSP is the target platform: Sql160 for 2022, Sql170 for 2025, SqlAzureV12 for Azure SQL Database, SqlDbFabric and others. Default globbing builds every .sql file in the folder; the output goes to bin, configuration, Name.dacpac.
- The build validates syntax for the platform and every object reference. An unresolved reference is SQL71501, an error. Warnings from code analysis or casing do not fail the build; errors cannot be suppressed away. That is why option c fails.

Then database references, the fix for unresolved references:

- System database: PackageReference to Microsoft.SqlServer.Dacpacs.Master, Msdb or Azure.Master, with a version matching the platform major, such as 170.x.
- Same database, a split project: a reference with no variables; deploy with IncludeCompositeObjects true.
- Different database, same server: DatabaseSqlCmdVariable plus a SqlCmdVariable item, and the object is written as square brackets dollar Var, dot schema dot object.
- Different database, different server: add ServerSqlCmdVariable, and the object is written with both variables.
- Prefer PackageReference or ProjectReference. ArtifactReference with a dacpac path is the legacy form. That is why option b fails on form, and it also fails on hygiene.
- DatabaseVariableLiteralValue exists for the rare case where a literal database name is wanted in a reference, set on the reference, not by editing the view. That is not what option d does.

Then git hygiene: commit the sqlproj, the .sql files, pre and post-deploy scripts and publish profiles without secrets; ignore bin, obj and dacpac files. To reverse-engineer an existing database into a project, use sqlpackage Action Extract with ExtractTarget SqlProject.

Memory hook: "Seven one five zero one is an error, so reference it, do not silence it. Package reference plus DatabaseSqlCmdVariable. Ignore bin and obj."

## 9. Follow-up oral questions (optional)

1. "How would you set the DonorHub variable at deployment time with sqlpackage?" (With the slash v argument, for example /v:DonorHub=DonorHub, or in the publish profile.)
2. "Which package would you reference instead of Msdb if the target were Azure SQL Database and the object lived in master?" (Microsoft.SqlServer.Dacpacs.Azure.Master.)
3. "If DonorHub lived on a different server, which extra element would the reference need, and how would the object be written in the view?" (ServerSqlCmdVariable; square brackets dollar server variable, dot square brackets dollar database variable, dot schema dot object.)

## 10. References

- SQL Database Projects overview, SDK-style: https://learn.microsoft.com/en-us/sql/tools/sql-database-projects/sql-database-projects
- Database references in SQL projects: https://learn.microsoft.com/en-us/sql/tools/sql-database-projects/concepts/database-references
- Build a SQL project: https://learn.microsoft.com/en-us/sql/tools/sql-database-projects/howto/build-database-project
- Microsoft.Build.Sql on GitHub: https://github.com/microsoft/DacFx
- Microsoft.SqlServer.Dacpacs.Msdb package: https://www.nuget.org/packages/Microsoft.SqlServer.Dacpacs.Msdb
- SqlPackage Publish parameters, including SQLCMD variables: https://learn.microsoft.com/en-us/sql/tools/sqlpackage/sqlpackage-publish
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
