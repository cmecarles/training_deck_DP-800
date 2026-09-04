# Instructor-Examiner guide — Generate Embeddings 1

Companion to [generate_embeddings_1.md](generate_embeddings_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a seven-statement lab question about the vector data type and JSON parsing. The JSON response body matters a lot: read piece 4 slowly and repeat it whenever the learner asks for a path. For S3 and S4 the learner must say both "succeeds, one row" and what value is stored; accept "succeeds" as half right and ask for the stored value. For S7, take the three rows one at a time. Accept "zero point six, zero point eight, zero" as a correct reading of the first vector; do not require the scientific notation digits, but mention them when teaching.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked. For JSON paths say "dollar dot result dot data, index zero, dot embedding".

## 1. Exam skill covered

- Functional group: Implement AI capabilities in database solutions (25–30%).
- Skill: Design and implement models and embeddings.
- Task bullet: Generate embeddings.
- What is tested: casting a JSON array to `vector(n)`, the dimension-mismatch error 42204, the difference between `JSON_VALUE` and `JSON_QUERY` on arrays, the `$.result` envelope of `sp_invoke_external_rest_endpoint`, parsing a batched response with `OPENJSON ... AS JSON` joined on `index`, and the `AI_GENERATE_EMBEDDINGS` alternative.

## 2. Scenario to read aloud

**Piece 1, the story.** "A patent office keeps patent abstracts in a SQL Server 2025 database called PatentVault. They produce embeddings by calling an Azure OpenAI embeddings endpoint with the stored procedure sp underscore invoke underscore external underscore rest underscore endpoint. They send several abstracts per request, in batches, and parse the JSON response in T-SQL. For this exercise the model is a toy one that returns only three dimensions."

**Piece 2, the abstracts table.** "There is a schema called Ip, as in intellectual property. In it, a table Ip dot Abstracts with three columns. PatentId, an integer, the primary key. AbstractText, an NVARCHAR MAX that allows NULL. And Embedding, of type VECTOR open paren three close paren, which also allows NULL."

**Piece 3, the abstract rows.** "Three rows are inserted, with only PatentId and AbstractText. Patent 1, text: a self-closing hinge with a damped return spring. Patent 2, the text is NULL. Patent 3, text: a pressure-relief valve for hydraulic lines. The Embedding column is NULL for all three."

**Piece 4, the saved response.** "There is a second table, Ip dot ModelResponses, with ResponseId, an integer primary key, and Body, an NVARCHAR MAX that is NOT NULL. One row is inserted, ResponseId 1. Its Body is the JSON that sp underscore invoke underscore external underscore rest underscore endpoint put into its at-response output parameter for one batch, the batch that contained abstracts 1 and 3, in PatentId order. The JSON looks like this. At the top level there are two keys. The first key is response, and inside it a status object with an http code of 200. The second key is result, and inside result there is an object key with value list, a data key that is an array of two elements, a model key with value text-embedding-3-small, and a usage object. The data array element with index zero has an embedding of open bracket zero point six, zero point eight, zero close bracket. The element with index one has an embedding of open bracket zero, zero, one close bracket."

**Piece 5, statements one and two.** "Seven statements then run in order, each in its own batch. S1 is a SELECT that casts the string literal open bracket zero point six, zero point eight, zero close bracket to VECTOR three, aliased V. S2 is a SELECT from ModelResponses where ResponseId is 1. It applies JSON underscore QUERY to Body with the path dollar dot result dot data index zero dot embedding, and casts the result to VECTOR four. Four, not three."

**Piece 6, statements three and four.** "S3 is an UPDATE of Abstracts, aliased a, cross joined to ModelResponses, aliased r, filtered to PatentId 1 and ResponseId 1. It sets a dot Embedding to the CAST as VECTOR three of JSON underscore VALUE, not QUERY, of r dot Body with the path dollar dot result dot data index zero dot embedding. S4 is the same UPDATE but with JSON underscore QUERY, and the path is dollar dot data index zero dot embedding. Notice: no result in that path."

**Piece 7, statement five, the batch update.** "S5 starts with a CTE called Batch. It selects PatentId and a column Idx, computed as ROW underscore NUMBER over order by PatentId, minus one, from Abstracts, where AbstractText is not NULL. So Idx is zero-based. Then it updates Abstracts aliased a, inner joined to Batch on PatentId, cross joined to ModelResponses aliased r, and cross applied to OPENJSON of r dot Body with path dollar dot result dot data, using a WITH clause that maps Idx as INT from dollar dot index and Emb as NVARCHAR MAX from dollar dot embedding AS JSON. The alias of that OPENJSON is j. The WHERE clause keeps ResponseId 1 and j dot Idx equal to b dot Idx. It sets a dot Embedding to CAST of j dot Emb as VECTOR three."

**Piece 8, statements six and seven.** "S6 is an UPDATE of Abstracts that sets Embedding to the string open bracket one, zero close bracket, two numbers, where PatentId is 2. S7 is a SELECT from Abstracts ordered by PatentId, with three columns: PatentId, the Embedding cast to VARCHAR sixty, aliased EmbeddingText, and VECTORPROPERTY of Embedding with the argument Dimensions, aliased Dims."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE PatentVault;
GO
USE PatentVault;
GO
CREATE SCHEMA Ip;
GO
CREATE TABLE Ip.Abstracts
(
    PatentId     INT           NOT NULL PRIMARY KEY,
    AbstractText NVARCHAR(MAX) NULL,
    Embedding    VECTOR(3)     NULL
);
GO
INSERT INTO Ip.Abstracts (PatentId, AbstractText) VALUES
    (1, N'A self-closing hinge with a damped return spring.'),
    (2, NULL),
    (3, N'A pressure-relief valve for hydraulic lines.');
GO
CREATE TABLE Ip.ModelResponses
(
    ResponseId INT           NOT NULL PRIMARY KEY,
    Body       NVARCHAR(MAX) NOT NULL
);
GO
INSERT INTO Ip.ModelResponses (ResponseId, Body) VALUES (1, N'{
  "response": { "status": { "http": { "code": 200, "description": "" } } },
  "result": {
    "object": "list",
    "data": [
      { "object": "embedding", "index": 0, "embedding": [0.6, 0.8, 0] },
      { "object": "embedding", "index": 1, "embedding": [0, 0, 1] }
    ],
    "model": "text-embedding-3-small",
    "usage": { "prompt_tokens": 41, "total_tokens": 41 }
  }
}');
GO
-- S1
SELECT CAST('[0.6, 0.8, 0]' AS VECTOR(3)) AS V;

-- S2
SELECT CAST(JSON_QUERY(Body, '$.result.data[0].embedding') AS VECTOR(4)) AS V
FROM Ip.ModelResponses WHERE ResponseId = 1;

-- S3
UPDATE a
SET a.Embedding = CAST(JSON_VALUE(r.Body, '$.result.data[0].embedding') AS VECTOR(3))
FROM Ip.Abstracts AS a
CROSS JOIN Ip.ModelResponses AS r
WHERE a.PatentId = 1 AND r.ResponseId = 1;

-- S4
UPDATE a
SET a.Embedding = CAST(JSON_QUERY(r.Body, '$.data[0].embedding') AS VECTOR(3))
FROM Ip.Abstracts AS a
CROSS JOIN Ip.ModelResponses AS r
WHERE a.PatentId = 1 AND r.ResponseId = 1;

-- S5
WITH Batch AS
(
    SELECT PatentId, ROW_NUMBER() OVER (ORDER BY PatentId) - 1 AS Idx
    FROM Ip.Abstracts
    WHERE AbstractText IS NOT NULL
)
UPDATE a
SET a.Embedding = CAST(j.Emb AS VECTOR(3))
FROM Ip.Abstracts AS a
INNER JOIN Batch AS b ON b.PatentId = a.PatentId
CROSS JOIN Ip.ModelResponses AS r
CROSS APPLY OPENJSON(r.Body, '$.result.data')
            WITH (Idx INT '$.index', Emb NVARCHAR(MAX) '$.embedding' AS JSON) AS j
WHERE r.ResponseId = 1 AND j.Idx = b.Idx;

-- S6
UPDATE Ip.Abstracts SET Embedding = '[1, 0]' WHERE PatentId = 2;

-- S7
SELECT PatentId,
       CAST(Embedding AS VARCHAR(60))          AS EmbeddingText,
       VECTORPROPERTY(Embedding, 'Dimensions') AS Dims
FROM Ip.Abstracts
ORDER BY PatentId;
```

## 4. The question (ask exactly this)

"For statements S1 to S6, tell me whether each one succeeds or raises an error. For the ones that succeed, tell me how many rows are affected, and for S3 and S4 also tell me what value is stored in the Embedding column. Let's go one at a time, starting with S1."

After S1 to S6: "Now give me the exact result of S7: for each of the three patents, the EmbeddingText and the Dims value."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

| Stmt | Outcome | Detail |
|---|---|---|
| S1 | Succeeds | Returns the vector `[6.0000002e-001,8.0000001e-001,0.0000000e+000]`, that is 0.6, 0.8, 0 shown as float32 |
| S2 | Fails, error 42204 | "The vector dimensions 4 and 3 do not match." Declared 4, supplied 3 |
| S3 | Succeeds, 1 row | Stores NULL. `JSON_VALUE` returns NULL for an array in lax mode |
| S4 | Succeeds, 1 row | Stores NULL again. The path skips the `result` wrapper, so `JSON_QUERY` finds nothing |
| S5 | Succeeds, 2 rows | Patents 1 and 3 get their vectors, matched by zero-based index |
| S6 | Fails, error 42204 | "The vector dimensions 3 and 2 do not match." Patent 2 stays NULL |

S7:

| PatentId | EmbeddingText | Dims |
|---|---|---|
| 1 | `[6.0000002e-001,8.0000001e-001,0.0000000e+000]` (0.6, 0.8, 0) | 3 |
| 2 | NULL | NULL |
| 3 | `[0.0000000e+000,0.0000000e+000,1.0000000e+000]` (0, 0, 1) | 3 |

Patent 1's NULL from S3 and S4 was overwritten by S5. Patent 3 was filled by S5. Patent 2 was never embedded.

## 6. Hint ladder (one hint per attempt, in order)

**S1**
1. "What does a vector literal look like in T-SQL? Is it a JSON array inside a string?"
2. "The string has three numbers and the target is VECTOR three. Does that match?"
3. "It succeeds. When the vector is shown as text, it is printed as single-precision floats. Zero point six is not exact in single precision. What might the display look like?"

**S2**
1. "First, does JSON underscore QUERY with that path find the array? Look at the path again: dollar dot result dot data zero dot embedding."
2. "The array is found. Now compare its length with the number in VECTOR open paren four close paren."
3. "Three values into a four-dimensional vector. Is that a silent success, a NULL, or an error?"
4. "It is an error. The message names the declared size first and the supplied size second."

**S3**
1. "Two things differ from S2. Look at the function name. Is it QUERY or VALUE?"
2. "What does JSON underscore VALUE return when the path points to an array, not a scalar, in the default lax mode?"
3. "It returns NULL, with no error. What does CAST of NULL as VECTOR three give? And does the UPDATE still count the row?"

**S4**
1. "Back to JSON underscore QUERY, so arrays are fine. Now read the path very carefully. Which top-level key is missing?"
2. "The procedure wraps the endpoint's JSON in an envelope. What are the two top-level keys of that envelope, from piece 4?"
3. "Dollar dot data does not exist at the top level. In lax mode, a missing path returns what? And the row is still updated."

**S5**
1. "Start with the CTE. Which patents have a non-NULL AbstractText, and what Idx does each one get?"
2. "Patent 1 gets Idx zero, patent 3 gets Idx one. Now, what does OPENJSON with path dollar dot result dot data return, and what is in its index column?"
3. "Note the AS JSON on the Emb column. Without it, an array would come back as NULL. With it, the array comes back as text. Can that text be cast to VECTOR three?"
4. "Each of the two batch rows matches one OPENJSON row on index. How many rows are updated?"

**S6**
1. "There is no CAST here. The string is converted implicitly to the column type. What is the column type?"
2. "Count the numbers in the string and compare with the column's dimension count. Is implicit conversion any more forgiving than an explicit CAST?"
3. "It is the same error as S2, but with the declared and supplied numbers of this case."

**S7**
1. "Which statements actually changed data and survived? Remember S5 ran after S3 and S4."
2. "Patent 2 had a NULL text, so it was never in the batch, and S6 failed. What is its Embedding?"
3. "What does VECTORPROPERTY with Dimensions return for a stored three-dimensional vector, and what for a NULL?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "S1 fails, you cannot cast a string to a vector" | Does not know a vector literal is a JSON array string | "How else would you write a vector literal in T-SQL? Think JSON." |
| "S2 returns NULL" or "S2 pads with a zero" | Thinks dimension mismatch is tolerated | "Is the engine lenient about the dimension count? Think of a production column of 1536 dimensions and a model that returns 3072." |
| "S2 error says three and four" | Reverses the order in the message | "Which number does the engine name first: the declared size or the supplied size?" |
| "S3 stores the vector zero point six, zero point eight, zero" | Confuses JSON_VALUE with JSON_QUERY | "Which function extracts scalars, and which extracts objects and arrays? Which one is used here?" |
| "S3 raises an error" | Assumes strict mode | "Is the path in strict mode or lax mode? What does lax mode do when the path is not a scalar?" |
| "S4 stores the vector" | Missed the missing `result` key in the path | "Read the path once more and compare with the top-level keys of the envelope." |
| "S5 updates one row" or "three rows" | Forgets the NULL filter or the index match | "How many rows does the Batch CTE contain? And how many elements does the data array have?" |
| "S5 stores NULL because OPENJSON returns NULL for arrays" | Missed the `AS JSON` option in the WITH clause | "Look at the Emb column definition in the WITH clause. Is there an option after the path?" |
| "S6 succeeds, it is just a string" | Thinks implicit conversion skips the dimension check | "The column is VECTOR three. Does the engine check dimensions on implicit conversion too?" |
| "Patent 1 is NULL in S7" | Forgets S5 overwrote the NULL | "S3 and S4 wrote NULL. Did anything write to patent 1 after that?" |
| "Dims for patent 2 is zero" | Thinks a property of NULL is zero | "What does a function return when its input is NULL?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the two kinds of failure, loud and silent:

- **Loud: dimension mismatch, error 42204.** A vector column or CAST target declares a dimension count. If the JSON array has a different length, the engine raises "The vector dimensions declared and supplied do not match", naming the declared size first. That is S2, four versus three, and S6, three versus two. Implicit conversion is checked exactly like explicit CAST. In production this is what happens when the column is vector 1536 and the model is switched to one returning 3072 or 1024 values.
- **Silent: NULL from the wrong JSON function or the wrong path.** `JSON_VALUE` extracts scalars only. In the default lax mode it returns NULL when the path points to an array or an object. In strict mode it would raise error 13623 instead. That is S3. `JSON_QUERY` is the right function for arrays and objects, but a path that does not exist also returns NULL in lax mode. That is S4. Both statements say "one row affected" and quietly wipe the embedding. This bug shows up weeks later as "all my distances are NULL".

Then the envelope rule:

- `sp_invoke_external_rest_endpoint` does not put the HTTP body straight into the at-response parameter. It wraps it. The top-level keys are `response`, with status and headers, and `result`, with the endpoint's own JSON. So every path must start with dollar dot result. The official Azure OpenAI example uses exactly `JSON_QUERY(@response, '$.result.data[0].embedding')`.

Then batching, S5:

- The request body for a batch is a JSON array of inputs, built for example with `JSON_OBJECT('input': JSON_ARRAYAGG(AbstractText ORDER BY PatentId))` over the non-NULL texts. The response carries one element per input with a zero-based `index`. S5 reproduces the same order with `ROW_NUMBER() OVER (ORDER BY PatentId) - 1` over the same filtered set, shreds dollar dot result dot data with `OPENJSON ... WITH`, and joins on the index. The `AS JSON` option on the Emb column is what returns the array as text instead of NULL. Two rows match, patents 1 and 3.
- Keep NULL texts out of the batch. A NULL input would shift every index after it. Batching saves HTTPS round trips; Azure OpenAI accepts up to 2,048 inputs per embeddings request, with a 300,000-token aggregate limit and 8,192 tokens per input.

Then the display detail, S1 and S7:

- Vectors are stored as float32. When converted back to text, each element prints in scientific notation. Zero point six is not exact in single precision, so it displays as 6.0000002e-001, and zero point eight as 8.0000001e-001. That is a display artefact, not corruption. Only numbers are allowed in the array; a string element fails with error 13670. `VECTORPROPERTY(v, 'Dimensions')` returns 3 for a stored vector and NULL for NULL; `'BaseType'` would return float32.

Finally, the inline alternative, `AI_GENERATE_EMBEDDINGS`:

- With an external model registered, the whole S5 pipeline becomes `SET Embedding = AI_GENERATE_EMBEDDINGS(AbstractText USE MODEL AbstractEmbedder)`. The column size must still match the model's output, same error 42204 otherwise. Three behaviours to remember: without `sp_configure 'external rest endpoint enabled', 1` it fails with error 31643; a NULL input is not skipped but raises error 31701, so filter `WHERE AbstractText IS NOT NULL`; and when the endpoint is unreachable the whole statement fails with error 31608 and the UPDATE rolls back as a unit, so batch the UPDATE by ranges of PatentId.

Memory hook: "JSON VALUE for scalars, JSON QUERY for arrays, always through dollar dot result. Wrong length is loud, wrong path is silent."

## 9. Follow-up oral questions (optional)

1. "If S3 had used a strict path, dollar dot result dot data zero dot embedding with the word strict in front, what would have happened?" (An error, 13623, scalar value cannot be found in the specified JSON path, instead of a silent NULL.)
2. "In S5, what would go wrong if patent 2 had been included in the batch with its NULL text?" (Every index after it would shift by one, so patent 3 would get the wrong embedding, or the request itself would be rejected for a null input.)
3. "You switch the model to one that returns 1536 values but the column is VECTOR 1998. What happens on the first UPDATE?" (Error 42204, the vector dimensions 1998 and 1536 do not match.)
4. "What error does AI underscore GENERATE underscore EMBEDDINGS raise for a NULL input, and how do you avoid it?" (Error 31701; filter with WHERE text IS NOT NULL.)

## 10. References

- Vector data type: https://learn.microsoft.com/en-us/sql/t-sql/data-types/vector-data-type
- VECTORPROPERTY: https://learn.microsoft.com/en-us/sql/t-sql/functions/vectorproperty-transact-sql
- JSON_VALUE: https://learn.microsoft.com/en-us/sql/t-sql/functions/json-value-transact-sql
- JSON_QUERY: https://learn.microsoft.com/en-us/sql/t-sql/functions/json-query-transact-sql
- OPENJSON with an explicit schema, including AS JSON: https://learn.microsoft.com/en-us/sql/t-sql/functions/openjson-transact-sql
- sp_invoke_external_rest_endpoint, response envelope and Azure OpenAI example: https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-invoke-external-rest-endpoint-transact-sql
- AI_GENERATE_EMBEDDINGS: https://learn.microsoft.com/en-us/sql/t-sql/functions/ai-generate-embeddings-transact-sql
- CREATE EXTERNAL MODEL: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-external-model-transact-sql
- Azure OpenAI embeddings request limits: https://learn.microsoft.com/en-us/azure/ai-services/openai/reference
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
