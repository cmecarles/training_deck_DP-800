# SQL Server question — Retrieval-Augmented Generation 1

## Statement

A travel agency runs its customer-support assistant on top of an Azure SQL Database named `TripDesk`.

The assistant answers customer questions by grounding an Azure OpenAI **chat-completions** model on two kinds of data:

- The customer's booking row, converted to JSON.
- A policy text chunk already selected by the retrieval step (the vector search itself is out of scope for this question).

The database contains the following objects:

```sql
CREATE SCHEMA Support;
GO

CREATE TABLE Support.Bookings
(
    BookingId     int NOT NULL PRIMARY KEY,
    CustomerName  nvarchar(100) NOT NULL,
    Destination   nvarchar(100) NOT NULL,
    DepartureDate date NOT NULL,
    TotalAmount   decimal(10,2) NOT NULL
);

CREATE TABLE Support.PolicyChunks
(
    ChunkId   int NOT NULL PRIMARY KEY,
    ChunkText nvarchar(max) NOT NULL
);
GO

INSERT INTO Support.Bookings
VALUES (1001, N'Elena Ruiz', N'Lisbon', '2026-09-14', 1480.00);

INSERT INTO Support.PolicyChunks
VALUES (7, N'Under the "FlexCancel" plan, a booking can be cancelled up to 24 hours before departure for a 90% refund.');
GO
```

Note that `ChunkText` for the retrieved chunk (`ChunkId = 7`) contains **double quotation marks** around `"FlexCancel"`.

A database scoped credential already exists and injects the Azure OpenAI API key as a request header:

```sql
CREATE DATABASE SCOPED CREDENTIAL [https://tripdesk-ai.openai.azure.com]
WITH IDENTITY = 'HTTPEndpointHeaders',
     SECRET = '{"api-key":"<key-value>"}';
GO
```

The executing principal has `EXECUTE ANY EXTERNAL ENDPOINT` permission and `REFERENCES` permission on the credential.

You must write one T-SQL batch that, for booking `1001` and policy chunk `7`:

1. Builds a **valid** chat-completions request payload, even though `ChunkText` contains double quotation marks.
2. Sends the request to the `support-gpt` deployment by using `sys.sp_invoke_external_rest_endpoint`, authenticating with the existing credential.
3. Returns the assistant's reply text — a scalar JSON string — in a column named `Answer`. Returning `NULL` or raising an error is not acceptable.

Which batch should you use?

### a.

```sql
DECLARE @bookingJson nvarchar(max) =
    (SELECT BookingId, CustomerName, Destination, DepartureDate, TotalAmount
     FROM Support.Bookings
     WHERE BookingId = 1001
     FOR JSON PATH, WITHOUT_ARRAY_WRAPPER);

DECLARE @context nvarchar(max) =
    (SELECT ChunkText FROM Support.PolicyChunks WHERE ChunkId = 7);

DECLARE @payload nvarchar(max) =
    N'{"messages":[{"role":"user","content":"Policy: ' + @context
    + N' Booking: ' + @bookingJson
    + N' Question: Can I cancel my trip?"}]}';

DECLARE @ret int, @response nvarchar(max);

EXEC @ret = sys.sp_invoke_external_rest_endpoint
    @url = N'https://tripdesk-ai.openai.azure.com/openai/deployments/support-gpt/chat/completions?api-version=2024-10-21',
    @method = N'POST',
    @credential = [https://tripdesk-ai.openai.azure.com],
    @payload = @payload,
    @response = @response OUTPUT;

SELECT JSON_VALUE(@response, '$.result.choices[0].message.content') AS Answer;
```

### b.

```sql
DECLARE @bookingJson nvarchar(max) =
    (SELECT BookingId, CustomerName, Destination, DepartureDate, TotalAmount
     FROM Support.Bookings
     WHERE BookingId = 1001
     FOR JSON PATH, WITHOUT_ARRAY_WRAPPER);

DECLARE @context nvarchar(max) =
    (SELECT ChunkText FROM Support.PolicyChunks WHERE ChunkId = 7);

DECLARE @payload nvarchar(max) =
    N'{"messages":[{"role":"user","content":"Policy: '
    + STRING_ESCAPE(@context, 'json')
    + N' Booking: '
    + STRING_ESCAPE(@bookingJson, 'json')
    + N' Question: Can I cancel my trip?"}]}';

DECLARE @ret int, @response nvarchar(max);

EXEC @ret = sys.sp_invoke_external_rest_endpoint
    @url = N'https://tripdesk-ai.openai.azure.com/openai/deployments/support-gpt/chat/completions?api-version=2024-10-21',
    @method = N'POST',
    @credential = [https://tripdesk-ai.openai.azure.com],
    @payload = @payload,
    @response = @response OUTPUT;

SELECT JSON_VALUE(@response, '$.choices[0].message.content') AS Answer;
```

### c.

```sql
DECLARE @bookingJson nvarchar(max) =
    (SELECT BookingId, CustomerName, Destination, DepartureDate, TotalAmount
     FROM Support.Bookings
     WHERE BookingId = 1001
     FOR JSON PATH, WITHOUT_ARRAY_WRAPPER);

DECLARE @context nvarchar(max) =
    (SELECT ChunkText FROM Support.PolicyChunks WHERE ChunkId = 7);

DECLARE @payload nvarchar(max) = JSON_OBJECT(
    'messages': JSON_ARRAY(
        JSON_OBJECT(
            'role':'system',
            'content':'Answer using only the provided policy and booking data.'),
        JSON_OBJECT(
            'role':'user',
            'content': N'Policy: ' + @context
                + N' Booking: ' + @bookingJson
                + N' Question: Can I cancel my trip?')));

DECLARE @ret int, @response nvarchar(max);

EXEC @ret = sys.sp_invoke_external_rest_endpoint
    @url = N'https://tripdesk-ai.openai.azure.com/openai/deployments/support-gpt/chat/completions?api-version=2024-10-21',
    @method = N'POST',
    @credential = [https://tripdesk-ai.openai.azure.com],
    @payload = @payload,
    @response = @response OUTPUT;

SELECT JSON_VALUE(@response, '$.result.choices[0].message.content') AS Answer;
```

### d.

```sql
DECLARE @bookingJson nvarchar(max) =
    (SELECT BookingId, CustomerName, Destination, DepartureDate, TotalAmount
     FROM Support.Bookings
     WHERE BookingId = 1001
     FOR JSON PATH, WITHOUT_ARRAY_WRAPPER);

DECLARE @context nvarchar(max) =
    (SELECT ChunkText FROM Support.PolicyChunks WHERE ChunkId = 7);

DECLARE @payload nvarchar(max) = JSON_OBJECT(
    'messages': JSON_ARRAY(
        JSON_OBJECT(
            'role':'system',
            'content':'Answer using only the provided policy and booking data.'),
        JSON_OBJECT(
            'role':'user',
            'content': N'Policy: ' + @context
                + N' Booking: ' + @bookingJson
                + N' Question: Can I cancel my trip?')));

DECLARE @ret int, @response nvarchar(max);

EXEC @ret = sys.sp_invoke_external_rest_endpoint
    @url = N'https://tripdesk-ai.openai.azure.com/openai/deployments/support-gpt/chat/completions?api-version=2024-10-21',
    @method = N'POST',
    @credential = [https://tripdesk-ai.openai.azure.com],
    @payload = @payload,
    @response = @response OUTPUT;

SELECT JSON_QUERY(@response, '$.result.choices[0].message.content') AS Answer;
```

## Correct Answer

**c**

## Explanation

The correct answer is **c**.

This question tests the three fragile points of the T-SQL "glue" of a RAG pipeline in Azure SQL Database:

1. The request payload must be **valid JSON**, even when the retrieved context contains characters that are special in JSON (here, the double quotation marks around `"FlexCancel"`).
2. `sp_invoke_external_rest_endpoint` does **not** put the HTTP body directly into `@response`. It wraps it in an envelope: the HTTP metadata is under the `response` property, and the JSON payload returned by the endpoint is under the **`result`** property.
3. The assistant's reply text is a **scalar** JSON string, so it must be extracted with `JSON_VALUE`, not `JSON_QUERY`.

### Why option c is correct

Walking the batch end to end:

- `FOR JSON PATH, WITHOUT_ARRAY_WRAPPER` converts the single booking row into one JSON object such as:

  ```json
  {"BookingId":1001,"CustomerName":"Elena Ruiz","Destination":"Lisbon","DepartureDate":"2026-09-14","TotalAmount":1480.00}
  ```

  This is the epigraph's "convert structured data to JSON for language model processing" step.

- The payload is assembled with the JSON constructors `JSON_OBJECT` and `JSON_ARRAY`. When a plain string expression is supplied as a value, the constructors **escape it automatically**: the `"` characters inside `ChunkText` and inside `@bookingJson` become `\"` in the serialized output. The result is a valid chat-completions request:

  ```json
  {"messages":[{"role":"system","content":"..."},{"role":"user","content":"Policy: Under the \"FlexCancel\" plan, ... Booking: {\"BookingId\":1001,...} Question: Can I cancel my trip?"}]}
  ```

  The chat-completions API expects exactly this shape: a `messages` array of objects with `role` and `content`.

- The procedure call is complete and correct:
  - `@url` targets the deployment's `chat/completions` endpoint (`nvarchar(4000)`, required).
  - `@method = N'POST'` (this is also the documented default, so it could even be omitted).
  - `@credential` names the existing `DATABASE SCOPED CREDENTIAL`; because it was created `WITH IDENTITY = 'HTTPEndpointHeaders'`, the `api-key` header is injected automatically. No `@headers` parameter is needed — the procedure already injects `content-type: application/json`.
  - `@response` is declared `nvarchar(max)` and passed with `OUTPUT`, which is how the procedure hands back the response document.

- The extraction accounts for the envelope. `@response` contains:

  ```json
  {
    "response": { "status": { "http": { "code": 200, "description": "OK" } }, "headers": { ... } },
    "result": {
      "id": "chatcmpl-...",
      "choices": [
        { "index": 0, "finish_reason": "stop",
          "message": { "role": "assistant", "content": "Yes — under the FlexCancel plan ..." } }
      ],
      "usage": { ... }
    }
  }
  ```

  `JSON_VALUE(@response, '$.result.choices[0].message.content')` navigates through `result` into the chat-completions body and returns the scalar string, so `Answer` contains the assistant's reply. (`JSON_VALUE` returns `nvarchar(4000)` by default, and in lax mode a longer value comes back as `NULL`; a support answer fits well within that limit. For longer completions you would switch to `OPENJSON`, or cast the response to the **json** type and use `JSON_VALUE ... RETURNING nvarchar(max)` — the `RETURNING` clause is only supported on **json**-typed input.)

### Why option a is wrong

Option a builds the payload by naive string concatenation:

```sql
N'{"messages":[{"role":"user","content":"Policy: ' + @context + ...
```

`@context` contains literal `"` characters (`"FlexCancel"`), and `@bookingJson` is itself full of `"` characters. Concatenated raw between the double quotation marks of the `content` string, the first embedded `"` terminates the JSON string early and everything after it is garbage:

```json
{"messages":[{"role":"user","content":"Policy: Under the "FlexCancel" plan, ...
```

This is **not a valid JSON document**. The documentation for `sp_invoke_external_rest_endpoint` is explicit: "*Payloads must be a valid JSON document, a well formed XML document, or text*" — and with the auto-injected `content-type: application/json`, this payload is rejected. The batch fails instead of returning an answer (and even if the bytes reached the service, Azure OpenAI would reject the malformed body with an HTTP 400, so `Answer` could never contain the reply). The extraction path in option a is actually correct — the payload construction is the fatal flaw.

### Why option b is wrong

Option b fixes the escaping problem correctly: `STRING_ESCAPE(@context, 'json')` and `STRING_ESCAPE(@bookingJson, 'json')` escape the embedded quotation marks, so the payload is valid JSON and the HTTP call succeeds with return code `0`.

The flaw is the extraction path:

```sql
JSON_VALUE(@response, '$.choices[0].message.content')
```

There is no top-level `choices` property in `@response`. The procedure wraps the endpoint's payload under `result`; the top-level properties of `@response` are `response` and `result`. Because `JSON_VALUE` uses **lax** path mode by default, a path that does not exist returns `NULL` rather than an error.

The batch therefore runs without any error and silently returns `Answer = NULL` — the most treacherous failure mode of the four, and explicitly disallowed by requirement 3.

### Why option d is wrong

Option d is byte-for-byte identical to option c except for one function:

```sql
JSON_QUERY(@response, '$.result.choices[0].message.content')
```

The path is correct, but the function is not. `JSON_QUERY` extracts **objects or arrays**; `JSON_VALUE` extracts **scalar values**. The `content` property is a scalar JSON string. In the default lax path mode, `JSON_QUERY` returns `NULL` when the path points to a scalar (in strict mode it would raise an error instead).

So option d also runs cleanly and returns `Answer = NULL`.

`JSON_QUERY` would be the right tool one level up — for example, `JSON_QUERY(@response, '$.result.choices[0].message')` returns the whole `{"role":"assistant","content":"..."}` object, and the documentation's own Azure OpenAI example uses `JSON_QUERY(@response, '$.result.data[0].embedding')` because an embedding is an **array**.

## DP-800 Exam Rule to Remember

The T-SQL glue of RAG with `sp_invoke_external_rest_endpoint` has three checkpoints:

```text
1. BUILD   → @payload must be valid JSON.
             Use JSON_OBJECT / JSON_ARRAY (auto-escape)
             or STRING_ESCAPE(<text>, 'json') in concatenation.
             Never splice raw text between quotation marks.

2. CALL    → @url (required), @method (default POST),
             @credential (DATABASE SCOPED CREDENTIAL),
             @response nvarchar(max) OUTPUT.

3. EXTRACT → the HTTP body is wrapped under "result":
             { "response": {...HTTP metadata...}, "result": {...body...} }

             scalar text  → JSON_VALUE('$.result.choices[0].message.content')
             object/array → JSON_QUERY  (returns NULL for scalars in lax mode)
```

If the answer comes back `NULL` with no error, suspect the path first: either the `result` wrapper is missing from the path, or `JSON_QUERY` was used where the target is a scalar.
