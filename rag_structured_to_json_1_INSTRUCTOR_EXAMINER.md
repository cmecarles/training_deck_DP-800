# Instructor-Examiner guide — RAG structured data to JSON 1

Companion to [rag_structured_to_json_1.md](rag_structured_to_json_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Read all four options before taking an answer. The options differ only in one expression, so read the shared part once and then describe each option's difference precisely. If the learner asks for the exact text of the JSON that the rows produce, read piece 5 again slowly.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Implement AI capabilities in database solutions (25–30%).
- Skill: Design and implement retrieval-augmented generation (RAG).
- Task bullet: Convert structured data to JSON and send it to a model.
- What is tested: how to turn retrieved rows into JSON, how to embed that JSON as a string inside a chat-completions payload built with `JSON_OBJECT`, and the three escaping traps: `JSON_QUERY` nests instead of quoting, raw concatenation never escapes, and `STRING_ESCAPE` plus a constructor escapes twice.

## 2. Scenario to read aloud

**Piece 1, the story.** "A bank runs a loan-advisor assistant on an Azure SQL Database called LoanAdvisor. The retrieval step is done: two loan products have already been picked for a customer. The next step has to turn those two rows into JSON, put that JSON inside a chat-completions request, and send the request to Azure OpenAI."

**Piece 2, the table.** "There is one table, in a schema called Advice, named LoanProduct. Five columns. ProductId, an integer, the primary key. Name, text up to sixty characters. RateApr, a decimal with five digits and two decimals, the annual rate. MaxTermMo, a small integer, the maximum term in months. And Notes, text up to two hundred characters, which allows NULL."

**Piece 3, the data.** "Three rows. Product 1, GreenHome 5, rate three point two five, sixty months, and a note that says: Requires an, open quote, A, close quote, energy certificate. So the letter A is wrapped in double quotation marks inside the text. Product 2, FlexCash, rate seven point nine zero, thirty-six months, Notes is NULL. Product 3, AutoDrive, rate five point one zero, eighty-four months, note: New vehicles only, semicolon, ten percent down."

**Piece 4, the retrieval variable.** "A variable called at rows, NVARCHAR MAX, is filled by one query. It selects ProductId, Name, RateApr, MaxTermMo and Notes from LoanProduct where ProductId is in one and three, ordered by ProductId, FOR JSON PATH. So at rows holds a JSON array with two objects, product 1 and product 3."

**Piece 5, what at rows contains.** "Let me describe that JSON text. It is an array, open square bracket. First object: ProductId one, Name GreenHome 5, RateApr three point two five, MaxTermMo sixty, and Notes. Inside the Notes value, FOR JSON has already escaped the two quotation marks around the letter A with a backslash each. Second object: ProductId three, Name AutoDrive, RateApr five point one zero, MaxTermMo eighty-four, Notes New vehicles only, semicolon, ten percent down. Close square bracket. That text is valid JSON on its own."

**Piece 6, the call.** "A second variable, at payload, NVARCHAR MAX, is set to an expression that differs per option. Then the script calls sys dot sp underscore invoke underscore external underscore rest underscore endpoint. The URL is the chat-completions endpoint of an Azure OpenAI deployment called advisor-gpt. Method is POST. Headers is a small JSON object with one custom header, x-ms-client-request-id. Credential is a database scoped credential named after the endpoint URL; it already exists and was created with IDENTITY equals HTTPEndpointHeaders and a secret holding the api-key. Payload is at payload. Timeout is sixty seconds. Response goes into an output variable. Finally the script selects the return code and JSON underscore VALUE of the response at path dollar dot response dot status dot http dot code."

**Piece 7, the requirements.** "Two requirements for at payload. One: it must be a valid JSON document with a messages array holding a system message and a user message, plus temperature zero point two and max underscore tokens three hundred. Two: the user message's content must be a JSON string. Its text must be the words Products colon space, then the retrieved rows exactly as FOR JSON PATH produced them, quotation marks and all, then space Question colon Which product suits a five-year car purchase, question mark."

**Piece 8, option a.** "Option a builds the payload with JSON underscore OBJECT. Key messages is a JSON underscore ARRAY of two JSON underscore OBJECT calls. The first has role system and content: You are a loan advisor. Use only the products provided. The second has role user and content set to a plain string concatenation: the literal Products colon space, plus at rows, plus the literal space Question colon Which product suits a five-year car purchase. Then key temperature with the numeric literal zero point two, and key max underscore tokens with the numeric literal three hundred."

**Piece 9, option b.** "Option b is the same JSON underscore OBJECT structure, same system message, same temperature and max underscore tokens. The only difference is the user content: it is JSON underscore QUERY of at rows. No Products prefix and no Question suffix."

**Piece 10, option c.** "Option c does not use the constructors at all. It concatenates three N-prefixed string literals with at rows in the middle. The first literal opens the JSON document: messages array, the system message object, then the start of the user object with role user and content, and it ends with the text Products colon space right after an opening quotation mark. Then plus at rows. Then the third literal: space Question colon Which product suits a five-year car purchase, closing quotation mark, closing braces and brackets, temperature zero point two, max underscore tokens three hundred, closing brace."

**Piece 11, option d.** "Option d is again the JSON underscore OBJECT structure, identical to option a in every way except one. The user content is the literal Products colon space, plus STRING underscore ESCAPE of at rows with the second argument json, plus the Question suffix."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE LoanAdvisor;
GO
USE LoanAdvisor;
GO
CREATE SCHEMA Advice;
GO
CREATE TABLE Advice.LoanProduct
(
    ProductId  INT           NOT NULL PRIMARY KEY,
    Name       NVARCHAR(60)  NOT NULL,
    RateApr    DECIMAL(5,2)  NOT NULL,
    MaxTermMo  SMALLINT      NOT NULL,
    Notes      NVARCHAR(200) NULL
);
INSERT INTO Advice.LoanProduct VALUES
 (1, N'GreenHome 5', 3.25, 60, N'Requires an "A" energy certificate'),
 (2, N'FlexCash',    7.90, 36, NULL),
 (3, N'AutoDrive',   5.10, 84, N'New vehicles only; 10% down');
GO

DECLARE @rows NVARCHAR(MAX) =
  (SELECT ProductId, Name, RateApr, MaxTermMo, Notes
   FROM Advice.LoanProduct WHERE ProductId IN (1, 3) ORDER BY ProductId FOR JSON PATH);

DECLARE @payload NVARCHAR(MAX) = /* ---- differs per option ---- */ ;

DECLARE @ret INT, @response NVARCHAR(MAX);
EXEC @ret = sys.sp_invoke_external_rest_endpoint
    @url        = N'https://loanadvisor-ai.openai.azure.com/openai/deployments/advisor-gpt/chat/completions?api-version=2024-10-21',
    @method     = N'POST',
    @headers    = N'{"x-ms-client-request-id":"loan-1001"}',
    @credential = [https://loanadvisor-ai.openai.azure.com],
    @payload    = @payload,
    @timeout    = 60,
    @response   = @response OUTPUT;
SELECT @ret AS ReturnCode, JSON_VALUE(@response, '$.response.status.http.code') AS HttpStatus;
```

Contents of `@rows` as produced by the engine:

```json
[{"ProductId":1,"Name":"GreenHome 5","RateApr":3.25,"MaxTermMo":60,"Notes":"Requires an \"A\" energy certificate"},{"ProductId":3,"Name":"AutoDrive","RateApr":5.10,"MaxTermMo":84,"Notes":"New vehicles only; 10% down"}]
```

Option a:

```sql
JSON_OBJECT(
  'messages': JSON_ARRAY(
     JSON_OBJECT('role':'system', 'content':'You are a loan advisor. Use only the products provided.'),
     JSON_OBJECT('role':'user',   'content': N'Products: ' + @rows
                                            + N' Question: Which product suits a 5-year car purchase?')),
  'temperature': 0.2,
  'max_tokens': 300)
```

Option b:

```sql
JSON_OBJECT(
  'messages': JSON_ARRAY(
     JSON_OBJECT('role':'system', 'content':'You are a loan advisor. Use only the products provided.'),
     JSON_OBJECT('role':'user',   'content': JSON_QUERY(@rows))),
  'temperature': 0.2,
  'max_tokens': 300)
```

Option c:

```sql
N'{"messages":[{"role":"system","content":"You are a loan advisor. Use only the products provided."},'
+ N'{"role":"user","content":"Products: ' + @rows
+ N' Question: Which product suits a 5-year car purchase?"}],"temperature":0.2,"max_tokens":300}'
```

Option d:

```sql
JSON_OBJECT(
  'messages': JSON_ARRAY(
     JSON_OBJECT('role':'system', 'content':'You are a loan advisor. Use only the products provided.'),
     JSON_OBJECT('role':'user',   'content': N'Products: ' + STRING_ESCAPE(@rows, 'json')
                                            + N' Question: Which product suits a 5-year car purchase?')),
  'temperature': 0.2,
  'max_tokens': 300)
```

## 4. The question (ask exactly this)

"Which expression for at payload is correct? Option a: JSON underscore OBJECT with the user content as a plain concatenation of Products colon, at rows, and the Question text. Option b: JSON underscore OBJECT with the user content as JSON underscore QUERY of at rows. Option c: hand-written string concatenation, with at rows spliced between the quotation marks of the content value. Option d: JSON underscore OBJECT with the user content as Products colon, plus STRING underscore ESCAPE of at rows with json, plus the Question text. Which one is correct, a, b, c or d?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

`JSON_OBJECT` receives `'content'` as an ordinary nvarchar expression and serializes it as a JSON string, escaping every `"` and `\` exactly once. `ISJSON(@payload)` returns 1. Reading it back with `JSON_VALUE(@payload, '$.messages[1].content')` returns `Products: [{"ProductId":1,...,"Notes":"Requires an \"A\" energy certificate"},...] Question: ...`, byte for byte the FOR JSON text with the prefix and suffix around it. Numbers stay numbers because they were passed as numeric literals.

Why each wrong option is wrong:

- **b** — `JSON_QUERY(@rows)` tells `JSON_OBJECT` to embed the value unescaped, as nested JSON. The document is valid (`ISJSON` = 1) but `content` becomes a JSON array of product objects, not a string. `JSON_VALUE` on it returns NULL. The chat-completions API accepts `content` only as a string or as an array of typed content parts, so the service returns HTTP 400. Requirement 2 is violated. It also drops the Products prefix and the Question suffix.
- **c** — Raw concatenation splices `@rows` into the string with zero escaping. The first `"` inside `@rows` terminates the content string early. `ISJSON(@payload)` = 0. `sp_invoke_external_rest_endpoint` requires a valid JSON payload, so the batch fails before any model is reached.
- **d** — `STRING_ESCAPE(@rows, 'json')` escapes once, then `JSON_OBJECT` escapes again. The payload is valid and the call succeeds, but the model receives backslashes everywhere: the text starts with `[{\"ProductId\":1,...`. `JSON_VALUE` returns that backslash-laden text, so requirement 2 ("exactly as FOR JSON PATH produced them") fails. Double escaping.

## 6. Hint ladder (one hint per attempt, in order)

1. "Think about the difference between JSON that is structure, and JSON that is text the model should read. Requirement 2 says the content must be a JSON string. Which options make content a string, and which make it something else?"
2. "There is exactly one rule here: a JSON fragment that travels inside a JSON string must be escaped exactly once. Count the escapes in each option. Zero, one, or two."
3. "Look at option c. It writes the quotation marks by hand and drops at rows between them. At rows contains quotation marks of its own. What happens to the document when the first one appears?"
4. "Look at option b. JSON underscore QUERY says to the constructor: this is already JSON, do not quote it. So what type does content end up being? Is that a string?"
5. "You are down to a and d. Both use JSON underscore OBJECT, which escapes its string arguments once. One of them also escapes before handing the string over. Which one escapes twice?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "c, because that is exactly the JSON shape the API wants" | Forgets that the embedded text contains quotation marks that must be escaped | "The shape is right. Now think about what is inside at rows. Does anything in there break a hand-written string?" |
| "b, because JSON underscore QUERY is how you embed JSON safely" | Confuses nesting JSON with quoting JSON as text | "JSON underscore QUERY embeds without quoting. Read requirement 2 again. Must content be a string or a structure?" |
| "d, because you must escape before putting text into JSON" | Does not know that the constructors already escape string arguments | "You are right that escaping is needed. Who does it in option d? Count how many times it happens." |
| "a, but the quotation marks around the letter A will break it" | Thinks the constructor leaves quotation marks unescaped | "What does JSON underscore OBJECT do to a quotation mark inside a string argument?" |
| "a and d are the same" | Assumes STRING underscore ESCAPE is a no-op when the input is already valid JSON | "STRING underscore ESCAPE does not check whether the text is already escaped. It escapes every quotation mark and backslash it finds. What happens to a backslash that is already there?" |
| "None, because temperature zero point two will be quoted as a string" | Thinks JSON underscore OBJECT quotes numbers | "The value was passed as a numeric literal, not a string. What does the constructor do with a numeric argument?" |

## 8. Teaching notes (after the answer is complete or revealed)

Start with the one rule: **a JSON fragment that must travel inside a JSON string has to be escaped exactly once.** Zero escapes break the document. Two escapes hand the model backslashes.

Then map the four options to the rule:

- **Option a, escaped once.** `JSON_OBJECT` takes the content expression as an nvarchar and serializes it as a JSON string. It escapes every quotation mark and backslash. Because at rows already contains one level of escaping around the letter A, the payload shows that spot as backslash backslash backslash quote, three backslashes, which is correct: read back with `JSON_VALUE`, it becomes backslash quote again, exactly the FOR JSON text. Numbers stay numbers because temperature and max underscore tokens are numeric literals.
- **Option b, not a string at all.** `JSON_QUERY` marks the value as JSON to be nested unescaped. The document is valid, but content is now an array of objects. `JSON_VALUE` on it returns NULL. The chat-completions API allows content to be a string or an array of typed content parts such as type text, not a bare array of arbitrary objects, so it answers HTTP 400. Use `JSON_QUERY` only when you want structure, for example a tools or response underscore format object.
- **Option c, escaped zero times.** Hand-written concatenation splices at rows between two quotation marks with no escaping. The first quotation mark inside at rows closes the content string early. `ISJSON` returns 0 and the procedure rejects the payload before any network call.
- **Option d, escaped twice.** `STRING_ESCAPE` with json does one round, then `JSON_OBJECT` does another. The call runs and the model answers, but it reads text full of backslashes, so its grounding is wrong. `STRING_ESCAPE` belongs with manual concatenation, as a fix for option c. Never combine it with the constructors.

Then the procedure itself. `sp_invoke_external_rest_endpoint`: at url is required. At method defaults to POST. At headers is a flat JSON object of string values; content-type application json and the credential's api-key header are injected automatically. At credential names the database scoped credential, whose name must match the URL. At timeout is in seconds and defaults to thirty. At response is nvarchar max OUTPUT. The return code is zero on HTTP success and non-zero otherwise. The response always carries an envelope: the status is under dollar dot response dot status dot http dot code, and the body is under dollar dot result.

Accepted alternatives for building the rows: `JSON_ARRAYAGG` of `JSON_OBJECT` with ORDER BY instead of FOR JSON PATH, remembering that `JSON_OBJECT` defaults to NULL ON NULL, so a NULL Notes becomes notes null unless you write ABSENT ON NULL. Or `STRING_AGG` of `CONCAT` for a plain-text context when the model does not need JSON.

Memory hook: "Strings for the model, JSON underscore QUERY only for structure. Escape once, never zero, never twice."

## 9. Follow-up oral questions (optional)

1. "How would you fix option c without using the constructors?" (Wrap at rows in STRING underscore ESCAPE with json, so the hand-built string gets its one escape.)
2. "In option a, if product 2 with a NULL Notes had been retrieved, what would FOR JSON PATH do with Notes?" (Omit the key by default; FOR JSON PATH skips NULL columns unless INCLUDE underscore NULL underscore VALUES is specified.)
3. "After the call, where do you read the HTTP status, and what does the return code tell you?" (JSON underscore VALUE of the response at dollar dot response dot status dot http dot code; the return code is zero on HTTP success and non-zero otherwise.)

## 10. References

- JSON_OBJECT: https://learn.microsoft.com/en-us/sql/t-sql/functions/json-object-transact-sql
- JSON_ARRAY: https://learn.microsoft.com/en-us/sql/t-sql/functions/json-array-transact-sql
- JSON_QUERY: https://learn.microsoft.com/en-us/sql/t-sql/functions/json-query-transact-sql
- STRING_ESCAPE: https://learn.microsoft.com/en-us/sql/t-sql/functions/string-escape-transact-sql
- FOR JSON PATH: https://learn.microsoft.com/en-us/sql/relational-databases/json/format-query-results-as-json-with-for-json-sql-server
- JSON_ARRAYAGG: https://learn.microsoft.com/en-us/sql/t-sql/functions/json-arrayagg-transact-sql
- sp_invoke_external_rest_endpoint: https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-invoke-external-rest-endpoint-transact-sql
- Azure OpenAI chat completions REST reference: https://learn.microsoft.com/en-us/azure/ai-services/openai/reference
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
