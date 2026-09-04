# Instructor-Examiner guide — Secrets and Branching 1

Companion to [secrets_and_branching_1.md](secrets_and_branching_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a, b, c and d. Each option has two halves: a GitHub Actions pipeline configuration and a git conflict resolution. Read the four requirements, the conflict, and all four options before taking an answer. Describe the YAML precisely in words: the trigger, the permissions block, the environment, and the keys passed to each action. This is a conceptual tooling question; nothing was executed against an engine.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Implement CI/CD by using SQL Database Projects.
- Task bullet: Manage secrets in pipelines; manage branching, pull requests and merge conflicts.
- What is tested: OpenID Connect federated credentials versus stored secrets, GitHub secrets versus variables and environment scoping, deploying only from main, and resolving a merge conflict in a declarative .sql object file so that both changes survive.

## 2. Scenario to read aloud

**Piece 1, the story.** "A ride-sharing startup keeps the schema of its Azure SQL Database FareSplit in an SDK-style SQL Database Project in a GitHub repository. Deployments run from GitHub Actions, using the azure slash login action followed by the azure slash sql-action action. The production database accepts Microsoft Entra authentication. A user-assigned managed identity named id dash faresplit dash deploy has been created in the subscription and granted db underscore owner on the database."

**Piece 2, the four requirements.** "The security review produced four requirements. One: the workflow must authenticate to Azure and to the database without any long-lived credential. No client secret, no SQL password, existing in GitHub or in the repository. Two: connection information must never be committed to the repository. Not in the sqlproj, not in a publish profile, not in the workflow YAML. Three: whatever secret material remains must be readable only by the job that deploys to production. Four: developers work on feature branches and merge into main through pull requests. The pipeline must deploy production only from main."

**Piece 3, the conflict.** "While the review was in progress, two pull requests touched the same table. Main already merged PR forty-one, which added a SurgeMultiplier column. PR forty-two, on branch feature slash tips, adds a TipAmount column to the same table. GitHub reports that PR forty-two has conflicts that must be resolved. The file Fare slash Tables slash Trip dot sql shows CREATE TABLE Fare dot Trip with TripId, an integer primary key, and BaseFare, decimal eight comma two. Then a conflict block. Between the marker with seven less-than signs and HEAD, and the marker of seven equals signs, is the main side: SurgeMultiplier, decimal four comma two, not null, with a default constraint named DF underscore Trip underscore Surge defaulting to one point zero zero. Between the equals marker and the marker of seven greater-than signs and feature slash tips is the branch side: TipAmount, decimal eight comma two, not null, with a default constraint DF underscore Trip underscore Tip defaulting to zero. Then the closing paren and semicolon."

**Piece 4, option a.** "Option a. Create a service principal with a client secret. Store its JSON in the repository secret AZURE underscore CREDENTIALS. Store the connection string in the repository secret SQL underscore CONNECTION. The workflow triggers on push to branches main. One job, deploy, on ubuntu-latest, with three steps: actions slash checkout at v4; azure slash login at v3 with the input creds set to secrets dot AZURE underscore CREDENTIALS; and azure slash sql-action at v2 dot 3 with connection-string set to secrets dot SQL underscore CONNECTION, where a comment shows the string contains User ID equals deploy and a Password, plus path set to FareSplit dot sqlproj and action set to publish. The conflict is resolved by taking the version of the file from main, with git checkout main, double dash, the file path, then committing and merging."

**Piece 5, option b.** "Option b. Configure a federated identity credential on id dash faresplit dash deploy that trusts the repository's production environment. The subject is repo colon faresplit slash db colon environment colon production. Store only identifiers as environment secrets. The workflow triggers on push to branches main, and also on workflow underscore dispatch. It has a top-level permissions block: id-token write, contents read. One job, deploy, on ubuntu-latest, with environment set to production. Steps: checkout at v4; azure slash login at v3 with client-id, tenant-id and subscription-id, each from a secret named AZURE underscore CLIENT underscore ID, AZURE underscore TENANT underscore ID and AZURE underscore SUBSCRIPTION underscore ID; and azure slash sql-action at v2 dot 3 with connection-string from secrets dot SQL underscore CONNECTION, where a comment shows Server equals tcp colon faresplit dot database dot windows dot net comma fourteen thirty three, Initial Catalog equals FareSplit, Authentication equals Active Directory Default, Encrypt equals True. Then path FareSplit dot sqlproj and action publish. The conflict is resolved on the feature branch by editing Trip dot sql to keep both columns and remove the markers, running dotnet build to confirm the project still builds, committing and pushing. The pull request checks re-run and the PR is merged when green."

**Piece 6, option c.** "Option c. Put the connection string, including the SQL login password, in FareSplit dot publish dot xml, in the TargetConnectionString element, and reference the profile from the action, so no GitHub secret is needed at all. Add the file to dot gitignore afterwards. The workflow triggers on push, with no branch filter. One job with checkout and azure slash sql-action at v2 dot 3, with path FareSplit dot sqlproj, action publish, and arguments slash Profile colon FareSplit dot publish dot xml. There is no azure slash login step. The conflict is resolved by rebasing feature slash tips onto main, dropping the conflicting commit from main during the rebase, and force-pushing main."

**Piece 7, option d.** "Option d. Use the same OpenID Connect login as option b, but store AZURE underscore CLIENT underscore ID, AZURE underscore TENANT underscore ID, AZURE underscore SUBSCRIPTION underscore ID and the connection string as repository variables, referenced as vars dot SQL underscore CONNECTION, so they are visible for troubleshooting. Trigger the deployment on pull underscore request against branches main, so that reviewers see the production result before approving. The conflict is resolved automatically with git merge, dash X theirs, main, on the feature branch."

## 3. Setup script (reference only; do not read verbatim unless asked)

The conflicted file:

```sql
CREATE TABLE Fare.Trip
(
    TripId          INT           NOT NULL PRIMARY KEY,
    BaseFare        DECIMAL(8,2)  NOT NULL,
<<<<<<< HEAD
    SurgeMultiplier DECIMAL(4,2)  NOT NULL CONSTRAINT DF_Trip_Surge DEFAULT (1.00)
=======
    TipAmount       DECIMAL(8,2)  NOT NULL CONSTRAINT DF_Trip_Tip DEFAULT (0)
>>>>>>> feature/tips
);
```

Option a workflow:

```yaml
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v3
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      - uses: azure/sql-action@v2.3
        with:
          connection-string: ${{ secrets.SQL_CONNECTION }}   # ...;User ID=deploy;Password=...;
          path: ./FareSplit.sqlproj
          action: publish
```

Option b workflow:

```yaml
on:
  push:
    branches: [main]
  workflow_dispatch:
permissions:
  id-token: write
  contents: read
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v3
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - uses: azure/sql-action@v2.3
        with:
          connection-string: ${{ secrets.SQL_CONNECTION }}
          # Server=tcp:faresplit.database.windows.net,1433;Initial Catalog=FareSplit;Authentication=Active Directory Default;Encrypt=True;
          path: ./FareSplit.sqlproj
          action: publish
```

Option c workflow:

```yaml
on: push
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/sql-action@v2.3
        with:
          path: ./FareSplit.sqlproj
          action: publish
          arguments: /Profile:FareSplit.publish.xml
```

Option d trigger:

```yaml
on:
  pull_request:
    branches: [main]
```

Resolved file for the correct option, both columns and no markers:

```sql
CREATE TABLE Fare.Trip
(
    TripId          INT           NOT NULL PRIMARY KEY,
    BaseFare        DECIMAL(8,2)  NOT NULL,
    SurgeMultiplier DECIMAL(4,2)  NOT NULL CONSTRAINT DF_Trip_Surge DEFAULT (1.00),
    TipAmount       DECIMAL(8,2)  NOT NULL CONSTRAINT DF_Trip_Tip DEFAULT (0)
);
```

## 4. The question (ask exactly this)

"Which combination of pipeline configuration and conflict resolution satisfies all four requirements and lands both columns in main? Option a, option b, option c, or option d?"

Options in full:

- **a.** Service principal client secret in repository secret AZURE_CREDENTIALS, connection string with SQL password in repository secret SQL_CONNECTION, azure/login with `creds:`, trigger on push to main. Resolve by taking main's version of Trip.sql.
- **b.** Federated identity credential on id-faresplit-deploy trusting `repo:faresplit/db:environment:production`; workflow with `permissions: id-token: write`, `environment: production`, azure/login with client-id, tenant-id, subscription-id from environment secrets, connection string with `Authentication=Active Directory Default` from an environment secret; trigger on push to main plus workflow_dispatch. Resolve by editing Trip.sql to keep both columns, removing markers, `dotnet build`, commit, push, PR checks re-run.
- **c.** Password inside FareSplit.publish.xml TargetConnectionString, referenced with `/Profile:`, file added to .gitignore afterwards, trigger `on: push` with no branch filter, no azure/login. Resolve by rebasing feature/tips onto main, dropping main's commit, force-pushing main.
- **d.** Same OIDC login as b but identifiers and connection string as repository variables (`vars.SQL_CONNECTION`), trigger on `pull_request` against main. Resolve with `git merge -X theirs main`.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: b.**

- Requirement one: OpenID Connect. GitHub issues a short-lived ID token, azure/login at v3 exchanges it because a federated identity credential on the managed identity trusts the subject `repo:faresplit/db:environment:production`. The workflow needs `permissions: id-token: write`. Client id, tenant id and subscription id are identifiers, not access-granting secrets. The database connection uses `Authentication=Active Directory Default`, no User ID or Password.
- Requirement two: the connection string is a GitHub secret, masked in logs, never in git.
- Requirement three: secrets on the production environment are readable only by jobs declaring `environment: production`, and environment secrets take precedence over repository and organisation secrets.
- Requirement four: `on: push: branches: [main]` plus `workflow_dispatch`; with branch protection, main changes only through reviewed pull requests.
- Conflict: a .sql object file is a declarative whole-object definition, so edit it to the intended final state with both columns, delete the three markers, `dotnet build` to validate, commit to the feature branch, let checks re-run, merge.

Why the others are wrong, one line each:

- **a.** `creds:` is a service principal client secret and the connection string carries a SQL password, both long-lived; repository secrets are readable by any workflow; and taking main's file silently drops TipAmount.
- **c.** The password is committed inside the publish profile, and gitignore afterwards does not remove it from history; `on: push` with no filter deploys from every branch; the rebase discards the SurgeMultiplier commit and force-pushes main.
- **d.** Variables are not masked, so the connection string prints in every log; deploying on `pull_request` ships unreviewed code, and fork PRs do not even receive secrets; `-X theirs` drops SurgeMultiplier without anyone looking.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with the conflict. The goal is both columns in main. For each option, ask: after this resolution, does Trip dot sql contain SurgeMultiplier and TipAmount, or only one of them?"
2. "Now requirement one, no long-lived credential. Which login mechanism lets GitHub prove its identity with a short-lived token, and which inputs to azure slash login does it use?"
3. "Requirement two says nothing committed. Does putting a file in dot gitignore after committing it remove it from the repository history?"
4. "Option a uses a client secret in creds and a SQL password in the string, and its resolution takes main's file wholesale. That eliminates a."
5. "Option c commits the password inside the publish profile and rewrites main with a force-push. That eliminates c."
6. "You are down to b and d. Both use OpenID Connect. Compare where the connection string lives, secrets versus variables, and what event triggers the production deployment, push to main versus pull underscore request. Which one keeps the string masked and deploys only reviewed code?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, the secrets are in GitHub secrets, so they are safe" | Confuses stored securely with not long-lived | "Requirement one is not about where the secret sits. Is a client secret or a SQL password long-lived? And what does the resolution do to TipAmount?" |
| "c, it removes secrets from GitHub entirely" | Misses that the password went into the repository | "Where did the password go instead? And does adding the file to gitignore afterwards remove it from history?" |
| "d, variables make troubleshooting easier" | Does not know variables are unmasked | "Are values from the vars context masked in logs? What will appear in every run's output?" |
| "d, deploying on pull request lets reviewers see the result" | Confuses validation on PR with production deployment | "Whose code runs against production when a PR from any branch opens? Has it been reviewed yet?" |
| "b, but dash X theirs would also keep both columns" | Misunderstands merge strategy options | "Dash X theirs resolves every conflicting hunk in favour of one side. Which side, and what happens to the other column?" |
| "b, but a federated credential is still a secret" | Confuses identifiers with credentials | "Can client id, tenant id and subscription id grant access by themselves without the trusted token?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the secrets half:

- Authenticate without a stored secret: a federated identity credential on the managed identity or app registration, trusting a subject such as `repo:owner/repo:environment:production`, or `...:ref:refs/heads/main`, or `...:pull_request`. The workflow declares `permissions: id-token: write` and calls azure/login at v3 with client-id, tenant-id and subscription-id. The `creds:` JSON form is a client secret and is long-lived. A SQL password in the string is long-lived. The database connection uses `Authentication=Active Directory Default`, which picks up the identity azure/login established.
- Where things live: secrets are masked, with precedence organisation, then repository, then environment; environment secrets are available only to jobs that declare `environment: name`, and they are not passed to pull requests from forks. Variables in the vars context are plain text for non-sensitive configuration. Never in git: connection strings, passwords, publish profiles with credentials. A gitignore added after the commit does not remove history.
- Requirement three is met by environment scoping. If the environment has required reviewers, the job cannot read the secrets until approved.

Then the branching half:

- Trunk-based flow: short-lived feature branch, pull request, main always deployable. Deploy on push to main or workflow_dispatch, never on pull_request.
- A conflict in a .sql object file is resolved like any source file: edit to the intended final definition, usually both changes, remove the `<<<<<<<`, `=======` and `>>>>>>>` markers, run `dotnet build` to validate, commit to the feature branch, let the checks re-run. A leftover marker is a syntax error and the build fails, which is a useful safety net.
- Never `-X theirs` or `-X ours`, which pick one side blindly, and never force-push main, which branch protection should forbid. Taking one version of the file wholesale drops the other change silently.

Memory hook: "Federated token, not stored secret. Environment secrets, not variables. Push to main, not pull request. Edit both columns, drop the markers, build."

## 9. Follow-up oral questions (optional)

1. "Which permission must the workflow declare so that a job can request an OpenID Connect token?" (permissions: id-token: write.)
2. "If the same secret name exists at repository level and at environment level, which value does a job with environment production see?" (The environment secret; environment takes precedence.)
3. "What happens at dotnet build if a conflict marker is left in Trip dot sql?" (The build fails with a syntax error, which catches the mistake before merge.)

## 10. References

- Use GitHub Actions to connect to Azure with OpenID Connect: https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect
- azure/login action: https://github.com/Azure/login
- azure/sql-action: https://github.com/Azure/sql-action
- Using secrets in GitHub Actions: https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions
- Store information in variables: https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/store-information-in-variables
- Managing environments for deployment: https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment
- Resolving a merge conflict using the command line: https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/addressing-merge-conflicts/resolving-a-merge-conflict-using-the-command-line
- SQL Database Projects, CI/CD with GitHub Actions: https://learn.microsoft.com/en-us/sql/tools/sql-database-projects/tutorials/create-deploy-sql-project
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
