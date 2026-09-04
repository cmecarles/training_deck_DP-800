# Instructor-Examiner guide — JSON_VALUE 1

Companion to [JSON_VALUE_1.md](JSON_VALUE_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a predict-the-output question with two queries. Query 1 has nine JSON columns over two rows; take it column by column, asking for policy 1 and policy 2 together for each column. Query 2 is a single question: does it return rows, or fail, and with what message. Be strict about the difference between NULL, an empty array as text, and the words true and false as text.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Implement programmability objects and data types.
- Task bullet: Work with JSON data using the native JSON functions.
- What is tested: the scalar-versus-fragment split between `JSON_VALUE` and `JSON_QUERY`, lax versus strict path mode, zero-based array indexes, quoted keys in a path, and the 4,000-character limit of `JSON_VALUE`.

## 2. Scenario to read aloud

**Piece 1, the story.** "A home-insurance carrier keeps flexible policy metadata as JSON documents, stored beside the relational core of each policy. Two queries read that metadata with JSON underscore VALUE and JSON underscore QUERY."

**Piece 2, the table.** "The database is PolicyVault on SQL Server 2025, at compatibility level one hundred seventy. There is a schema Ins and one table, Ins dot Policies, with two columns. PolicyID, an integer primary key. And Meta, an NVARCHAR MAX column, not null, with a CHECK constraint that ISJSON of Meta equals one."

**Piece 3, policy 1.** "Two policies are inserted. Policy 1's document has these keys. A key named policy, space, no, with a space in the key name, holding the string HP dash 2044 dash X. A key holder, an object with a key name holding Marta Voss. A key riders, an array of two strings: flood, then quake. A key limits, an object with one key dwelling holding four hundred fifty thousand. A key active holding the boolean true. And a key prior underscore claims, an array of two objects: the first has yr 2021 and paid twelve thousand five hundred point seven five; the second has yr 2024 and paid null, a JSON null."

**Piece 4, policy 2.** "Policy 2's document has these keys. Policy no, with a space, holding HP dash 2189 dash B. Holder, an object with name Ade Kaplan. Riders, an empty array. Limits, an object with dwelling three hundred thousand and deductible two thousand five hundred. Active, the boolean false. And a key notes, holding a string of four thousand one hundred letter x characters. The document is built with REPLICATE and is valid JSON; the ISJSON check passed."

**Piece 5, what each policy lacks.** "Note the differences. Policy 1 has no notes key and no deductible under limits. Policy 2 has an empty riders array and no prior claims key at all."

**Piece 6, query 1.** "Query 1 selects PolicyID and nine columns, ordered by PolicyID. I will read them one at a time.

- PolicyNo: JSON VALUE at path dollar dot, then the key policy no in double quotes.
- SecondRider: JSON VALUE at dollar dot riders open bracket one close bracket.
- RidersV: JSON VALUE at dollar dot riders.
- RidersQ: JSON QUERY at dollar dot riders.
- HolderQ: JSON QUERY at dollar dot holder dot name.
- ActiveV: JSON VALUE at dollar dot active.
- Deductible: JSON VALUE at dollar dot limits dot deductible.
- Paid2: JSON VALUE at dollar dot prior claims open bracket one close bracket dot paid.
- Notes: JSON VALUE at dollar dot notes."

**Piece 7, query 2.** "Query 2 selects PolicyID and one column, Deductible, which is JSON VALUE at the path strict, space, dollar dot limits dot deductible. Ordered by PolicyID. The only difference from the Deductible column in query 1 is the word strict at the start of the path."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE PolicyVault;
GO
ALTER DATABASE PolicyVault SET COMPATIBILITY_LEVEL = 170;
GO
USE PolicyVault;
GO
CREATE SCHEMA Ins;
GO
CREATE TABLE Ins.Policies
(
    PolicyID INT           NOT NULL PRIMARY KEY,
    Meta     NVARCHAR(MAX) NOT NULL CHECK (ISJSON(Meta) = 1)
);
GO
INSERT INTO Ins.Policies (PolicyID, Meta) VALUES
 (1, N'{"policy no":"HP-2044-X","holder":{"name":"Marta Voss"},"riders":["flood","quake"],"limits":{"dwelling":450000},"active":true,"prior_claims":[{"yr":2021,"paid":12500.75},{"yr":2024,"paid":null}]}'),
 (2, CONVERT(NVARCHAR(MAX), N'{"policy no":"HP-2189-B","holder":{"name":"Ade Kaplan"},"riders":[],"limits":{"dwelling":300000,"deductible":2500},"active":false,"notes":"')
     + REPLICATE(CONVERT(NVARCHAR(MAX), N'x'), 4100) + N'"}');
GO
-- Query 1
SELECT
    p.PolicyID,
    JSON_VALUE(p.Meta, '$."policy no"')          AS PolicyNo,
    JSON_VALUE(p.Meta, '$.riders[1]')            AS SecondRider,
    JSON_VALUE(p.Meta, '$.riders')               AS RidersV,
    JSON_QUERY(p.Meta, '$.riders')               AS RidersQ,
    JSON_QUERY(p.Meta, '$.holder.name')          AS HolderQ,
    JSON_VALUE(p.Meta, '$.active')               AS ActiveV,
    JSON_VALUE(p.Meta, '$.limits.deductible')    AS Deductible,
    JSON_VALUE(p.Meta, '$.prior_claims[1].paid') AS Paid2,
    JSON_VALUE(p.Meta, '$.notes')                AS Notes
FROM Ins.Policies AS p
ORDER BY p.PolicyID;
-- Query 2
SELECT
    p.PolicyID,
    JSON_VALUE(p.Meta, 'strict $.limits.deductible') AS Deductible
FROM Ins.Policies AS p
ORDER BY p.PolicyID;
```

## 4. The question (ask exactly this)

"Predict the exact output of each query, every value and every NULL, or the exact failure. Let's start with query 1, one column at a time, giving the value for policy 1 and then policy 2. First column: PolicyNo."

Continue through SecondRider, RidersV, RidersQ, HolderQ, ActiveV, Deductible, Paid2, Notes. Then: "Now query 2. Does it return rows? If so, which values. If not, what happens?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Query 1, two rows** (verified on SQL Server 2025 RTM):

| PolicyID | PolicyNo | SecondRider | RidersV | RidersQ | HolderQ | ActiveV | Deductible | Paid2 | Notes |
|---|---|---|---|---|---|---|---|---|---|
| 1 | HP-2044-X | quake | NULL | ["flood","quake"] | NULL | true | NULL | NULL | NULL |
| 2 | HP-2189-B | NULL | NULL | [] | NULL | false | 2500 | NULL | NULL |

Reasons per column:

- PolicyNo: the key with a space is double-quoted in the path and resolves normally.
- SecondRider: indexes are zero-based, so index 1 is quake. Policy 2's array is empty, lax mode gives NULL.
- RidersV: the path lands on an array; JSON_VALUE returns scalars only, lax gives NULL for both rows.
- RidersQ: JSON_QUERY returns the fragment. Policy 2 gives the two-character text of an empty array, not NULL.
- HolderQ: the path lands on a string scalar; JSON_QUERY returns fragments only, lax gives NULL for both.
- ActiveV: booleans are scalars, returned as the text true and false, nvarchar not bit.
- Deductible: exists only for policy 2, 2500 as text. Missing for policy 1, lax gives NULL.
- Paid2: policy 1's second claim has a JSON null, which surfaces as SQL NULL. Policy 2 has no prior_claims, path not found, also NULL.
- Notes: policy 2's notes is 4,100 characters. JSON_VALUE returns nvarchar(4000); in lax mode an oversized scalar returns NULL rather than truncating. Policy 1 lacks the key. NULL on both rows.

**Query 2, fails at run time, no rows:**

```text
Msg 13608, Level 16, State 2
Property cannot be found on the specified JSON path.
```

Strict mode on policy 1's missing deductible aborts the whole statement. Policy 2's 2500 is never returned.

## 6. Hint ladder (one hint per attempt, in order)

**PolicyNo**
1. "The key has a space in it. How is such a key written inside a path?"
2. "Quoted keys resolve like any other. What is the value in each document?"

**SecondRider**
1. "Are JSON array indexes zero-based or one-based?"
2. "Index 1 is the second element. Policy 1 has two riders. Policy 2 has none. What does lax mode do when the index does not exist?"

**RidersV and RidersQ**
1. "JSON VALUE and JSON QUERY split the world in two. Which one returns scalars and which one returns objects and arrays?"
2. "The path lands on an array. Which of the two functions can return it, and what does the other return in lax mode?"
3. "For policy 2 the array is empty but it exists. Does JSON QUERY return NULL or the empty array as text?"

**HolderQ**
1. "The path holder dot name lands on a string. Is a string a scalar or a fragment?"
2. "JSON QUERY was asked for a scalar. Lax mode. What comes back?"

**ActiveV**
1. "A JSON boolean is a scalar. What SQL type does JSON VALUE always return?"
2. "It returns nvarchar. So what text comes back for policy 1 and for policy 2?"

**Deductible**
1. "Which policy has a deductible under limits?"
2. "For the other policy the property is missing and the path is lax. What is returned?"

**Paid2**
1. "Policy 1's second claim has paid colon null. What does a JSON null become in SQL?"
2. "Policy 2 has no prior claims at all. Path not found in lax mode. Same result, different reason."

**Notes**
1. "Policy 2 has a notes key and it is a valid string. But how long is it?"
2. "What is the return type of JSON VALUE, and what is its length limit?"
3. "The value is longer than four thousand characters. In lax mode, does JSON VALUE truncate, or return NULL?"

**Query 2**
1. "What does the word strict change about a path that does not resolve?"
2. "Policy 1 has no deductible. In strict mode that is not a NULL. What is it?"
3. "One row fails. Does the statement still return the other row, or does it return nothing?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| SecondRider is flood | Thinks indexes are one-based | "Which element is index zero?" |
| RidersV returns the array text | Thinks JSON_VALUE returns anything at the path | "What kind of thing does JSON VALUE extract: scalars, or fragments?" |
| RidersQ is NULL for policy 2 | Treats an empty array as missing | "Does the riders key exist in policy 2? What is its value, as text?" |
| HolderQ returns the name | Thinks JSON_QUERY is a superset of JSON_VALUE | "JSON QUERY was pointed at a scalar. Can it return one?" |
| ActiveV is 1 and 0 | Expects a bit | "What data type does JSON VALUE return, always?" |
| Notes for policy 2 is 4,000 x's | Expects truncation | "Does JSON VALUE silently truncate an oversized scalar, or do something else in lax mode?" |
| Query 2 returns one row with 2500 | Thinks strict fails per row | "When one row raises an error in a SELECT, what happens to the result set?" |
| Query 2 returns NULL and 2500 | Forgets the strict prefix | "Read the path again. What is the first word?" |

## 8. Teaching notes (after the answer is complete or revealed)

Split the world in two and never cross the line. JSON VALUE returns scalars only: strings, numbers, true and false. JSON QUERY returns fragments only: objects and arrays. Asked for the wrong kind, each returns NULL in lax mode, the default, and raises an error in strict mode. RidersV and HolderQ are the two mirror-image traps.

Then stack the JSON VALUE specifics:

- Return type is nvarchar 4000. A scalar over four thousand characters gives NULL in lax mode and error 13625, "String value in the specified JSON path would be truncated", in strict mode. To read the full value, use OPENJSON WITH a column of NVARCHAR MAX, or JSON VALUE with RETURNING nvarchar max over a json-typed input.
- Booleans and numbers come back as text, true and 2500, unless RETURNING casts them. RETURNING is a SQL Server 2025 feature that needs the native json type as input; on this build it is rejected over a plain nvarchar input.
- Array indexes are zero-based. Quote unusual keys, as in dollar dot quote policy no quote.
- Lax mode makes four different situations collapse into the same NULL: a missing path, a wrong-kind path, a JSON null, and an oversized scalar. Paid2 shows two of them side by side.
- Strict mode on a mixed document set fails the entire statement, error 13608, not just the offending row. Strict belongs in data-quality checks, not in general reports.

Equivalent alternatives: the results are identical if Meta is the native json type instead of NVARCHAR MAX with ISJSON. RidersQ, RidersV and HolderQ behave like OPENJSON WITH columns with and without AS JSON.

Memory hook: "Value for scalars, query for fragments. Lax says NULL, strict says error. Four thousand is the wall."

## 9. Follow-up oral questions (optional)

1. "What would JSON VALUE at dollar dot holder dot name return for policy 1?" (Marta Voss, as nvarchar.)
2. "What would JSON VALUE at strict dollar dot notes raise for policy 2?" (Error 13625, string value in the specified JSON path would be truncated.)
3. "How can you read the full 4,100-character notes value?" (OPENJSON with a WITH clause declaring Notes as NVARCHAR MAX at dollar dot notes, or JSON VALUE with RETURNING nvarchar max over a json-typed column.)

## 10. References

- JSON_VALUE (Transact-SQL): https://learn.microsoft.com/en-us/sql/t-sql/functions/json-value-transact-sql
- JSON_QUERY (Transact-SQL): https://learn.microsoft.com/en-us/sql/t-sql/functions/json-query-transact-sql
- JSON path expressions, lax and strict modes: https://learn.microsoft.com/en-us/sql/relational-databases/json/json-path-expressions-sql-server
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
