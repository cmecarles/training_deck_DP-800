# Instructor-Examiner guide — REGEXP_SPLIT_TO_TABLE 1

Companion to [REGEXP_SPLIT_TO_TABLE_1.md](REGEXP_SPLIT_TO_TABLE_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a prediction question with two queries and four explicit sub-questions. Take Query 1 photo by photo, then Query 2, then the four sub-questions. The learner must say "empty string" or "NULL" explicitly for the blank cells; do not accept "blank" without asking which of the two it is. Be ready to repeat the raw tag strings, spelling out every comma, semicolon and space.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked. For the pattern, say "backslash s star" for `\s*` and "open square bracket comma semicolon close square bracket" for `[,;]`.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Implement programmability objects and query data.
- Task bullet: Use built-in functions, including string splitting and the SQL Server 2025 regular expression table-valued functions.
- What is tested: the two output columns of `REGEXP_SPLIT_TO_TABLE`, the empty-string versus NULL edge cases of splitting, how `CROSS APPLY` treats a NULL input, and the differences from `STRING_SPLIT`.

## 2. Scenario to read aloud

**Piece 1, the story.** "LensFolio is the catalog database of a photography portfolio site on SQL Server 2025. For years, photographers typed keyword tags into a single free-text column, and the delimiters are a mess. Some used commas, some semicolons, some left stray spaces around the delimiter, some left double delimiters or trailing delimiters, and one photo has no tags at all. The data team wants one row per keyword using the new table-valued function REGEXP underscore SPLIT underscore TO underscore TABLE, and to compare it with the older STRING underscore SPLIT."

**Piece 2, the database and the table.** "The script creates a database LensFolio, sets compatibility level 170, and creates a schema Photo. One table, Photo dot Portfolio, with three columns. PhotoID, an integer, the primary key. Title, an nvarchar of sixty, not null. And Tags, an nvarchar of two hundred, which allows NULL."

**Piece 3, the data.** "Four rows. I will spell the tags carefully.

- Photo 1, Harbor Dusk. Tags: longexposure, comma, harbor, comma, comma, dusk. Note the two commas in a row between harbor and dusk. No spaces anywhere.
- Photo 2, Alpine Ridge. Tags: alpine, semicolon, space, ridge, space, semicolon, sunrise, semicolon. So there is a space after the first semicolon and a space before the second semicolon, and the string ends with a semicolon.
- Photo 3, Street Solo. Tags: the single word monochrome. No delimiters at all.
- Photo 4, Untagged. Tags is NULL."

**Piece 4, Query 1.** "Query 1 selects p dot PhotoID, s dot ordinal, and s dot value aliased Keyword, from Photo dot Portfolio aliased p, CROSS APPLY REGEXP_SPLIT_TO_TABLE of p dot Tags with the pattern backslash s star, open square bracket comma semicolon close square bracket, backslash s star. In words: any whitespace, then one comma or one semicolon, then any whitespace. The result is aliased s. It orders by PhotoID and then ordinal."

**Piece 5, Query 2.** "Query 2 selects value and ordinal from STRING_SPLIT of the literal string alpine semicolon space ridge space semicolon sunrise semicolon, which is photo 2's tags, with the separator semicolon, and a third argument of 1. It orders by ordinal."

**Piece 6, what is asked.** "Predict the exact result of each query: every row, every value, every ordinal, in order. And answer four explicit points: how many rows each query returns; which cells are the empty string and which are NULL; whether any PhotoID is missing from Query 1 and why; and whether the keyword ridge comes back identically from both queries."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE LensFolio;
GO
ALTER DATABASE LensFolio SET COMPATIBILITY_LEVEL = 170;
GO
USE LensFolio;
GO
CREATE SCHEMA Photo;
GO
CREATE TABLE Photo.Portfolio
(
    PhotoID  INT           NOT NULL PRIMARY KEY,
    Title    NVARCHAR(60)  NOT NULL,
    Tags     NVARCHAR(200) NULL
);
GO
INSERT INTO Photo.Portfolio (PhotoID, Title, Tags) VALUES
  (1, N'Harbor Dusk',  N'longexposure,harbor,,dusk'),
  (2, N'Alpine Ridge', N'alpine; ridge ;sunrise;'),
  (3, N'Street Solo',  N'monochrome'),
  (4, N'Untagged',     NULL);
GO
-- Query 1
SELECT p.PhotoID,
       s.ordinal,
       s.value AS Keyword
FROM Photo.Portfolio AS p
CROSS APPLY REGEXP_SPLIT_TO_TABLE(p.Tags, N'\s*[,;]\s*') AS s
ORDER BY p.PhotoID, s.ordinal;
-- Query 2
SELECT value, ordinal
FROM STRING_SPLIT(N'alpine; ridge ;sunrise;', N';', 1)
ORDER BY ordinal;
```

## 4. The question (ask exactly this)

"Predict the exact result set of each query. One part at a time.

Part 1: Query 1, photo 1. How many rows, and what is each ordinal and Keyword?

Part 2: Query 1, photo 2. Same thing.

Part 3: Query 1, photo 3. Same thing.

Part 4: Query 1, photo 4. What comes back?

Part 5: So how many rows does Query 1 return in total, and is any PhotoID missing? Why?

Part 6: Query 2. How many rows, and what is each value and ordinal? Say exactly which cells are the empty string and which are NULL.

Part 7: Does the keyword ridge come back identically from both queries?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Query 1 returns exactly 9 rows.**

| PhotoID | ordinal | Keyword |
|---|---|---|
| 1 | 1 | longexposure |
| 1 | 2 | harbor |
| 1 | 3 | empty string |
| 1 | 4 | dusk |
| 2 | 1 | alpine |
| 2 | 2 | ridge |
| 2 | 3 | sunrise |
| 2 | 4 | empty string |
| 3 | 1 | monochrome |

- The two empty cells are empty strings, not NULLs.
- Photo 4 is missing entirely. Tags is NULL, the function returns an empty table, zero rows, and CROSS APPLY eliminates the outer row.
- Photo 2's keywords come back without surrounding spaces because `\s*` on both sides of `[,;]` makes the whitespace part of the consumed delimiter.

**Query 2 returns exactly 4 rows.**

| value | ordinal |
|---|---|
| alpine | 1 |
| space ridge space, seven characters | 2 |
| sunrise | 3 |
| empty string | 4 |

STRING_SPLIT splits on the literal single character semicolon only, so the spaces around ridge survive. So no: ridge is not identical. Query 1 yields ridge, five characters; Query 2 yields space ridge space, seven characters.

Part answers: Part 1: 4 rows, ordinal 3 is the empty string. Part 2: 4 rows, ordinal 4 is the empty string, values trimmed. Part 3: 1 row, monochrome, ordinal 1. Part 4: zero rows. Part 5: 9 rows, PhotoID 4 missing because NULL input gives an empty table and CROSS APPLY drops the outer row. Part 6: 4 rows, ordinal 4 is the empty string, nothing is NULL. Part 7: No.

## 6. Hint ladder (one hint per attempt, in order)

**Part 1, photo 1**
1. "Count the delimiter matches in longexposure comma harbor comma comma dusk. Each match is a split point."
2. "The character class matches exactly one character. Can one match swallow two adjacent commas, or are they two separate split points?"
3. "Two split points next to each other. What lies between them: nothing at all, so what value is produced, an empty string or a NULL?"

**Part 2, photo 2**
1. "The pattern has backslash s star on both sides of the delimiter class. What happens to the spaces next to the semicolons?"
2. "The string ends with a semicolon. That is a delimiter match at the very end. What comes after it?"
3. "A trailing delimiter always yields a trailing element. Is that element NULL or the empty string?"

**Part 3, photo 3**
1. "The pattern never matches in monochrome. Does the function return zero rows, or the whole string as one row?"

**Part 4, photo 4**
1. "Tags is NULL. Does the function return one NULL row, or an empty table?"
2. "Now think about CROSS APPLY. It behaves like an inner join. What does an inner join do with an outer row that has no inner rows?"

**Part 5, total**
1. "Four plus four plus one plus zero."

**Part 6, Query 2**
1. "STRING_SPLIT takes a separator of one literal character. Can it absorb spaces?"
2. "The third argument, 1, turns on a second column. What is it called?"
3. "The string ends with a semicolon. Does STRING_SPLIT also produce a trailing empty element?"

**Part 7, ridge**
1. "Compare what each function consumed as the delimiter around ridge. Which one ate the spaces?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "Photo 1 gives three rows; the double comma collapses" | Thinks adjacent delimiters merge | "How many characters can one match of the delimiter class consume? So how many split points are there between harbor and dusk?" |
| "The empty cells are NULL" | Confuses empty string with NULL | "The function returned a substring of zero length. Is a zero-length substring NULL?" |
| "Photo 2 gives space ridge space" | Ignores `\s*` in the pattern | "Read the pattern again. What sits on either side of the comma-or-semicolon class, and is it part of the delimiter?" |
| "Photo 4 gives one row with NULL keyword" | Confuses CROSS APPLY with OUTER APPLY | "Which APPLY operator keeps outer rows when the inner table is empty? Is that the one in the query?" |
| "Photo 3 gives zero rows because nothing matched" | Thinks no match means no rows | "The docs say what happens when the pattern never matches. Does the input disappear, or come back whole?" |
| "Query 2 returns only one column" | Forgets that enable_ordinal 1 adds the ordinal column | "What does the third argument of STRING_SPLIT do?" |
| "Query 1 has ten rows" | Counts photo 4 | "Go back to what CROSS APPLY does with an empty inner table." |
| "The ordinal column is called position" | Wrong column name | "The two columns of REGEXP_SPLIT_TO_TABLE are lowercase. One is value. What is the other?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the output shape:

- `REGEXP_SPLIT_TO_TABLE(string, pattern [, flags])` always returns a two-column table, no opt-in flag: `value`, same type as the input, and `ordinal`, a bigint, 1-based. The names are lowercase. Its optional third argument is flags, `i`, `m`, `s`, `c`, default `c`, not an ordinal switch.

Then walk Query 1:

- **Photo 1.** Three delimiter matches produce four substrings. Between the adjacent commas there is nothing, so ordinal 3 is the empty string. Adjacent delimiters are not collapsed; each match of the character class is a separate split point because the class matches exactly one character.
- **Photo 2.** The first delimiter match is semicolon plus the following space; the second is space plus semicolon. So the values are trimmed to alpine, ridge, sunrise. The trailing semicolon is the third match, at the very end, and what follows is the empty string at ordinal 4. Symmetrically, a leading delimiter would yield a leading empty element at ordinal 1.
- **Photo 3.** The pattern never matches. Per the documentation, if there is no match the function returns the string: one row, the whole input, ordinal 1. Never zero rows for a non-NULL input.
- **Photo 4.** A NULL input produces an empty table. CROSS APPLY behaves like an inner join between each outer row and its table-valued result, so photo 4 vanishes. OUTER APPLY would keep it as one row with ordinal NULL and Keyword NULL, engine-verified, giving 10 rows.

Then the STRING_SPLIT contrast:

- The separator is a literal single character, nvarchar(1). It cannot be a class, cannot absorb whitespace, cannot do "comma or semicolon" in one call. That is why ordinal 2 is space ridge space.
- The ordinal column is opt-in. Without the third argument STRING_SPLIT returns a single column, value. Passing enable_ordinal 1, SQL Server 2022 and later, adds the bigint ordinal. It must be a constant.
- Order is not guaranteed from STRING_SPLIT without ORDER BY.
- Behaviors that agree: both return empty-string rows for consecutive and trailing delimiters, and both return an empty table for NULL input.
- Compatibility level: STRING_SPLIT needs at least 130, REGEXP_SPLIT_TO_TABLE needs 170. At a lower level the regex function is an invalid object name, unless the database-scoped configuration `ALLOW_BUILTIN_TVF_IN_ALL_COMPAT_LEVELS` is enabled, a preview option shipped in SQL Server 2025 CU5 and Azure SQL, not on RTM.

Equivalent alternative, engine-verified: splitting on just `[,;]` and applying TRIM to the value gives the same 9 rows for this data set, because no keyword has interior edge whitespace that must survive.

Memory hook: "Adjacent delimiters give an empty string, a trailing delimiter gives an empty string, no match gives the whole string, NULL gives no rows. And CROSS APPLY drops what OUTER APPLY keeps."

## 9. Follow-up oral questions (optional)

1. "How would you keep photo 4 in the Query 1 output, and what would its row look like?" (Use OUTER APPLY. The row is PhotoID 4, ordinal NULL, Keyword NULL, for 10 rows total.)
2. "What is the minimum compatibility level for REGEXP_SPLIT_TO_TABLE, and for STRING_SPLIT?" (170 for REGEXP_SPLIT_TO_TABLE, 130 for STRING_SPLIT.)
3. "What is the third argument of REGEXP_SPLIT_TO_TABLE, and what is the third argument of STRING_SPLIT?" (Regex flags for REGEXP_SPLIT_TO_TABLE; enable_ordinal for STRING_SPLIT.)

## 10. References

- REGEXP_SPLIT_TO_TABLE: https://learn.microsoft.com/en-us/sql/t-sql/functions/regexp-split-to-table-transact-sql
- STRING_SPLIT: https://learn.microsoft.com/en-us/sql/t-sql/functions/string-split-transact-sql
- Regular expressions in SQL Server 2025, overview: https://learn.microsoft.com/en-us/sql/relational-databases/regular-expressions/regular-expressions-overview
- APPLY operator, CROSS APPLY and OUTER APPLY: https://learn.microsoft.com/en-us/sql/t-sql/queries/from-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
