# Instructor-Examiner guide — External Models Evaluate 1

Companion to [external_models_evaluate_1.md](external_models_evaluate_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Each option decides two things, an embedding model with its dimensions, and a chat-model output setting. Read the two features, the three constraints and the model table before the options, and read all four options before taking an answer. Name the JSON keys of response underscore format precisely.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Implement AI capabilities in database solutions (25–30%).
- Skill: Evaluate and integrate external AI models.
- Task bullet: Evaluate embedding and chat models for multilingual retrieval, dimension limits of the vector type, and structured outputs.
- What is tested: third-generation embeddings versus ada-002 on multilingual retrieval, the dimensions parameter, the 1,998-dimension limit of the SQL vector type, one model and one dimension count per column, and JSON mode versus structured outputs.

## 2. Scenario to read aloud

**Piece 1, the story.** "GlobalBazaar is a cross-border marketplace whose catalog lives in an Azure SQL Database. Listings are written by sellers in twelve languages: Spanish, German, Japanese, English and others. The table Market dot Listings has five columns. ListingId, an integer, the primary key. SellerLang, a two-character code. Title, up to two hundred characters. Description, NVARCHAR max. And Embedding, a VECTOR of one thousand five hundred thirty-six dimensions, nullable. Ten million rows already carry an embedding made with text dash embedding dash ada dash 002. Thirty million more rows are still to be embedded."

**Piece 2, feature one.** "Feature one is cross-language semantic search. A buyer typing an English query must find listings written in Japanese or German, and vice versa. The team evaluates embedding models on multilingual retrieval, using the MIRACL benchmark. No extra model call may be added to the query path: one embedding call for the query text, then a VECTOR underscore DISTANCE search."

**Piece 3, feature two.** "Feature two is structured attribute extraction. A chat model must turn each free-text Description into a JSON document with exactly the properties brand, colour and size, all strings, no extras. Downstream code deserializes it without validation, so the model output must be guaranteed to conform to the schema, not merely be valid JSON."

**Piece 4, the constraints.** "Three constraints. One: every vector stored in one vector column must come from the same model and the same dimension count, otherwise similarity between rows is meaningless. Two: the vector column must stay at one thousand five hundred thirty-six dimensions or fewer; the storage budget is four bytes times dimensions plus eight bytes per row. Three: the SQL vector type is limited to one thousand nine hundred ninety-eight dimensions. On SQL Server 2025 this is enforced at declaration time: DECLARE at v VECTOR of 1999 fails with message 2717, the size 1999 given to the type vector exceeds the maximum allowed, 1998."

**Piece 5, the candidate models.** "Four candidates from the Azure OpenAI model reference. text dash embedding dash ada dash 002, version 2: one thousand five hundred thirty-six output dimensions, fixed, no dimensions parameter, eight thousand one hundred ninety-two max input tokens. text dash embedding dash 3 dash small: one thousand five hundred thirty-six dimensions, dimensions parameter supported to shorten, same max tokens. text dash embedding dash 3 dash large: three thousand seventy-two dimensions, dimensions parameter supported to shorten, same max tokens. And gpt dash 4o, version 2024 dash 08 dash 06 or later: a chat model with structured outputs and one hundred twenty-eight thousand max input tokens."

**Piece 6, option a.** "Option a. Deploy text dash embedding dash 3 dash large with its default output and write the new vectors into the existing Embedding VECTOR 1536 column, next to the ada dash 002 vectors, re-embedding only the thirty million rows that have no embedding yet. For feature two, call gpt dash 4o with JSON mode, meaning response underscore format with type json underscore object, and a system prompt that lists the three properties."

**Piece 7, option b.** "Option b. Deploy text dash embedding dash 3 dash large and request dimensions equal to 1536 on every embedding call, in the PARAMETERS of the external model or in the request body. Re-embed all forty million listings into Embedding VECTOR 1536, discarding the ada dash 002 vectors. For feature two, call gpt dash 4o version 2024 dash 08 dash 06 with response underscore format of type json underscore schema, with a json underscore schema object named attrs, strict true, and a schema that declares the three string properties, marks all three as required, and sets additionalProperties to false."

**Piece 8, option c.** "Option c. Keep text dash embedding dash ada dash 002 for all rows, so no re-embedding cost. To make feature one work across languages, first send every buyer query to gpt dash 4o to translate it into English, then embed the translation with ada dash 002 and search. For feature two, use gpt dash 4o with structured outputs, json underscore schema and strict true."

**Piece 9, option d.** "Option d. Deploy text dash embedding dash 3 dash large at its full three thousand seventy-two dimensions for maximum quality, add a new column EmbeddingLarge VECTOR 3072, and re-embed all forty million rows into it, keeping the old column untouched. For feature two, use gpt dash 4o with structured outputs, json underscore schema and strict true."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE TABLE Market.Listings
(
    ListingId   INT            NOT NULL PRIMARY KEY,
    SellerLang  CHAR(2)        NOT NULL,
    Title       NVARCHAR(200)  NOT NULL,
    Description NVARCHAR(MAX)  NOT NULL,
    Embedding   VECTOR(1536)   NULL      -- 10 million rows already embedded with text-embedding-ada-002
);
```

Engine limit check:

```text
DECLARE @v VECTOR(1999);
Msg 2717, Level 15, State 3
The size (1999) given to the type 'vector' exceeds the maximum allowed (1998).
```

Option b, response format:

```json
"response_format": {"type": "json_schema", "json_schema": {"name": "attrs", "strict": true, "schema": {...}}}
```

Option a, response format: `"response_format": {"type": "json_object"}`.

Where the dimensions choice is wired in:

```sql
CREATE EXTERNAL MODEL ListingEmbedder
WITH (
    LOCATION   = 'https://globalbazaar-oai.openai.azure.com/openai/deployments/text-embedding-3-large/embeddings?api-version=2024-02-01',
    API_FORMAT = 'Azure OpenAI',
    MODEL_TYPE = EMBEDDINGS,
    MODEL      = 'text-embedding-3-large',
    CREDENTIAL = [https://globalbazaar-oai.openai.azure.com/],
    PARAMETERS = '{"dimensions": 1536}'          -- every request asks for 1,536 values
);
-- Embedding VECTOR(1536): the column width equals the requested dimension count
UPDATE Market.Listings
SET Embedding = AI_GENERATE_EMBEDDINGS(CONCAT_WS(N' | ', Title, Description) USE MODEL ListingEmbedder)
WHERE Embedding IS NULL;
```

Storage arithmetic, 4 bytes per element plus 8 bytes per row:

```text
vector(1024): 4,104 bytes per row, about 164 GB for 40 million rows
vector(1536): 6,152 bytes per row, about 246 GB for 40 million rows
vector(3072): not declarable (max 1998)
```

## 4. The question (ask exactly this)

"Which choice of models and settings satisfies both features and all constraints?

a. text dash embedding dash 3 dash large with its default output, written into the existing VECTOR 1536 column next to the ada dash 002 vectors, re-embedding only the thirty million missing rows. gpt dash 4o with JSON mode, response underscore format type json underscore object, and a system prompt listing the three properties.

b. text dash embedding dash 3 dash large with dimensions 1536 on every call, re-embedding all forty million rows into VECTOR 1536 and discarding the ada dash 002 vectors. gpt dash 4o 2024 dash 08 dash 06 with response underscore format type json underscore schema, strict true, three string properties all required, additionalProperties false.

c. Keep ada dash 002 for all rows. Translate every buyer query to English with gpt dash 4o first, then embed with ada dash 002 and search. gpt dash 4o with structured outputs, json underscore schema, strict true.

d. text dash embedding dash 3 dash large at full 3072 dimensions in a new column EmbeddingLarge VECTOR 3072, re-embedding all forty million rows, old column untouched. gpt dash 4o with structured outputs, json underscore schema, strict true.

Which letter, and why do the other three fail?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: b.**

| Option | Verdict | Why |
|---|---|---|
| a | Wrong | Default output of 3-large is 3,072 values; storing it in VECTOR 1536 fails with Msg 42204, the vector dimensions 1536 and 3072 do not match. Mixing ada-002 and 3-large vectors in one column violates constraint 1. JSON mode guarantees valid JSON only, not schema adherence; feature 2 needs a guarantee. |
| b | Correct | Third-generation models beat ada-002 on MIRACL multilingual retrieval, so no translation step; query path stays at one embedding call. dimensions 1536 fits the column, 6,152 bytes per row, under the 1,998 cap. Models cannot be upgraded in place, so all forty million rows are re-embedded, one model, one dimension count. Structured outputs with json underscore schema, strict true, all properties required, additionalProperties false, on gpt-4o 2024-08-06, guarantee the schema. |
| c | Wrong | Adds a chat-model call to the query path, forbidden. Translating the query does nothing for the listings: a Japanese description is still embedded by ada-002, the weakest on multilingual retrieval. Feature 2 design is fine. |
| d | Wrong | VECTOR 3072 cannot be declared; the type allows at most 1,998. Even if it could, 12,296 bytes per row doubles the storage budget, constraint 2. The only way to use 3-large in SQL is to shorten it with dimensions. |

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with constraint three, the vector type's maximum. One option declares a column wider than that maximum. Can it even be created?"
2. "Feature one says no extra model call in the query path. One option translates every query with a chat model before embedding. Also ask: does translating the query change how the Japanese listings were embedded?"
3. "Now feature two. Two output settings appear: json underscore object, and json underscore schema with strict true. Which one only promises valid JSON, and which one enforces the schema?"
4. "Two options remain on the embedding side. One writes 3-large vectors at their default size into a 1536 column, next to the old ada vectors. What is the default size of 3-large? And can two models' vectors share one column?"
5. "The remaining option shortens 3-large to 1536 with the dimensions parameter, re-embeds everything, and uses json underscore schema strict. Which letter?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "Option a saves re-embedding ten million rows" | Thinks vectors from two models can coexist | "Would a distance between an ada-002 vector and a 3-large vector mean anything? Read constraint one." |
| "Option a, the driver truncates 3072 to 1536" | Expects silent truncation | "What does SQL Server do when you assign a 3072-dimension vector to a VECTOR 1536 column?" |
| "Option a, JSON mode plus a prompt is enough" | Confuses valid JSON with schema conformance | "Can JSON mode return a missing key, an extra key or a number where a string was expected? What does the downstream code do with that?" |
| "Option c, translation fixes cross-language search" | Believes preprocessing one side fixes retrieval | "The listings were embedded in Japanese by ada-002. Does translating the English query change those vectors? And how many model calls are now in the query path?" |
| "Option d, full 3072 is best quality" | Ignores the vector type limit and the storage budget | "What is the maximum size of the vector type? And what is 4 times 3072 plus 8, per row, times forty million?" |
| "Option b wastes money re-embedding ten million rows" | Expects in-place model upgrades | "Can you upgrade between embedding models without regenerating vectors? What does the documentation say?" |

## 8. Teaching notes (after the answer is complete or revealed)

Evaluate an external model on four axes, in this order:

- **Modality.** Text-only embedding, multimodal text plus image, or chat. If the marketplace later needs image search, a text embedding model cannot help; the model catalog lists multimodal embedding models such as Cohere embed dash v dash 4 dash 0, which takes text of 512 tokens and images and can emit 256, 512, 1,024 or 1,536 dimensions in ten languages. The same rules apply: one model, one dimension count, never mixed.
- **Language.** The documentation says both third-generation models, 3-small and 3-large, offer better average multi-language retrieval on the MIRACL benchmark than ada-002. ada-002 has no dimensions parameter. Cross-language recall is decided by the embedding model on both sides, not by translating one side. That is why option c fails feature 1, and it also adds a forbidden call to the query path.
- **Size.** 3-large outputs 3,072 dimensions and must be shortened with the dimensions parameter, because the SQL vector type allows at most 1,998, Msg 2717 at declaration time, and storage is 4 bytes times dimensions plus 8 per row. 1,536 gives 6,152 bytes per row, about 246 gigabytes for forty million rows. Even reduced below 1,536, 3-large performance remains slightly better than ada-002. One column equals one model equals one dimension count; you cannot upgrade between embedding models, you must generate new embeddings. That is why option a fails, Msg 42204 on the mismatch and mixed vector spaces, and why option d fails.
- **Output shape.** JSON mode, response underscore format type json underscore object, guarantees valid JSON only. Structured outputs, type json underscore schema with strict true, make the model follow the schema. Azure's rules: all properties listed as required, additionalProperties false, a supported model such as gpt-4o 2024-08-06. That is feature 2, and the second failure of option a.
- **Where the dimensions choice lives in SQL.** On the external model, PARAMETERS equal to a JSON with dimensions 1536, or overridden per call; the column width must equal the requested count. AI underscore GENERATE underscore EMBEDDINGS with USE MODEL fills the rows. Fewer dimensions mean cheaper storage and faster distance computations at a small quality cost, so the dimensions parameter, not a weaker model, is the lever when the budget is tight.

Memory hook: "Third generation beats ada. Shorten with dimensions, never exceed 1998. One column, one model, one size. Schema guarantee needs json underscore schema strict, not JSON mode."

## 9. Follow-up oral questions (optional)

1. "What is the per-row storage of a VECTOR 1024 column, and roughly how much for forty million rows?" (4,104 bytes; about 164 gigabytes.)
2. "Name the three schema rules Azure requires for strict structured outputs." (All properties listed in required, additionalProperties false, and a supported model version such as gpt-4o 2024-08-06.)
3. "The team wants the same dimension count on every call without changing application code. Where in SQL do they set it?" (In the PARAMETERS clause of CREATE EXTERNAL MODEL, as a JSON with dimensions 1536.)

## 10. References

- Azure OpenAI models, embeddings and dimensions parameter: https://learn.microsoft.com/en-us/azure/ai-foundry/openai/concepts/models
- Azure OpenAI embeddings and the dimensions parameter: https://learn.microsoft.com/en-us/azure/ai-foundry/openai/how-to/embeddings
- Structured outputs in Azure OpenAI: https://learn.microsoft.com/en-us/azure/ai-foundry/openai/how-to/structured-outputs
- Vector data type in SQL Server: https://learn.microsoft.com/en-us/sql/t-sql/data-types/vector-data-type
- CREATE EXTERNAL MODEL: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-external-model-transact-sql
- AI_GENERATE_EMBEDDINGS: https://learn.microsoft.com/en-us/sql/t-sql/functions/ai-generate-embeddings-transact-sql
- VECTOR_DISTANCE: https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-distance-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
