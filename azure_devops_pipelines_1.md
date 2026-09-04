# SQL Server question — Azure DevOps Pipelines 1

## Statement

A harbour authority keeps the schema of its berth-booking Azure SQL Database `MarinaBerth` in an SDK-style SQL Database Project (`MarinaBerth/MarinaBerth.sqlproj`) in an **Azure Repos** Git repository, and deploys it with **Azure Pipelines** (YAML). There are two databases on the logical server `marinaberth-sql.database.windows.net`: `MarinaBerth_Staging` and `MarinaBerth`. Two DBAs form the Azure DevOps group `[Harbour]\DBA Team`.

The release policy is:

1. Every pull request into `main`, and every push to `main`, must **build** the project and publish the `.dacpac` as a pipeline artifact.
2. Deployments must authenticate to Azure and to the databases through the pipeline's **service connection** with **no stored secret** — no SQL password, no client secret, nothing that expires.
3. Production is deployed **only from `main`**, **only after a member of `DBA Team` approves**, and **only between 20:00 and 06:00**. A developer who can edit the YAML file must **not** be able to remove or weaken these three controls.
4. A pull request into `main` requires at least **two** reviewers, approval votes must be **reset when new commits are pushed**, the build must succeed before completion, and any change under `MarinaBerth/Tables/` must be approved by `DBA Team`.
5. A deployment that could **lose data** must fail.

Which configuration implements the policy?

### a.

Pipeline:

```yaml
trigger: { branches: { include: [ main ] } }
pr:      { branches: { include: [ main ] } }
stages:
- stage: Build
  jobs:
  - job: build
    pool: { vmImage: windows-latest }
    steps:
    - task: DotNetCoreCLI@2
      inputs: { command: build, projects: MarinaBerth/MarinaBerth.sqlproj, arguments: '-c Release' }
    - task: PublishPipelineArtifact@1
      inputs: { targetPath: MarinaBerth/bin/Release/MarinaBerth.dacpac, artifact: dacpac }
- stage: Staging
  dependsOn: Build
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: deploy_staging
    environment: marinaberth-staging
    pool: { vmImage: windows-latest }
    strategy:
      runOnce:
        deploy:
          steps:
          - download: current
            artifact: dacpac
          - task: SqlAzureDacpacDeployment@1
            inputs:
              azureSubscription: sc-marinaberth          # ARM service connection, workload identity federation
              AuthenticationType: servicePrincipal
              ServerName: marinaberth-sql.database.windows.net
              DatabaseName: MarinaBerth_Staging
              deployType: DacpacTask
              DeploymentAction: Publish
              DacpacFile: $(Pipeline.Workspace)/dacpac/MarinaBerth.dacpac
              AdditionalArguments: /p:BlockOnPossibleDataLoss=True
- stage: Production
  dependsOn: Staging
  jobs:
  - deployment: deploy_production
    environment: marinaberth-production
    pool: { vmImage: windows-latest }
    strategy:
      runOnce:
        deploy:
          steps:
          - download: current
            artifact: dacpac
          - task: SqlAzureDacpacDeployment@1
            inputs:
              azureSubscription: sc-marinaberth
              AuthenticationType: servicePrincipal
              ServerName: marinaberth-sql.database.windows.net
              DatabaseName: MarinaBerth
              deployType: DacpacTask
              DeploymentAction: Publish
              DacpacFile: $(Pipeline.Workspace)/dacpac/MarinaBerth.dacpac
              AdditionalArguments: /p:BlockOnPossibleDataLoss=True
```

Service connection `sc-marinaberth`: Azure Resource Manager, **App registration (automatic)** with credential **Workload identity federation**; its identity is created in each database with `CREATE USER [sc-marinaberth] FROM EXTERNAL PROVIDER` and added to `db_owner`.

Environment `marinaberth-production` → **Approvals and checks**: **Approvals** (`DBA Team`), **Branch control** (allowed branches `refs/heads/main`, *Verify branch protection* on), **Business hours** (20:00–06:00).

Branch policies on `main`: **Require a minimum number of reviewers** = 2 with *When new changes are pushed → Reset all approval votes*; **Build validation** on the pipeline, trigger *Automatic*, policy requirement *Required*; **Automatically include reviewers** → `DBA Team`, *Required*, path filter `/MarinaBerth/Tables/*`.

### b.

Same branch policies as option a. The deployment stages use `SqlDacpacDeploymentOnMachineGroup@0` with `TargetMethod: server`, `AuthScheme: sqlServerAuthentication`, `SqlUsername: marina_deploy` and `SqlPassword: $(MarinaSqlPassword)` (a secret pipeline variable), and the production stage carries `condition: eq(variables['approvedByDba'], 'true')` where a DBA sets the variable at queue time; a `schedules:` cron trigger at 21:00 runs the deployment inside the window.

### c.

Same pipeline, service connection and branch policies as option a, except that production gating is done in YAML: the production stage starts with an agentless job running `ManualValidation@1` (`notifyUsers: [Harbour]\DBA Team`, `onTimeout: reject`), the stage `condition` checks `Build.SourceBranch` for `refs/heads/main`, and the business-hours window is implemented with a `schedules:` cron trigger at 22:00. The `marinaberth-production` environment has no checks.

### d.

Same pipeline as option a, but with `AuthenticationType: aadAuthenticationIntegrated` on the hosted `windows-latest` agent and `DeploymentAction: DeployReport` in the production stage "so the DBAs can review it". Reviewer requirements are expressed with a `CODEOWNERS` file at the repository root (`/MarinaBerth/Tables/  @DBA-Team`), and the production approval is written into the YAML as `environment: { name: marinaberth-production, approvals: [ DBA Team ] }`.

## Correct Answer

**a**

## Explanation

The policy maps onto four Azure DevOps control surfaces, and only option a uses each one for what it does: the **YAML** builds and deploys, the **service connection** authenticates, **environment checks** gate the production stage outside the YAML, and **branch policies** gate the merge.

### Why option a is correct

- **Rule 1 — build on PR and on `main`.** `trigger`/`pr` filters on `main`; `DotNetCoreCLI@2` with `command: build` builds the SDK-style project (`dotnet build` restores the `Microsoft.Build.Sql` SDK and produces `bin/Release/MarinaBerth.dacpac`); `PublishPipelineArtifact@1` publishes it as the `dacpac` artifact, and the deployment jobs `download: current` it, so every environment deploys the *same* artifact that was built. The `condition` on the Staging stage keeps pull-request runs at the build stage.
- **Rule 2 — no stored secret.** The recommended Azure Resource Manager service connection uses **workload identity federation** ("eliminates the need for secrets and secret management"); the automatic option creates the app registration and federated credential for you (a user-assigned managed identity is the alternative when you cannot create app registrations). `SqlAzureDacpacDeployment@1` with `AuthenticationType: servicePrincipal` then connects to the database as that identity — no `SqlUsername`/`SqlPassword`, no connection string — provided the identity exists as a contained user (`FROM EXTERNAL PROVIDER`) with the needed rights. The task adds a firewall rule for the agent's IP (`IpDetectionMethod: AutoDetect`, default) and removes it afterwards (`DeleteFirewallRule: true`, default). Note that this task runs **only on Windows agents** (demand `sqlpackage`), hence `windows-latest`.
- **Rule 3 — controls the YAML author cannot weaken.** "Approvals and other checks aren't defined in the yaml file. Users modifying the pipeline yaml file can't modify the checks performed before start of a stage." They are configured by the resource owner on the **environment** (`marinaberth-production`) that the deployment job references: **Approvals** (a group counts as one approval from any member; optional timeout, deferred approvals), **Branch control** (allowed branches must be fully qualified, `refs/heads/main`; *Verify branch protection* fails the check if `main` has no policy), and **Business hours** (the stage waits for the window; if it cannot start inside it the check is re-evaluated the next day and eventually times out). Checks run in a fixed order — static checks (branch control, required template) first, then approvals, then dynamic checks (business hours, Azure Function, REST, Azure Monitor) — and *all* checks on *all* resources used by the stage must pass. They can also be placed on the service connection itself (for example a **Required template** check).
- **Rule 4 — merge gates.** Azure Repos branch policies: *Require a minimum number of reviewers* (2) with the *Reset all approval votes* option under "When new changes are pushed"; *Build validation* linked to this pipeline (Automatic trigger, Required) so the PR cannot complete until the build passes; *Automatically include reviewers* with `DBA Team`, marked **Required**, restricted with a path filter — the Azure Repos equivalent of GitHub's `CODEOWNERS` + "Require review from Code Owners". Once any required policy exists on `main`, direct pushes are blocked and all changes go through pull requests.
- **Rule 5 — data loss.** `/p:BlockOnPossibleDataLoss=True` in `AdditionalArguments` (the SqlPackage default, stated explicitly) makes `Publish` stop during validation when the change could lose data; `AdditionalArguments` override a publish profile if one is also supplied.

### Why option b is wrong

`SqlDacpacDeploymentOnMachineGroup@0` is the **SQL Server database deploy** task: it targets SQL Server instances on machines in a **deployment group** (on-premises or IaaS VMs), uses Windows or SQL authentication, and is "supported on classic release pipelines only" — it cannot be used in a YAML pipeline at all, let alone against Azure SQL Database through a service connection. A SQL password in a secret variable is a stored, expiring secret (rule 2). A queue-time variable is not an approval: anyone who can queue the run can set `approvedByDba`, and a cron `schedules:` trigger is not a business-hours gate — it deploys whatever is on the branch at 21:00 without waiting for anyone (rule 3).

### Why option c is wrong

This is the subtle distractor: `ManualValidation@1` really does pause a pipeline and wait for a person, and the branch condition and cron trigger look like the other two controls. The problem is *where* they live. All three are in the YAML, so a developer with write access to the repository can delete the validation step, loosen the condition or change the schedule in the same pull request that carries the schema change — exactly what rule 3 forbids. Environment checks exist so that "administrators of resources manage checks using the web interface", independently of the pipeline definition. A schedule is also not a window: it fires once at 22:00 regardless of whether the approval happened, while the **Business hours** check holds an approved stage until the window opens.

### Why option d is wrong

- `aadAuthenticationIntegrated` connects as the agent's Active Directory account; the task documentation states that "Azure AD integrated authentication is not supported for hosted agents" — it requires a private agent joined to the directory.
- `DeploymentAction: DeployReport` generates the XML report of what a publish *would* do (the task exposes it as `SqlDeploymentOutputFile`); it changes nothing, so production is never deployed.
- `CODEOWNERS` is a **GitHub** convention; Azure Repos does not read it. The equivalent is the *Automatically include reviewers* policy with a path filter.
- Environments have no `approvals:` key in YAML; approvals are checks configured on the environment resource, precisely so that they cannot be edited from the pipeline file.

Conceptual question (Azure / tooling); not executed against an engine.

## DP-800 Exam Rule to Remember

```text
GitHub construct                         Azure DevOps equivalent
  dotnet build step                        DotNetCoreCLI@2 (command: build) + PublishPipelineArtifact@1
  azure/login (OIDC) + azure/sql-action    ARM service connection with WORKLOAD IDENTITY FEDERATION (no secret)
                                           + SqlAzureDacpacDeployment@1 (AuthenticationType: servicePrincipal,
                                             deployType: DacpacTask, DeploymentAction: Publish|Script|DeployReport|
                                             DriftReport|Extract|Export|Import, AdditionalArguments: /p:...;
                                             Windows agent only; auto firewall rule)
  environment required reviewers           Environment -> Approvals and checks: Approvals, Branch control
  (branch-limited, wait timer)               (refs/heads/main, verify protection), Business hours, Required template,
                                             Invoke Azure Function / REST API, Exclusive lock -> NOT in YAML
  branch protection + CODEOWNERS           Branch policies: minimum reviewers (reset votes on push), build validation
                                             (required/automatic), comment resolution, linked work items,
                                             Automatically include reviewers (Required + path filter), status checks
  SQL Server on a VM / deployment group    SqlDacpacDeploymentOnMachineGroup@0 (classic release only)
```

Whenever a requirement says a control "must not be editable by whoever edits the pipeline", the answer is a **check on the environment or service connection**, never a task, condition or schedule in the YAML.
