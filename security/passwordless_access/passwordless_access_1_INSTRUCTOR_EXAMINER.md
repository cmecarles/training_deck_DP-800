# Instructor-Examiner guide — Passwordless Access 1

Companion to [passwordless_access_1.md](passwordless_access_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a, b, c and d. Read the five requirements and all four options before taking an answer. Each option is a configuration, so describe the connection string keys, the identity type and the T-SQL precisely in words. This is a conceptual Azure question; nothing was executed against an engine.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Implement data security and compliance.
- Task bullet: Implement passwordless access with managed identities and Microsoft Entra authentication.
- What is tested: telling apart a secret-based credential from a token-based one, a system-assigned from a user-assigned managed identity, and the different `Authentication=` modes of the SqlClient connection string, plus the `CREATE USER ... FROM EXTERNAL PROVIDER` database step.

## 2. Scenario to read aloud

**Piece 1, the story.** "A motorway operator runs its toll-processing API as an Azure Functions app, Linux, dot NET isolated. The app reads and writes an Azure SQL Database named TollGrid on the logical server tollgrid dash sql dot database dot windows dot net. The security team has written a baseline with five requirements, and you must pick the configuration that meets all five."

**Piece 2, requirements one and two.** "Requirement one: the application must connect without any secret. No password, no client secret, no connection-string key, not in app settings, not in Key Vault, not in source code. Requirement two: the same code and the same connection string must work on a developer laptop, where the developer is signed in with az login, and in production in Azure, with no code branch per environment."

**Piece 3, requirements three, four and five.** "Requirement three: in production, the Functions app authenticates as the user-assigned managed identity named id dash tollgrid dash api. That identity is shared by two Function apps and must survive the recreation of either app. Requirement four: the logical server has Microsoft Entra-only authentication enabled. SERVERPROPERTY IsExternalAuthenticationOnly returns one, and a Microsoft Entra administrator is configured. Requirement five: the identity must be able to read and write data only. It must not be db underscore owner."

**Piece 4, option a.** "Option a. Create a SQL login named tollapi with a strong password. Store the password in Azure Key Vault. Reference it from app settings with a Key Vault reference, at Microsoft dot KeyVault open paren SecretUri equals dot dot dot close paren. The connection string is Server equals tollgrid dash sql dot database dot windows dot net, Database equals TollGrid, User Id equals tollapi, Password equals the value from Key Vault, Encrypt equals True. In the database: CREATE USER tollapi FOR LOGIN tollapi, then ALTER ROLE db underscore datareader ADD MEMBER tollapi, and ALTER ROLE db underscore datawriter ADD MEMBER tollapi."

**Piece 5, option b.** "Option b. Assign the user-assigned identity id dash tollgrid dash api to both Function apps. The connection string is Server equals tollgrid dash sql dot database dot windows dot net, Database equals TollGrid, Authentication equals Active Directory Default, User Id equals the client ID of id dash tollgrid dash api, Encrypt equals True. The driver is Microsoft dot Data dot SqlClient with the Microsoft dot Data dot SqlClient dot Extensions dot Azure package. As the Entra admin, in TollGrid, run three statements: CREATE USER, in square brackets id dash tollgrid dash api, FROM EXTERNAL PROVIDER. ALTER ROLE db underscore datareader ADD MEMBER that user. ALTER ROLE db underscore datawriter ADD MEMBER that user. Developers are added to an Entra group that has its own contained user, created with CREATE USER, square brackets sg dash tollgrid dash dev, FROM EXTERNAL PROVIDER."

**Piece 6, option c.** "Option c. Enable the system-assigned managed identity on each Function app. The connection string is Server equals tollgrid dash sql dot database dot windows dot net, Database equals TollGrid, Authentication equals Active Directory Managed Identity, Encrypt equals True. No User Id. As the Entra admin, run CREATE USER, square brackets func dash tollgrid dash api, FROM EXTERNAL PROVIDER, and add it to db underscore datareader and db underscore datawriter. The option argues that because the identity is created together with the app, no separate identity resource needs to be managed."

**Piece 7, option d.** "Option d. Register the API as a Microsoft Entra application and connect with Server equals tollgrid dash sql dot database dot windows dot net, Database equals TollGrid, Authentication equals Active Directory Password, User Id equals tollapi at tollgrid dot example, Password equals an Entra password, Encrypt equals True. Create the user with CREATE USER, square brackets tollapi at tollgrid dot example, FROM EXTERNAL PROVIDER, and add it to db underscore datareader and db underscore datawriter. The option argues that because the account is an Entra identity, the connection is compliant with Entra-only authentication."

## 3. Setup script (reference only; do not read verbatim unless asked)

Option a, connection string and T-SQL:

```text
Server=tollgrid-sql.database.windows.net;Database=TollGrid;User Id=tollapi;Password=<from Key Vault>;Encrypt=True
App setting value: @Microsoft.KeyVault(SecretUri=...)
```

```sql
CREATE USER tollapi FOR LOGIN tollapi;
ALTER ROLE db_datareader ADD MEMBER tollapi;
ALTER ROLE db_datawriter ADD MEMBER tollapi;
```

Option b, connection string and T-SQL:

```text
Server=tollgrid-sql.database.windows.net;Database=TollGrid;Authentication=Active Directory Default;User Id=<client ID of id-tollgrid-api>;Encrypt=True
```

```sql
CREATE USER [id-tollgrid-api] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [id-tollgrid-api];
ALTER ROLE db_datawriter ADD MEMBER [id-tollgrid-api];
CREATE USER [sg-tollgrid-dev] FROM EXTERNAL PROVIDER;
```

Option c, connection string and T-SQL:

```text
Server=tollgrid-sql.database.windows.net;Database=TollGrid;Authentication=Active Directory Managed Identity;Encrypt=True
```

```sql
CREATE USER [func-tollgrid-api] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [func-tollgrid-api];
ALTER ROLE db_datawriter ADD MEMBER [func-tollgrid-api];
```

Option d, connection string and T-SQL:

```text
Server=tollgrid-sql.database.windows.net;Database=TollGrid;Authentication=Active Directory Password;User Id=tollapi@tollgrid.example;Password=<Entra password>;Encrypt=True
```

```sql
CREATE USER [tollapi@tollgrid.example] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [tollapi@tollgrid.example];
ALTER ROLE db_datawriter ADD MEMBER [tollapi@tollgrid.example];
```

Catalog check from the explanation:

```sql
SELECT name, type, type_desc, CAST(sid AS UNIQUEIDENTIFIER) AS EntraObjectId
FROM sys.database_principals
WHERE type IN ('E', 'X');       -- E = external user/app, X = external group
```

## 4. The question (ask exactly this)

"Which configuration meets all five requirements? Option a, option b, option c, or option d?"

Options in full:

- **a.** SQL login `tollapi` with a password stored in Key Vault, referenced from app settings; connection string with `User Id=tollapi;Password=...`; `CREATE USER tollapi FOR LOGIN tollapi` plus db_datareader and db_datawriter.
- **b.** User-assigned identity `id-tollgrid-api` assigned to both apps; connection string with `Authentication=Active Directory Default;User Id=<client ID>`; `CREATE USER [id-tollgrid-api] FROM EXTERNAL PROVIDER` plus db_datareader and db_datawriter; developers via an Entra group user `sg-tollgrid-dev`.
- **c.** System-assigned identity on each app; connection string with `Authentication=Active Directory Managed Identity`; `CREATE USER [func-tollgrid-api] FROM EXTERNAL PROVIDER` plus the two roles.
- **d.** Entra application plus `Authentication=Active Directory Password;User Id=tollapi@tollgrid.example;Password=...`; `CREATE USER [tollapi@tollgrid.example] FROM EXTERNAL PROVIDER` plus the two roles.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: b.**

- A managed identity is passwordless: the app obtains an access token for `https://database.windows.net/` from the identity endpoint, no secret anywhere. Requirement one.
- `Authentication=Active Directory Default` uses the DefaultAzureCredential chain: EnvironmentCredential, WorkloadIdentityCredential, ManagedIdentityCredential, SharedTokenCacheCredential, VisualStudioCredential, VisualStudioCodeCredential, AzurePowerShellCredential, AzureCliCredential, AzureDeveloperCliCredential. In Azure the managed identity step wins; on the laptop it falls through to the Azure CLI token. One connection string for both. Requirement two.
- The user-assigned identity is a standalone resource, shareable across apps, surviving app recreation. Its client ID goes in `User Id=`. Requirement three.
- `CREATE USER [id-tollgrid-api] FROM EXTERNAL PROVIDER` creates a contained user of type E, EXTERNAL_USER, run by an Entra principal with ALTER ANY USER, usually the Entra admin. Token-based, so allowed under Entra-only. Requirement four. Membership in db_datareader and db_datawriter only. Requirement five.

Why the others are wrong, one line each:

- **a.** A password in Key Vault is still a secret (requirement one), and Entra-only authentication disables SQL authentication, so `tollapi` can be created but can never connect (requirement four).
- **c.** A system-assigned identity is one per resource and dies with it, so two apps need two identities and recreating an app orphans the database user (requirement three); and `Active Directory Managed Identity` only calls the managed-identity endpoint, which does not exist on a laptop (requirement two).
- **d.** `Active Directory Password` is the ROPC flow: the app holds a user password in configuration (requirement one), it is incompatible with mandatory MFA, and it is deprecated and marked Obsolete in Microsoft.Data.SqlClient 7.0. Entra users must not be used as service accounts.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with requirement one. Read each option and ask: is there a password or a secret anywhere, even one that is stored somewhere safe?"
2. "Now requirement four. Entra-only authentication disables one whole kind of authentication. Which kind, and which option depends on it?"
3. "Requirement three talks about an identity shared by two apps that survives recreation. Which kind of managed identity is tied to a single resource and deleted with it?"
4. "Option a stores a SQL password in Key Vault. A secret in a vault is still a secret, and SQL logins cannot connect when Entra-only is on. That eliminates a."
5. "Option d uses Authentication equals Active Directory Password. Whose password is that, and where does the app keep it? That eliminates d."
6. "You are down to b and c. One uses a system-assigned identity with Active Directory Managed Identity. The other uses a user-assigned identity with Active Directory Default and a client ID in User Id. Which one works both on a laptop with az login and in Azure, and can be shared by two apps?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, the password is in Key Vault so it is not in code" | Equates secret-not-in-code with passwordless | "Does a Key Vault reference remove the secret, or just move it? And can a SQL login connect at all under Entra-only authentication?" |
| "c, managed identity is the recommended passwordless pattern" | Stops at the technology, ignores the scenario | "It is a good pattern in Azure. Now check requirement three: how many identities would two apps have, and what happens when one app is recreated?" |
| "c, the laptop can use the same string" | Thinks the Managed Identity mode has a fallback | "Active Directory Managed Identity calls exactly one endpoint. Does a developer laptop have that endpoint?" |
| "d, it is an Entra identity so Entra-only allows it" | Confuses Entra authentication with passwordless | "Entra-only does allow it. Now read requirement one again. Is there a password in the connection string?" |
| "b, but User Id should be the object ID" | Version confusion | "In Microsoft dot Data dot SqlClient three and later, which identifier of a user-assigned identity goes in User Id?" (Client ID.) |
| "b needs an Azure RBAC role like SQL DB Contributor as well" | Confuses control-plane RBAC with database access | "Does an Azure RBAC role ever grant permissions inside the database? What statement actually creates the principal?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the three things the exam wants you to keep apart:

- **Credential type.** Passwordless means token-based Entra authentication with a managed identity or workload identity. A password in Key Vault or app settings is still a secret. An Entra user plus password is the ROPC flow, deprecated and MFA-incompatible. The guidance says: do not use an Entra user account as a service account. Use a managed identity in Azure, a service principal, preferably with a certificate, outside Azure, and Active Directory Interactive only for humans.
- **Identity type.** System-assigned: one per resource, shares the resource lifecycle, cannot be shared across resources. User-assigned: a standalone Azure resource, attachable to several apps, survives app recreation. Its client ID goes in the User Id property of the connection string, in Microsoft.Data.SqlClient 3.0 and later.
- **Authentication mode in the connection string.** Active Directory Managed Identity, alias Active Directory MSI, calls only the managed-identity endpoint, Azure only. Active Directory Default runs the DefaultAzureCredential chain, environment variables, workload identity, managed identity, Visual Studio, VS Code, Azure PowerShell, Azure CLI, Azure Developer CLI, so one string serves laptop and cloud, at the cost of some connection latency. Active Directory Interactive is for humans with browser and MFA. Active Directory Service Principal takes an app registration client id plus secret or certificate. Active Directory Password is deprecated.

Then the database side:

- `CREATE USER [display name] FROM EXTERNAL PROVIDER` creates a contained user. In sys.database_principals it has type E for an external user or application, or X for an external group. The SID is the binary form of the Entra object id; cast it to UNIQUEIDENTIFIER to map it back. Only an Entra principal with ALTER ANY USER can run it, because the engine validates the identity through Microsoft Graph. Then ALTER ROLE db_datareader and db_datawriter ADD MEMBER.
- Azure RBAC roles such as SQL Server Contributor or SQL DB Contributor never grant database access. Entra authentication for Azure SQL does not integrate with Azure RBAC.
- Entra-only authentication disables SQL logins. They still exist, but they cannot connect. SERVERPROPERTY IsExternalAuthenticationOnly returns one.
- The same pattern exists outside Azure: SQL Server 2022 and 2025 get Entra authentication once the host is connected to Azure Arc, or through the SQL IaaS Agent extension on an Azure VM. Failover cluster instances are not supported. The statements are the same.

Memory hook: "Vault is still a secret. System-assigned dies with the app. Default falls through to az login. FROM EXTERNAL PROVIDER, then roles."

## 9. Follow-up oral questions (optional)

1. "Which SqlClient authentication mode would you use for an on-premises service that must connect to Azure SQL without a managed identity?" (Active Directory Service Principal, preferably with a certificate.)
2. "How can you find the Entra object id of a contained external user from inside the database?" (Cast the sid column of sys.database_principals to UNIQUEIDENTIFIER, filtering type E or X.)
3. "Does granting the SQL DB Contributor RBAC role to the identity let it read the TollGrid tables?" (No. RBAC is control plane only; the identity still needs a database user and database permissions.)

## 10. References

- Managed identities for Azure SQL: https://learn.microsoft.com/en-us/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity
- Microsoft Entra authentication for Azure SQL overview: https://learn.microsoft.com/en-us/azure/azure-sql/database/authentication-aad-overview
- Microsoft Entra-only authentication: https://learn.microsoft.com/en-us/azure/azure-sql/database/authentication-azure-ad-only-authentication
- Using Microsoft Entra authentication with SqlClient, Authentication keyword values: https://learn.microsoft.com/en-us/sql/connect/ado-net/sql/azure-active-directory-authentication
- DefaultAzureCredential chain, Azure Identity for .NET: https://learn.microsoft.com/en-us/dotnet/api/azure.identity.defaultazurecredential
- CREATE USER, FROM EXTERNAL PROVIDER: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-user-transact-sql
- Managed identities overview: https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview
- Microsoft Entra authentication for SQL Server 2022 and later via Azure Arc: https://learn.microsoft.com/en-us/sql/relational-databases/security/authentication-access/azure-ad-authentication-sql-server-overview
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
