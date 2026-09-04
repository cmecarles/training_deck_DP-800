# Instructor-Examiner guide — Deploy and Pipeline Controls 1

Companion to [deploy_and_pipeline_controls_1.md](deploy_and_pipeline_controls_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Option a contains a GitHub Actions workflow of about forty lines; do not read it line by line. Describe its triggers, its three jobs and the SqlPackage actions and properties as written in section 2, and say "I can read any line on request". Read the four policy rules and all four options before taking an answer.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Deploy database solutions.
- Task bullet: Implement CI/CD for SQL Database Projects with GitHub Actions, SqlPackage and repository controls.
- What is tested: SqlPackage DeployReport versus Script versus Publish, BlockOnPossibleDataLoss, branch protection with Require review from Code Owners and no bypass, CODEOWNERS, environments with required reviewers, and workflow triggers including workflow underscore dispatch.

## 2. Scenario to read aloud

**Piece 1, the story.** "A city's parking department maintains the Azure SQL Database ParkPermit as an SDK-style SQL Database Project in the GitHub repository cityparking slash parkpermit. The team has a staging and a production database, and two database administrators who form the GitHub team at cityparking slash dba dash team. A developer opens a pull request that changes the file Permits slash Tables slash Permit dot sql, so that the single column HolderName, NVARCHAR one hundred twenty, is replaced by HolderFirstName, NVARCHAR sixty, and HolderLastName, NVARCHAR sixty. Production holds four hundred eighty thousand permits."

**Piece 2, the release policy.** "Four rules. One: every pull request must build the project and show reviewers what the deployment would do to the staging database, without changing any database. Two: changes to anything under Permits slash Tables cannot be merged until a member of the dba dash team has approved the pull request; nobody, administrators included, may push directly to main. Three: a merge to main deploys staging automatically. Production is deployed only after two named approvers accept it, and a deployment that would lose data must fail rather than proceed. Four: it must be possible to re-run the production deployment on demand without a new commit."

**Piece 3, option a, repository controls.** "Option a. Branch protection on main: require a pull request with one approval, tick Require review from Code Owners, require the build status check, and tick Do not allow bypassing the above settings. Add a CODEOWNERS file under dot github with one line: the path slash Permits slash Tables slash, mapped to the team at cityparking slash dba dash team. Create a production environment with the two DBAs as required reviewers."

**Piece 4, option a, the workflow.** "The workflow triggers on pull underscore request to main, on push to main, and on workflow underscore dispatch. Three jobs. Job build: checks out, runs dotnet build on ParkPermit dot sqlproj in Release, then, only when the event is a pull request, runs sqlpackage with Action DeployReport, source file the dacpac, target connection string the STAGING underscore SQL underscore CONNECTION secret, output path report dot xml. It uploads the dacpac as an artifact. Job deploy dash staging: runs only when the event is not a pull request, needs build, uses environment staging, downloads the dacpac and runs sqlpackage Action Publish with the environment's SQL underscore CONNECTION secret. Job deploy dash production: needs deploy dash staging, uses environment production, downloads the dacpac, and runs sqlpackage Action Publish with the property BlockOnPossibleDataLoss equals True."

**Piece 5, option b.** "Option b. Same branch protection and CODEOWNERS as option a, but no environments. A single deploy job runs on push to main and publishes staging and then production with the properties BlockOnPossibleDataLoss equals False and DropObjectsNotInSource equals True, quote, so that releases never fail on a column change and the databases always match the project. Approval is provided by the pull request review."

**Piece 6, option c.** "Option c. Add the same CODEOWNERS file but leave Require review from Code Owners unchecked, quote, because GitHub automatically requests the owners' review anyway. Trigger the production deployment on pull underscore request so reviewers can inspect the deployed result before approving, and run sqlpackage Action Publish with BlockOnPossibleDataLoss equals True."

**Piece 7, option d.** "Option d. Generate the deployment script instead of deploying: sqlpackage Action Script with the source file, the target connection string and output path deploy dot sql, on every push to main. Upload it as an artifact and let the two DBAs run it by hand in SQL Server Management Studio after reading it. Gate the workflow with a workflow underscore dispatch input named approved, set to true, that the DBAs must set when they trigger it."

## 3. Setup script (reference only; do not read verbatim unless asked)

Option a CODEOWNERS:

```text
# .github/CODEOWNERS
/Permits/Tables/   @cityparking/dba-team
```

Option a workflow:

```yaml
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
  workflow_dispatch:
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: dotnet build ParkPermit.sqlproj -c Release
      - if: github.event_name == 'pull_request'
        run: >
          sqlpackage /Action:DeployReport /SourceFile:bin/Release/ParkPermit.dacpac
          /TargetConnectionString:"${{ secrets.STAGING_SQL_CONNECTION }}" /OutputPath:report.xml
      - uses: actions/upload-artifact@v4
        with: { name: dacpac, path: bin/Release/ParkPermit.dacpac }
  deploy-staging:
    if: github.event_name != 'pull_request'
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/download-artifact@v4
        with: { name: dacpac }
      - run: sqlpackage /Action:Publish /SourceFile:ParkPermit.dacpac /TargetConnectionString:"${{ secrets.SQL_CONNECTION }}"
  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/download-artifact@v4
        with: { name: dacpac }
      - run: >
          sqlpackage /Action:Publish /SourceFile:ParkPermit.dacpac
          /TargetConnectionString:"${{ secrets.SQL_CONNECTION }}" /p:BlockOnPossibleDataLoss=True
```

Option b publish properties: `/p:BlockOnPossibleDataLoss=False /p:DropObjectsNotInSource=True`.
Option d command: `sqlpackage /Action:Script /SourceFile:... /TargetConnectionString:... /OutputPath:deploy.sql`.

## 4. The question (ask exactly this)

"Which configuration implements the policy?

a. Branch protection on main with one required approval, Require review from Code Owners, the build status check, and Do not allow bypassing. A CODEOWNERS file mapping slash Permits slash Tables slash to the dba dash team. A production environment with the two DBAs as required reviewers. A workflow triggered on pull underscore request, push to main and workflow underscore dispatch, with a build job that runs DeployReport against staging on pull requests, a deploy dash staging job that publishes on non pull request events, and a deploy dash production job in the production environment that publishes with BlockOnPossibleDataLoss True.

b. Same branch protection and CODEOWNERS, no environments. One deploy job on push to main publishes staging then production with BlockOnPossibleDataLoss False and DropObjectsNotInSource True. Approval is the pull request review.

c. CODEOWNERS file but Require review from Code Owners unchecked, because GitHub requests the owners' review anyway. Production deployment triggered on pull underscore request so reviewers can inspect the deployed result, with BlockOnPossibleDataLoss True.

d. Generate the script with Action Script on every push to main, upload it, and let the DBAs run it by hand in SSMS. Gate the workflow with a workflow underscore dispatch input approved equals true.

Which letter, and which rule does each of the other three break?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

| Option | Verdict | Why |
|---|---|---|
| a | Correct | Rule 1: on pull request, build plus DeployReport, an XML of what Publish would do, no changes; the if guards keep deploy jobs off PR runs. Rule 2: CODEOWNERS plus Require review from Code Owners makes an owner approval mandatory for matching files; require a pull request blocks direct pushes; Do not allow bypassing extends it to admins. Rule 3: push to main runs build, staging, then production; the production environment's required reviewers pause the job and hide its secrets until approved; BlockOnPossibleDataLoss True stops a publish that would lose data. Rule 4: workflow underscore dispatch re-runs the same workflow with the same gates. |
| b | Wrong | BlockOnPossibleDataLoss False lets the publish drop HolderName and its 480,000 values, forbidden by rule 3. DropObjectsNotInSource True silently deletes every target object absent from the project. A PR review approves code, not this deployment to production at this time; no environment gate, no per-job secret scoping. |
| c | Wrong | CODEOWNERS only requests the review; without the Require review from Code Owners checkbox, a PR can merge with any other approval while the DBAs' request is pending, so rule 2 fails. Deploying production on pull underscore request runs unreviewed, unmerged code, and from a fork the secrets are unavailable. |
| d | Wrong | Action Script is useful for reading and archiving, but running scripts by hand in SSMS bypasses every control: no approval trail, no artifact lineage, a script from yesterday's target state run against today's. A workflow underscore dispatch input is not an approval gate; anyone with write access can type approved true. Environment required reviewers are enforced and recorded by GitHub. |

For this specific change the production publish will stop on data loss, and that is the point: use expand and contract, add the two columns and back-fill in a post-deployment script now, drop HolderName in a later release, or a pre-deployment script that preserves the data.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with rule three, a deployment that would lose data must fail. One option turns off the data-loss block and also drops objects not in source. Which rule does that break?"
2. "Rule two says the DBA team's approval is mandatory. CODEOWNERS makes GitHub request their review. Is a request the same as a requirement? Which branch protection checkbox turns it into one?"
3. "Rule one says show what the deployment would do, without changing anything. Which SqlPackage action produces a report of what Publish would do? And which option runs that on pull requests?"
4. "Rule three also wants two named approvers before production. Which GitHub feature pauses a job until named reviewers approve: a workflow input, a pull request review, or an environment with required reviewers?"
5. "Only one option uses DeployReport on pull requests, Require review from Code Owners, an environment with required reviewers, BlockOnPossibleDataLoss True and workflow underscore dispatch. Which letter?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "Option b, the release must never fail" | Treats the data-loss block as an obstacle | "Rule three says a deployment that would lose data must fail. What happens to the 480,000 HolderName values under option b?" |
| "Option b, the PR review is the approval" | Confuses code review with deployment approval | "Who approved this deployment, to production, at this time? Where is that recorded?" |
| "Option c, GitHub requests the owners automatically" | Thinks a review request blocks merging | "If another reviewer approves while the DBAs have not answered, can the PR merge? Which checkbox would stop it?" |
| "Option c, deploying on pull request lets reviewers see the result" | Deploys unmerged code to production | "Which code runs in a pull request event: merged code or the proposal? And what is DeployReport for?" |
| "Option d, a script the DBAs read is safer" | Replaces automation with a manual procedure | "Does running a script by hand in SSMS leave an approval trail? Is the script still valid if the target changed since it was generated?" |
| "Option d, the approved input is a gate" | Mistakes an input for an enforced approval | "Who can set that input to true? Is that limited to the two named DBAs?" |
| "Option a, workflow underscore dispatch skips the approval" | Thinks manual runs bypass environments | "Does a manual run still pass through the production environment? What happens when the job reaches it?" |

## 8. Teaching notes (after the answer is complete or revealed)

Map the policy onto the tools:

- **SqlPackage actions.** Publish applies changes. Script writes the T-SQL without running it. DeployReport writes an XML of what Publish would do, with no changes, including the HolderName drop flagged as a data-loss operation. DeployReport is the pull request check, rule 1.
- **Publish properties.** BlockOnPossibleDataLoss defaults to True and makes the publish stop during validation if data could be lost; option a states it explicitly. DropObjectsNotInSource defaults to False; True deletes every target object absent from the project, indexes a DBA added, monitoring views, diagnostic tables. IncludeCompositeObjects handles same-database references. A publish profile, dot publish dot xml, stores saved properties. Never disable the data-loss block as a blanket setting.
- **Branch protection on main.** Require a pull request with N approvals blocks direct pushes. Require status checks makes the build mandatory. Require review from Code Owners turns CODEOWNERS into a requirement. Do not allow bypassing applies the rules to administrators. Also restrict who can push and forbid force pushes.
- **CODEOWNERS.** Valid locations: repository root, docs, or dot github. Each line is a pattern and one or more owners, users or teams with write access. The last matching pattern wins. Owners are automatically requested, but required only with the branch protection checkbox. That is why option c fails.
- **Environments.** A job-level environment equals production key gives required reviewers, up to six people or teams, a wait timer, deployment branch restrictions and environment-scoped secrets. The job pauses before starting and the secrets are not exposed until approved. That is the two-approver gate of rule 3, and why options b and d fail.
- **Triggers.** pull underscore request validates: build and DeployReport. push to main deploys staging then production. workflow underscore dispatch runs the same workflow on demand, through the same gates, from the Actions tab or gh workflow run. That is rule 4.
- **The column change itself.** A drop with data is what the block is for. Deliver in two releases: expand, add the two new columns and back-fill in a post-deployment script; contract, drop HolderName later once nothing reads it. Or use a pre-deployment script to preserve the data.

Memory hook: "DeployReport shows, Publish does, Script writes. Code Owners require only with the checkbox. Environments approve. Never switch off the data-loss block."

## 9. Follow-up oral questions (optional)

1. "Where can a CODEOWNERS file live, and which pattern wins when several match?" (Repository root, docs, or dot github; the last matching pattern wins.)
2. "What is the default value of BlockOnPossibleDataLoss, and what does it do?" (True; the publish stops during validation if the change could lose data.)
3. "How would you deliver the HolderName split without losing data?" (Expand and contract: add HolderFirstName and HolderLastName and back-fill them in a post-deployment script, then drop HolderName in a later release; or preserve the data in a pre-deployment script.)

## 10. References

- SqlPackage Publish and its properties: https://learn.microsoft.com/en-us/sql/tools/sqlpackage/sqlpackage-publish
- SqlPackage DeployReport: https://learn.microsoft.com/en-us/sql/tools/sqlpackage/sqlpackage-deploy-drift-report
- SqlPackage Script: https://learn.microsoft.com/en-us/sql/tools/sqlpackage/sqlpackage-script
- SQL Database Projects overview: https://learn.microsoft.com/en-us/sql/tools/sql-database-projects/sql-database-projects
- About protected branches, including Require review from Code Owners and bypass settings: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches
- About code owners: https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners
- Using environments for deployment, required reviewers: https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment
- Events that trigger workflows, including workflow_dispatch: https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
