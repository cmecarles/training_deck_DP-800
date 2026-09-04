# SQL Server question — Foundry Embedding Maintenance 1

## Statement

A regional transit agency keeps its rider-facing service alerts in an Azure SQL Database named `TransitTimes`. Alerts are embedded for semantic search with an **Azure OpenAI** deployment named `text-embedding-3-small` on the Azure OpenAI resource `transittimes-aoai` (endpoint `https://transittimes-aoai.openai.azure.com`). The database authenticates to that resource with an API key today.

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

The agency now wants to search station-sign **photos** as well as text, using the Cohere multimodal embedding model `embed-v-4-0` from the Microsoft Foundry model catalog (text and image input; 256/512/1,024/1,536-dimension output). The vectors will be written into `SignEmbedding` by an Azure Function that calls the model, because `CREATE EXTERNAL MODEL` has no API format for that model family. The nightly text sweep above must keep running.

The redesign must satisfy **all** of the following:

1. Both embedding deployments (`text-embedding-3-small` and `embed-v-4-0`) must live on the **single existing Azure AI resource `transittimes-aoai`**, which keeps its name and endpoints. Governance forbids a second AI account, hub, or workspace for this workload.
2. The T-SQL side must keep working **unchanged**: `AlertEmbedder` keeps its current `LOCATION` and `API_FORMAT`, and `Ops.usp_RefreshStaleTextEmbeddings` is not rewritten.
3. **No API keys anywhere.** The database authenticates to the model endpoint with the logical server's managed identity, and key-based (local) authentication is disabled on the resource.
4. The nightly sweep is orchestrated by a **Foundry prompt agent** (the ops team wants run history and traces in the Foundry project). The schedule must live **inside the Foundry project** — no separate scheduler resource — and must not depend on a feature that Microsoft has already announced for retirement. Preview features are acceptable for the scheduler.

Which design meets all four requirements?

### a.

Upgrade `transittimes-aoai` **in place** from an Azure OpenAI resource to a Foundry resource (`kind: 'AIServices'`, `allowProjectManagement: true`, `disableLocalAuth: true`, system-assigned identity on), and create a Foundry project in it. Deploy `embed-v-4-0` from the model catalog as a standard deployment on that resource. Leave `AlertEmbedder` and the procedure untouched. Replace the key credential:

```sql
DROP DATABASE SCOPED CREDENTIAL [https://transittimes-aoai.openai.azure.com];
CREATE DATABASE SCOPED CREDENTIAL [https://transittimes-aoai.openai.azure.com]
WITH IDENTITY = 'Managed Identity', SECRET = '{"resourceid":"https://cognitiveservices.azure.com"}';
```

and grant the logical server's managed identity the **Cognitive Services OpenAI User** role on `transittimes-aoai`. In the project, create a prompt agent whose tool is an Azure Function (exposed through an OpenAPI/MCP tool definition) that runs `Ops.usp_RefreshStaleTextEmbeddings` and then embeds any alert with a `SignPhotoUrl` and a `NULL` `SignEmbedding` through the `embed-v-4-0` deployment. Create a **routine** with a *recurring* trigger (cron-style, 02:00 daily) whose action invokes that agent.

### b.

Same upgrade, Cohere deployment, managed-identity credential, role assignment, agent, and routine as option a — but because the upgraded resource now exposes a Foundry endpoint, point the external model at it so both models share one host name:

```sql
ALTER EXTERNAL MODEL AlertEmbedder
SET (LOCATION = 'https://transittimes-aoai.services.ai.azure.com/openai/deployments/text-embedding-3-small/embeddings?api-version=2024-02-01');
DROP DATABASE SCOPED CREDENTIAL [https://transittimes-aoai.openai.azure.com];
CREATE DATABASE SCOPED CREDENTIAL [https://transittimes-aoai.services.ai.azure.com]
WITH IDENTITY = 'Managed Identity', SECRET = '{"resourceid":"https://cognitiveservices.azure.com"}';
```

### c.

Keep `transittimes-aoai` as an Azure OpenAI resource (no upgrade) so the text deployment is not disturbed. Create a Foundry **hub** with a hub-based project and deploy `embed-v-4-0` there as a **serverless API endpoint** (pay-as-you-go); keep an API-key credential for that endpoint, because serverless API endpoints do not support keyless authentication. Orchestrate the nightly sweep with a **prompt flow** in the hub-based project (an LLM node plus a Python node that calls the database), and schedule the flow's batch run nightly.

### d.

Same upgrade, Cohere deployment, agent, and unchanged `AlertEmbedder` as option a. Because the upgrade preserves the resource's API key, keep the existing `HTTPEndpointHeaders` credential unchanged for the database, and set `disableLocalAuth: true` on the resource so that the agent runtime uses Microsoft Entra ID. Schedule the sweep with an **Azure Logic Apps** Consumption workflow: a *Recurrence* trigger followed by the Foundry Agent Service connector actions *Create Thread*, *Create Run*, *Get Run*, and *List Messages*.

## Correct Answer

**a**

## Explanation

The bullet "choose an embedding maintenance method, including Microsoft Foundry" is about where the models are hosted, how the database reaches them, and what orchestrates the refresh. Evaluate every option against the four requirements:

| Requirement | a | b | c | d |
| --- | --- | --- | --- | --- |
| 1. One resource, existing name and endpoints | satisfied | satisfied | **violated** (hub + serverless endpoint) | satisfied |
| 2. `LOCATION` / `API_FORMAT` unchanged | satisfied | **violated** | satisfied | satisfied |
| 3. No keys; managed identity; local auth off | satisfied | satisfied | **violated** | **violated** |
| 4. Agent + in-project schedule, nothing retiring | satisfied | satisfied | **violated** (prompt flow) | **violated** (Logic Apps, classic connector) |

### Why option a is correct

- **Foundry resource, not a new resource.** The Microsoft Learn article *Upgrade Azure OpenAI to Microsoft Foundry* states that upgrading "keeps your existing Azure OpenAI API endpoint, state of work, and security configurations", and lists what stays the same: resource name, tags, network configuration, access and identity configuration, "API endpoint and API key", custom domain, and existing state. In Bicep the upgrade is a patch that changes `kind` from `OpenAI` to `AIServices` and sets `allowProjectManagement: true`; `disableLocalAuth: true` is the documented property for Entra-only access. That is exactly requirement 1 plus the "local auth disabled" half of requirement 3.
- **Why Foundry and not a plain Azure OpenAI resource.** The same article's benefit table shows the difference: an Azure OpenAI resource offers "Azure OpenAI only" models, while a Foundry resource adds models sold by Azure from other providers and "Partner and community models sold through Marketplace - Stability, Cohere, and others", plus the Agent service. The deployment-options documentation is equally explicit: "If you use Azure OpenAI resources, the model catalog shows only Azure OpenAI in Foundry Models for deployment. You can get the full list of Foundry Models by upgrading to a Foundry resource." `embed-v-4-0` is listed in the catalog's Cohere collection with "Input: text (512 tokens) and images (2MM pixels)" and "Output: Vector (256, 512, 1024, 1536 dim.)" — hence the separate `SignEmbedding VECTOR(1024)` column (one column, one model, one dimension count; it never shares `TextEmbedding` with the 1,536-dimension OpenAI vectors).
- **T-SQL keeps working.** After the upgrade the resource still answers on `https://transittimes-aoai.openai.azure.com`, and the `CREATE EXTERNAL MODEL` reference lists the Azure OpenAI location format as `https://{endpoint}/openai/deployments/{deployment-id}/embeddings?api-version={date}` with `API_FORMAT` limited to `Azure OpenAI`, `OpenAI`, `Ollama`, and `ONNX Runtime`. `AlertEmbedder` already matches that, so neither the model object nor the procedure changes.
- **Keyless from the database.** The `sp_invoke_external_rest_endpoint` reference (whose credential rules `CREATE EXTERNAL MODEL` reuses) shows the Azure OpenAI managed-identity pattern verbatim: `IDENTITY = 'Managed Identity', SECRET = '{"resourceid":"https://cognitiveservices.azure.com"}'` with the identity granted **Cognitive Services OpenAI User**. The upgrade article's governance table confirms that role keeps working after the upgrade ("Cognitive Services OpenAI User: OpenAI features" before and after). Foundry's own authentication guidance says API keys "grant full access to the resource" and are "not recommended by Microsoft" for production; its feature matrix marks the **Agents service as "API key: No, Microsoft Entra ID: Yes"**, so the agent side is Entra-only regardless.
- **Agent plus routine.** Foundry Agent Service prompt agents are "defined entirely through configuration, including instructions, model selection, and tools", and custom tools can be added "through functions, OpenAPI specs, and MCP servers", including "custom MCP servers hosted on Azure Functions". Routines (preview) "let you run an agent automatically when a defined trigger fires" with three trigger types — timer, **recurring** ("Run an agent repeatedly on a cron-style schedule"), and event — and are explicitly positioned as "No separate scheduler resource to manage", with run history and traces recorded in the project. The scenario allowed preview features for the scheduler, so this meets requirement 4 without prompt flow or Logic Apps.

### Why option b is wrong

This is the subtle distractor: every Azure-side decision is right, and the new host name is real — the upgrade article says a Foundry resource "exposes its capabilities over three FQDNs": `{custom-domain}.openai.azure.com`, `{custom-domain}.services.ai.azure.com`, and `{custom-domain}.cognitiveservices.azure.com`. But the option rewrites `LOCATION`, which requirement 2 forbids outright. It also would not work from Azure SQL Database: the allowed-endpoints table for `sp_invoke_external_rest_endpoint` (which `CREATE EXTERNAL MODEL` follows — "The URL domain must be one of those domains included in the allow list") contains `*.openai.azure.com` and `*.cognitiveservices.azure.com` / `*.api.cognitive.microsoft.com`, but **no `*.services.ai.azure.com` entry**. The credential name rules add that the credential's protocol + FQDN must match the called URL, so the new credential would be rejected for the same reason. The Foundry endpoint is for the SDK, the Responses API, and agents; the database keeps talking to the Azure OpenAI FQDN.

### Why option c is wrong

Three independent failures:

- **Requirement 1.** Serverless API endpoints are "available only in AI Hub resources" and their "Deployment resource" is an "AI project (in AI hub resource)", so the Cohere deployment lands on a second resource, not on `transittimes-aoai`. The deployment-options article also says to "Use Foundry resources for deployment whenever possible" and that standard deployment in Foundry resources "is the preferred deployment option".
- **Requirement 3.** The option is honest about the consequence: the capability table lists "Key-less authentication: Yes" for standard deployments in Foundry resources but "No" for serverless API endpoints, so this design forces a key back into the pipeline.
- **Requirement 4.** The prompt flow documentation carries a retirement warning: "Prompt flow in Microsoft Foundry and Azure Machine Learning will be retired on April 20, 2027. Prompt flow is no longer recommended for new development", its runtime images "are no longer receiving updates, including security and package updates", and it "provides legacy support for hub-based projects. It will not work for Foundry projects." Choosing it for a new orchestration is exactly what the requirement excludes.

### Why option d is wrong

- **Requirement 3.** Keeping the `HTTPEndpointHeaders` api-key credential violates the no-keys rule on its face, and the option contradicts itself: `disableLocalAuth: true` turns key authentication off for the whole resource, so the very credential it keeps stops working and `AI_GENERATE_EMBEDDINGS` fails to authenticate. Foundry's guidance is to "remove key-based authentication after all callers use token authentication" and *then* "optionally disable local authentication" — the callers must be migrated first, which is what option a does with the managed-identity credential.
- **Requirement 4.** A Logic Apps workflow is a separate scheduler resource outside the Foundry project, which the requirement rules out. Worse, the *Create Thread / Create Run / Get Run / List Messages* sequence is the **classic** Agent Service connector built on threads and runs; that documentation now opens with "Agents (classic) are now deprecated and will be retired on March 31, 2027", and the Assistants API it depends on "retires August 26, 2026". Logic Apps with a Recurrence trigger remains a valid maintenance pattern in general (embeddings_1 covers it), but not under this scenario's constraints.

Conceptual question (Azure / tooling); not executed against an engine.

## DP-800 Exam Rule to Remember

Microsoft Foundry enters the embedding pipeline in three places — the resource, the endpoint, and the orchestrator:

```text
Resource   Azure OpenAI resource  = Azure OpenAI models only, no agents
           Foundry resource       = same endpoint/keys/RBAC after in-place upgrade
                                    (kind OpenAI -> AIServices) + full catalog
                                    (Cohere, Mistral, DeepSeek, ...) + projects + agents
           Hub / serverless API   = separate hub resource, regional, NO keyless auth

Endpoint   T-SQL (CREATE EXTERNAL MODEL, sp_invoke_external_rest_endpoint) stays on
           https://<res>.openai.azure.com/openai/deployments/<dep>/embeddings?api-version=...
           API_FORMAT in {Azure OpenAI, OpenAI, Ollama, ONNX Runtime}
           *.services.ai.azure.com is NOT in Azure SQL's outbound allow list
           Keyless: Managed Identity credential, resourceid https://cognitiveservices.azure.com,
                    Cognitive Services OpenAI User on the resource, disableLocalAuth: true

Orchestrator  Prompt agent + routine (recurring/timer/event, preview) = schedule inside the project
              Logic Apps Recurrence / Functions timer               = external scheduler
              Prompt flow                                           = retiring 2027-04-20, hub-only
```

Non-OpenAI embedding models (Cohere `embed-v-4-0`, multimodal, 256–1,536 dims) are deployed on the Foundry resource and called by application code or an agent tool, never by `AI_GENERATE_EMBEDDINGS`; store their vectors in their own `vector(n)` column. If an option moves the database to the Foundry FQDN, keeps a key while disabling local auth, or schedules new work on prompt flow or the classic threads/runs connector, it is wrong no matter how correct the rest of the design looks.
