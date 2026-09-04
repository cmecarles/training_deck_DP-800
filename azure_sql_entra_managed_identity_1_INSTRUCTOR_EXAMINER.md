# Instructor-Examiner guide — Azure SQL Entra Managed Identity 1

Companion to [azure_sql_entra_managed_identity_1.md](azure_sql_entra_managed_identity_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**This is a hands-on Azure lab question.** Before anything else, ask: "Have you already run this lab in your own subscription?" If yes, go through the runs R1 to R7 and ask what the learner observed at each step before you quiz; treat the observations as their answers and correct them against section 5. If no, walk through the provisioning and the runs in words using section 2, so the question can still be answered from the documented facts. Either way the learner must give, for each run, success or failure and the error number for the failures.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "dash dash" for `--`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line from section 3 only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Implement data security and compliance.
- Task bullet: Configure Microsoft Entra authentication, managed identities and Entra-only authentication for Azure SQL Database.
- What is tested: that a managed identity needs a database principal created FROM EXTERNAL PROVIDER before it can connect, what error a missing principal produces, and that Entra-only authentication blocks SQL logins from connecting but not from being created.

## 2. Scenario to read aloud

**Piece 1, the story.** "A marina publishes its berth availability through a small worker that runs on demand in Azure. The worker reads an Azure SQL Database called BerthBook. The security baseline has three parts. No SQL login can ever connect. The worker authenticates as a user-assigned managed identity. And nothing that looks like a password exists anywhere. This is a hands-on lab: you provision the resources in your own subscription, run seven steps, R1 to R7, and predict what each does."

**Piece 2, the logical server.** "The provisioning is an Azure CLI script in bash, after az login. It picks a region, West Europe, and a random suffix. It creates a resource group named rg dash dp800 dash berthbook dash suffix. It reads your own user principal name and object id with az ad signed-in-user show. Then the key command: az sql server create with the flag dash dash enable dash ad dash only dash auth, and three external admin flags: external admin principal type User, external admin name equal to your UPN, and external admin sid equal to your object id. Notice what is missing: there is no dash u and no dash p. No SQL admin password is given. The server is born with Entra-only authentication on, and you are its Entra admin."

**Piece 3, firewall and database.** "Two firewall rules are added. One named client, for your own public IP. One named AllowAzureServices with start and end address zero dot zero dot zero dot zero, which is the special rule that lets Azure services reach the server. Then az sql db create makes the database BerthBook in the cheapest vCore shape: General Purpose, Gen5, one vCore, compute model Serverless, auto-pause delay sixty minutes, min capacity zero point five, local backup redundancy. The script saves the server's fully qualified domain name in a variable called SQL underscore FQDN."

**Piece 4, the managed identity.** "az identity create makes a user-assigned managed identity named id dash berthbook dash worker in the resource group. The script saves two values from az identity show: its resource id, in UAMI underscore ID, and its client id, in UAMI underscore CLIENT underscore ID. Keep in mind that a managed identity also has a principal id, also called object id. That is a different GUID from the client id."

**Piece 5, the schema.** "You connect as yourself with the Go version of sqlcmd, using dash dash authentication dash method ActiveDirectoryDefault, which reuses your az login session. You create a schema Marina and a table Marina dot Berths with four columns: BerthId, an integer primary key; Pontoon, one character; LengthM, a decimal with four digits and one decimal place; and IsFree, a bit. Three rows go in: berth 1, pontoon A, eight metres, free. Berth 2, pontoon A, ten metres, not free. Berth 3, pontoon B, twelve point five metres, free. Finally you run SELECT SERVERPROPERTY of IsExternalAuthenticationOnly."

**Piece 6, the worker.** "The worker is an Azure Container Instance. A helper function called run underscore worker creates a container named aci dash berthbook from the image ubuntu 22.04, one CPU, one gigabyte, restart policy Never, and the important flag: dash dash assign dash identity with the resource id of the user-assigned identity. It passes three environment variables: SQL underscore FQDN, UAMI underscore CLIENT underscore ID, and a base64-encoded script. Inside, the script installs sqlcmd from the Microsoft package feed, then runs sqlcmd against BerthBook with dash dash authentication dash method ActiveDirectoryManagedIdentity and dash U set to the identity's client id. The query returns three columns: USER underscore NAME, the type underscore desc of that principal from sys dot database underscore principals, and COUNT star from Marina dot Berths. The function waits for the container to stop, prints its logs and deletes it."

**Piece 7, runs R1 to R3.** "R1 calls run underscore worker before any database principal exists for the identity. R2, as the Entra admin inside BerthBook, runs three statements: CREATE USER, in square brackets, id dash berthbook dash worker, FROM EXTERNAL PROVIDER. Then ALTER ROLE db underscore datareader ADD MEMBER that user. Then a SELECT from sys dot database underscore principals showing name, type, type underscore desc and the sid cast to UNIQUEIDENTIFIER, filtered to types E and X. R3 calls run underscore worker again."

**Piece 8, runs R4 to R7.** "R4, still as the Entra admin: in master, CREATE LOGIN berth underscore sql WITH PASSWORD, a strong password. Then in BerthBook, CREATE USER berth underscore sql FOR LOGIN berth underscore sql, and add it to db underscore datareader. R5 runs sqlcmd from your machine with dash U berth underscore sql and dash P the password, selecting COUNT star from Marina dot Berths. R6 runs az sql server ad dash only dash auth disable on the server, then repeats R5. R7 runs az sql server ad dash only dash auth get, and in T-SQL, SELECT SERVERPROPERTY of IsExternalAuthenticationOnly."

**Piece 9, cost and cleanup.** "Serverless one vCore bills per vCore-second only while the database is running, cents for this lab, plus storage. The container costs cents per run. At the end, az group delete on the resource group with dash dash yes and dash dash no dash wait."

## 3. Setup script (reference only; do not read verbatim unless asked)

```bash
LOCATION="westeurope"                       # any region with Azure SQL and Container Instances
SUFFIX=$RANDOM
RG="rg-dp800-berthbook-$SUFFIX"
SQL="sql-berthbook-$SUFFIX"                  # globally unique, lowercase
DB="BerthBook"
UAMI="id-berthbook-worker"
ADMIN_UPN=$(az ad signed-in-user show --query userPrincipalName -o tsv)
ADMIN_OID=$(az ad signed-in-user show --query id -o tsv)
MYIP=$(curl -s https://api.ipify.org)

az group create -n $RG -l $LOCATION
# Logical server with NO SQL admin: Entra admin = you, Entra-only authentication ON from birth
az sql server create -g $RG -n $SQL -l $LOCATION --enable-ad-only-auth \
  --external-admin-principal-type User --external-admin-name "$ADMIN_UPN" --external-admin-sid $ADMIN_OID
az sql server firewall-rule create -g $RG -s $SQL -n client --start-ip-address $MYIP --end-ip-address $MYIP
az sql server firewall-rule create -g $RG -s $SQL -n AllowAzureServices --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0
# Cheapest vCore option: serverless GP Gen5 1 vCore, auto-pause after 60 idle minutes
az sql db create -g $RG -s $SQL -n $DB -e GeneralPurpose -f Gen5 -c 1 --compute-model Serverless \
  --auto-pause-delay 60 --min-capacity 0.5 --backup-storage-redundancy Local
SQL_FQDN=$(az sql server show -g $RG -n $SQL --query fullyQualifiedDomainName -o tsv)

# User-assigned managed identity for the worker
az identity create -g $RG -n $UAMI
UAMI_ID=$(az identity show -g $RG -n $UAMI --query id -o tsv)
UAMI_CLIENT_ID=$(az identity show -g $RG -n $UAMI --query clientId -o tsv)
```

Connect: `sqlcmd -S $SQL_FQDN -d $DB --authentication-method ActiveDirectoryDefault` (sqlcmd Go). With the ODBC sqlcmd use `-G -U "$ADMIN_UPN"`.

```sql
CREATE SCHEMA Marina;
GO
CREATE TABLE Marina.Berths (BerthId INT NOT NULL PRIMARY KEY, Pontoon CHAR(1) NOT NULL, LengthM DECIMAL(4,1) NOT NULL, IsFree BIT NOT NULL);
INSERT INTO Marina.Berths VALUES (1, 'A', 8.0, 1), (2, 'A', 10.0, 0), (3, 'B', 12.5, 1);
GO
SELECT SERVERPROPERTY('IsExternalAuthenticationOnly') AS EntraOnly;
```

```bash
cat > worker.sh <<'EOF'
set -e
apt-get update -qq && apt-get install -y -qq curl >/dev/null
curl -fsSL https://packages.microsoft.com/keys/microsoft.asc -o /etc/apt/trusted.gpg.d/microsoft.asc
curl -fsSL https://packages.microsoft.com/config/ubuntu/22.04/prod.list -o /etc/apt/sources.list.d/mssql-release.list
apt-get update -qq && apt-get install -y -qq sqlcmd >/dev/null
sqlcmd -S "$SQL_FQDN" -d BerthBook --authentication-method ActiveDirectoryManagedIdentity -U "$UAMI_CLIENT_ID" -W \
  -Q "SELECT USER_NAME() AS db_user, (SELECT type_desc FROM sys.database_principals WHERE name = USER_NAME()) AS principal_type, COUNT(*) AS berths FROM Marina.Berths;"
EOF
SCRIPT_B64=$(base64 worker.sh | tr -d '\n')

run_worker () {   # creates the container, waits for it to finish, prints its output, deletes it
  az container create -g $RG -n aci-berthbook --image ubuntu:22.04 --os-type Linux --cpu 1 --memory 1 \
    --restart-policy Never --assign-identity $UAMI_ID \
    --environment-variables SQL_FQDN=$SQL_FQDN UAMI_CLIENT_ID=$UAMI_CLIENT_ID SCRIPT_B64=$SCRIPT_B64 \
    --command-line "/bin/bash -c 'echo \$SCRIPT_B64 | base64 -d | bash'" -o none
  while [ "$(az container show -g $RG -n aci-berthbook --query instanceView.state -o tsv)" = "Running" ]; do sleep 15; done
  az container logs -g $RG -n aci-berthbook
  az container delete -g $RG -n aci-berthbook -y -o none
}
```

```text
R1  run_worker                                   -- BEFORE any database principal exists for the identity
R2  (as the Entra admin, in BerthBook)
      CREATE USER [id-berthbook-worker] FROM EXTERNAL PROVIDER;
      ALTER ROLE db_datareader ADD MEMBER [id-berthbook-worker];
      SELECT name, type, type_desc, CAST(sid AS UNIQUEIDENTIFIER) AS oid FROM sys.database_principals WHERE type IN ('E','X');
R3  run_worker
R4  (as the Entra admin) in master:   CREATE LOGIN berth_sql WITH PASSWORD = 'Str0ng!Passw0rd#2026';
                         in BerthBook: CREATE USER berth_sql FOR LOGIN berth_sql; ALTER ROLE db_datareader ADD MEMBER berth_sql;
R5  sqlcmd -S $SQL_FQDN -d $DB -U berth_sql -P 'Str0ng!Passw0rd#2026' -Q "SELECT COUNT(*) FROM Marina.Berths;"
R6  az sql server ad-only-auth disable -g $RG -n $SQL ; then repeat R5
R7  az sql server ad-only-auth get -g $RG -n $SQL ; and in T-SQL: SELECT SERVERPROPERTY('IsExternalAuthenticationOnly');
```

Cleanup: `az group delete -n $RG --yes --no-wait`

## 4. The question (ask exactly this)

"For each run, R1 to R7, tell me whether it succeeds or fails. For the failures, give the SQL error number the client receives. For R3, give the three column values the worker prints. Let's go one at a time, starting with R1."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

| Run | Outcome | Detail |
|---|---|---|
| R1 | Fails, error 18456 | "Login failed for user '<token-identified principal>'." The token is valid, but no database principal has a sid equal to the identity's object id |
| R2 | Succeeds | One row: name id-berthbook-worker, type E, type_desc EXTERNAL_USER, oid equal to the identity's principal (object) id |
| R3 | Succeeds | db_user = id-berthbook-worker, principal_type = EXTERNAL_USER, berths = 3 |
| R4 | Succeeds | Both statements run. Entra-only authentication does not prevent creating SQL principals |
| R5 | Fails, error 18456 | "Login failed for user 'berth_sql'." SQL authentication is disabled server-wide |
| R6 | Succeeds | After ad-only-auth disable, the repeated R5 returns 3 |
| R7 | Succeeds | azureAdOnlyAuthentication: false, and IsExternalAuthenticationOnly = 0 (it was 1 right after provisioning) |

Key facts behind the key: the client always sees the generic 18456 text; the detailed state stays server-side. Azure RBAC roles on the server never grant database access. The system-generated CloudSA server admin cannot connect either while Entra-only is on.

## 6. Hint ladder (one hint per attempt, in order)

**R1**
1. "Separate two things: does Entra issue a token for the identity, and does the database know that identity? Which of the two is missing at this point?"
2. "A token proves who you are. Inside BerthBook, is there any row in sys dot database underscore principals whose sid matches the identity's object id?"
3. "This is a login failure. Which is the classic login failure error number in SQL Server, the one that starts with eighteen?"
4. "Error 18456. And the user name in the message is not the identity's name. It is a placeholder that says the login was identified by a token."

**R2**
1. "Who is running these statements, and can that principal resolve an Entra name through Microsoft Graph?"
2. "Think about what FROM EXTERNAL PROVIDER creates: a contained user, no login in master, no password. What single-letter type does an external user get in the catalog?"
3. "Type E, EXTERNAL underscore USER. And the sid cast to a GUID equals which of the identity's two GUIDs: the client id or the principal id?"

**R3**
1. "Nothing changed on the container. What changed on the database side between R1 and R3?"
2. "The identity now maps to a database user. What does USER underscore NAME return for it, and what type does the catalog report?"
3. "The third column is just a count. How many rows did the INSERT put into Marina dot Berths?"

**R4**
1. "Read the documented rule carefully: Entra-only authentication blocks SQL principals from doing what, exactly? Connecting, or being created?"
2. "The Entra admin has the permissions to run CREATE LOGIN and CREATE USER. Does the server-level switch remove that permission?"

**R5**
1. "berth underscore sql exists. But what does the server-level setting say about SQL authentication right now?"
2. "Same family of failure as R1, and the same number. The message this time names the login."

**R6**
1. "What does ad dash only dash auth disable change, and for which scope: one database or the whole logical server?"
2. "Did berth underscore sql need to be recreated? It already exists, so what does the repeated R5 return?"

**R7**
1. "Two views of the same switch: the CLI property and the SERVERPROPERTY. After R6, what value should each show?"
2. "Compare with the value you saw right after provisioning, in piece 5. It was one then."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "R1 succeeds because the identity has a valid token" | Confuses authentication by Entra with authorization in the database | "A valid token gets you to the gateway. Does the database have a principal that maps to that token?" |
| "R1 fails because the identity needs an Azure RBAC role on the server" | Thinks Azure RBAC grants database access | "Does Microsoft Entra authentication for Azure SQL integrate with Azure RBAC? Check that assumption." |
| "R2 needs CREATE LOGIN in master first" | Applies the SQL login pattern to Entra users | "What kind of user does FROM EXTERNAL PROVIDER create? Does it need a login at all?" |
| "R2 shows type X" | Mixes up external user and external group | "Type X is for external groups. What is the type for a single external user?" |
| "R3 prints the client id as the user name" | Confuses the sid mapping with the name | "USER underscore NAME returns the name of the database user. What name did R2 give it?" |
| "R4 fails because SQL authentication is disabled" | Believes Entra-only blocks creation of SQL principals | "The documentation distinguishes creating from connecting. Which one does the switch block?" |
| "R5 succeeds because the login and user exist" | Ignores the server-level switch | "The login exists. What does the server say about SQL authentication as a connection method?" |
| "R6 needs the login recreated" | Thinks disabling the switch drops SQL principals | "Were the SQL principals ever removed, or just unable to connect?" |
| "R7 still shows one" | Forgets R6 flipped the server-level switch | "What did R6 do to the server, and does SERVERPROPERTY reflect the server-level setting?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the three independent gates for a managed identity reaching Azure SQL:

- **Gate 1, the token.** Entra issues it. For a user-assigned identity the client passes the identity's client id: in sqlcmd Go, `--authentication-method ActiveDirectoryManagedIdentity -U <client id>`; in Microsoft.Data.SqlClient, `Authentication=Active Directory Managed Identity;User Id=<client id>`. For a system-assigned identity you omit the client id. No password anywhere.
- **Gate 2, the principal.** `CREATE USER [<identity display name>] FROM EXTERNAL PROVIDER`, run by an Entra principal with ALTER ANY USER. It creates a contained user, type E, EXTERNAL_USER, whose sid equals the identity's object id. No login in master. If this is missing the client gets error 18456 with the placeholder user name `<token-identified principal>`. That is R1 versus R3.
- **Gate 3, the permission.** `ALTER ROLE db_datareader ADD MEMBER`. Azure RBAC roles such as SQL DB Contributor or Owner never grant access inside the database.

Then the server-level switch, Entra-only authentication:

- Created on from birth with `az sql server create --enable-ad-only-auth --external-admin-*`, no SQL admin values given. Managed later with `az sql server ad-only-auth enable`, `disable` and `get`. In T-SQL, `SERVERPROPERTY('IsExternalAuthenticationOnly')` returns 1 or 0.
- While on, SQL logins and users can still be created, they just cannot connect. That is R4 succeeding and R5 failing with 18456. The system-generated CloudSA server admin is not a back door either.
- An Entra admin must exist before the switch can be enabled, and it cannot be removed while the switch is on. Flipping the switch needs Owner, Contributor or SQL Security Manager. SQL Server Contributor can set the Entra admin but cannot flip the switch. That is separation of duties.
- Disabling the switch re-enables SQL authentication for the whole logical server at once, so R6 succeeds immediately with the existing login, and R7 shows false and 0.

Local note: on a plain SQL Server 2025 instance without Azure Arc, CREATE USER FROM EXTERNAL PROVIDER fails with message 37525, because Entra authentication is not configured, and IsExternalAuthenticationOnly returns 0.

Memory hook: "Token, principal, permission. Entra-only blocks connecting, not creating."

## 9. Follow-up oral questions (optional)

1. "If the worker used a system-assigned managed identity instead, what changes in the sqlcmd command?" (Drop the -U client id; the identity is implicit. The database user is still created FROM EXTERNAL PROVIDER, named after the resource.)
2. "Could a SQL login with ALTER ANY USER run the CREATE USER FROM EXTERNAL PROVIDER statement in R2?" (No. The engine must resolve the name through Microsoft Graph, which requires an Entra principal to run it.)
3. "Which Azure role can set the Entra admin but cannot enable or disable Entra-only authentication?" (SQL Server Contributor. Flipping the switch needs Owner, Contributor or SQL Security Manager.)

## 10. References

- Microsoft Entra-only authentication with Azure SQL: https://learn.microsoft.com/en-us/azure/azure-sql/database/authentication-azure-ad-only-authentication
- Managed identities in Microsoft Entra for Azure SQL: https://learn.microsoft.com/en-us/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity
- Configure and manage Microsoft Entra authentication with Azure SQL: https://learn.microsoft.com/en-us/azure/azure-sql/database/authentication-aad-configure
- CREATE USER, FROM EXTERNAL PROVIDER: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-user-transact-sql
- SERVERPROPERTY, IsExternalAuthenticationOnly: https://learn.microsoft.com/en-us/sql/t-sql/functions/serverproperty-transact-sql
- sqlcmd (Go) authentication methods: https://learn.microsoft.com/en-us/sql/tools/sqlcmd/sqlcmd-authentication
- az sql server ad-only-auth: https://learn.microsoft.com/en-us/cli/azure/sql/server/ad-only-auth
- MSSQLSERVER error 18456: https://learn.microsoft.com/en-us/sql/relational-databases/errors-events/mssqlserver-18456-database-engine-error
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
