# Instructor-Examiner guide — RAG use cases 1

Companion to [rag_use_cases_1.md](rag_use_cases_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a, b, c and d. Read the three constraints from the partners and all four projects before taking an answer. This is a conceptual architecture question; nothing was executed against an engine. The learner needs to reason about which tool fits each project, not about syntax.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Implement AI capabilities in database solutions (25–30%).
- Skill: Design and implement retrieval-augmented generation (RAG).
- Task bullet: Identify use cases for RAG.
- What is tested: telling a genuine RAG use case apart from a SQL reporting job, a fine-tuning or few-shot style job, and a prompt-only summarisation job.

## 2. Scenario to read aloud

**Piece 1, the story.** "A mid-sized law firm runs its practice-management data in an Azure SQL Database named LumenLegal. The firm has an Azure OpenAI deployment with a chat model and an embeddings model, and both are registered in the database as external models. The innovation team proposes four projects. They ask the database team which one should be built as a retrieval-augmented generation pipeline, RAG for short. RAG means: retrieve relevant private data from LumenLegal at question time, place it in the prompt, and have the chat model answer from it."

**Piece 2, the Matter table.** "There is a schema named Practice with three tables. The first is Practice dot Matter. Five columns. MatterId, an integer, primary key. ClientId, an integer. BilledHours, decimal eight comma two. RatePerHour, decimal eight comma two. And OpenedOn, a date. All not null."

**Piece 3, the ContractClause table.** "The second table is Practice dot ContractClause. It holds about fifty thousand rows and is updated every day. Six columns. ClauseId, integer, primary key. ContractNo, varchar twenty. Heading, nvarchar two hundred. ClauseText, nvarchar max. RevisedOn, datetime2 with zero fractional digits. And Embedding, a vector column of dimension fifteen thirty six, nullable."

**Piece 4, the IntakeForm table.** "The third table is Practice dot IntakeForm. Three columns. FormId, integer, primary key. SubmittedOn, datetime2 zero. And FormText, nvarchar max, which holds about two pages of text per form."

**Piece 5, the partners' constraints.** "The partners have stated three constraints. First: answers about contracts must cite the exact clause they rely on, meaning the ContractNo and the ClauseId, and must reflect revisions made the same day. Second: anything that a deterministic query can answer must be answered by the query. The firm does not want model cost or non-determinism where none is needed. Third: fine-tuning a model is acceptable only for style or format goals, never as a way to teach the model the firm's documents, because documents change daily and fine-tuned knowledge cannot be cited or revoked."

**Piece 6, option a.** "Option a, monthly client statements. For each client, compute total billed hours and total fees for the matters opened in the month. That is SUM of BilledHours, and SUM of BilledHours times RatePerHour, grouped by ClientId. Then produce the statement table."

**Piece 7, option b.** "Option b, house-style letter writer. Every outgoing letter drafted by associates must be rewritten in the firm's distinctive tone, structure and sign-off conventions. A curated set of four hundred approved letters is available as examples. The letters do not reference the contract archive."

**Piece 8, option c.** "Option c, contract-clause assistant. Attorneys ask free-text questions such as: which of our supply contracts allow termination for convenience on less than sixty days' notice? The assistant must answer from the firm's fifty thousand private, daily-revised clauses in Practice dot ContractClause, and cite each clause it relied on."

**Piece 9, option d.** "Option d, intake-form summarizer. As each new Practice dot IntakeForm row arrives, produce a five-line summary of that form's FormText for the intake coordinator. Each form is self-contained. Nothing else needs to be consulted."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE SCHEMA Practice;
GO
CREATE TABLE Practice.Matter
(
    MatterId    int NOT NULL PRIMARY KEY,
    ClientId    int NOT NULL,
    BilledHours decimal(8,2) NOT NULL,
    RatePerHour decimal(8,2) NOT NULL,
    OpenedOn    date NOT NULL
);

CREATE TABLE Practice.ContractClause          -- ~50,000 rows, updated every day
(
    ClauseId    int NOT NULL PRIMARY KEY,
    ContractNo  varchar(20) NOT NULL,
    Heading     nvarchar(200) NOT NULL,
    ClauseText  nvarchar(max) NOT NULL,
    RevisedOn   datetime2(0) NOT NULL,
    Embedding   vector(1536) NULL
);

CREATE TABLE Practice.IntakeForm
(
    FormId      int NOT NULL PRIMARY KEY,
    SubmittedOn datetime2(0) NOT NULL,
    FormText    nvarchar(max) NOT NULL          -- about two pages of text per form
);
```

Sketch of the RAG pipeline for the correct option, from the explanation:

```sql
-- 1. Embed the question with the registered embeddings model
DECLARE @q vector(1536) = AI_GENERATE_EMBEDDINGS(
    N'supply contracts that allow termination for convenience on less than 60 days notice'
    USE MODEL ClauseEmbeddingModel);

-- 2. Retrieve the top clauses (exact nearest-neighbour search; VECTOR_SEARCH for ANN)
SELECT TOP (5) ClauseId, ContractNo, Heading, ClauseText,
       VECTOR_DISTANCE('cosine', @q, Embedding) AS Distance
FROM Practice.ContractClause
WHERE RevisedOn >= DATEADD(year, -3, SYSUTCDATETIME())
ORDER BY Distance;

-- 3. Serialize the rows to JSON, build the chat payload with the clauses as grounding
--    context and the instruction "cite ClauseId/ContractNo for every statement",
--    call sys.sp_invoke_external_rest_endpoint, then extract $.result.choices[0].message.content.
```

## 4. The question (ask exactly this)

"Which project is the proper RAG use case? Option a, option b, option c, or option d?"

Options in full:

- **a.** Monthly client statements: SUM(BilledHours) and SUM(BilledHours * RatePerHour) grouped by ClientId for matters opened in the month.
- **b.** House-style letter writer: rewrite associates' letters in the firm's tone, structure and sign-off, with 400 approved letters as examples; the contract archive is not involved.
- **c.** Contract-clause assistant: answer free-text questions from 50,000 private, daily-revised clauses in Practice.ContractClause and cite each clause used.
- **d.** Intake-form summarizer: a five-line summary of each new self-contained IntakeForm's FormText; nothing else is consulted.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: c.**

- The knowledge is private, so no foundation model has seen it. It changes daily, and the partners demand same-day freshness. Answers must cite ContractNo and ClauseId. The corpus, fifty thousand clauses, is far too large for one prompt. And the question is semantic, over free text, so no SQL predicate expresses it. All of that is exactly what retrieve, ground, generate gives.

Why the others are wrong, one line each:

- **a.** A deterministic SQL reporting query, GROUP BY ClientId with SUMs. It must be exact to the cent; the second constraint forbids a model here.
- **b.** The goal is style and format, not knowledge. That is fine-tuning on the 400 letters, or few-shot prompting. Nothing needs to be looked up; the third constraint reserves fine-tuning for exactly this.
- **d.** Legitimate generative work, but prompt-only. Each two-page form fits in the prompt; there is nothing to retrieve, rank or cite. A vector index would be pure overhead.

## 6. Hint ladder (one hint per attempt, in order)

1. "Ask one question of each project: does answering need knowledge that the model does not have, that is too large or too volatile to put in every prompt, and that must be traceable to a source row?"
2. "Read the partners' second constraint again. Which project is fully answered by a GROUP BY query with exact totals?"
3. "Read the third constraint. Which project is about how the model writes, its voice and format, rather than what it knows?"
4. "Option a is a plain SUM and GROUP BY over Matter. No model needed. That eliminates a."
5. "Option b is about tone, structure and sign-off, with four hundred example letters and no archive lookup. That is a fine-tuning or few-shot job. That eliminates b."
6. "You are down to c and d. Both call a chat model. In one, the whole input is already known and is two pages long. In the other, the answer lives somewhere inside fifty thousand daily-revised rows and must be cited. Which one actually needs a retrieval step?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "d, it uses the LLM inside the database" | Equates any LLM call with RAG | "A model call is not retrieval. What would the pipeline need to search for, and from where, if the whole form is already in hand?" |
| "b, we can retrieve similar letters as examples" | Confuses few-shot examples with grounding knowledge | "Would retrieved letters add facts the answer needs, or only tokens? And what does the third constraint say fine-tuning is for?" |
| "a, the model can write the statement narrative" | Misses that the numbers come from the query, not from retrieval | "Where would the totals come from, the model or a GROUP BY? Does the second constraint allow a model where a query suffices?" |
| "c, but fine-tuning on the clauses would be simpler" | Ignores freshness and citation | "If a clause is revised this morning, when does a fine-tuned model learn it? And can it cite a ClauseId?" |
| "c and d are both RAG" | Thinks summarisation always needs retrieval | "Retrieval selects a small subset from a large corpus. How big is one intake form, and does it fit in the prompt?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the decision ladder from the question:

- Can a deterministic query answer it exactly? Then it is SQL reporting, no model. That is option a: SUM of BilledHours, SUM of BilledHours times RatePerHour, GROUP BY ClientId, filtered by OpenedOn. A model here adds cost, latency and the risk of arithmetic hallucination.
- Is the goal how the model writes, tone and format? Then fine-tune, or use few-shot prompting with several examples. That is option b: four hundred approved letters, no archive involved. A RAG pipeline would retrieve similar letters that add tokens without adding facts.
- Is all the needed text already known and small? Then prompt-only: send it in the prompt. That is option d: one self-contained two-page form goes straight into the chat payload through sp underscore invoke underscore external underscore rest underscore endpoint, and back comes a summary. No search, no ranking, no citation.
- Is the knowledge private, large, changing, and must the answer cite where it came from? Then RAG: retrieve, ground, generate. That is option c.

Then the pipeline for option c, in words:

- Step one, embed the question with AI underscore GENERATE underscore EMBEDDINGS and the registered embeddings model, into a vector of dimension fifteen thirty six.
- Step two, retrieve the top clauses with VECTOR underscore DISTANCE using cosine distance against the Embedding column, ordered by distance, optionally filtered by RevisedOn or ContractNo. VECTOR underscore SEARCH gives approximate nearest neighbours when needed. Because this reads the table at question time, a clause revised this morning and re-embedded by the maintenance job is already what the model sees.
- Step three, serialise the rows to JSON, build the chat payload with the clauses as grounding context and the instruction to cite ClauseId and ContractNo for every statement, call sys dot sp underscore invoke underscore external underscore rest underscore endpoint, and extract the message content from the result.

RAG's selling points are exactly what fine-tuning and prompt-only cannot give: freshness, proprietary data, citations and grounding, hallucination reduction, and bounded cost through top-k.

Memory hook: "Our documents, updated daily, must cite: RAG. Our voice: fine-tune. Exact totals: SQL. Fits in the prompt: just prompt."

## 9. Follow-up oral questions (optional)

1. "Why can a fine-tuned model not satisfy the same-day revision constraint?" (Its knowledge is frozen at training time; it cannot unlearn a revoked clause and cannot cite a row.)
2. "What does the application do with the ClauseId returned alongside each retrieved chunk?" (Puts it in the prompt so the model cites it, then verifies the citation with a database lookup.)
3. "Which two T-SQL functions appear in the retrieval step of the sketched pipeline?" (AI_GENERATE_EMBEDDINGS to embed the question, and VECTOR_DISTANCE to rank clauses; VECTOR_SEARCH for approximate search.)

## 10. References

- Retrieval-augmented generation with SQL Server and Azure SQL: https://learn.microsoft.com/en-us/sql/relational-databases/vectors/retrieval-augmented-generation
- Intelligent applications with Azure SQL Database: https://learn.microsoft.com/en-us/azure/azure-sql/database/ai-artificial-intelligence-intelligent-applications
- AI_GENERATE_EMBEDDINGS: https://learn.microsoft.com/en-us/sql/t-sql/functions/ai-generate-embeddings-transact-sql
- VECTOR_DISTANCE: https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-distance-transact-sql
- sp_invoke_external_rest_endpoint: https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-invoke-external-rest-endpoint-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
