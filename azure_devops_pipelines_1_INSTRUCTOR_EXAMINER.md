# Instructor-Examiner guide — Azure DevOps Pipelines 1

Companion to [azure_devops_pipelines_1.md](azure_devops_pipelines_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**This question.** It is a multiple-choice question with four options, a to d, and only one is correct. Read all five policy rules and all four options before taking an answer. Option a contains a long YAML pipeline; summarise it as written in section 2, name the task names and input values that matter, and say "I can read any line on request". Options b, c and d are described as differences from option a. This is a conceptual Azure DevOps question: nothing is executed.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Manage database solutions with CI/CD (20–25%).
- Skill: Deploy database solutions.
- Task bullet: Implement CI/CD for SQL Database Projects with Azure Pipelines, including service connections, environment checks and branch policies.
- What is tested: which Azure DevOps surface owns each control, namely the YAML for build and deploy, the service connection with workload identity federation for secretless authentication, environment checks for approvals, branch control and business hours that a YAML author cannot remove, and Azure Repos branch policies for the pull request gates; plus the SqlAzureDacpacDeployment task inputs and BlockOnPossibleDataLoss.

## 2. Scenario to read aloud

**Piece 1, the story.** "A harbour authority keeps the schema of its berth-booking Azure SQL Database, MarinaBerth, in an SDK-style SQL Database Project. The project file is MarinaBerth slash MarinaBerth dot sqlproj, in an Azure Repos Git repository, and it deploys with Azure Pipelines in YAML. The logical server marinaberth dash sql dot database dot windows dot net has two databases: MarinaBerth underscore Staging and MarinaBerth. Two DBAs form the Azure DevOps group Harbour backslash DBA Team."

**Piece 2, the policy, rules one and two.** "The release policy has five rules. Rule one: every pull request into main, and every push to main, must build the project and publish the dacpac as a pipeline artifact. Rule two: deployments must authenticate to Azure and to the databases through the pipeline's service connection with no stored secret. No SQL password, no client secret, nothing that expires."

**Piece 3, the policy, rules three to five.** "Rule three: production is deployed only from main, only after a member of DBA Team approves, and only between eight in the evening and six in the morning. A developer who can edit the YAML file must not be able to remove or weaken these three controls. Rule four: a pull request into main requires at least two reviewers, approval votes must be reset when new commits are pushed, the build must succeed before completion, and any change under MarinaBerth slash Tables must be approved by DBA Team. Rule five: a deployment that could lose data must fail. Which configuration implements the policy?"

**Piece 4, option a, the pipeline.** "Option a. The YAML pipeline has a trigger on branch main and a pr trigger on branch main. Three stages. Stage Build: a job on the windows dash latest image with two steps. A DotNetCoreCLI at 2 task with command build, projects MarinaBerth slash MarinaBerth dot sqlproj, arguments dash c Release. Then a PublishPipelineArtifact at 1 task whose targetPath is MarinaBerth slash bin slash Release slash MarinaBerth dot dacpac and whose artifact name is dacpac. Stage Staging depends on Build and has a condition: succeeded, and Build dot SourceBranch equals refs slash heads slash main. Stage Production depends on Staging. I can read any line on request."

**Piece 5, option a, the deployment jobs.** "Staging and Production are deployment jobs. Staging references the environment marinaberth dash staging, Production references marinaberth dash production. Both use the runOnce strategy, download the current run's artifact called dacpac, and run a SqlAzureDacpacDeployment at 1 task. Its inputs: azureSubscription sc dash marinaberth, which is an Azure Resource Manager service connection with workload identity federation. AuthenticationType servicePrincipal. ServerName marinaberth dash sql dot database dot windows dot net. DatabaseName MarinaBerth underscore Staging or MarinaBerth. deployType DacpacTask. DeploymentAction Publish. DacpacFile is Pipeline dot Workspace slash dacpac slash MarinaBerth dot dacpac. And AdditionalArguments slash p colon BlockOnPossibleDataLoss equals True."

**Piece 6, option a, the surrounding configuration.** "The service connection sc dash marinaberth is Azure Resource Manager, App registration automatic, credential Workload identity federation. Its identity is created in each database with CREATE USER sc dash marinaberth FROM EXTERNAL PROVIDER and added to db underscore owner. On the environment marinaberth dash production, under Approvals and checks, three checks: Approvals by DBA Team; Branch control with allowed branches refs slash heads slash main and Verify branch protection on; and Business hours from eight in the evening to six in the morning. Branch policies on main: Require a minimum number of reviewers equals two, with When new changes are pushed set to Reset all approval votes; Build validation on this pipeline, trigger Automatic, requirement Required; and Automatically include reviewers with DBA Team, Required, path filter slash MarinaBerth slash Tables slash star."

**Piece 7, option b.** "Option b. Same branch policies as option a. But the deployment stages use the task SqlDacpacDeploymentOnMachineGroup at 0, with TargetMethod server, AuthScheme sqlServerAuthentication, SqlUsername marina underscore deploy and SqlPassword taken from a secret pipeline variable called MarinaSqlPassword. The production stage carries a condition that the variable approvedByDba equals true, which a DBA sets at queue time. And a schedules cron trigger at nine in the evening runs the deployment inside the window."

**Piece 8, option c.** "Option c. Same pipeline, service connection and branch policies as option a, except that production gating is done in YAML. The production stage starts with an agentless job running ManualValidation at 1, with notifyUsers set to DBA Team and onTimeout reject. The stage condition checks Build dot SourceBranch for refs slash heads slash main. The business-hours window is implemented with a schedules cron trigger at ten in the evening. The marinaberth dash production environment has no checks."

**Piece 9, option d.** "Option d. Same pipeline as option a, but with AuthenticationType aadAuthenticationIntegrated on the hosted windows dash latest agent, and DeploymentAction DeployReport in the production stage, so the DBAs can review it. Reviewer requirements are expressed with a CODEOWNERS file at the repository root, containing the line slash MarinaBerth slash Tables slash, at DBA dash Team. And the production approval is written into the YAML as environment, name marinaberth dash production, approvals, DBA Team."

## 3. Setup script (reference only; do not read verbatim unless asked)

Option a pipeline:

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

Option a, database user for the service connection identity:

```sql
CREATE USER [sc-marinaberth] FROM EXTERNAL PROVIDER;
ALTER ROLE db_owner ADD MEMBER [sc-marinaberth];
```

Option d, CODEOWNERS line: `/MarinaBerth/Tables/  @DBA-Team`. Option d, YAML approval: `environment: { name: marinaberth-production, approvals: [ DBA Team ] }`.

Summary of the differences:

| Control | a | b | c | d |
|---|---|---|---|---|
| Deploy task | SqlAzureDacpacDeployment@1, servicePrincipal | SqlDacpacDeploymentOnMachineGroup@0, SQL login and password | as a | SqlAzureDacpacDeployment@1, aadAuthenticationIntegrated, DeployReport |
| Approval | Environment check | Queue-time variable | ManualValidation@1 in YAML | approvals key in YAML |
| Branch control | Environment check | none | YAML condition | as a |
| Window | Environment Business hours check | cron schedule 21:00 | cron schedule 22:00 | as a |
| PR reviewers | Branch policies | Branch policies | Branch policies | CODEOWNERS file |

## 4. The question (ask exactly this)

"Which configuration implements the policy? Option a, option b, option c, or option d?"

If the learner wants a reminder, re-read any option piece from section 2.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

- Rule 1: trigger and pr on main, DotNetCoreCLI build of the sqlproj, PublishPipelineArtifact named dacpac, and every deployment job downloads that same artifact. The Staging condition keeps pull-request runs at the Build stage.
- Rule 2: an ARM service connection with workload identity federation has no secret; SqlAzureDacpacDeployment with AuthenticationType servicePrincipal connects as that identity, which exists in the database as a contained user FROM EXTERNAL PROVIDER. The task runs only on Windows agents and manages the firewall rule itself.
- Rule 3: Approvals, Branch control with refs slash heads slash main, and Business hours are checks on the environment resource, configured in the web interface, outside the YAML; a YAML author cannot modify them.
- Rule 4: branch policies on main: minimum two reviewers with reset votes on push, build validation Required and Automatic, Automatically include reviewers DBA Team as Required with a path filter.
- Rule 5: slash p colon BlockOnPossibleDataLoss equals True in AdditionalArguments.
- **b is wrong:** SqlDacpacDeploymentOnMachineGroup targets deployment-group machines, is supported in classic release pipelines only, and uses a SQL password stored as a secret; a queue-time variable is not an approval; a cron schedule is not a business-hours gate.
- **c is wrong:** ManualValidation, the branch condition and the schedule all live in the YAML, so a developer can remove them in the same pull request; a schedule fires once at a fixed time regardless of approval.
- **d is wrong:** aadAuthenticationIntegrated is not supported on hosted agents; DeployReport only produces a report and never deploys; CODEOWNERS is a GitHub convention that Azure Repos does not read; environments have no approvals key in YAML.

## 6. Hint ladder (one hint per attempt, in order)

1. "Rule three ends with: a developer who can edit the YAML must not be able to weaken these controls. Where would a control have to live so that editing the YAML cannot touch it?"
2. "Think about the four surfaces of Azure DevOps: the YAML file, the service connection, the environment, and the branch policies. Which of the four options puts approvals, branch control and the time window on the environment?"
3. "Rule two says no stored secret. One option uses a SQL password in a secret variable. What kind of service connection credential has no secret at all?"
4. "One option uses SqlDacpacDeploymentOnMachineGroup. What does the phrase on machine group tell you about where that task deploys, and in which kind of pipeline it is supported?"
5. "Another option uses a CODEOWNERS file and an approvals key inside the environment YAML. Which product reads CODEOWNERS? Does Azure Repos? And does the YAML environment keyword accept an approvals list?"
6. "You are between a and c. Both have the same pipeline and the same service connection. The difference is only where the production gates live: in the YAML, or on the environment. Which one survives a malicious pull request?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "c, ManualValidation pauses the pipeline until a DBA approves, that is an approval" | Confuses a pause step with a check outside the YAML | "It does pause. Now imagine a developer deletes that step in the same pull request that changes a table. What stops them?" |
| "c, the cron schedule at ten in the evening satisfies the window" | Schedule versus window | "A schedule starts a run at a fixed time. Does it wait for an approval? What happens if the approval arrives at eleven in the morning?" |
| "b, a secret variable is encrypted, so there is no stored secret" | Encrypted secret is still a secret | "Is a SQL password that must be rotated something that expires? What does rule two say about that?" |
| "b, the queue-time variable approvedByDba is set by a DBA" | Variable is not an approval | "Who else can queue a run and set that variable to true?" |
| "d, DeployReport lets the DBAs review before production" | Confuses report with deployment | "After DeployReport runs, has the production database changed? Does the policy want a report or a deployment?" |
| "d, CODEOWNERS handles the Tables folder" | GitHub versus Azure Repos | "Which product invented CODEOWNERS? What is the Azure Repos policy that does the same job with a path filter?" |
| "a is wrong, the Production stage has no branch condition" | Forgets the environment check | "Look at the checks on the marinaberth dash production environment. Is there one about branches?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the four control surfaces and what each one owns:

- **YAML: build and deploy.** DotNetCoreCLI at 2 with command build restores the Microsoft dot Build dot Sql SDK and produces bin slash Release slash MarinaBerth dot dacpac. PublishPipelineArtifact at 1 publishes it; deployment jobs download current so every environment deploys the very artifact that was built. SqlAzureDacpacDeployment at 1 with deployType DacpacTask and DeploymentAction Publish deploys to Azure SQL Database. Other actions are Script, DeployReport, DriftReport, Extract, Export and Import. AdditionalArguments passes SqlPackage properties, and slash p colon BlockOnPossibleDataLoss equals True makes Publish stop when the change could lose data. The task runs on Windows agents only and adds and removes a firewall rule for the agent IP.
- **Service connection: authentication.** Azure Resource Manager with workload identity federation, App registration automatic, has no secret and nothing to rotate. AuthenticationType servicePrincipal makes the task connect as that identity, which must exist in each database as a contained user created FROM EXTERNAL PROVIDER with the needed role. aadAuthenticationIntegrated needs a directory-joined private agent and is not supported on hosted agents.
- **Environment checks: production gates.** Approvals and checks are not defined in the YAML; users who modify the pipeline YAML cannot modify the checks. Approvals by a group count as one approval from any member. Branch control takes fully qualified names such as refs slash heads slash main, and Verify branch protection fails if the branch has no policy. Business hours holds the stage until the window opens. Checks run in a fixed order: static checks first, then approvals, then dynamic checks, and all checks on all resources must pass. Checks can also sit on the service connection, for example Required template.
- **Branch policies: merge gates.** Minimum reviewers with Reset all approval votes when new changes are pushed. Build validation, Automatic and Required, so the pull request cannot complete until the build passes. Automatically include reviewers, Required, with a path filter, which is the Azure Repos equivalent of GitHub's CODEOWNERS plus Require review from Code Owners. Once a required policy exists, direct pushes to main are blocked.
- **The wrong task.** SqlDacpacDeploymentOnMachineGroup at 0 deploys to SQL Server on machines in a deployment group, with Windows or SQL authentication, and is supported on classic release pipelines only.

Memory hook: "YAML builds and deploys. The service connection authenticates without secrets. The environment holds the gates the YAML author cannot touch. Branch policies gate the merge."

## 9. Follow-up oral questions (optional)

1. "What is the Azure Repos equivalent of a GitHub CODEOWNERS file with Require review from Code Owners?" (The Automatically include reviewers branch policy, marked Required, with a path filter.)
2. "Which SqlPackage property, passed in AdditionalArguments, makes a publish fail when data could be lost?" (BlockOnPossibleDataLoss equals True, which is also the SqlPackage default.)
3. "In which order are environment checks evaluated?" (Static checks such as branch control and required template first, then approvals, then dynamic checks such as business hours, Azure Function, REST API and Azure Monitor.)

## 10. References

- SqlAzureDacpacDeployment@1 task: https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/sql-azure-dacpac-deployment-v1
- SqlDacpacDeploymentOnMachineGroup@0 task: https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/sql-dacpac-deployment-on-machine-group-v0
- ManualValidation@1 task: https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/manual-validation-v1
- Approvals and checks: https://learn.microsoft.com/en-us/azure/devops/pipelines/process/approvals
- Environments in Azure Pipelines: https://learn.microsoft.com/en-us/azure/devops/pipelines/process/environments
- Connect to Azure with an ARM service connection and workload identity federation: https://learn.microsoft.com/en-us/azure/devops/pipelines/library/connect-to-azure
- Branch policies in Azure Repos: https://learn.microsoft.com/en-us/azure/devops/repos/git/branch-policies
- Pipeline triggers: https://learn.microsoft.com/en-us/azure/devops/pipelines/build/triggers
- SQL Database Projects: https://learn.microsoft.com/en-us/sql/tools/sql-database-projects/sql-database-projects
- SqlPackage Publish parameters and properties: https://learn.microsoft.com/en-us/sql/tools/sqlpackage/sqlpackage-publish
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
