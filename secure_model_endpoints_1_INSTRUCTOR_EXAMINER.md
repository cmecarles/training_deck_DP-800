# Instructor-Examiner guide — Secure Model Endpoints 1

Companion to [secure_model_endpoints_1.md](secure_model_endpoints_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a, b, c and d. Read the four requirements and all four options before taking an answer. Options c and d are described as variations of option a, so read option a slowly, naming the identity type, the RBAC role, the outbound rule, the credential name, the IDENTITY value and the SECRET JSON. This is a conceptual Azure question; nothing was executed against an engine.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop AI solutions with SQL (20–25%).
- Skill: Integrate external AI models.
- Task bullet: Secure access to model endpoints from the database.
- What is tested: the managed-identity form of a database scoped credential, the credential naming rules, the minimum Azure OpenAI RBAC role for inference, and outbound firewall rules on an Azure SQL logical server.

## 2. Scenario to read aloud

**Piece 1, the story.** "A patent-search startup stores patent abstracts in an Azure SQL Database named PatentScout, on the logical server patentscout dash sql dot database dot windows dot net. The database must call an Azure OpenAI embedding deployment named embed dash 3 dash small, on the resource patentscout dash aoai, whose endpoint is https colon slash slash patentscout dash aoai dot openai dot azure dot com. It must call it in two ways: from stored procedures through sp underscore invoke underscore external underscore rest underscore endpoint, and through an EXTERNAL MODEL used by AI underscore GENERATE underscore EMBEDDINGS."

**Piece 2, the four requirements.** "Requirement one: no API key for Azure OpenAI may be stored anywhere, not in the database, not in app settings, not in Key Vault. The database must authenticate to Azure OpenAI with its own Microsoft Entra identity. Requirement two: the identity must hold the minimum Azure RBAC role that allows inference, meaning embedding, calls on patentscout dash aoai. Requirement three: outbound traffic from the logical server must be restricted so that sp underscore invoke underscore external underscore rest underscore endpoint can reach patentscout dash aoai dot openai dot azure dot com and nothing else. Requirement four: both the procedure call and the external model must reuse one database scoped credential."

**Piece 3, option a, the Azure side.** "Option a. Enable the system-assigned managed identity of the logical server patentscout dash sql. On the patentscout dash aoai resource, assign that identity the role Cognitive Services OpenAI User. Enable Restrict outbound networking on the server and add the allowed fully qualified domain name patentscout dash aoai dot openai dot azure dot com."

**Piece 4, option a, the T-SQL.** "Still option a. In PatentScout, three statements. First, CREATE DATABASE SCOPED CREDENTIAL, and the credential name, in square brackets, is the URL https colon slash slash patentscout dash aoai dot openai dot azure dot com, nothing after the host. WITH IDENTITY equals the string Managed Identity, and SECRET equals a JSON string with one key, resourceid, whose value is https colon slash slash cognitiveservices dot azure dot com. Second, CREATE EXTERNAL MODEL PatentEmbedder, WITH LOCATION equal to the full deployment URL, that is the host, then slash openai slash deployments slash embed dash 3 dash small slash embeddings, question mark api dash version equals 2024 dash 02 dash 01. API underscore FORMAT is Azure OpenAI, MODEL underscore TYPE is EMBEDDINGS, MODEL is text dash embedding dash 3 dash small, and CREDENTIAL is the credential named after the host URL. Third, EXEC sp underscore invoke underscore external underscore rest underscore endpoint with at url equal to the same full deployment URL, at credential equal to the host-named credential, a payload JSON with input laser diode cooling, and at response as output."

**Piece 5, option b.** "Option b. Store the Azure OpenAI key in the database. CREATE MASTER KEY ENCRYPTION BY PASSWORD. Then CREATE DATABASE SCOPED CREDENTIAL named AzureOpenAI, in square brackets, WITH IDENTITY equals the string HTTPEndpointHeaders, and SECRET equals a JSON with the key api dash key and the value key one of patentscout dash aoai. Pass at credential equals AzureOpenAI to the procedure and CREDENTIAL equals AzureOpenAI to the model. The option argues that because the key is encrypted by the database master key it is not stored in plaintext, and that no RBAC role or outbound rule is needed."

**Piece 6, option c.** "Option c. Same as option a, but assign the server's managed identity the role Cognitive Services Contributor on patentscout dash aoai, since a contributor can do everything a user can do, including creating deployments the team may need later."

**Piece 7, option d.** "Option d. Same as option a, but name the credential after the full deployment URL so that it matches the call exactly. So the credential name, in square brackets, is https colon slash slash patentscout dash aoai dot openai dot azure dot com slash openai slash deployments slash embed dash 3 dash small slash embeddings question mark api dash version equals 2024 dash 02 dash 01. Same IDENTITY Managed Identity and the same resourceid secret."

## 3. Setup script (reference only; do not read verbatim unless asked)

Option a:

```sql
CREATE DATABASE SCOPED CREDENTIAL [https://patentscout-aoai.openai.azure.com]
    WITH IDENTITY = 'Managed Identity',
         SECRET = '{"resourceid":"https://cognitiveservices.azure.com"}';
GO
CREATE EXTERNAL MODEL PatentEmbedder
WITH (LOCATION = 'https://patentscout-aoai.openai.azure.com/openai/deployments/embed-3-small/embeddings?api-version=2024-02-01',
      API_FORMAT = 'Azure OpenAI', MODEL_TYPE = EMBEDDINGS, MODEL = 'text-embedding-3-small',
      CREDENTIAL = [https://patentscout-aoai.openai.azure.com]);
GO
EXEC sp_invoke_external_rest_endpoint
     @url = 'https://patentscout-aoai.openai.azure.com/openai/deployments/embed-3-small/embeddings?api-version=2024-02-01',
     @credential = [https://patentscout-aoai.openai.azure.com],
     @payload = N'{"input":"laser diode cooling"}', @response = @r OUTPUT;
```

Option a, Azure CLI for the outbound rule:

```text
az sql server update ... --set restrictOutboundNetworkAccess="Enabled"
az sql server outbound-firewall-rule create --outbound-rule-fqdn patentscout-aoai.openai.azure.com ...
```

Option b:

```sql
CREATE MASTER KEY ENCRYPTION BY PASSWORD = '...';
CREATE DATABASE SCOPED CREDENTIAL [AzureOpenAI]
    WITH IDENTITY = 'HTTPEndpointHeaders', SECRET = '{"api-key":"<key1 of patentscout-aoai>"}';
```

Option d:

```sql
CREATE DATABASE SCOPED CREDENTIAL
  [https://patentscout-aoai.openai.azure.com/openai/deployments/embed-3-small/embeddings?api-version=2024-02-01]
    WITH IDENTITY = 'Managed Identity',
         SECRET = '{"resourceid":"https://cognitiveservices.azure.com"}';
```

Least-privilege grants from the explanation:

```sql
GRANT EXECUTE ANY EXTERNAL ENDPOINT TO [app-patentscout];
GRANT REFERENCES ON DATABASE SCOPED CREDENTIAL::[https://patentscout-aoai.openai.azure.com]
    TO [app-patentscout];
GRANT EXECUTE ON EXTERNAL MODEL::PatentEmbedder TO [app-patentscout];

SELECT name, credential_identity, create_date FROM sys.database_scoped_credentials;
SELECT name, location, api_format, model_type FROM sys.external_models;
```

## 4. The question (ask exactly this)

"Which configuration meets all four requirements? Option a, option b, option c, or option d?"

Options in full:

- **a.** Server system-assigned managed identity; Cognitive Services OpenAI User on patentscout-aoai; Restrict outbound networking plus allowed FQDN patentscout-aoai.openai.azure.com; credential named `https://patentscout-aoai.openai.azure.com` with IDENTITY 'Managed Identity' and SECRET resourceid `https://cognitiveservices.azure.com`; the same credential in CREATE EXTERNAL MODEL and in sp_invoke_external_rest_endpoint.
- **b.** Master key plus credential `[AzureOpenAI]` with IDENTITY 'HTTPEndpointHeaders' and SECRET api-key; no RBAC role, no outbound rule.
- **c.** Same as a but with the Cognitive Services Contributor role.
- **d.** Same as a but the credential is named after the full deployment URL including `?api-version=2024-02-01`.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

- IDENTITY 'Managed Identity' makes the engine obtain an Entra token with the server's managed identity and send it as a bearer token. The SECRET JSON's resourceid is the token audience, `https://cognitiveservices.azure.com` for Azure OpenAI. No key stored. Requirement one. The system-assigned identity is used when no user-assigned identity is attached; otherwise the server's primary identity.
- Cognitive Services OpenAI User is the least-privileged role that can make inference calls with Entra ID; it cannot see keys, create deployments or fine-tune. Requirement two.
- Outbound firewall rules limit traffic from the logical server to a defined list and explicitly govern sp_invoke_external_rest_endpoint. Restrict outbound networking plus the allowed FQDN. Requirement three. Azure SQL Database also only allows an allow-list of Azure domains, `*.openai.azure.com` among them.
- The credential name must be a valid URL whose scheme plus FQDN match the call, whose path is a prefix of the called path, with no query string. `https://patentscout-aoai.openai.azure.com` is the most generic valid name, so it serves both the procedure and the model. Requirement four.

Why the others are wrong, one line each:

- **b.** HTTPEndpointHeaders with api-key is the key-based pattern; the key is a long-lived secret stored in the database, violating requirement one; `[AzureOpenAI]` is not a URL, so it is not the documented binding; and "no outbound rule" contradicts requirement three.
- **c.** Cognitive Services Contributor is a management-plane role; the role table marks inference calls with Entra ID as not allowed, so the call gets 401 or 403, and it grants key-management rights, violating least privilege.
- **d.** The credential URL must not contain a query string, and it must be more generic than the request URL, not more specific; a deployment-scoped name would also need a second credential for any second deployment, defeating requirement four.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with requirement one. Which IDENTITY value of a database scoped credential makes the engine use its own Entra identity, and which values carry a stored key?"
2. "Requirement two says minimum role for inference. Think about the difference between a management-plane role and a data-plane role on an Azure OpenAI resource."
3. "Requirement four says one credential for two calls. Recall the three naming rules for an endpoint credential: what must match, what may be a prefix, and what is forbidden."
4. "Option b stores key one of the resource inside the credential and says no outbound rule is needed. That fails requirement one and requirement three. That eliminates b."
5. "Option d names the credential after a URL that ends with question mark api dash version. Is a query string allowed in a credential name? That eliminates d."
6. "You are down to a and c. They differ only in the role. One is Cognitive Services OpenAI User. The other is Cognitive Services Contributor. Which one can actually make an inference call, and which one is the smaller grant?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "b, the key is encrypted by the master key so it is not stored in plaintext" | Confuses encrypted-at-rest with not stored | "Is an encrypted key still a long-lived secret that someone must rotate? What does requirement one say about storing a key anywhere?" |
| "c, Contributor includes everything User can do" | Assumes RBAC roles nest by name | "Check the Azure OpenAI role table. Is making inference calls with Entra ID allowed for Cognitive Services Contributor?" |
| "c, the team may need to create deployments later" | Ignores least privilege | "Requirement two says minimum. Should a database identity be able to regenerate keys?" |
| "d, the name should match the call exactly" | Does not know the prefix and no-query-string rules | "Which part of the credential name must match the call exactly, and which part may only be a prefix? Is a question mark allowed?" |
| "a, but it also needs Cognitive Services OpenAI Contributor" | Mixes up the SQL Server 2025 Arc note with Azure SQL Database | "That role appears in the docs for SQL Server 2025 with an Azure Arc identity. For Azure SQL Database, what does the procedure example prescribe?" |
| "a is incomplete without Key Vault" | Thinks a vault is always needed | "With IDENTITY Managed Identity, what secret is there to keep in a vault?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the credential first:

- CREATE DATABASE SCOPED CREDENTIAL for a model endpoint: the name is a URL. Scheme plus FQDN must match the call, the path may only be a prefix, and there must be no query string. The generic `https://host` name is the intended pattern and serves every deployment on that resource.
- IDENTITY 'Managed Identity' with SECRET `{"resourceid":"https://cognitiveservices.azure.com"}` is passwordless: the engine fetches an Entra token for that audience with the server's system-assigned identity, or the primary user-assigned identity, and sends it as a bearer token. Other audiences: `https://eventhubs.azure.net` for Event Hubs, an app's APP_ID for an Entra-protected App Service.
- IDENTITY 'HTTPEndpointHeaders' with `{"api-key":"..."}` is key-based, as are HTTPEndpointQueryString and Shared Access Signature. Valid, but a stored secret.
- The same credential serves sp_invoke_external_rest_endpoint through at credential and CREATE EXTERNAL MODEL through CREDENTIAL.

Then the role:

- Cognitive Services OpenAI User: inference only, the minimum. Cognitive Services OpenAI Contributor: adds deployments and fine-tuning; this is the role the CREATE EXTERNAL MODEL docs name for SQL Server 2025 with an Azure Arc managed identity. Cognitive Services Contributor: management plane, keys and resources, and it cannot call inference. A wrong role shows up as a 401 or 403 in the procedure's return value, not as a T-SQL error; the procedure returns the HTTP status code, zero for any 2xx, and waits on HTTP_EXTERNAL_CONNECTION.

Then the network:

- Azure SQL Database allows only allow-listed domains, such as `*.openai.azure.com` and `*.cognitiveservices.azure.com`. Optional outbound firewall rules narrow that further: Restrict outbound networking on the server plus allowed FQDNs, set with az sql server update and az sql server outbound-firewall-rule create. The feature governs sp_invoke_external_rest_endpoint, auditing, vulnerability assessment, import and export, OPENROWSET and BULK INSERT.
- SQL Server 2025 has no outbound firewall feature; egress is an OS and network matter. It needs sp_configure external rest endpoint enabled set to one, and allow server scoped db credentials set to one for the Azure Arc identity, both followed by RECONFIGURE WITH OVERRIDE.

Then least privilege inside the database: GRANT EXECUTE ANY EXTERNAL ENDPOINT for the procedure, REFERENCES on the credential, and EXECUTE ON EXTERNAL MODEL for AI_GENERATE_EMBEDDINGS. Creating the credential needs CONTROL on the database; creating the model needs CREATE EXTERNAL MODEL or ALTER ANY EXTERNAL MODEL. sys.database_scoped_credentials never shows the secret.

Memory hook: "Name it by the host, no query string. Managed Identity plus resourceid. OpenAI User is enough. Contributor manages but cannot infer. Restrict outbound, allow the FQDN."

## 9. Follow-up oral questions (optional)

1. "What is the resourceid value in the secret for, and what would you put there to call Event Hubs?" (It is the token audience; `https://eventhubs.azure.net` for Event Hubs.)
2. "Which three permissions does an application principal need to use both the procedure and the external model?" (EXECUTE ANY EXTERNAL ENDPOINT, REFERENCES on the database scoped credential, and EXECUTE ON EXTERNAL MODEL.)
3. "On SQL Server 2025, which two sp_configure options gate this scenario?" (external rest endpoint enabled, and allow server scoped db credentials, each set to one with RECONFIGURE WITH OVERRIDE.)

## 10. References

- sp_invoke_external_rest_endpoint, including credential naming rules and managed identity: https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-invoke-external-rest-endpoint-transact-sql
- CREATE EXTERNAL MODEL: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-external-model-transact-sql
- CREATE DATABASE SCOPED CREDENTIAL: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-database-scoped-credential-transact-sql
- Role-based access control for Azure OpenAI: https://learn.microsoft.com/en-us/azure/ai-foundry/openai/how-to/role-based-access-control
- Outbound firewall rules for Azure SQL Database: https://learn.microsoft.com/en-us/azure/azure-sql/database/outbound-firewall-rule-overview
- Managed identities for Azure SQL logical servers: https://learn.microsoft.com/en-us/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity
- AI_GENERATE_EMBEDDINGS: https://learn.microsoft.com/en-us/sql/t-sql/functions/ai-generate-embeddings-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
