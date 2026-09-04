# Instructor-Examiner guide — JSON_ARRAY 1

Companion to [JSON_ARRAY_1.md](JSON_ARRAY_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Read all four options before taking an answer. The options differ only in small details, so read each one slowly and name the element count, whether a null appears, and how the 3D flag is written. Accept the letter as the answer. If the learner wants to say the full string instead, accept it and map it to the letter.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Write advanced T-SQL code.
- Task bullet: Work with JSON data using the native JSON functions.
- What is tested: the default null handling of `JSON_ARRAY` (`ABSENT ON NULL`), the effect of an explicit `NULL ON NULL` on a nested constructor, how `bit` renders in JSON, and how an embedded double quote is escaped.

## 2. Scenario to read aloud

**Piece 1, the story.** "A cinema multiplex publishes its showtimes as compact JSON arrays. There is one array per showing, and the mobile app reads it by position. Element zero is the film title. Element one is the screen number. Element two is the rating. Element three is the runtime in minutes. Element four is the 3D flag. And element five is the list of subtitle languages, itself a nested array."

**Piece 2, the table.** "The database is CineSlate, on SQL Server 2025 at compatibility level one hundred seventy. There is a schema called Cine and one table, Cine dot Showings, with eight columns. ShowingId, an integer primary key. FilmTitle, text up to sixty characters, not null. ScreenNo, an integer, not null. Rating, text up to four characters, which allows null. RuntimeMin, an integer, not null. Is3D, a bit, not null. SubLang1 and SubLang2, each text up to five characters, both allowing null."

**Piece 3, the data.** "Three rows are inserted. Row 1: Solar Drift, screen 4, rating twelve A, one hundred twenty-eight minutes, 3D flag one, subtitles fr and de. Row 2: the title is Midnight, a space, then the digit eight wrapped in double quotes, so the title itself contains two double-quote characters. Screen 1, rating null, ninety-five minutes, 3D flag zero, subtitle one is es, subtitle two is null. Row 3: Paper Lanterns, screen 6, rating PG, one hundred one minutes, 3D flag zero, both subtitle columns null."

**Piece 4, the query.** "One query selects ShowingId and a column called FeedRow, ordered by ShowingId. FeedRow is an outer JSON underscore ARRAY with six arguments in this order: FilmTitle, ScreenNo, Rating, RuntimeMin, Is3D, and a nested JSON ARRAY. The nested JSON ARRAY has two arguments, SubLang1 and SubLang2, followed by the clause NULL ON NULL. The outer JSON ARRAY has no null clause at all."

**Piece 5, what is asked.** "You will be asked for the exact FeedRow string for ShowingId 2, the Midnight row. I will read four options. Listen for three things: how many elements the outer array has, whether the word null appears anywhere, and how the 3D flag is written."

**Piece 6, option a.** "Option a. An array of six elements: the escaped title Midnight backslash-quote eight backslash-quote, then the number one, then null, then the number ninety-five, then false, then a nested array of es and null."

**Piece 7, option b.** "Option b. An array of five elements: the escaped title, then the number one, then ninety-five, then false, then a nested array of es and null. There is no null for the rating; the rating position is simply missing."

**Piece 8, option c.** "Option c. An array of five elements: the escaped title, then one, then ninety-five, then false, then a nested array containing only es. The inner array has one element, no null."

**Piece 9, option d.** "Option d. An array of five elements: the escaped title, then one, then ninety-five, then the number zero, then a nested array of es and null. The 3D flag is written as the digit zero, not as false."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE CineSlate;
GO
ALTER DATABASE CineSlate SET COMPATIBILITY_LEVEL = 170;
GO
USE CineSlate;
GO
CREATE SCHEMA Cine;
GO
CREATE TABLE Cine.Showings (
    ShowingId  INT          PRIMARY KEY,
    FilmTitle  NVARCHAR(60) NOT NULL,
    ScreenNo   INT          NOT NULL,
    Rating     NVARCHAR(4)  NULL,
    RuntimeMin INT          NOT NULL,
    Is3D       BIT          NOT NULL,
    SubLang1   NVARCHAR(5)  NULL,
    SubLang2   NVARCHAR(5)  NULL
);
GO
INSERT INTO Cine.Showings (ShowingId, FilmTitle, ScreenNo, Rating, RuntimeMin, Is3D, SubLang1, SubLang2) VALUES
  (1, N'Solar Drift',    4, N'12A', 128, 1, N'fr', N'de'),
  (2, N'Midnight "8"',   1, NULL,    95, 0, N'es', NULL),
  (3, N'Paper Lanterns', 6, N'PG',  101, 0, NULL,  NULL);
GO
SELECT s.ShowingId,
       JSON_ARRAY(s.FilmTitle, s.ScreenNo, s.Rating, s.RuntimeMin, s.Is3D,
                  JSON_ARRAY(s.SubLang1, s.SubLang2 NULL ON NULL)) AS FeedRow
FROM Cine.Showings AS s
ORDER BY s.ShowingId;
```

## 4. The question (ask exactly this)

"What is the exact FeedRow string returned for ShowingId equals 2? Here are the four options."

- a. `["Midnight \"8\"",1,null,95,false,["es",null]]`
- b. `["Midnight \"8\"",1,95,false,["es",null]]`
- c. `["Midnight \"8\"",1,95,false,["es"]]`
- d. `["Midnight \"8\"",1,95,0,["es",null]]`

"Which option, a, b, c or d?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: b.** `["Midnight \"8\"",1,95,false,["es",null]]`

- The outer JSON_ARRAY has no null clause, so it uses the default ABSENT ON NULL. The NULL Rating is dropped and the outer array has five elements, not six.
- The inner JSON_ARRAY has NULL ON NULL, so the NULL SubLang2 is kept as a JSON null: `["es",null]`.
- Is3D is a bit, which renders as the JSON boolean `false`.
- The embedded double quotes in the title are escaped as `\"`.

Why each wrong option is wrong:

- a: keeps a null for Rating, six elements. That would need NULL ON NULL on the outer array. NULL ON NULL is the default of JSON_OBJECT, not of JSON_ARRAY.
- c: drops the null from the inner array. The inner array explicitly says NULL ON NULL, so the null must stay.
- d: renders Is3D as the number 0. A bit maps to a JSON boolean, false, never to 0 or 1.

Full result set for reference:

```json
["Solar Drift",4,"12A",128,true,["fr","de"]]
["Midnight \"8\"",1,95,false,["es",null]]
["Paper Lanterns",6,"PG",101,false,[null,null]]
```

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with the outer array. It has no null clause. Do you remember which null behaviour JSON ARRAY uses when nothing is written?"
2. "JSON ARRAY and JSON OBJECT have opposite defaults. One keeps nulls, the other drops them. Which one is the array?"
3. "Now the inner array. It says NULL ON NULL explicitly. Does the null subtitle stay or go? That rules out one option."
4. "Next, the 3D flag. The column is a bit. Under the FOR JSON conversion rules, does a bit become a number or a boolean? That rules out another option."
5. "Two options remain, and they differ in one thing only: whether the outer array has five elements or six. Apply the outer default to the null rating."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| a | Assumes JSON_ARRAY keeps nulls by default | "Is the outer array using a written null clause, or the default? And which constructor has NULL ON NULL as its default?" |
| c | Thinks ABSENT ON NULL on the outer array also empties the inner one | "The inner array has its own clause. Does an outer default reach inside a nested constructor?" |
| d | Expects bit to render as 0 or 1 | "How does the bit type map under the FOR JSON type rules: number, string, or boolean?" |
| "The inner array vanishes because everything in it could be null" | Thinks a nested constructor can be SQL NULL | "Does a JSON ARRAY call ever return SQL NULL, or always a string?" |
| "The quotes in the title break the JSON" | Does not know embedded quotes are escaped | "How does JSON write a double quote inside a string?" |
| "It is an error because the array is shorter than six" | Thinks the engine validates positions | "Is a five-element array invalid JSON? What would the engine complain about?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the core rule first:

- **JSON_ARRAY defaults to ABSENT ON NULL.** A SQL NULL argument disappears without a trace, so the array changes length. For ShowingId 2 the declared positions were title, screen, rating, runtime, 3D, subtitles. The actual output is title, screen, ninety-five, false, subtitles. The mobile app reading element three as runtime now gets false, and element two as rating gets ninety-five. Nothing fails; the JSON is valid, just one element shorter. That is why a positional feed over nullable columns is unsafe unless the outer constructor says NULL ON NULL, or the feed emits key-value objects instead.
- **JSON_OBJECT defaults to the opposite, NULL ON NULL.** The key survives with a null value. Memorize the pair: JSON OBJECT of key k with NULL gives k colon null; JSON ARRAY of NULL gives an empty array.
- **One null clause per constructor.** It is written once, after the last argument, and governs all arguments of that constructor and only that constructor. The inner NULL ON NULL is not attached to SubLang2 alone; it covers SubLang1 too. That is why row 3 gives a nested array of null, null.
- **A nested constructor is never SQL NULL.** Worst case it returns an empty array or an empty object as text. So the enclosing constructor can never absent it.

Then the two character-level rules:

- Embedded double quotes are escaped as backslash quote, so the title renders as Midnight backslash-quote eight backslash-quote.
- Type mapping follows FOR JSON rules: int renders unquoted, bit renders as the boolean true or false, never as 0 or 1 and never as a quoted string.

Equivalent forms worth mentioning: spelling out ABSENT ON NULL on the outer array changes nothing. Adding NULL ON NULL to the outer array would produce option a's string, which is the correct output for a different query.

Memory hook: "Array absents, object keeps. Bit becomes true or false."

## 9. Follow-up oral questions (optional)

1. "What is the FeedRow string for ShowingId 3, Paper Lanterns?" (Six elements: Paper Lanterns, 6, PG, 101, false, and a nested array of null, null.)
2. "How would you change the query so that the rating position never disappears?" (Add NULL ON NULL after the last argument of the outer JSON_ARRAY.)
3. "If the outer array used NULL ON NULL and the inner array used nothing, what would row 3's nested array be?" (An empty array, because the inner default is ABSENT ON NULL.)

## 10. References

- JSON_ARRAY (Transact-SQL), including the ABSENT ON NULL default: https://learn.microsoft.com/en-us/sql/t-sql/functions/json-array-transact-sql
- JSON_OBJECT (Transact-SQL), for the opposite default: https://learn.microsoft.com/en-us/sql/t-sql/functions/json-object-transact-sql
- FOR JSON type conversion rules, including bit to boolean: https://learn.microsoft.com/en-us/sql/relational-databases/json/how-for-json-converts-sql-server-data-types-to-json-data-types-sql-server
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
