# Instructor-Examiner guide — Embeddings 1

Companion to [embeddings_1.md](embeddings_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a, b, c and d. Read the service objective, the table, the external model, the five requirements and all four options before taking an answer. The service objective, Standard S1 in the DTU model, is a load-bearing fact; make sure the learner has heard it. This is a conceptual Azure architecture question; nothing was executed against an engine.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Implement AI capabilities in database solutions (25–30%).
- Skill: Design and implement models and embeddings.
- Task bullet: Design embedding maintenance for changing data.
- What is tested: choosing an embedding maintenance method by matching constraints: no model call inside the user transaction, near-real-time freshness, no missed changes across an outage, changed rows only, and feature support on the current service tier.

## 2. Scenario to read aloud

**Piece 1, the story.** "A cooking-recipe platform stores its catalog in an Azure SQL Database named RecipeBox. The database uses the DTU purchasing model at the Standard S1 service objective, and the team is not allowed to change the service objective. Users constantly add and edit recipes through the website. You must design the embedding maintenance architecture that keeps the Embedding column up to date with the text in Title, Ingredients and Steps."

**Piece 2, the table.** "The table is Kitchen dot Recipes. Five columns. RecipeId, an integer, primary key. Title, nvarchar two hundred. Ingredients, nvarchar max. Steps, nvarchar max. And Embedding, a vector column of dimension fifteen thirty six, nullable."

**Piece 3, the external model.** "An external model named RecipeEmbeddingModel is already registered and works. Its LOCATION is the Azure OpenAI endpoint recipebox dash openai dot cognitiveservices dot azure dot com, path openai slash deployments slash text dash embedding dash ada dash 002 slash embeddings, api dash version 2023 dash 05 dash 15. API underscore FORMAT is Azure OpenAI, MODEL underscore TYPE is EMBEDDINGS, MODEL is text dash embedding dash ada dash 002, and it uses a database scoped credential named after the endpoint host."

**Piece 4, the five requirements.** "Requirement one: a user's INSERT or UPDATE transaction must not wait for the embedding model call, and the user's write must succeed even when the Azure OpenAI endpoint is unreachable. Requirement two: a new or edited recipe must get a refreshed embedding in near real time, within about a minute under normal operation, not on a daily cadence. Requirement three: if the embedding processor is offline for up to one day, every recipe changed during that time must still be embedded automatically after the processor comes back. No change may be missed. Requirement four: only the rows that actually changed may be reprocessed. Re-embedding the entire table is not acceptable because every model call is billed. Requirement five: every feature used must be supported on the current service objective, Azure SQL Database, DTU model, Standard S1."

**Piece 5, option a.** "Option a. Generate the embedding inside the database transaction with a DML trigger. CREATE TRIGGER Kitchen dot trg underscore Recipes underscore Embed on Kitchen dot Recipes, AFTER INSERT, UPDATE. Inside, it updates Recipes joined to the inserted pseudo-table on RecipeId, setting Embedding to AI underscore GENERATE underscore EMBEDDINGS of CONCAT underscore WS of Title, Ingredients and Steps, USE MODEL RecipeEmbeddingModel."

**Piece 6, option b.** "Option b. Enable change tracking and process changes with an Azure Function that uses the SQL trigger binding. ALTER DATABASE RecipeBox SET CHANGE underscore TRACKING equals ON, with CHANGE underscore RETENTION two days and AUTO underscore CLEANUP on. ALTER TABLE Kitchen dot Recipes ENABLE CHANGE underscore TRACKING. Deploy an Azure Function whose trigger attribute is SqlTrigger with the table name Kitchen dot Recipes and the connection setting name SqlConnectionString. For each change batch that the binding delivers, the function calls the embedding model for the changed rows only and updates Embedding for those rows."

**Piece 7, option c.** "Option c. Enable change data capture and poll the change table from a timer-based Azure Function. EXEC sys dot sp underscore cdc underscore enable underscore db. Then EXEC sys dot sp underscore cdc underscore enable underscore table with source schema Kitchen, source name Recipes, role name NULL, and supports net changes one. A timer-triggered Azure Function runs every thirty seconds, reads cdc dot fn underscore cdc underscore get underscore net underscore changes underscore Kitchen underscore Recipes for the new LSN range, calls the embedding model for the changed rows only, and updates Embedding."

**Piece 8, option d.** "Option d. Use an Azure Logic Apps workflow with a Recurrence trigger that runs every night at two in the morning and executes an UPDATE of Kitchen dot Recipes, with no WHERE clause, setting Embedding to AI underscore GENERATE underscore EMBEDDINGS of the concatenated Title, Ingredients and Steps, USE MODEL RecipeEmbeddingModel."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE TABLE Kitchen.Recipes
(
    RecipeId    int NOT NULL PRIMARY KEY,
    Title       nvarchar(200) NOT NULL,
    Ingredients nvarchar(max) NOT NULL,
    Steps       nvarchar(max) NOT NULL,
    Embedding   vector(1536) NULL
);

CREATE EXTERNAL MODEL RecipeEmbeddingModel
WITH (
    LOCATION = 'https://recipebox-openai.cognitiveservices.azure.com/openai/deployments/text-embedding-ada-002/embeddings?api-version=2023-05-15',
    API_FORMAT = 'Azure OpenAI',
    MODEL_TYPE = EMBEDDINGS,
    MODEL = 'text-embedding-ada-002',
    CREDENTIAL = [https://recipebox-openai.cognitiveservices.azure.com/]
);
```

Option a:

```sql
CREATE TRIGGER Kitchen.trg_Recipes_Embed
ON Kitchen.Recipes
AFTER INSERT, UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    UPDATE r
    SET Embedding = AI_GENERATE_EMBEDDINGS(
            CONCAT_WS(N' ', i.Title, i.Ingredients, i.Steps)
            USE MODEL RecipeEmbeddingModel)
    FROM Kitchen.Recipes AS r
    INNER JOIN inserted AS i
        ON i.RecipeId = r.RecipeId;
END;
```

Option b:

```sql
ALTER DATABASE RecipeBox
SET CHANGE_TRACKING = ON
(CHANGE_RETENTION = 2 DAYS, AUTO_CLEANUP = ON);

ALTER TABLE Kitchen.Recipes
ENABLE CHANGE_TRACKING;
```

```text
[SqlTrigger("[Kitchen].[Recipes]", "SqlConnectionString")]
```

Option c:

```sql
EXEC sys.sp_cdc_enable_db;
GO

EXEC sys.sp_cdc_enable_table
    @source_schema = N'Kitchen',
    @source_name   = N'Recipes',
    @role_name     = NULL,
    @supports_net_changes = 1;
GO
```

Option d:

```sql
UPDATE r
SET Embedding = AI_GENERATE_EMBEDDINGS(
        CONCAT_WS(N' ', r.Title, r.Ingredients, r.Steps)
        USE MODEL RecipeEmbeddingModel)
FROM Kitchen.Recipes AS r;
```

## 4. The question (ask exactly this)

"Which architecture should you implement? Option a, option b, option c, or option d?"

Options in full:

- **a.** An AFTER INSERT, UPDATE trigger on Kitchen.Recipes that calls AI_GENERATE_EMBEDDINGS for the inserted rows inside the transaction.
- **b.** Change tracking on the database, CHANGE_RETENTION 2 DAYS, AUTO_CLEANUP ON, enabled on Kitchen.Recipes; an Azure Function with the SqlTrigger binding that embeds only the changed rows of each batch.
- **c.** Change data capture enabled on the database and on Kitchen.Recipes with net changes; a timer Function every 30 seconds reading cdc.fn_cdc_get_net_changes_Kitchen_Recipes and embedding the changed rows.
- **d.** A Logic Apps Recurrence trigger at 02:00 running an UPDATE without WHERE that re-embeds every row.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: b.**

- Requirement one: change tracking records only the primary-key values of changed rows, synchronously and cheaply, in internal side tables; the user's write never calls the model. The Function polls outside the user transaction, default PollingIntervalMs 1000, so an Azure OpenAI outage delays refresh but never fails a write.
- Requirement two: the binding polls about once a second and delivers batches, so embeddings refresh within seconds.
- Requirement three: the binding persists its progress in leases and state tables in the az_func schema and picks up where it left off after a restart. CHANGE_RETENTION of two days covers the one-day maximum outage.
- Requirement four: change tracking answers exactly which rows changed since version X, so only modified recipes are re-embedded.
- Requirement five: change tracking is supported in Azure SQL Database with no service-tier restriction, so it works at Standard S1.

Why the others are wrong, one line each:

- **a.** An AFTER trigger runs inside the user's transaction; AI_GENERATE_EMBEDDINGS makes an HTTPS call at execution time, so every write waits for Azure OpenAI, and if the endpoint is unreachable the trigger errors and the user's transaction fails. Violates requirement one.
- **c.** In the DTU purchasing model, CDC requires S3 or higher; Basic, S0, S1 and S2 are not supported, and the tier cannot change. Violates requirement five. CDC is also heavier than needed: it scans the log on a twenty-second scheduler with no latency SLA and copies full column values, including the nvarchar max columns, into change tables.
- **d.** A nightly recurrence leaves a recipe edited at 08:00 stale for about eighteen hours, violating requirement two; and the UPDATE with no WHERE re-embeds every row every night, paying one model call per recipe, violating requirement four.

## 6. Hint ladder (one hint per attempt, in order)

1. "Treat this as constraint matching. Take each option and check it against the five requirements one by one. Most options fail exactly one requirement."
2. "Requirement one: which option makes the model call inside the user's own transaction? What happens to that transaction if the endpoint is down?"
3. "Requirement five is about the tier. Standard S1 in the DTU model. Which of the two change-detection features has a minimum DTU tier in Azure SQL Database?"
4. "Option d runs once a night and updates every row without a WHERE. That fails freshness and changed-rows-only at the same time. That eliminates d."
5. "Option a is an AFTER trigger that calls AI underscore GENERATE underscore EMBEDDINGS. Where does that call run relative to the user's INSERT? That eliminates a."
6. "You are down to b and c. Both detect changes and both embed only changed rows. One uses change tracking, the other change data capture. Which of those two is supported on Standard S1?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, a trigger is the simplest and always fresh" | Ignores that AFTER triggers run inside the transaction | "It is fresh. Now, if Azure OpenAI is unreachable when a user saves a recipe, what happens to that user's save?" |
| "c, CDC is the enterprise change-detection feature" | Forgets the DTU tier restriction | "Reread the service objective. Which DTU tiers support CDC in Azure SQL Database?" |
| "c, CDC captures net changes so it is more precise" | Confuses richness with fit | "The processor only needs to know which rows changed; it can read current values from the table. Which feature records just primary keys?" |
| "d, Logic Apps is a valid embedding maintenance tool" | Judges the tool, not the design | "Logic Apps can be fine. What drives this particular workflow, detected changes or a blind schedule? And what does the UPDATE re-embed?" |
| "b, but a one-day outage would lose changes" | Does not connect CHANGE_RETENTION and the az_func leases | "What is CHANGE underscore RETENTION set to, and where does the binding store its position?" |
| "b, but change tracking is not supported on S1" | Mixes up change tracking with CDC | "Which of the two features has the tier restriction? Check change tracking's support statement." |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the constraint-matching frame with three anchor questions:

- May the user's write transaction pay for the model call? If not, rule out synchronous DML triggers that call the model. An AFTER trigger executes inside the user's transaction; AI_GENERATE_EMBEDDINGS performs an HTTPS REST call at execution time, adding model latency to the write path and turning an endpoint outage into a failed user transaction. That is option a, the classic anti-pattern.
- How fresh must embeddings be, and may unchanged rows be reprocessed? Scheduled full re-embeds fail near-real-time and cost constraints. That is option d.
- What does the platform and tier actually support? Change tracking is supported on all Azure SQL Database tiers and records primary keys only, answering which rows changed. CDC gives full before and after data from an asynchronous log scan; in Azure SQL Database it is supported on any vCore tier, but in the DTU model it needs S3 or higher, not Basic, S0, S1 or S2. That is option c.

Then the packaged answer, the Azure Functions SQL trigger binding:

- It is built on change tracking. Enable it at database level with CHANGE_RETENTION and AUTO_CLEANUP, then per table with ENABLE CHANGE_TRACKING.
- It polls outside the user transaction, default PollingIntervalMs 1000, and delivers change batches to the function, which calls the model for those rows and updates Embedding.
- It persists its position in leases and state tables in the az_func schema and resumes where it left off after a restart or broken connection, as long as CHANGE_RETENTION covers the longest expected processor downtime. Two days covers the one-day requirement.

Memory hook: "Never call the model in the user's transaction. Never re-embed the whole table on a clock. Change tracking works on every tier; CDC needs S3 or vCore. The SQL trigger binding resumes from az underscore func."

## 9. Follow-up oral questions (optional)

1. "If the processor might be down for three days, what would you change in option b?" (Raise CHANGE_RETENTION above three days so the change history outlives the outage.)
2. "What does change tracking store for a changed row, compared with CDC?" (Only the primary key and version, not column values; CDC stores full before and after column values from the log.)
3. "At which DTU service objective would option c become supportable?" (Standard S3 or higher, or any vCore tier.)

## 10. References

- Azure SQL trigger for Azure Functions: https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-azure-sql-trigger
- About change tracking: https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/about-change-tracking-sql-server
- Enable and disable change tracking: https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/enable-and-disable-change-tracking-sql-server
- About change data capture, including Azure SQL Database tier support: https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/about-change-data-capture-sql-server
- AI_GENERATE_EMBEDDINGS: https://learn.microsoft.com/en-us/sql/t-sql/functions/ai-generate-embeddings-transact-sql
- CREATE EXTERNAL MODEL: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-external-model-transact-sql
- Azure SQL bindings for Azure Functions on GitHub: https://github.com/Azure/azure-functions-sql-extension
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
