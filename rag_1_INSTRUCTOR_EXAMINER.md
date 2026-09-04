# Instructor-Examiner guide — Retrieval-Augmented Generation 1

Companion to [rag_1.md](rag_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a, b, c and d. Each option is a T-SQL batch of about twenty-five lines. All four share the same first two variables and the same procedure call; they differ only in how the payload is built and in the final SELECT. Read all four options before taking an answer. Pieces 6 to 9 describe each option's two differences precisely; say "I can read any line on request" and read from section 3 if asked. Accept the answer as a letter.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked. Say "dollar dot result dot choices, index zero, dot message dot content" for the JSON path.

## 1. Exam skill covered

- Functional group: Implement AI capabilities in database solutions (25–30%).
- Skill: Design and implement retrieval-augmented generation (RAG).
- Task bullet: Convert structured data to JSON for language model processing and call a model with sp_invoke_external_rest_endpoint.
- What is tested: building a valid JSON payload when the context contains quotation marks, the parameters of `sp_invoke_external_rest_endpoint`, the `result` envelope of its response, and `JSON_VALUE` for scalars versus `JSON_QUERY` for objects and arrays.

## 2. Scenario to read aloud

**Piece 1, the story.** "A travel agency runs its customer-support assistant on top of an Azure SQL Database named TripDesk. The assistant answers customer questions by grounding an Azure OpenAI chat-completions model on two kinds of data: the customer's booking row, converted to JSON, and a policy text chunk already selected by the retrieval step. The vector search itself is out of scope here."

**Piece 2, the tables.** "There is a schema Support. Support dot Bookings has BookingId, an integer primary key; CustomerName, Destination, both NVARCHAR one hundred; DepartureDate, a date; and TotalAmount, a DECIMAL ten comma two. Support dot PolicyChunks has ChunkId, an integer primary key, and ChunkText, an NVARCHAR MAX."

**Piece 3, the data.** "One booking: 1001, Elena Ruiz, Lisbon, departing the fourteenth of September twenty twenty-six, total fourteen hundred eighty. One policy chunk, ChunkId 7, whose text is: Under the, open double quote, FlexCancel, close double quote, plan, a booking can be cancelled up to 24 hours before departure for a 90 percent refund. Note that the chunk text contains double quotation marks around the word FlexCancel."

**Piece 4, the credential and the task.** "A database scoped credential already exists, named after the endpoint URL https colon slash slash tripdesk-ai dot openai dot azure dot com. It was created WITH IDENTITY equals HTTPEndpointHeaders and a secret that is a JSON object with an api-key. So the API key is injected as a request header automatically. The executing principal has EXECUTE ANY EXTERNAL ENDPOINT and REFERENCES on the credential. You must write one batch that, for booking 1001 and chunk 7, first builds a valid chat-completions payload even though the chunk contains double quotes; second, sends it to the support-gpt deployment with sys dot sp underscore invoke underscore external underscore rest underscore endpoint using the credential; and third, returns the assistant's reply text, a scalar JSON string, in a column named Answer. NULL or an error is not acceptable."

**Piece 5, what all four options share.** "All four batches start the same way. A variable at bookingJson is filled by selecting the five booking columns for BookingId 1001 with FOR JSON PATH, WITHOUT underscore ARRAY underscore WRAPPER, so it holds one JSON object full of double quotes. A variable at context holds the ChunkText of chunk 7. Then all four call sys dot sp underscore invoke underscore external underscore rest underscore endpoint with at url pointing to the deployment's chat slash completions endpoint with an api-version, at method POST, at credential set to the existing credential, at payload, and at response as an NVARCHAR MAX OUTPUT parameter. That call is identical in all four. The differences are in how at payload is built and in the last SELECT."

**Piece 6, option a.** "Option a builds the payload by plain string concatenation. It writes the literal text open brace, messages, open bracket, open brace, role user, content, and then a double quote, then Policy colon, then plus at context, plus the text Booking colon, plus at bookingJson, plus the text Question: Can I cancel my trip? and the closing quotes and braces. No escaping of any kind. The final SELECT is JSON underscore VALUE of at response with the path dollar dot result dot choices index zero dot message dot content, aliased Answer."

**Piece 7, option b.** "Option b is the same concatenation, but at context is wrapped in STRING underscore ESCAPE with the json option, and at bookingJson is also wrapped in STRING underscore ESCAPE json. The final SELECT is JSON underscore VALUE of at response with the path dollar dot choices index zero dot message dot content. Notice: no result in that path."

**Piece 8, option c.** "Option c builds the payload with JSON constructors. JSON underscore OBJECT with the key messages, whose value is JSON underscore ARRAY of two JSON underscore OBJECTs. The first has role system and content: Answer using only the provided policy and booking data. The second has role user and content: the concatenation Policy colon plus at context plus Booking colon plus at bookingJson plus Question: Can I cancel my trip? The concatenation is passed as a value to JSON underscore OBJECT, not spliced into a literal. The final SELECT is JSON underscore VALUE of at response with the path dollar dot result dot choices index zero dot message dot content, aliased Answer."

**Piece 9, option d.** "Option d is identical to option c, same constructors, same system and user messages, except for one thing: the final SELECT uses JSON underscore QUERY instead of JSON underscore VALUE, with the same path dollar dot result dot choices index zero dot message dot content."

## 3. Setup script (reference only; do not read verbatim unless asked)

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

CREATE DATABASE SCOPED CREDENTIAL [https://tripdesk-ai.openai.azure.com]
WITH IDENTITY = 'HTTPEndpointHeaders',
     SECRET = '{"api-key":"<key-value>"}';
GO
```

Shared prefix of all four options:

```sql
DECLARE @bookingJson nvarchar(max) =
    (SELECT BookingId, CustomerName, Destination, DepartureDate, TotalAmount
     FROM Support.Bookings
     WHERE BookingId = 1001
     FOR JSON PATH, WITHOUT_ARRAY_WRAPPER);

DECLARE @context nvarchar(max) =
    (SELECT ChunkText FROM Support.PolicyChunks WHERE ChunkId = 7);
```

Shared call of all four options:

```sql
DECLARE @ret int, @response nvarchar(max);

EXEC @ret = sys.sp_invoke_external_rest_endpoint
    @url = N'https://tripdesk-ai.openai.azure.com/openai/deployments/support-gpt/chat/completions?api-version=2024-10-21',
    @method = N'POST',
    @credential = [https://tripdesk-ai.openai.azure.com],
    @payload = @payload,
    @response = @response OUTPUT;
```

Option a, payload and extraction:

```sql
DECLARE @payload nvarchar(max) =
    N'{"messages":[{"role":"user","content":"Policy: ' + @context
    + N' Booking: ' + @bookingJson
    + N' Question: Can I cancel my trip?"}]}';
-- ... call ...
SELECT JSON_VALUE(@response, '$.result.choices[0].message.content') AS Answer;
```

Option b, payload and extraction:

```sql
DECLARE @payload nvarchar(max) =
    N'{"messages":[{"role":"user","content":"Policy: '
    + STRING_ESCAPE(@context, 'json')
    + N' Booking: '
    + STRING_ESCAPE(@bookingJson, 'json')
    + N' Question: Can I cancel my trip?"}]}';
-- ... call ...
SELECT JSON_VALUE(@response, '$.choices[0].message.content') AS Answer;
```

Option c, payload and extraction:

```sql
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
-- ... call ...
SELECT JSON_VALUE(@response, '$.result.choices[0].message.content') AS Answer;
```

Option d, payload identical to c, extraction:

```sql
SELECT JSON_QUERY(@response, '$.result.choices[0].message.content') AS Answer;
```

## 4. The question (ask exactly this)

"Which batch should you use? Choose one option.

- a. Plain concatenation of the context and booking JSON into a string literal, no escaping; extract with JSON underscore VALUE at dollar dot result dot choices zero dot message dot content.
- b. Concatenation with STRING underscore ESCAPE json on the context and on the booking JSON; extract with JSON underscore VALUE at dollar dot choices zero dot message dot content.
- c. JSON underscore OBJECT and JSON underscore ARRAY constructors with a system and a user message; extract with JSON underscore VALUE at dollar dot result dot choices zero dot message dot content.
- d. The same constructors as c; extract with JSON underscore QUERY at dollar dot result dot choices zero dot message dot content."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: c.**

Three fragile points are tested:

1. The payload must be valid JSON even when the context contains double quotes. JSON_OBJECT and JSON_ARRAY escape string values automatically, so the quotes around FlexCancel and all the quotes inside the booking JSON become backslash-quote. The result is a valid chat-completions request: a messages array of objects with role and content.
2. sp_invoke_external_rest_endpoint wraps the HTTP body in an envelope: HTTP metadata under `response`, the endpoint's JSON under `result`. The path must start with dollar dot result.
3. The reply text is a scalar JSON string, so JSON_VALUE, not JSON_QUERY.

The call itself is correct in all four: at url required, at method POST which is also the default, at credential names the DATABASE SCOPED CREDENTIAL whose HTTPEndpointHeaders identity injects the api-key header, no at headers needed because content-type application slash json is injected, at response NVARCHAR MAX OUTPUT.

Why the wrong options are wrong:

- a: naive concatenation. The first embedded double quote in the context terminates the JSON string early; the payload is not valid JSON. The documentation says payloads must be a valid JSON document, well-formed XML or text, and with the auto-injected content-type application slash json this payload is rejected; the batch fails. The extraction path in a is actually correct.
- b: the escaping is right and the call succeeds, but the path dollar dot choices skips the result wrapper. In lax mode JSON_VALUE returns NULL for a missing path. No error, Answer is NULL, the most treacherous failure.
- d: correct payload, correct path, wrong function. JSON_QUERY extracts objects or arrays and returns NULL in lax mode when the path points to a scalar. Answer is NULL.

## 6. Hint ladder (one hint per attempt, in order)

1. "Three things can go wrong here: building the payload, calling the endpoint, and extracting the answer. The call is identical in all four options, so focus on the other two."
2. "Start with the payload. The chunk text contains double quotes around FlexCancel, and the booking JSON is full of double quotes. If you splice that raw text between the quotes of a JSON string literal, what happens to the document?"
3. "One option does no escaping at all. That one produces an invalid payload. Cross it off."
4. "Now the extraction. The procedure does not put the HTTP body straight into at response. It wraps it. What are the two top-level keys of the wrapped document, and which one holds the endpoint's JSON?"
5. "One option's path has no result in it. What does JSON underscore VALUE return in lax mode for a path that does not exist? Cross that option off."
6. "Two options are left, identical except for one function name. The content property is a scalar string. Which function extracts scalars, and which extracts objects and arrays?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, the path is right and the concatenation looks fine" | Does not notice the embedded quotes break the JSON | "Say the chunk text aloud. What is the first character after the words Under the? What does that do inside a JSON string?" |
| "b, STRING underscore ESCAPE fixes the quotes" | Stops checking after the payload; misses the envelope | "The payload is fine. Now read the path in the last SELECT. What is the top-level key of at response?" |
| "d, JSON underscore QUERY is for extracting from JSON" | Confuses the two extraction functions | "Is the content property an object, an array, or a scalar string? Which function is for which?" |
| "c and d are the same, so neither can be right" | Missed the single-function difference | "Compare the last line of each. One word differs." |
| "None work, the call needs an at headers parameter for the API key" | Does not know the HTTPEndpointHeaders identity | "How was the credential created? What does IDENTITY equals HTTPEndpointHeaders do with the secret?" |
| "The system message in c is wrong, the API needs only a user message" | Thinks a system message is invalid | "Is a messages array with a system message and a user message a valid chat-completions request?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the three checkpoints of the T-SQL glue of RAG:

- **Build.** The payload must be valid JSON. FOR JSON PATH with WITHOUT_ARRAY_WRAPPER turns the booking row into one JSON object; that is the "convert structured data to JSON" step. To put text with special characters into a JSON string, use JSON_OBJECT and JSON_ARRAY, which escape values automatically, or STRING_ESCAPE with the json option inside concatenation. Never splice raw text between quotation marks. Option a fails here: the first embedded quote ends the string, the document is invalid, and the procedure rejects it because the auto-injected content-type is application slash json. Option c's payload is a messages array with a system message and a user message, exactly the shape the chat-completions API expects.
- **Call.** at url is required, NVARCHAR 4000. at method defaults to POST. at credential names a DATABASE SCOPED CREDENTIAL; with IDENTITY equals HTTPEndpointHeaders the secret's headers, here api-key, are injected automatically, so no at headers is needed. at response is NVARCHAR MAX OUTPUT. This part is identical and correct in all four options.
- **Extract.** at response is an envelope: response with status and headers, and result with the endpoint's body. So the reply is at dollar dot result dot choices index zero dot message dot content. That property is a scalar string, so JSON_VALUE. JSON_QUERY is for objects and arrays and returns NULL for a scalar in lax mode; it would be right one level up, for the whole message object, and the documentation's own Azure OpenAI example uses JSON_QUERY for an embedding because that is an array. Option b skips result and gets NULL; option d uses JSON_QUERY and gets NULL. Both run without error, which makes them the most treacherous failures.

One more detail: JSON_VALUE returns NVARCHAR 4000 by default, and in lax mode a longer value comes back as NULL. A support answer fits. For long completions, use OPENJSON, or cast the response to the json type and use JSON_VALUE with RETURNING NVARCHAR MAX, which is supported only on json-typed input.

Memory hook: "Build with constructors or STRING ESCAPE. Call with the credential. Extract through dollar dot result, VALUE for scalars, QUERY for objects. NULL with no error means the path or the function is wrong."

## 9. Follow-up oral questions (optional)

1. "How would you get the whole message object, role and content together, out of at response?" (JSON_QUERY of at response at dollar dot result dot choices index zero dot message.)
2. "How would you check the HTTP status code of the call from at response?" (JSON_VALUE at dollar dot response dot status dot http dot code.)
3. "The model returns a two-thousand-word answer and JSON underscore VALUE gives NULL. Why, and what do you do?" (JSON_VALUE returns NVARCHAR 4000 and NULL in lax mode when the value is longer; use OPENJSON, or cast to json and use RETURNING NVARCHAR MAX.)

## 10. References

- sp_invoke_external_rest_endpoint, parameters, response envelope and Azure OpenAI example: https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-invoke-external-rest-endpoint-transact-sql
- CREATE DATABASE SCOPED CREDENTIAL, HTTPEndpointHeaders identity: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-database-scoped-credential-transact-sql
- JSON_OBJECT: https://learn.microsoft.com/en-us/sql/t-sql/functions/json-object-transact-sql
- JSON_ARRAY: https://learn.microsoft.com/en-us/sql/t-sql/functions/json-array-transact-sql
- STRING_ESCAPE: https://learn.microsoft.com/en-us/sql/t-sql/functions/string-escape-transact-sql
- FOR JSON PATH and WITHOUT_ARRAY_WRAPPER: https://learn.microsoft.com/en-us/sql/relational-databases/json/format-query-results-as-json-with-for-json-sql-server
- JSON_VALUE: https://learn.microsoft.com/en-us/sql/t-sql/functions/json-value-transact-sql
- JSON_QUERY: https://learn.microsoft.com/en-us/sql/t-sql/functions/json-query-transact-sql
- Azure OpenAI chat completions REST reference: https://learn.microsoft.com/en-us/azure/ai-services/openai/reference
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
