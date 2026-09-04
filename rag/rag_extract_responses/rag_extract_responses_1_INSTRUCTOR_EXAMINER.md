# Instructor-Examiner guide — RAG extract responses 1

Companion to [rag_extract_responses_1.md](rag_extract_responses_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a five-statement prediction question about parsing a chat-completions response in T-SQL. The JSON document in pieces 2 and 3 is the whole puzzle; repeat any part of it on request, and offer to repeat the structure before each statement. Several statements return more than one column or more than one result set; take them column by column. Accept "NULL, no error" as one answer. For S5, require the three outputs separately: the lax value, the OPENJSON length, and the strict outcome with its error number.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked. For JSON paths say "dollar dot result dot choices, index zero, dot message dot content".

## 1. Exam skill covered

- Functional group: Implement AI capabilities in database solutions (25–30%).
- Skill: Design and implement retrieval-augmented generation (RAG).
- Task bullet: Extract and process language model responses in T-SQL.
- What is tested: navigating the `response` and `result` envelope of `sp_invoke_external_rest_endpoint`, choosing `JSON_VALUE`, `JSON_QUERY` or `OPENJSON` by the type of the target, parsing a structured-output answer twice, and the 4,000-character limit of `JSON_VALUE` with its lax NULL versus strict error 13625.

## 2. Scenario to read aloud

**Piece 1, the story.** "A consumer-electronics maker runs a warranty-support assistant on an Azure SQL Database named RepairPilot. The RAG pipeline calls Azure OpenAI chat completions through sys dot sp underscore invoke underscore external underscore rest underscore endpoint with response underscore format set to a JSON schema, so the model's answer is itself a JSON document. The procedure returned a value into at response, and the same text was stored in a variable to verify the queries."

**Piece 2, the response envelope.** "At response is a JSON document with two top-level keys. The first is response. Inside it, a status object with an http object holding code 200 and description OK, and a headers object with Content-Type application slash json and x-ms-region West Europe. The second top-level key is result, which holds the body the API returned."

**Piece 3, inside result.** "Inside result: an id, chatcmpl dash 9x1RepairPilot. An object key with value chat dot completion. A created timestamp. A model key, gpt-4o-2024-08-06. A choices array with one element. That element has index 0, finish underscore reason stop, a message object, and logprobs null. The message object has role assistant, a content key, and a refusal key whose value is JSON null. The content value is a string. Inside that string, with the quotes escaped by backslashes, is a JSON object: eligible true, reason, Purchased 14 months ago; the 24-month warranty is still active, and claim underscore steps, an array of two strings: Register the serial number, and Ship the unit to the service centre. After choices there is a usage object with prompt underscore tokens 812, completion underscore tokens 57 and total underscore tokens 869."

**Piece 4, S1.** "Five statements run in order in one batch. S1 selects five JSON underscore VALUE calls on at response: http underscore code from dollar dot response dot status dot http dot code; finish underscore reason from dollar dot result dot choices zero dot finish underscore reason; prompt underscore tokens and completion underscore tokens from dollar dot result dot usage; and refusal from dollar dot result dot choices zero dot message dot refusal."

**Piece 5, S2 and S3.** "S2 selects two JSON underscore QUERY calls: content underscore query from dollar dot result dot choices zero dot message dot content, and usage underscore obj from dollar dot result dot usage. S3 selects two JSON underscore VALUE calls: eligible underscore direct from dollar dot result dot choices zero dot message dot content dot eligible, stepping into content as if it were an object; and no underscore envelope from dollar dot choices zero dot message dot content, with no result in the path."

**Piece 6, S4.** "S4 declares at content as JSON underscore VALUE of at response at dollar dot result dot choices zero dot message dot content. Then three selects. First, ISJSON of at content, aliased content underscore isjson. Second, OPENJSON of at content WITH three columns: eligible as BIT from dollar dot eligible, reason as NVARCHAR two hundred from dollar dot reason, and claim underscore steps as NVARCHAR MAX from dollar dot claim underscore steps AS JSON. Third, key and value from OPENJSON of at content with the path dollar dot claim underscore steps and no WITH clause."

**Piece 7, S5.** "S5 builds a second response. At long is 4,100 copies of the letter x, as NVARCHAR MAX. At resp2 is built with JSON underscore OBJECT and JSON underscore ARRAY so that dollar dot result dot choices zero dot message dot content is that 4,100-character string, and finish underscore reason is length. Then three selects. First, JSON underscore VALUE of at resp2 at the content path, aliased jv underscore lax. Second, LEN of a column answer from OPENJSON of at resp2 at dollar dot result dot choices zero dot message WITH answer as NVARCHAR MAX from dollar dot content, aliased openjson underscore len. Third, JSON underscore VALUE of at resp2 with the same content path but prefixed with the word strict, aliased jv underscore strict."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
DECLARE @response NVARCHAR(MAX) = N'{
  "response": {
    "status": { "http": { "code": 200, "description": "OK" } },
    "headers": { "Content-Type": "application/json", "x-ms-region": "West Europe" }
  },
  "result": {
    "id": "chatcmpl-9x1RepairPilot",
    "object": "chat.completion",
    "created": 1756900000,
    "model": "gpt-4o-2024-08-06",
    "choices": [
      {
        "index": 0,
        "finish_reason": "stop",
        "message": {
          "role": "assistant",
          "content": "{\"eligible\":true,\"reason\":\"Purchased 14 months ago; the 24-month warranty is still active.\",\"claim_steps\":[\"Register the serial number\",\"Ship the unit to the service centre\"]}",
          "refusal": null
        },
        "logprobs": null
      }
    ],
    "usage": { "prompt_tokens": 812, "completion_tokens": 57, "total_tokens": 869 }
  }
}';

-- S1
SELECT JSON_VALUE(@response, '$.response.status.http.code')          AS http_code,
       JSON_VALUE(@response, '$.result.choices[0].finish_reason')     AS finish_reason,
       JSON_VALUE(@response, '$.result.usage.prompt_tokens')          AS prompt_tokens,
       JSON_VALUE(@response, '$.result.usage.completion_tokens')      AS completion_tokens,
       JSON_VALUE(@response, '$.result.choices[0].message.refusal')   AS refusal;

-- S2
SELECT JSON_QUERY(@response, '$.result.choices[0].message.content')  AS content_query,
       JSON_QUERY(@response, '$.result.usage')                        AS usage_obj;

-- S3
SELECT JSON_VALUE(@response, '$.result.choices[0].message.content.eligible') AS eligible_direct,
       JSON_VALUE(@response, '$.choices[0].message.content')                 AS no_envelope;

-- S4
DECLARE @content NVARCHAR(MAX) = JSON_VALUE(@response, '$.result.choices[0].message.content');
SELECT ISJSON(@content) AS content_isjson;
SELECT eligible, reason, claim_steps
FROM OPENJSON(@content)
WITH (eligible BIT '$.eligible', reason NVARCHAR(200) '$.reason', claim_steps NVARCHAR(MAX) '$.claim_steps' AS JSON);
SELECT [key], value FROM OPENJSON(@content, '$.claim_steps');

-- S5: a second response whose content is 4,100 characters long
DECLARE @long NVARCHAR(MAX) = REPLICATE(CAST(N'x' AS NVARCHAR(MAX)), 4100);
DECLARE @resp2 NVARCHAR(MAX) = JSON_OBJECT('result': JSON_OBJECT('choices': JSON_ARRAY(
    JSON_OBJECT('message': JSON_OBJECT('content': @long), 'finish_reason': 'length'))));
SELECT JSON_VALUE(@resp2, '$.result.choices[0].message.content') AS jv_lax;
SELECT LEN(answer) AS openjson_len
FROM OPENJSON(@resp2, '$.result.choices[0].message') WITH (answer NVARCHAR(MAX) '$.content');
SELECT JSON_VALUE(@resp2, 'strict $.result.choices[0].message.content') AS jv_strict;
```

## 4. The question (ask exactly this)

"Predict the exact output of each statement, S1 to S5, run in order in the same batch. Where a statement raises an error, give the error number. Let's go one at a time. S1: the five columns."

Then: "S2: content underscore query and usage underscore obj." Then: "S3: eligible underscore direct and no underscore envelope." Then: "S4: the ISJSON result, the typed row, and the key value rows." Then: "S5: jv underscore lax, openjson underscore len, and jv underscore strict."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

| Stmt | Output |
|---|---|
| S1 | One row: http_code 200, finish_reason stop, prompt_tokens 812, completion_tokens 57, refusal NULL. All five columns are nvarchar |
| S2 | content_query NULL; usage_obj is the original object text with its whitespace preserved: prompt_tokens 812, completion_tokens 57, total_tokens 869 |
| S3 | eligible_direct NULL; no_envelope NULL. No error |
| S4 | content_isjson 1. Then one row: eligible 1, reason "Purchased 14 months ago; the 24-month warranty is still active.", claim_steps the array text ["Register the serial number","Ship the unit to the service centre"]. Then two rows: key 0, Register the serial number; key 1, Ship the unit to the service centre |
| S5 | jv_lax NULL; openjson_len 4100; the strict query fails with error 13625, String value in the specified JSON path would be truncated |

## 6. Hint ladder (one hint per attempt, in order)

**S1**
1. "All five paths point at scalars. Do they all exist? Walk each one through the envelope: response for HTTP metadata, result for the body."
2. "What type does JSON underscore VALUE always return, whatever the JSON type of the value?"
3. "The refusal key exists, but its value is JSON null. What does JSON underscore VALUE return for JSON null?"

**S2**
1. "JSON underscore QUERY is for objects and arrays. Is content an object, or a string that happens to look like one? Look at the backslash-escaped quotes."
2. "In lax mode, what does JSON underscore QUERY return when the path points to a scalar?"
3. "Now usage: is that a real object? If so, does JSON underscore QUERY return it reformatted or exactly as written?"

**S3**
1. "The first path tries to step inside content with dot eligible. Can you step into a string?"
2. "The second path has no result in it. What are the only two top-level keys of at response?"
3. "Both paths are wrong. In lax mode, is a wrong path an error or a NULL?"

**S4**
1. "JSON underscore VALUE returns the string and unescapes it. What does at content look like now? Is it valid JSON?"
2. "OPENJSON WITH maps to typed columns. What does eligible true become as a BIT? And what does the AS JSON option do for claim underscore steps?"
3. "OPENJSON without a WITH clause on an array: what are the key and value columns, and is the key zero-based?"

**S5**
1. "What is the return type of JSON underscore VALUE, and how many characters can it hold?"
2. "The content is 4,100 characters. In lax mode, does JSON underscore VALUE truncate, return NULL, or fail?"
3. "OPENJSON WITH answer NVARCHAR MAX has no such limit. How many characters does it return?"
4. "Strict mode does not hide problems. What happens instead of a NULL?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "S1's http code is the integer 200" | Expects JSON_VALUE to keep the JSON type | "What data type does JSON underscore VALUE return for every scalar?" |
| "S1's refusal is the string null" | Confuses JSON null with a string | "The value is a JSON null literal, not quoted. How is that mapped to SQL?" |
| "S2's content underscore query returns the eligible object" | Treats the content string as an object | "Look at the quotes inside content. Are they backslash-escaped? What does that make content?" |
| "S2's usage underscore obj is NULL or an error" | Thinks JSON_QUERY only works on arrays | "Is usage an object? What is JSON underscore QUERY for?" |
| "S3 raises an error" | Assumes strict mode | "Is the path prefixed with strict? What does lax mode do for a missing path?" |
| "S3's eligible underscore direct returns true" | Thinks a JSON path can step into a string containing JSON | "Strings have no properties. What would you have to do first to read eligible?" |
| "S4's ISJSON is 0" | Thinks the escaped string is not valid JSON | "JSON underscore VALUE unescapes on the way out. What does at content contain after that?" |
| "S4's claim underscore steps is NULL" | Missed the AS JSON option | "Look at the WITH clause. What option follows the claim underscore steps path?" |
| "S5's jv underscore lax returns 4,000 x's, truncated" | Expects truncation | "Does JSON underscore VALUE ever truncate silently, or does it choose between NULL and an error?" |
| "S5's strict query returns NULL too" | Does not distinguish lax from strict | "Strict mode turns lax NULLs into what?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the three skills:

- **Navigate the envelope.** at response has two top-level keys: response, with HTTP metadata, status dot http dot code and headers; and result, with the API body. Chat-completions paths: the answer at dollar dot result dot choices zero dot message dot content; the stop reason at dollar dot result dot choices zero dot finish underscore reason, which is stop, length or content underscore filter; the token counts at dollar dot result dot usage; the HTTP status at dollar dot response dot status dot http dot code. A path without result is silently NULL in lax mode. That is S3's no underscore envelope.
- **Choose the function by the type of the target.** Scalars, text, number or boolean, come out of JSON underscore VALUE, always as NVARCHAR 4000; cast before arithmetic. JSON null maps to SQL NULL, indistinguishable from a missing path. That is S1. Objects and arrays come out of JSON underscore QUERY, which returns the original text with whitespace preserved, useful for storing the usage block for billing, and which returns NULL on a scalar in lax mode. That is S2: content is a string, type 1 in OPENJSON's default output, so JSON underscore QUERY gives NULL; usage is a real object. Many fields or long text: OPENJSON WITH, with NVARCHAR MAX columns and AS JSON for arrays and objects; without AS JSON an array target returns NULL.
- **Parse structured output twice.** With response underscore format equal to json underscore schema, content is a string containing JSON. JSON underscore VALUE unescapes it, so at content becomes a valid document and ISJSON returns 1. Then OPENJSON WITH maps it to typed columns: eligible BIT gives 1, reason gives the text, claim underscore steps AS JSON keeps the array. A second OPENJSON with the array path and no schema explodes it into key and value rows, key being the zero-based index as a string. Trying to step into the string with a path, content dot eligible, gives NULL because strings have no properties. That is S3's eligible underscore direct and S4.

Then the 4,000-character trap, S5:

- JSON underscore VALUE returns NVARCHAR 4000. When the scalar is longer, lax mode returns NULL silently, so a long model answer simply disappears, and strict mode raises error 13625, String value in the specified JSON path would be truncated. The fixes: OPENJSON WITH answer NVARCHAR MAX, which returned the full 4,100 characters, or in SQL Server 2025 JSON underscore VALUE of the response cast to the json type with RETURNING NVARCHAR MAX; the RETURNING clause is available only on json-typed input. Also notice finish underscore reason equal to length: the API is telling you the answer was cut off by max underscore tokens, so check it before trusting content.

Then the production pattern: read the HTTP status and finish underscore reason first, THROW if the code is not 200 or the reason is length, and only then parse content. And a single OPENJSON WITH over dollar dot result can replace five JSON underscore VALUE calls, sidestep the 4,000 limit, and give real INT columns for the token counts.

Memory hook: "Scalar, VALUE. Object, QUERY. Many or long, OPENJSON WITH. Structured output is a string: parse it twice. NULL with no error means wrong path, wrong function, or over 4,000 characters."

## 9. Follow-up oral questions (optional)

1. "How would you read eligible from at response in one expression, without the at content variable?" (JSON underscore VALUE of JSON underscore VALUE of at response at the content path, at dollar dot eligible; it returns true as text.)
2. "The answer column is NULL and nothing failed. Name three possible causes." (A path missing the result wrapper, JSON underscore QUERY used on a scalar or JSON underscore VALUE on an object, or a scalar longer than 4,000 characters in lax mode.)
3. "Which finish underscore reason value tells you the answer was cut off, and what do you do about it?" (length; increase max underscore tokens or shorten the context, and do not trust content until then.)

## 10. References

- sp_invoke_external_rest_endpoint, response envelope: https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-invoke-external-rest-endpoint-transact-sql
- JSON_VALUE, including the nvarchar(4000) limit, lax and strict modes and RETURNING: https://learn.microsoft.com/en-us/sql/t-sql/functions/json-value-transact-sql
- JSON_QUERY: https://learn.microsoft.com/en-us/sql/t-sql/functions/json-query-transact-sql
- OPENJSON with default schema and with an explicit schema, AS JSON: https://learn.microsoft.com/en-us/sql/t-sql/functions/openjson-transact-sql
- ISJSON: https://learn.microsoft.com/en-us/sql/t-sql/functions/isjson-transact-sql
- JSON path expressions, lax and strict: https://learn.microsoft.com/en-us/sql/relational-databases/json/json-path-expressions-sql-server
- JSON data type (SQL Server 2025): https://learn.microsoft.com/en-us/sql/t-sql/data-types/json-data-type
- Azure OpenAI chat completions REST reference, finish_reason and structured outputs: https://learn.microsoft.com/en-us/azure/ai-services/openai/reference
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
