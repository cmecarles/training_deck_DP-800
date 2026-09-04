# SQL Server question — Generate Embeddings 1

## Statement

A patent office stores abstracts in a SQL Server 2025 database named `PatentVault`. Embeddings are produced by calling an Azure OpenAI embeddings endpoint with `sp_invoke_external_rest_endpoint` in **batches** (several abstracts per request) and parsing the JSON response in T-SQL.

For this exercise the model is a toy 3-dimensional one, and the JSON that `sp_invoke_external_rest_endpoint` placed in its `@response` output parameter for one batch (abstracts 1 and 3, in `PatentId` order) has been saved into a table so that it can be reused across batches:

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
```

The following seven statements are executed **in order, each in its own batch**:

```sql
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

For S1–S6 state whether each **succeeds or raises an error** (for successes: rows affected and, for S3/S4, the value stored). Then give the exact result of S7.

## Correct Answer

| Stmt | Outcome | Detail |
|---|---|---|
| S1 | **Succeeds** | returns `[6.0000002e-001,8.0000001e-001,0.0000000e+000]` |
| S2 | **Fails** | `Msg 42204` — `The vector dimensions 4 and 3 do not match.` |
| S3 | **Succeeds** | `(1 rows affected)` — but stores **`NULL`**: `JSON_VALUE` returns `NULL` for an array |
| S4 | **Succeeds** | `(1 rows affected)` — stores **`NULL`** again: the path skips the `result` wrapper |
| S5 | **Succeeds** | `(2 rows affected)` — patents 1 and 3 get their vectors |
| S6 | **Fails** | `Msg 42204` — `The vector dimensions 3 and 2 do not match.` |

S7:

| PatentId | EmbeddingText | Dims |
|---|---|---|
| 1 | `[6.0000002e-001,8.0000001e-001,0.0000000e+000]` | 3 |
| 2 | NULL | NULL |
| 3 | `[0.0000000e+000,0.0000000e+000,1.0000000e+000]` | 3 |

## Explanation

### S1 — a JSON array string casts to `vector(n)`

A `vector` literal is a JSON array in a string; `CAST('[0.6, 0.8, 0]' AS VECTOR(3))` succeeds. When a vector is converted back to text the engine prints each element as a float32 in scientific notation: `0.6` is not exactly representable in single precision, so it displays as `6.0000002e-001` (and `0.8` as `8.0000001e-001`). Those digits are display artefacts of the 4-byte storage, not data corruption. Only numbers are allowed inside the array: `CAST('[1,2,"a"]' AS VECTOR(3))` fails with `Msg 13670 Input JSON is not a valid Vector : 'String not Supported'.`

### S2 — the declared dimension count must match the array

`JSON_QUERY(Body, '$.result.data[0].embedding')` correctly returns the array `[0.6, 0.8, 0]`, but the target type is `VECTOR(4)`. The engine reports the declared size first and the supplied size second: `The vector dimensions 4 and 3 do not match.` This is the error you get in production when the column is `vector(1536)` and the model was switched to one that returns 3,072 (or 1,024) values.

### S3 — `JSON_VALUE` on an array silently yields `NULL`

This is the trap. `JSON_VALUE` extracts **scalars**; in the default *lax* path mode it returns `NULL` when the path points to an object or an array (in *strict* mode it would raise `Msg 13623 Scalar value cannot be found in the specified JSON path.`). `CAST(NULL AS VECTOR(3))` is `NULL`, the `UPDATE` affects one row, and the embedding is quietly wiped. No error, `(1 rows affected)` — the kind of bug that surfaces weeks later as "all my distances are `NULL`". For arrays and objects use `JSON_QUERY`.

### S4 — the wrong path is also silently `NULL`

`sp_invoke_external_rest_endpoint` does not put the HTTP body straight into `@response`: it wraps it in an envelope whose top-level properties are `response` (status, headers) and `result` (the endpoint's JSON). The path must therefore start at `$.result.data[0].embedding`; `$.data[0].embedding` does not exist, so `JSON_QUERY` returns `NULL` in lax mode and the row is updated to `NULL` once more. The engine's own example for Azure OpenAI uses exactly `JSON_QUERY(@response, '$.result.data[0].embedding')`.

### S5 — batching: one request, many embeddings, matched by `index`

The request body for a batch is built with a JSON array of inputs — for these rows `SELECT JSON_OBJECT('input': JSON_ARRAYAGG(AbstractText ORDER BY PatentId)) FROM Ip.Abstracts WHERE AbstractText IS NOT NULL` returns `{"input":["A self-closing hinge with a damped return spring.","A pressure-relief valve for hydraulic lines."]}`. The response carries one element per input with its 0-based `index`. S5 reproduces that ordering with `ROW_NUMBER() OVER (ORDER BY PatentId) - 1` over the **same filtered, same-ordered** set, shreds `$.result.data` with `OPENJSON ... WITH (... AS JSON)` (the `AS JSON` option is what returns the array as text rather than `NULL`), and joins on the index. Two rows match — patents 1 and 3 — and each array is cast to `VECTOR(3)`.

Batching is how cost and latency are controlled: fewer HTTPS round trips, and Azure OpenAI accepts up to 2,048 inputs per embeddings request (with a 300,000-token aggregate limit and 8,192 tokens per input). Keep `NULL` texts out of the batch: they would shift every index after them.

### S6 — dimension check on implicit conversion too

The string `'[1, 0]'` is converted implicitly to the column type `VECTOR(3)` and fails the same way as an explicit `CAST`: `The vector dimensions 3 and 2 do not match.` (declared 3, supplied 2). Patent 2 keeps its `NULL` embedding.

### S7 — final state

Patent 1's `NULL` from S3/S4 was overwritten by S5; patent 3 was filled by S5; patent 2 was never embedded (`NULL` text, S6 failed). `VECTORPROPERTY(v, 'Dimensions')` returns 3 for stored vectors and `NULL` for `NULL` (`'BaseType'` would return `float32`).

### The inline alternative: `AI_GENERATE_EMBEDDINGS`

When an external model is registered, the whole S5 pipeline collapses to one expression:

```sql
UPDATE Ip.Abstracts
SET Embedding = AI_GENERATE_EMBEDDINGS(AbstractText USE MODEL AbstractEmbedder)
WHERE AbstractText IS NOT NULL AND Embedding IS NULL;
```

The function returns the vector array ready to store in a `vector(n)` column (the column size must still match the model's output — the same `Msg 42204` otherwise). Three behaviours verified on this instance with a registered model are worth remembering:

- Without `sp_configure 'external rest endpoint enabled', 1` it fails with `Msg 31643 'ai_generate_embeddings' is disabled on this instance of SQL Server. Use sp_configure 'external rest endpoint enabled' to enable it.`
- A `NULL` input is **not** skipped: it raises `Msg 31701 Parameter 'input data' must be specified. This parameter cannot be NULL.` — hence the `WHERE AbstractText IS NOT NULL` filter above.
- When the endpoint cannot be reached the whole statement fails (`Msg 31608 An error occurred, failed to communicate with the external rest endpoint. HRESULT: 0x80072ee7.`), so the `UPDATE` rolls back as a unit; batch the `UPDATE` by ranges of `PatentId` to bound the cost of a failed call.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

## DP-800 Exam Rule to Remember

```text
Two ways to generate embeddings in T-SQL
  1. AI_GENERATE_EMBEDDINGS(text USE MODEL m [PARAMETERS @json])   -- inline in UPDATE/INSERT/SELECT
       needs: CREATE EXTERNAL MODEL, EXECUTE on it, 'external rest endpoint enabled' = 1
       NULL input → error 31701 → filter WHERE text IS NOT NULL
  2. sp_invoke_external_rest_endpoint ... @response OUTPUT            -- manual, batchable
       body   = JSON_OBJECT('input': JSON_ARRAYAGG(text ORDER BY key))
       vector = CAST(JSON_QUERY(@response, '$.result.data[i].embedding') AS VECTOR(n))
       batch  = OPENJSON(@response, '$.result.data') WITH (index INT, embedding NVARCHAR(MAX) AS JSON)
```

Silent-`NULL` traps: `JSON_VALUE` on an array, and a path without the `$.result` wrapper. Loud errors: declared `vector(n)` ≠ array length → `Msg 42204 The vector dimensions <declared> and <supplied> do not match.`
