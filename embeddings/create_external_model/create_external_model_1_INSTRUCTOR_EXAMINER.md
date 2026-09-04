# Instructor-Examiner guide — Create External Model 1

Companion to [create_external_model_1.md](create_external_model_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. The four CREATE statements are nearly identical; the differences are one option value in the CREATE, which grants follow, and how the later change is made. Read all four options before taking an answer, and stress the differing lines. Take one letter as the answer.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Implement AI capabilities in database solutions (25–30%).
- Skill: Design and implement models and embeddings.
- Task bullet: Create and manage external models for embeddings.
- What is tested: the accepted option values of CREATE EXTERNAL MODEL, the two permission layers needed to use a model, and the in-place ALTER EXTERNAL MODEL SET syntax versus drop and recreate.

## 2. Scenario to read aloud

**Piece 1, the story.** "An insurer stores adjuster notes in a SQL Server 2025 database called ClaimsDesk. Notes must be embedded with the Azure OpenAI deployment text dash embedding dash 3 dash large, shortened to one thousand twenty-four dimensions. The embeddings are generated in T-SQL with AI underscore GENERATE underscore EMBEDDINGS."

**Piece 2, what already exists.** "The database, table, role and credential already exist. There is a schema called Claims and a table Claims dot Notes with three columns: NoteId, an integer, the primary key; NoteText, an NVARCHAR MAX, nullable; and Embedding, a VECTOR of 1024, nullable. There is a role called ClaimsAnalysts, and a user called Dana, created without login, who is a member of that role. There is a database master key. And there is a database scoped credential whose name is the URL of the Azure OpenAI endpoint, https colon slash slash claimsdesk dash oai dot openai dot azure dot com slash, with identity HTTPEndpointHeaders and a secret that holds the api key as JSON."

**Piece 3, requirement 1.** "Three requirements. Requirement 1. Register the model as an external model object named NotesEmbedder, which sends dimensions equals 1024 with every request."

**Piece 4, requirement 2.** "Requirement 2. Members of ClaimsAnalysts, such as Dana, must be able to run SELECT AI underscore GENERATE underscore EMBEDDINGS of some text USE MODEL NotesEmbedder, themselves."

**Piece 5, requirement 3.** "Requirement 3. Later, the operations team must add a retry policy, in the form of a JSON key sql underscore rest underscore options with retry underscore count 3, in place. The model may already be referenced by name from other objects, so dropping and recreating it is not acceptable."

**Piece 6, the common CREATE.** "All four options start with a CREATE EXTERNAL MODEL NotesEmbedder WITH six options. LOCATION is the HTTPS URL of the deployment's embeddings endpoint with an api dash version query string. API underscore FORMAT is a string I will name per option. MODEL underscore TYPE is EMBEDDINGS. MODEL is text dash embedding dash 3 dash large. CREDENTIAL is the credential named after the URL. And PARAMETERS is the JSON string, dimensions colon 1024. Options b, c and d use API underscore FORMAT equals Azure OpenAI. Option a is different."

**Piece 7, option a.** "Option a. The CREATE uses API underscore FORMAT equals Azure AI Foundry. Then GRANT EXECUTE ON EXTERNAL MODEL double colon NotesEmbedder TO ClaimsAnalysts. Then GRANT REFERENCES ON DATABASE SCOPED CREDENTIAL double colon, the credential, TO ClaimsAnalysts. Later: ALTER EXTERNAL MODEL NotesEmbedder SET, open paren, PARAMETERS equals the JSON with dimensions 1024 and sql underscore rest underscore options retry underscore count 3, close paren."

**Piece 8, option b.** "Option b. The CREATE uses API underscore FORMAT equals Azure OpenAI. Then the same two grants as option a: EXECUTE on the external model to ClaimsAnalysts, and REFERENCES on the database scoped credential to ClaimsAnalysts. Later: the same ALTER EXTERNAL MODEL NotesEmbedder SET PARAMETERS with the retry policy added."

**Piece 9, option c.** "Option c. The CREATE uses Azure OpenAI, identical to option b. Then one grant only: GRANT ALTER ANY EXTERNAL MODEL TO ClaimsAnalysts. Later: ALTER EXTERNAL MODEL NotesEmbedder WITH, open paren, PARAMETERS equals the JSON with the retry policy, close paren. Note the word WITH instead of SET."

**Piece 10, option d.** "Option d. The CREATE uses Azure OpenAI, identical to option b. Then one grant: GRANT EXECUTE ON EXTERNAL MODEL double colon NotesEmbedder TO ClaimsAnalysts. No grant on the credential. Later: DROP EXTERNAL MODEL NotesEmbedder, and then CREATE EXTERNAL MODEL NotesEmbedder again with the same six options, but PARAMETERS now includes the retry policy."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE ClaimsDesk;
GO
USE ClaimsDesk;
GO
CREATE SCHEMA Claims;
GO
CREATE TABLE Claims.Notes
(
    NoteId    INT           NOT NULL PRIMARY KEY,
    NoteText  NVARCHAR(MAX) NULL,
    Embedding VECTOR(1024)  NULL
);
GO
CREATE ROLE ClaimsAnalysts;
CREATE USER Dana WITHOUT LOGIN;
ALTER ROLE ClaimsAnalysts ADD MEMBER Dana;
GO
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'Cl@imsDesk!2026';
GO
CREATE DATABASE SCOPED CREDENTIAL [https://claimsdesk-oai.openai.azure.com/]
WITH IDENTITY = 'HTTPEndpointHeaders', SECRET = '{"api-key":"REDACTED"}';
GO
```

Option a:

```sql
CREATE EXTERNAL MODEL NotesEmbedder
WITH (
    LOCATION   = 'https://claimsdesk-oai.openai.azure.com/openai/deployments/text-embedding-3-large/embeddings?api-version=2024-02-01',
    API_FORMAT = 'Azure AI Foundry',
    MODEL_TYPE = EMBEDDINGS,
    MODEL      = 'text-embedding-3-large',
    CREDENTIAL = [https://claimsdesk-oai.openai.azure.com/],
    PARAMETERS = '{"dimensions":1024}'
);
GRANT EXECUTE ON EXTERNAL MODEL::NotesEmbedder TO ClaimsAnalysts;
GRANT REFERENCES ON DATABASE SCOPED CREDENTIAL::[https://claimsdesk-oai.openai.azure.com/] TO ClaimsAnalysts;
-- later
ALTER EXTERNAL MODEL NotesEmbedder SET (PARAMETERS = '{"dimensions":1024, "sql_rest_options": {"retry_count": 3}}');
```

Option b:

```sql
CREATE EXTERNAL MODEL NotesEmbedder
WITH (
    LOCATION   = 'https://claimsdesk-oai.openai.azure.com/openai/deployments/text-embedding-3-large/embeddings?api-version=2024-02-01',
    API_FORMAT = 'Azure OpenAI',
    MODEL_TYPE = EMBEDDINGS,
    MODEL      = 'text-embedding-3-large',
    CREDENTIAL = [https://claimsdesk-oai.openai.azure.com/],
    PARAMETERS = '{"dimensions":1024}'
);
GRANT EXECUTE ON EXTERNAL MODEL::NotesEmbedder TO ClaimsAnalysts;
GRANT REFERENCES ON DATABASE SCOPED CREDENTIAL::[https://claimsdesk-oai.openai.azure.com/] TO ClaimsAnalysts;
-- later
ALTER EXTERNAL MODEL NotesEmbedder SET (PARAMETERS = '{"dimensions":1024, "sql_rest_options": {"retry_count": 3}}');
```

Option c:

```sql
CREATE EXTERNAL MODEL NotesEmbedder
WITH (
    LOCATION   = 'https://claimsdesk-oai.openai.azure.com/openai/deployments/text-embedding-3-large/embeddings?api-version=2024-02-01',
    API_FORMAT = 'Azure OpenAI',
    MODEL_TYPE = EMBEDDINGS,
    MODEL      = 'text-embedding-3-large',
    CREDENTIAL = [https://claimsdesk-oai.openai.azure.com/],
    PARAMETERS = '{"dimensions":1024}'
);
GRANT ALTER ANY EXTERNAL MODEL TO ClaimsAnalysts;
-- later
ALTER EXTERNAL MODEL NotesEmbedder WITH (PARAMETERS = '{"dimensions":1024, "sql_rest_options": {"retry_count": 3}}');
```

Option d:

```sql
CREATE EXTERNAL MODEL NotesEmbedder
WITH (
    LOCATION   = 'https://claimsdesk-oai.openai.azure.com/openai/deployments/text-embedding-3-large/embeddings?api-version=2024-02-01',
    API_FORMAT = 'Azure OpenAI',
    MODEL_TYPE = EMBEDDINGS,
    MODEL      = 'text-embedding-3-large',
    CREDENTIAL = [https://claimsdesk-oai.openai.azure.com/],
    PARAMETERS = '{"dimensions":1024}'
);
GRANT EXECUTE ON EXTERNAL MODEL::NotesEmbedder TO ClaimsAnalysts;
-- later
DROP EXTERNAL MODEL NotesEmbedder;
CREATE EXTERNAL MODEL NotesEmbedder
WITH (
    LOCATION   = 'https://claimsdesk-oai.openai.azure.com/openai/deployments/text-embedding-3-large/embeddings?api-version=2024-02-01',
    API_FORMAT = 'Azure OpenAI',
    MODEL_TYPE = EMBEDDINGS,
    MODEL      = 'text-embedding-3-large',
    CREDENTIAL = [https://claimsdesk-oai.openai.azure.com/],
    PARAMETERS = '{"dimensions":1024, "sql_rest_options": {"retry_count": 3}}'
);
```

## 4. The question (ask exactly this)

"Which script meets all three requirements? Option a, API format Azure AI Foundry, EXECUTE plus REFERENCES grants, ALTER with SET. Option b, API format Azure OpenAI, EXECUTE plus REFERENCES grants, ALTER with SET. Option c, API format Azure OpenAI, GRANT ALTER ANY EXTERNAL MODEL, ALTER with WITH. Option d, API format Azure OpenAI, EXECUTE grant only, then DROP and CREATE. Give me one letter, and one sentence on why."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct: b.** API_FORMAT Azure OpenAI is an accepted value, so the CREATE succeeds and sys.external_models shows the model with parameters dimensions 1024. Using the model requires two grants: EXECUTE ON EXTERNAL MODEL and REFERENCES on the database scoped credential; with both, Dana's call passes every catalog and permission check. ALTER EXTERNAL MODEL SET is the documented in-place syntax; nothing is dropped.

- **a is wrong.** Azure AI Foundry is not an accepted API_FORMAT. The CREATE fails with error 46508, "Incorrect syntax on external DDL option API_FORMAT", so the model is never created.
- **c is wrong.** ALTER ANY EXTERNAL MODEL is a DDL permission; it makes the model visible but Dana's call still fails with error 15151, cannot find the external model. And ALTER EXTERNAL MODEL WITH is not valid syntax, error 156, incorrect syntax near WITH; ALTER uses SET.
- **d is wrong.** Without REFERENCES on the credential, Dana is stopped with error 15151, cannot find the credential. And DROP plus CREATE violates requirement 3 by design; with a schemabound view referencing the model, the DROP fails with error 3729.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with the CREATE statements. Three of them are identical. One uses a different API underscore FORMAT value. What are the accepted values for that option?"
2. "Requirement 3 says the change must be in place. Which option drops the model and creates it again? Cross that one out, whatever else it does."
3. "Now permissions. There are two layers: managing models, and using one in a query. Is ALTER ANY EXTERNAL MODEL about managing or using? Does it let Dana call the model?"
4. "Two options left. Look at the later ALTER statement in each. One says SET, the other says WITH. Which keyword belongs to CREATE, and which to ALTER?"
5. "Also count the grants. To use a model, Dana needs to reach both the model and the credential it uses. Which options grant both?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, Azure AI Foundry is the new name for the service" | Confuses the product name with the accepted option value | "What does the engine accept for API underscore FORMAT? Is Azure AI Foundry in that list?" |
| "c, ALTER ANY EXTERNAL MODEL covers everything on models" | Mixes up DDL permission and usage permission | "If Dana can alter models, can she run AI underscore GENERATE underscore EMBEDDINGS with one? Which permission does that require?" |
| "c, WITH and SET are both fine on ALTER" | Assumes CREATE syntax carries over | "Which clause does the ALTER EXTERNAL MODEL statement use to change options?" |
| "d, the recreate is fine because it ends up with the same name" | Ignores requirement 3 and schemabound references | "What does requirement 3 forbid? And what happens to the DROP if a schemabound view references the model?" |
| "d, EXECUTE on the model is enough for Dana" | Forgets the credential permission | "After the model check passes, what other object does the call need to reach? Does Dana have permission on it?" |
| "b is wrong because you need PREVIEW_FEATURES first" | Over-applies the ONNX Runtime requirement | "Which API format needs PREVIEW underscore FEATURES on? Is that the one used here?" |

## 8. Teaching notes (after the answer is complete or revealed)

The DDL contract:

- CREATE EXTERNAL MODEL takes LOCATION (HTTPS only), API_FORMAT which is one of Azure OpenAI, OpenAI, Ollama or ONNX Runtime, MODEL_TYPE which is EMBEDDINGS and nothing else, MODEL, optional CREDENTIAL, and optional PARAMETERS, a JSON string appended to every request body. After option b's CREATE, sys.external_models shows one row with api_format Azure OpenAI, model_type_desc EMBEDDINGS, and a parameters column of type json holding dimensions 1024. No PREVIEW_FEATURES setting was needed on the RTM build; the docs ask for it only for the local ONNX Runtime path. An invalid MODEL_TYPE such as COMPLETIONS fails with error 102.
- Option a's Azure AI Foundry is rejected with error 46508. Nothing after it can work.

The two permission layers:

- Managing models needs CREATE EXTERNAL MODEL or ALTER ANY EXTERNAL MODEL. That is DDL.
- Using a model in AI_GENERATE_EMBEDDINGS needs EXECUTE ON EXTERNAL MODEL double colon name, and REFERENCES on the database scoped credential the model uses. With no grants, Dana sees no rows in sys.external_models and gets error 15151, cannot find the external model. With EXECUTE only, the model is visible but the call fails with 15151, cannot find the credential. With both, the call reaches the network layer and gets the same error the owner gets on a lab box with no route to Azure, error 31608, which proves authorization is complete. Option c grants the DDL permission, which makes the model visible but does not let Dana call it. Option d grants EXECUTE but not REFERENCES.

The in-place change:

- ALTER EXTERNAL MODEL name SET, open paren, options, close paren, is the documented syntax. It succeeded and the catalog immediately showed the merged parameters with a new modify_time. WITH belongs to CREATE; on ALTER it gives error 156.
- DROP plus CREATE violates requirement 3 by design, and can fail outright: a schemabound view that selects AI_GENERATE_EMBEDDINGS with USE MODEL NotesEmbedder blocks the drop with error 3729. Related guard: a credential used by a model cannot be dropped either, error 46556.

Other verified behaviours: AI_GENERATE_EMBEDDINGS needs the instance option external rest endpoint enabled set to 1, otherwise error 31643. LOCATION must be HTTPS; an http Ollama model is created but every call fails with 31610. A NULL input is not skipped; it fails with 31701, so filter WHERE NoteText IS NOT NULL. The function's runtime PARAMETERS argument must be a json-typed value, not a plain string, or you get error 8116; runtime parameters override the same key in the model's PARAMETERS. Model names are unique per database, error 46502 on a duplicate, and there is no DROP EXTERNAL MODEL IF EXISTS.

Memory hook: "Azure OpenAI, OpenAI, Ollama, ONNX. EXECUTE on the model and REFERENCES on the credential to use it. ALTER uses SET, never WITH, never drop."

## 9. Follow-up oral questions (optional)

1. "Dana has EXECUTE on the model but nothing on the credential. What exact message does she get?" (Error 15151, cannot find the credential, because it does not exist or you do not have permission.)
2. "What happens if you run UPDATE Notes SET Embedding = AI_GENERATE_EMBEDDINGS(NoteText USE MODEL NotesEmbedder) and one row has a NULL NoteText?" (Error 31701, the input data parameter cannot be NULL. Filter with WHERE NoteText IS NOT NULL.)
3. "Which instance-level setting must be on before AI_GENERATE_EMBEDDINGS works on SQL Server 2025?" (sp_configure external rest endpoint enabled, set to 1.)

## 10. References

- CREATE EXTERNAL MODEL: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-external-model-transact-sql
- ALTER EXTERNAL MODEL: https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-external-model-transact-sql
- DROP EXTERNAL MODEL: https://learn.microsoft.com/en-us/sql/t-sql/statements/drop-external-model-transact-sql
- AI_GENERATE_EMBEDDINGS: https://learn.microsoft.com/en-us/sql/t-sql/functions/ai-generate-embeddings-transact-sql
- sys.external_models: https://learn.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-external-models-transact-sql
- CREATE DATABASE SCOPED CREDENTIAL: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-database-scoped-credential-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
