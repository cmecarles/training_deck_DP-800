# Instructor-Examiner guide — Foundry Embedding Maintenance 1

Companion to [foundry_embedding_maintenance_1.md](foundry_embedding_maintenance_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a conceptual multiple-choice design question about Azure resources, endpoints, credentials and orchestration. Nothing runs against an engine. There are four options, a to d, and four requirements. Read all four requirements and all four options before taking an answer. When the learner picks a letter, ask them to say which requirement each of the other three options fails; that is where the learning is.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Implement AI capabilities in database solutions (25–30%).
- Skill: Design and implement models and embeddings.
- Task bullet: Choose an embedding maintenance method, including Microsoft Foundry.
- What is tested: where the embedding models are hosted (Azure OpenAI resource, Foundry resource, hub with serverless endpoint), how the database reaches them (the CREATE EXTERNAL MODEL location, the allowed FQDN, the managed identity credential), and what orchestrates the refresh (prompt agent plus routine, versus Logic Apps or prompt flow, and what is retiring).

## 2. Scenario to read aloud

**Piece 1, the story.** "A regional transit agency keeps its rider-facing service alerts in an Azure SQL Database called TransitTimes. Alerts are embedded for semantic search with an Azure OpenAI deployment named text dash embedding dash 3 dash small. That deployment lives on an Azure OpenAI resource named transittimes dash aoai. Its endpoint is https colon slash slash transittimes dash aoai dot openai dot azure dot com. Today the database authenticates to that resource with an API key."

**Piece 2, the table.** "There is one table, in a schema called Ops, named Alerts. It has eight columns. AlertId, an integer, the primary key. Title, an NVARCHAR of two hundred, not null. Body, an NVARCHAR MAX, not null. SignPhotoUrl, an NVARCHAR of four hundred, nullable; it points to a photo of the printed station sign. ContentHash, a BINARY of thirty two, not null; it is the SHA2 256 hash of Title plus Body, maintained by the application. EmbeddedHash, a BINARY of thirty two, nullable; it is the hash that the text embedding was computed from. TextEmbedding, a VECTOR of one thousand five hundred thirty six, nullable, for text dash embedding dash 3 dash small. And SignEmbedding, a VECTOR of one thousand twenty four, nullable, reserved for the new image plus text model."

**Piece 3, the credential and the external model.** "A database master key is created with a password. Then a database scoped credential is created. Its name is the URL itself, https colon slash slash transittimes dash aoai dot openai dot azure dot com. Its identity is the string HTTPEndpointHeaders, and its secret is a JSON document with one key, api dash key, whose value is redacted. Then an external model named AlertEmbedder is created. Its LOCATION is the Azure OpenAI URL: the resource endpoint, then slash openai slash deployments slash text dash embedding dash 3 dash small slash embeddings, with api dash version 2024 dash 02 dash 01. Its API underscore FORMAT is Azure OpenAI. Its MODEL underscore TYPE is EMBEDDINGS. Its MODEL is text dash embedding dash 3 dash small. And its CREDENTIAL is that same credential named after the URL."

**Piece 4, the nightly sweep procedure.** "A stored procedure, Ops dot usp underscore RefreshStaleTextEmbeddings, does the nightly sweep. It updates Ops dot Alerts. It sets TextEmbedding to AI underscore GENERATE underscore EMBEDDINGS of Title and Body concatenated with a pipe separator, USE MODEL AlertEmbedder. In the same update it sets EmbeddedHash equal to ContentHash. Its WHERE clause keeps only rows where EmbeddedHash is null or EmbeddedHash differs from ContentHash. So it re-embeds only the alerts whose text changed since they were last embedded."

**Piece 5, the new need.** "The agency now wants to search station sign photos as well as text. They will use the Cohere multimodal embedding model embed dash v dash 4 dash 0 from the Microsoft Foundry model catalog. It takes text and image input and can output vectors of 256, 512, 1,024 or 1,536 dimensions. Those vectors will be written into SignEmbedding by an Azure Function that calls the model, because CREATE EXTERNAL MODEL has no API format for that model family. The nightly text sweep must keep running."

**Piece 6, the four requirements.** "The redesign must satisfy all four of these. Requirement 1. Both deployments, text dash embedding dash 3 dash small and embed dash v dash 4 dash 0, must live on the single existing Azure AI resource transittimes dash aoai, which keeps its name and its endpoints. Governance forbids a second AI account, hub or workspace. Requirement 2. The T-SQL side must keep working unchanged: AlertEmbedder keeps its current LOCATION and API FORMAT, and the procedure is not rewritten. Requirement 3. No API keys anywhere. The database authenticates to the model endpoint with the logical server's managed identity, and key based, that is local, authentication is disabled on the resource. Requirement 4. The nightly sweep is orchestrated by a Foundry prompt agent, because the ops team wants run history and traces in the Foundry project. The schedule must live inside the Foundry project, no separate scheduler resource, and must not depend on a feature Microsoft has already announced for retirement. Preview features are acceptable for the scheduler."

**Piece 7, option a.** "Option a. Upgrade transittimes dash aoai in place from an Azure OpenAI resource to a Foundry resource. In the resource definition that means kind equals AIServices, allowProjectManagement equals true, disableLocalAuth equals true, and the system assigned identity turned on. Create a Foundry project in that resource. Deploy embed dash v dash 4 dash 0 from the model catalog as a standard deployment on that same resource. Leave AlertEmbedder and the procedure untouched. Replace the key credential: drop the database scoped credential named after the openai dot azure dot com URL, and create it again with the same name, but now with IDENTITY equals Managed Identity and a SECRET JSON with one key, resourceid, whose value is https colon slash slash cognitiveservices dot azure dot com. Grant the logical server's managed identity the Cognitive Services OpenAI User role on transittimes dash aoai. In the project, create a prompt agent whose tool is an Azure Function, exposed through an OpenAPI or MCP tool definition, that runs the refresh procedure and then embeds any alert with a SignPhotoUrl and a null SignEmbedding through the embed dash v dash 4 dash 0 deployment. Finally create a routine with a recurring trigger, cron style, two a.m. daily, whose action invokes that agent."

**Piece 8, option b.** "Option b. Same upgrade, same Cohere deployment, same managed identity credential, same role assignment, same agent and same routine as option a. But because the upgraded resource now exposes a Foundry endpoint, option b points the external model at it so both models share one host name. It runs ALTER EXTERNAL MODEL AlertEmbedder SET LOCATION to https colon slash slash transittimes dash aoai dot services dot ai dot azure dot com, then the same path, slash openai slash deployments slash text dash embedding dash 3 dash small slash embeddings, same api dash version. It drops the credential named after the openai dot azure dot com URL and creates a new credential named after the services dot ai dot azure dot com URL, with IDENTITY equals Managed Identity and the same resourceid secret, cognitiveservices dot azure dot com."

**Piece 9, option c.** "Option c. Keep transittimes dash aoai as an Azure OpenAI resource, with no upgrade, so the text deployment is not disturbed. Create a Foundry hub with a hub based project and deploy embed dash v dash 4 dash 0 there as a serverless API endpoint, pay as you go. Keep an API key credential for that endpoint, because serverless API endpoints do not support keyless authentication. Orchestrate the nightly sweep with a prompt flow in the hub based project: an LLM node plus a Python node that calls the database. Schedule the flow's batch run nightly."

**Piece 10, option d.** "Option d. Same upgrade, same Cohere deployment, same agent, and unchanged AlertEmbedder as option a. But because the upgrade preserves the resource's API key, option d keeps the existing HTTPEndpointHeaders credential unchanged for the database, and sets disableLocalAuth equals true on the resource so that the agent runtime uses Microsoft Entra ID. It schedules the sweep with an Azure Logic Apps Consumption workflow: a Recurrence trigger followed by the Foundry Agent Service connector actions Create Thread, Create Run, Get Run and List Messages."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE TransitTimes;
GO
USE TransitTimes;
GO
CREATE SCHEMA Ops;
GO
CREATE TABLE Ops.Alerts
(
    AlertId        INT            NOT NULL PRIMARY KEY,
    Title          NVARCHAR(200)  NOT NULL,
    Body           NVARCHAR(MAX)  NOT NULL,
    SignPhotoUrl   NVARCHAR(400)  NULL,          -- photo of the printed station sign
    ContentHash    BINARY(32)     NOT NULL,      -- SHA2_256 of Title + Body, maintained by the app
    EmbeddedHash   BINARY(32)     NULL,          -- hash that TextEmbedding was computed from
    TextEmbedding  VECTOR(1536)   NULL,          -- text-embedding-3-small
    SignEmbedding  VECTOR(1024)   NULL           -- reserved for the new image+text model
);
GO
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'Tr@nsitTimes!2026';
GO
CREATE DATABASE SCOPED CREDENTIAL [https://transittimes-aoai.openai.azure.com]
WITH IDENTITY = 'HTTPEndpointHeaders', SECRET = '{"api-key":"REDACTED"}';
GO
CREATE EXTERNAL MODEL AlertEmbedder
WITH (
    LOCATION   = 'https://transittimes-aoai.openai.azure.com/openai/deployments/text-embedding-3-small/embeddings?api-version=2024-02-01',
    API_FORMAT = 'Azure OpenAI',
    MODEL_TYPE = EMBEDDINGS,
    MODEL      = 'text-embedding-3-small',
    CREDENTIAL = [https://transittimes-aoai.openai.azure.com]
);
GO
-- Nightly sweep: re-embed only the alerts whose text changed since they were last embedded.
CREATE PROCEDURE Ops.usp_RefreshStaleTextEmbeddings
AS
BEGIN
    SET NOCOUNT ON;
    UPDATE a
    SET TextEmbedding = AI_GENERATE_EMBEDDINGS(CONCAT_WS(N' | ', a.Title, a.Body) USE MODEL AlertEmbedder),
        EmbeddedHash  = a.ContentHash
    FROM Ops.Alerts AS a
    WHERE a.EmbeddedHash IS NULL OR a.EmbeddedHash <> a.ContentHash;
END;
GO
```

Option a, credential replacement:

```sql
DROP DATABASE SCOPED CREDENTIAL [https://transittimes-aoai.openai.azure.com];
CREATE DATABASE SCOPED CREDENTIAL [https://transittimes-aoai.openai.azure.com]
WITH IDENTITY = 'Managed Identity', SECRET = '{"resourceid":"https://cognitiveservices.azure.com"}';
```

Option b, external model and credential changes:

```sql
ALTER EXTERNAL MODEL AlertEmbedder
SET (LOCATION = 'https://transittimes-aoai.services.ai.azure.com/openai/deployments/text-embedding-3-small/embeddings?api-version=2024-02-01');
DROP DATABASE SCOPED CREDENTIAL [https://transittimes-aoai.openai.azure.com];
CREATE DATABASE SCOPED CREDENTIAL [https://transittimes-aoai.services.ai.azure.com]
WITH IDENTITY = 'Managed Identity', SECRET = '{"resourceid":"https://cognitiveservices.azure.com"}';
```

Option a resource properties, in words: `kind: 'AIServices'`, `allowProjectManagement: true`, `disableLocalAuth: true`, system-assigned managed identity enabled. Role: Cognitive Services OpenAI User for the logical server's managed identity on `transittimes-aoai`. Scheduler: a Foundry routine with a recurring (cron-style) trigger at 02:00 daily invoking the prompt agent.

## 4. The question (ask exactly this)

"Which design meets all four requirements?"

- a. Upgrade `transittimes-aoai` in place from an Azure OpenAI resource to a Foundry resource (`kind: 'AIServices'`, `allowProjectManagement: true`, `disableLocalAuth: true`, system-assigned identity on), and create a Foundry project in it. Deploy `embed-v-4-0` from the model catalog as a standard deployment on that resource. Leave `AlertEmbedder` and the procedure untouched. Replace the key credential with `DROP DATABASE SCOPED CREDENTIAL [https://transittimes-aoai.openai.azure.com]; CREATE DATABASE SCOPED CREDENTIAL [https://transittimes-aoai.openai.azure.com] WITH IDENTITY = 'Managed Identity', SECRET = '{"resourceid":"https://cognitiveservices.azure.com"}';` and grant the logical server's managed identity the Cognitive Services OpenAI User role on `transittimes-aoai`. In the project, create a prompt agent whose tool is an Azure Function (exposed through an OpenAPI/MCP tool definition) that runs `Ops.usp_RefreshStaleTextEmbeddings` and then embeds any alert with a `SignPhotoUrl` and a `NULL` `SignEmbedding` through the `embed-v-4-0` deployment. Create a routine with a recurring trigger (cron-style, 02:00 daily) whose action invokes that agent.
- b. Same upgrade, Cohere deployment, managed-identity credential, role assignment, agent, and routine as option a, but because the upgraded resource now exposes a Foundry endpoint, point the external model at it so both models share one host name: `ALTER EXTERNAL MODEL AlertEmbedder SET (LOCATION = 'https://transittimes-aoai.services.ai.azure.com/openai/deployments/text-embedding-3-small/embeddings?api-version=2024-02-01'); DROP DATABASE SCOPED CREDENTIAL [https://transittimes-aoai.openai.azure.com]; CREATE DATABASE SCOPED CREDENTIAL [https://transittimes-aoai.services.ai.azure.com] WITH IDENTITY = 'Managed Identity', SECRET = '{"resourceid":"https://cognitiveservices.azure.com"}';`
- c. Keep `transittimes-aoai` as an Azure OpenAI resource (no upgrade) so the text deployment is not disturbed. Create a Foundry hub with a hub-based project and deploy `embed-v-4-0` there as a serverless API endpoint (pay-as-you-go); keep an API-key credential for that endpoint, because serverless API endpoints do not support keyless authentication. Orchestrate the nightly sweep with a prompt flow in the hub-based project (an LLM node plus a Python node that calls the database), and schedule the flow's batch run nightly.
- d. Same upgrade, Cohere deployment, agent, and unchanged `AlertEmbedder` as option a. Because the upgrade preserves the resource's API key, keep the existing `HTTPEndpointHeaders` credential unchanged for the database, and set `disableLocalAuth: true` on the resource so that the agent runtime uses Microsoft Entra ID. Schedule the sweep with an Azure Logic Apps Consumption workflow: a Recurrence trigger followed by the Foundry Agent Service connector actions Create Thread, Create Run, Get Run, and List Messages.

After the learner picks a letter: "Now, for each of the other three options, tell me which requirement it fails."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

| Requirement | a | b | c | d |
|---|---|---|---|---|
| 1. One resource, existing name and endpoints | satisfied | satisfied | violated (hub plus serverless endpoint) | satisfied |
| 2. LOCATION and API_FORMAT unchanged | satisfied | violated | satisfied | satisfied |
| 3. No keys, managed identity, local auth off | satisfied | satisfied | violated | violated |
| 4. Agent plus in-project schedule, nothing retiring | satisfied | satisfied | violated (prompt flow) | violated (Logic Apps, classic connector) |

- **a is correct.** The in-place upgrade from `kind: OpenAI` to `kind: AIServices` with `allowProjectManagement: true` keeps the resource name, the `openai.azure.com` endpoint, the RBAC and the network settings, and unlocks the full catalog (Cohere included), projects and agents. `disableLocalAuth: true` turns keys off. The database switches to `IDENTITY = 'Managed Identity'` with `resourceid` `https://cognitiveservices.azure.com` and the Cognitive Services OpenAI User role, so `AlertEmbedder` keeps its LOCATION and API_FORMAT and the procedure is untouched. A prompt agent with a Function tool, invoked by a routine with a recurring cron-style trigger, is a schedule inside the project with run history and traces, and routines are preview, not retiring.
- **b is wrong, requirement 2.** It rewrites `LOCATION` to the `services.ai.azure.com` host, which the requirement forbids. It would also not work: `*.services.ai.azure.com` is not in Azure SQL Database's outbound allow list for `sp_invoke_external_rest_endpoint` and `CREATE EXTERNAL MODEL`, and the credential's protocol plus FQDN must match the called URL, so the new credential is rejected for the same reason.
- **c is wrong, requirements 1, 3 and 4.** A serverless API endpoint lives only in a hub resource, so the Cohere deployment lands on a second resource. Serverless API endpoints have no keyless authentication, so a key comes back. Prompt flow is announced for retirement on April 20, 2027, is no longer recommended for new development, and only works with hub-based projects.
- **d is wrong, requirements 3 and 4.** It keeps the `HTTPEndpointHeaders` api-key credential, which violates the no-keys rule, and it contradicts itself: `disableLocalAuth: true` disables key authentication for the whole resource, so that credential stops working and `AI_GENERATE_EMBEDDINGS` fails to authenticate. Logic Apps is a separate scheduler resource outside the project, and the Create Thread, Create Run, Get Run, List Messages sequence is the classic threads-and-runs Agent Service connector, deprecated and retiring March 31, 2027, on top of the Assistants API that retires August 26, 2026.

## 6. Hint ladder (one hint per attempt, in order)

1. "Do not judge the options as a whole yet. Take requirement 2 alone. It says AlertEmbedder keeps its current LOCATION. Which option changes that LOCATION? Cross it out."
2. "Now requirement 3, no API keys anywhere. Two options still keep a key somewhere. One admits it openly because of the kind of endpoint it chose. The other keeps the old credential and at the same time flips a switch on the resource. Find both."
3. "Think about that switch. disableLocalAuth equals true turns off key authentication for the whole resource. If the database still sends an api dash key header, what happens to the nightly sweep?"
4. "Now requirement 1, one resource, the existing name. Which option creates a hub with a serverless endpoint? Where does that Cohere deployment actually live, on transittimes dash aoai or somewhere else?"
5. "Requirement 4 has three parts: a prompt agent, a schedule inside the project, and nothing already announced for retirement. Prompt flow has a published retirement date. The Create Thread, Create Run, Get Run, List Messages connector is the classic threads and runs model, also deprecated. And a Logic App is a separate resource. Which option is left with an agent and a schedule that lives inside the project?"
6. "You are down to two options that share everything except one thing: the host name in the LOCATION. Does the T-SQL side need to move to the services dot ai dot azure dot com endpoint after the upgrade, or does the old openai dot azure dot com endpoint keep answering?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "b, because after the upgrade everything should use the Foundry endpoint" | Believes the upgrade replaces the Azure OpenAI FQDN | "Does the upgrade remove the old endpoint, or add new ones beside it? And re-read requirement 2 word for word." |
| "b, one host name for both models is cleaner" | Judges elegance, not the requirements and the allow list | "Cleaner, perhaps. Is that host name one that Azure SQL Database is allowed to call from T-SQL at all?" |
| "c, because an in-place upgrade might break the text deployment" | Thinks the upgrade is a migration that disturbs deployments | "What does the upgrade change in the resource definition? Just the kind and one property. What stays the same? And count the resources in option c." |
| "c, prompt flow is the Foundry way to orchestrate" | Not aware of the prompt flow retirement and hub-only support | "Requirement 4 mentions features already announced for retirement. Does prompt flow have such an announcement? And does it work with a Foundry project, or only a hub-based one?" |
| "d, the upgrade keeps the API key so the credential can stay" | Misses that disableLocalAuth kills key auth for every caller | "The key still exists, true. But the option also sets disableLocalAuth to true. Who else was using a key on that resource?" |
| "d, Logic Apps with Recurrence is a valid scheduler" | Ignores the in-project constraint and the classic connector | "Valid in general, yes. Where does the requirement say the schedule must live? And are Create Thread and Create Run the current agent model or the classic one?" |
| "a, but the routine is preview so it cannot be right" | Confuses preview with retirement | "Re-read the last sentence of requirement 4. What does it allow for the scheduler?" |
| "a, but the Cohere model should be called by AI GENERATE EMBEDDINGS too" | Thinks CREATE EXTERNAL MODEL covers every catalog model | "Which API formats does CREATE EXTERNAL MODEL accept? Is there one for Cohere?" |
| "None. The Cohere vectors are 1,024 wide and the column is 1,536" | Mixes up the two vector columns | "There are two vector columns. Which one is reserved for the sign embeddings, and how wide is it?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the three places where Microsoft Foundry enters an embedding pipeline: the resource, the endpoint, and the orchestrator.

**The resource.**

- An Azure OpenAI resource hosts Azure OpenAI models only and has no agents.
- A Foundry resource is the same account after an in-place upgrade. In Bicep or ARM the patch changes `kind` from `OpenAI` to `AIServices` and sets `allowProjectManagement: true`. The upgrade keeps the resource name, tags, network configuration, identity and access configuration, API endpoint and API key, custom domain and existing state. It adds the full model catalog (Cohere, Mistral, DeepSeek and others), projects and the Agent service. `disableLocalAuth: true` is the documented property for Entra-only access. That is why option a satisfies requirement 1 and the local-auth half of requirement 3.
- A hub with a serverless API endpoint is a separate hub resource, regional, and has no keyless authentication. That is why option c fails requirement 1 and requirement 3 at once.
- Cohere `embed-v-4-0` takes text and images and outputs 256, 512, 1,024 or 1,536 dimensions. It is deployed on the Foundry resource and called by application code or an agent tool, never by `AI_GENERATE_EMBEDDINGS`, because `CREATE EXTERNAL MODEL` only knows the formats Azure OpenAI, OpenAI, Ollama and ONNX Runtime. Its vectors go in their own `VECTOR(1024)` column, `SignEmbedding`. One column, one model, one dimension count.

**The endpoint.**

- T-SQL keeps talking to `https://<resource>.openai.azure.com/openai/deployments/<deployment>/embeddings?api-version=...`. After the upgrade the resource exposes three FQDNs: `openai.azure.com`, `services.ai.azure.com` and `cognitiveservices.azure.com`. The `services.ai.azure.com` one is for the SDK, the Responses API and agents. It is not in the outbound allow list for `sp_invoke_external_rest_endpoint`, which `CREATE EXTERNAL MODEL` follows, and the credential name must match the called URL's protocol and FQDN. That is why option b fails: it breaks requirement 2 by rewriting LOCATION, and it would not connect anyway.
- Keyless from the database means a credential with `IDENTITY = 'Managed Identity'` and `SECRET = '{"resourceid":"https://cognitiveservices.azure.com"}'`, plus the Cognitive Services OpenAI User role for the logical server's managed identity on the resource. That role keeps working after the upgrade. Only after every caller uses tokens do you set `disableLocalAuth: true`. Option d does it backwards: it disables local auth while the database still sends an api-key header, so the sweep stops authenticating.

**The orchestrator.**

- A prompt agent is defined by configuration: instructions, model and tools. Tools can be functions, OpenAPI specs and MCP servers, including MCP servers hosted on Azure Functions.
- A routine, in preview, runs an agent when a trigger fires: timer, recurring on a cron-style schedule, or event. No separate scheduler resource. Run history and traces are recorded in the project. That is a schedule inside the project, and preview is allowed by the requirement.
- Logic Apps Recurrence and a Functions timer are external schedulers. They are fine patterns in general, but not under this scenario. And the Create Thread, Create Run, Get Run, List Messages actions are the classic agents connector, deprecated and retiring in March 2027, on top of an Assistants API that retires in August 2026.
- Prompt flow retires April 20, 2027, is no longer recommended for new development, and only works with hub-based projects.

Memory hook: "Upgrade in place, keep the openai dot azure dot com URL, managed identity plus OpenAI User role, then flip local auth off. Agent plus routine schedules inside the project. Prompt flow and threads and runs are on their way out."

## 9. Follow-up oral questions (optional)

1. "After the in-place upgrade, which three host names does the resource answer on, and which one may Azure SQL Database call from CREATE EXTERNAL MODEL?" (openai dot azure dot com, services dot ai dot azure dot com and cognitiveservices dot azure dot com; the database uses openai dot azure dot com, the services dot ai host is not in the allow list.)
2. "What are the four API formats that CREATE EXTERNAL MODEL accepts, and why does that force an Azure Function for the Cohere model?" (Azure OpenAI, OpenAI, Ollama and ONNX Runtime; there is no Cohere format, so the model must be called from code or an agent tool.)
3. "What is the safe order of steps to go keyless on the resource?" (First switch every caller to token authentication, here the managed identity credential and the role assignment; then set disableLocalAuth to true.)

## 10. References

- Upgrade Azure OpenAI to Microsoft Foundry: https://learn.microsoft.com/en-us/azure/ai-foundry/openai/how-to/upgrade-azure-openai
- Deployment options for Foundry Models (standard deployment versus serverless API endpoint, keyless authentication): https://learn.microsoft.com/en-us/azure/ai-foundry/concepts/deployments-overview
- Foundry Models from partners and community, including Cohere embed-v-4-0: https://learn.microsoft.com/en-us/azure/ai-foundry/foundry-models/concepts/models-from-partners
- Authentication in Microsoft Foundry (keys versus Microsoft Entra ID, disableLocalAuth): https://learn.microsoft.com/en-us/azure/ai-foundry/concepts/authentication
- Foundry Agent Service overview (prompt agents, tools): https://learn.microsoft.com/en-us/azure/ai-foundry/agents/overview
- Routines in Foundry Agent Service (timer, recurring and event triggers): https://learn.microsoft.com/en-us/azure/ai-foundry/agents/concepts/routines
- Prompt flow in Microsoft Foundry, retirement notice: https://learn.microsoft.com/en-us/azure/ai-foundry/concepts/prompt-flow
- Azure Logic Apps, Foundry Agent Service connector (classic, threads and runs): https://learn.microsoft.com/en-us/azure/logic-apps/connectors/built-in/reference/azureaifoundryagentservice
- CREATE EXTERNAL MODEL (LOCATION format, API_FORMAT values, credential rules): https://learn.microsoft.com/en-us/sql/t-sql/statements/create-external-model-transact-sql
- AI_GENERATE_EMBEDDINGS: https://learn.microsoft.com/en-us/sql/t-sql/functions/ai-generate-embeddings-transact-sql
- sp_invoke_external_rest_endpoint (allowed endpoints, managed identity credential example): https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-invoke-external-rest-endpoint-transact-sql
- Role-based access control for Azure OpenAI, Cognitive Services OpenAI User: https://learn.microsoft.com/en-us/azure/ai-foundry/openai/how-to/role-based-access-control
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
