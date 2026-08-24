# SQL Server question — SQL Database Projects 1

## Statement

A supermarket chain's development team maintains the schema of the Azure SQL Database `StoreCatalog` as an SDK-style SQL Database Project stored in GitHub.

The project file begins as follows:

```xml
<Project Sdk="Microsoft.Build.Sql/1.0.0">
  <PropertyGroup>
    <Name>StoreCatalog</Name>
    <DSP>Microsoft.Data.Tools.Schema.Sql.SqlAzureV12DatabaseSchemaProvider</DSP>
  </PropertyGroup>
</Project>
```

A GitHub Actions workflow runs on every merge to `main`:

```yaml
- name: Build database project
  run: dotnet build StoreCatalog.sqlproj --configuration Release

- name: Deploy to production
  run: >
    sqlpackage /Action:Publish
    /SourceFile:"bin/Release/StoreCatalog.dacpac"
    /TargetConnectionString:"${{ secrets.PROD_SQL_CONNECTION }}"
```

The publish step runs with default SqlPackage publish properties. The `StoreCatalog` database has **never been registered as a data-tier application**.

During a weekend incident, a DBA connected directly to production and hot-fixed the database:

- Created a nonclustered index `IX_Product_SupplierId` on `dbo.Product`.
- Rewrote the body of the stored procedure `dbo.usp_GetProductsBySupplier`.

Neither change was made in the SQL Database Project. Several unrelated feature branches are about to be merged to `main`.

Before the next deployment runs, the team lead requires that you:

1. Produce a reviewable report of the exact actions the next deployment would perform against the **current** state of production, **without modifying production**.
2. Ensure the hot-fix is preserved by making it part of the project source through the normal pull-request process, so that future deployments do not revert it.

Which approach should you use?

### a.

```yaml
- name: Deploy with safety check
  run: >
    sqlpackage /Action:Publish
    /SourceFile:"bin/Release/StoreCatalog.dacpac"
    /TargetConnectionString:"${{ secrets.PROD_SQL_CONNECTION }}"
    /p:BlockOnPossibleDataLoss=True
```

Rely on `BlockOnPossibleDataLoss=True` to terminate the publish operation because the target schema no longer matches the dacpac. Review the error output to identify the hot-fixed objects, then add them to the project in a pull request.

### b.

```yaml
- name: Detect drift
  run: >
    sqlpackage /Action:DriftReport
    /TargetConnectionString:"${{ secrets.PROD_SQL_CONNECTION }}"
    /OutputPath:"drift-report.xml"
```

Run `DriftReport` against production to generate an XML report of the changes the DBA made, review the report, then add the hot-fixed objects to the project in a pull request.

### c.

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

Build the dacpac from `main` with `dotnet build`, run `DeployReport` with the dacpac as source and production as target, and review the XML report of the actions a publish would take. The report reveals that publish would drop `IX_Product_SupplierId` and alter `dbo.usp_GetProductsBySupplier` back to the old body. Add the index definition and the new procedure body to the project source in a pull request before the next deployment.

### d.

```yaml
- name: Capture production baseline
  run: >
    sqlpackage /Action:Extract
    /SourceConnectionString:"${{ secrets.PROD_SQL_CONNECTION }}"
    /TargetFile:"bin/Release/StoreCatalog.dacpac"
```

Run `Extract` against production and overwrite the pipeline's build artifact `bin/Release/StoreCatalog.dacpac` with the extracted dacpac, so that the next publish step deploys a dacpac that already contains the hot-fix and nothing is reverted.

## Correct Answer

**c**

## Explanation

The correct answer is **c**.

This question tests three facts about SQL Database Projects and SqlPackage:

1. An SDK-style (`Microsoft.Build.Sql`) project is built with `dotnet build`, and the build output artifact is a `.dacpac` file that represents the intended state of the schema.
2. `SqlPackage /Action:DeployReport` creates an **XML report of the changes that would be made by a publish action**, comparing a source (here, the dacpac built from `main`) against a target database — **without executing anything against the target**.
3. Out-of-band changes are only safe once they are merged back into the project source, because the project — not the live database — is the source of truth that every future build and publish is derived from.

### Why option c is correct

`DeployReport` accepts `/SourceFile` (a dacpac) and `/TargetConnectionString` (the production database) and writes the would-be deployment plan to the file given by `/OutputPath`. It is a read-only comparison: production is not modified, which satisfies requirement 1.

Because the dacpac built from `main` does not contain the hot-fix, the report surfaces exactly how the next publish would collide with it:

- `IX_Product_SupplierId` exists only in the target. The publish default `DropIndexesNotInSource=True` means the report shows the index being **dropped**.
- `dbo.usp_GetProductsBySupplier` differs from the source, so the report shows an `ALTER PROCEDURE` that would **revert** the DBA's rewrite.

Reviewing the report tells the team precisely which objects drifted. Adding the index and the new procedure body to the project's declarative T-SQL source through a pull request makes the hot-fix part of the model, so subsequent builds produce a dacpac that already contains it and future publishes no longer revert it. That satisfies requirement 2.

### Why option a is wrong

`BlockOnPossibleDataLoss=True` (which is already the default) does **not** block a publish because the target schema differs from the dacpac — differing is the normal case for every incremental deployment. The property only terminates the operation *"if the resulting schema changes could incur a loss of data"*, such as a data type narrowing that requires a cast.

Dropping a nonclustered index and altering a stored procedure body incur no data loss, so the publish proceeds normally: it silently drops `IX_Product_SupplierId` (default `DropIndexesNotInSource=True`) and reverts `dbo.usp_GetProductsBySupplier`. Production is modified (violating requirement 1) and the hot-fix is destroyed instead of preserved (violating requirement 2).

### Why option b is wrong

The action name sounds exactly right, which is the trap. Per the documentation, `DriftReport` *"creates an XML report of the changes that have been made to the **registered** database since it was **last registered**"* — it compares the live database against the snapshot stored when the database was registered as a data-tier application, and it accepts only target parameters (there is no `/SourceFile`).

The statement says `StoreCatalog` has **never been registered as a data-tier application**, so there is no registration baseline to compare against and `DriftReport` cannot report the DBA's changes. Even in a registered database, `DriftReport` compares production against its registration snapshot, not against the dacpac built from `main`, so it would not show what the *next deployment* would do — which is what requirement 1 asks for.

### Why option d is wrong

`Extract` runs in the wrong direction for this workflow: it builds a dacpac **from** the live database. Overwriting the pipeline artifact with the extracted dacpac makes the next publish an effective no-op, so:

- No report of differences is ever produced or reviewed — requirement 1 is not met.
- The `.sqlproj` T-SQL source in GitHub is never updated. The project remains the source of truth for every future build, so the very next merge to `main` runs `dotnet build` again, produces a dacpac **without** the hot-fix, and the following publish drops the index and reverts the procedure after all — requirement 2 is not met.

Bypassing the pull-request process by swapping binary artifacts inside the pipeline also defeats the purpose of keeping the schema under source control.

## DP-800 Exam Rule to Remember

Match the SqlPackage action to the question being asked:

```text
"What WOULD a publish change on this target?"
    → /Action:DeployReport   (source dacpac + target DB → XML report, read-only)

"What changed since this database was REGISTERED?"
    → /Action:DriftReport    (registered data-tier application only, target-only)

"Apply the source to the target now."
    → /Action:Publish        (modifies the target)

"Turn a live database INTO a dacpac."
    → /Action:Extract        (capture, not compare)
```

And remember the defaults that make undetected drift dangerous:

- `BlockOnPossibleDataLoss=True` blocks **data loss only**, never schema drift in general.
- `DropIndexesNotInSource=True` means a hot-fix index added directly in production is silently dropped by the next publish.

When production has been changed out-of-band, the recovery is always the same: **detect** the difference with a read-only report, then **merge the change into the SQL Database Project source** — the project, not the database, is the source of truth in CI/CD.
