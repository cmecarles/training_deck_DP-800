# Instructor-Examiner guide — JSON_CONTAINS 1

Companion to [JSON_CONTAINS_1.md](JSON_CONTAINS_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Each option is a result grid of four rows by seven columns. Do not read every cell of every option. Instead, describe the shared skeleton once, then describe how each option differs from the others, as written in section 2. Read all four options before taking an answer. A good approach is to let the learner work out each column for themselves first, then match to a letter.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Write advanced T-SQL code.
- Task bullet: Work with JSON data using the native JSON functions.
- What is tested: the three-valued result of `JSON_CONTAINS`, typed comparison between the SQL search value and the JSON value, the need for the `[*]` wildcard on arrays, NULL for a path that is not found, and the fourth argument that switches to LIKE semantics.

## 2. Scenario to read aloud

**Piece 1, the story.** "A tech job board stores each candidate's profile as a native json document. Recruiters match candidates against vacancies with the JSON underscore CONTAINS function, which is new in SQL Server 2025."

**Piece 2, the table.** "The database is TalentPool, at compatibility level one hundred seventy. There is a schema Jobs and one table, Jobs dot Candidates, with three columns. CandidateID, an identity integer primary key starting at one. FullName, text up to eighty characters. And Profile, of the native json type, not null."

**Piece 3, candidate 1.** "Four candidates are inserted, so they get IDs one to four. Candidate 1 is Ada Lindqvist. Her profile has a skills array with three strings: sql, python, azure. A key years with the number seven. A key remote with the boolean true. And a certs array with one object: id DP dash 800, active true."

**Piece 4, candidate 2.** "Candidate 2 is Bruno Ferrer. Skills array: java, sql. Years, the number twelve. Remote, false. Bruno has no certs key at all."

**Piece 5, candidate 3.** "Candidate 3 is Chen Wei. Skills is an empty array. Years, the number three. Remote, true. And certs is an empty array."

**Piece 6, candidate 4.** "Candidate 4 is Dara Okonkwo. Skills array: python, spark. Years is the JSON string quote seven quote, not the number seven. Remote, true. Certs has one object: id DP dash 800, active false."

**Piece 7, the query.** "The recruiter's query selects CandidateID and seven JSON CONTAINS columns, ordered by CandidateID. I will read them one at a time.

- HasSql: JSON CONTAINS of Profile, the string sql, at path dollar dot skills open bracket star close bracket.
- HasSqlNoWild: same, but the path is dollar dot skills, with no wildcard.
- Yrs7Int: search value is the integer seven, path dollar dot years.
- Yrs7Txt: search value is the string quote seven quote, path dollar dot years.
- RemoteBit: search value is one cast as bit, path dollar dot remote.
- RemoteInt: search value is the integer one, path dollar dot remote.
- CertLike: search value is the string DP dash percent, path dollar dot certs open bracket star close bracket dot id, and a fourth argument, one."

**Piece 8, the options, shared skeleton.** "All four options agree on these cells. HasSql is one, one, zero, zero for candidates 1 to 4. Yrs7Int is one for Ada. Yrs7Txt is zero for Bruno and Chen. RemoteBit is one, zero, one, one. CertLike is one for Ada and Dara. The options differ in the remaining cells."

**Piece 9, option a.** "Option a. HasSqlNoWild is zero everywhere. Yrs7Int is one for Ada and one for Dara. Yrs7Txt is one for Ada and one for Dara. RemoteInt equals RemoteBit: one, zero, one, one. CertLike is NULL for Bruno and Chen. In short, option a treats the number seven and the string seven as equal, and the integer one as equal to true."

**Piece 10, option b.** "Option b. HasSqlNoWild is zero everywhere. Yrs7Int is one for Ada and zero for Dara. Yrs7Txt is zero for Ada and one for Dara. RemoteInt is zero on every row. CertLike is NULL for Bruno and Chen."

**Piece 11, option c.** "Option c is identical to option b in every cell except CertLike. In option c, CertLike is zero for Bruno and zero for Chen, instead of NULL."

**Piece 12, option d.** "Option d is identical to option b in every cell except HasSqlNoWild. In option d, HasSqlNoWild is one for Ada and one for Bruno, so it equals HasSql."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE TalentPool;
GO
ALTER DATABASE TalentPool SET COMPATIBILITY_LEVEL = 170;
GO
USE TalentPool;
GO
CREATE SCHEMA Jobs;
GO
CREATE TABLE Jobs.Candidates
(
    CandidateID INT IDENTITY(1,1) PRIMARY KEY,
    FullName    NVARCHAR(80) NOT NULL,
    Profile     JSON         NOT NULL
);
GO
INSERT INTO Jobs.Candidates (FullName, Profile) VALUES
 (N'Ada Lindqvist', N'{"skills":["sql","python","azure"],"years":7,"remote":true,"certs":[{"id":"DP-800","active":true}]}'),
 (N'Bruno Ferrer',  N'{"skills":["java","sql"],"years":12,"remote":false}'),
 (N'Chen Wei',      N'{"skills":[],"years":3,"remote":true,"certs":[]}'),
 (N'Dara Okonkwo',  N'{"skills":["python","spark"],"years":"7","remote":true,"certs":[{"id":"DP-800","active":false}]}');
GO
SELECT
    c.CandidateID,
    JSON_CONTAINS(c.Profile, 'sql', '$.skills[*]')       AS HasSql,
    JSON_CONTAINS(c.Profile, 'sql', '$.skills')          AS HasSqlNoWild,
    JSON_CONTAINS(c.Profile, 7,   '$.years')             AS Yrs7Int,
    JSON_CONTAINS(c.Profile, '7', '$.years')             AS Yrs7Txt,
    JSON_CONTAINS(c.Profile, CAST(1 AS bit), '$.remote') AS RemoteBit,
    JSON_CONTAINS(c.Profile, 1, '$.remote')              AS RemoteInt,
    JSON_CONTAINS(c.Profile, 'DP-%', '$.certs[*].id', 1) AS CertLike
FROM Jobs.Candidates AS c
ORDER BY c.CandidateID;
```

## 4. The question (ask exactly this)

"Which result set does the query return? Option a, b, c or d?"

Option a:

| CandidateID | HasSql | HasSqlNoWild | Yrs7Int | Yrs7Txt | RemoteBit | RemoteInt | CertLike |
|---|---|---|---|---|---|---|---|
| 1 | 1 | 0 | 1 | 1 | 1 | 1 | 1 |
| 2 | 1 | 0 | 0 | 0 | 0 | 0 | NULL |
| 3 | 0 | 0 | 0 | 0 | 1 | 1 | NULL |
| 4 | 0 | 0 | 1 | 1 | 1 | 1 | 1 |

Option b:

| CandidateID | HasSql | HasSqlNoWild | Yrs7Int | Yrs7Txt | RemoteBit | RemoteInt | CertLike |
|---|---|---|---|---|---|---|---|
| 1 | 1 | 0 | 1 | 0 | 1 | 0 | 1 |
| 2 | 1 | 0 | 0 | 0 | 0 | 0 | NULL |
| 3 | 0 | 0 | 0 | 0 | 1 | 0 | NULL |
| 4 | 0 | 0 | 0 | 1 | 1 | 0 | 1 |

Option c:

| CandidateID | HasSql | HasSqlNoWild | Yrs7Int | Yrs7Txt | RemoteBit | RemoteInt | CertLike |
|---|---|---|---|---|---|---|---|
| 1 | 1 | 0 | 1 | 0 | 1 | 0 | 1 |
| 2 | 1 | 0 | 0 | 0 | 0 | 0 | 0 |
| 3 | 0 | 0 | 0 | 0 | 1 | 0 | 0 |
| 4 | 0 | 0 | 0 | 1 | 1 | 0 | 1 |

Option d:

| CandidateID | HasSql | HasSqlNoWild | Yrs7Int | Yrs7Txt | RemoteBit | RemoteInt | CertLike |
|---|---|---|---|---|---|---|---|
| 1 | 1 | 1 | 1 | 0 | 1 | 0 | 1 |
| 2 | 1 | 1 | 0 | 0 | 0 | 0 | NULL |
| 3 | 0 | 0 | 0 | 0 | 1 | 0 | NULL |
| 4 | 0 | 0 | 0 | 1 | 1 | 0 | 1 |

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: b.** Verified on SQL Server 2025 RTM.

| CandidateID | HasSql | HasSqlNoWild | Yrs7Int | Yrs7Txt | RemoteBit | RemoteInt | CertLike |
|---|---|---|---|---|---|---|---|
| 1 | 1 | 0 | 1 | 0 | 1 | 0 | 1 |
| 2 | 1 | 0 | 0 | 0 | 0 | 0 | NULL |
| 3 | 0 | 0 | 0 | 0 | 1 | 0 | NULL |
| 4 | 0 | 0 | 0 | 1 | 1 | 0 | 1 |

Why each wrong option is wrong:

- a: assumes value coercion. It treats the number 7 and the string "7" as equal, and the integer 1 as equal to true. JSON_CONTAINS compares by SQL type; int never equals a JSON string, and only bit compares with JSON true or false.
- c: gives 0 for Bruno's and Chen's CertLike. That confuses "not contained" with "path not found". Bruno has no certs key; Chen's certs is empty so the wildcard step finds nothing. Both are path not found, which returns NULL, not 0.
- d: gives 1 for HasSqlNoWild on Ada and Bruno. Without the wildcard the path points at the array itself, and a scalar search value is not searched inside the array. The result is 0 on every row.

## 6. Hint ladder (one hint per attempt, in order)

1. "JSON CONTAINS returns an int, but it has three possible results, not two. What are they, and when does the third one appear?"
2. "Look at CertLike for Bruno, who has no certs key, and for Chen, whose certs is an empty array. Does the path resolve to anything in those two documents? That rules out one option."
3. "Now HasSqlNoWild. The path is dollar dot skills with no wildcard, so it points at the array as a whole. Does the engine search inside the array for a scalar? That rules out another option."
4. "Two options remain. They differ in Yrs7Int, Yrs7Txt and RemoteInt. Does JSON CONTAINS compare by value, like JSON VALUE followed by equals, or by SQL type?"
5. "The integer seven versus the JSON string seven: are they comparable? The integer one versus JSON true: are they comparable? Which SQL type is the only one that matches JSON true?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| a | Expects JSON_VALUE-style string comparison | "With JSON VALUE everything becomes a string. Is JSON CONTAINS the same, or does the SQL type of the search value matter?" |
| c | Thinks a missing key is "not contained", so 0 | "When the path itself does not exist in the document, what does the function return: zero, or something else?" |
| d | Expects automatic unwrapping of arrays | "The documentation requires something on the path when it points to an array. What is it?" |
| "RemoteInt is 1 because 1 means true" | Assumes int coerces to boolean | "Which SQL type does JSON true compare against? Is int on that list?" |
| "Chen's CertLike is 0 because the array exists" | Thinks an empty array is a resolved path for the wildcard | "The path continues after the star with dot id. Is there any id under an empty array?" |
| "CertLike is 0 everywhere because DP dash percent is not a real id" | Ignores the fourth argument | "What does the fourth argument, one, change about how the string is compared?" |

## 8. Teaching notes (after the answer is complete or revealed)

The function is JSON CONTAINS of target, search value, optional path, optional search mode. It returns an int: 1 if the value is contained at the path, 0 if the path resolves but the value is not there, and NULL when the path is not found or the target is NULL.

Four rules explain the grid:

- **Rule 1, the SQL type of the search value drives the comparison.** Unlike JSON VALUE equals seven, where everything is compared as text, JSON CONTAINS pairs the SQL type with the JSON type. Int and decimal pair with JSON number. Nvarchar pairs with JSON string. Bit pairs with JSON true and false. Cross pairings are not comparable and give 0. So Yrs7Int is 1 for Ada's number 7 and 0 for Dara's string "7". Yrs7Txt is the reverse. RemoteBit matches true on rows 1, 3 and 4, and RemoteInt is 0 everywhere.
- **Rule 2, a path that lands on an array needs the star wildcard.** HasSql with skills star finds sql in Ada's and Bruno's arrays. HasSqlNoWild points at the array itself and is 0 on every row, even where sql is plainly present.
- **Rule 3, path not found returns NULL, not 0.** Bruno has no certs key. Chen's certs is empty, so the wildcard step has nothing to stand on and no id exists. Both give NULL. Zero is reserved for "the path exists, the value is absent". In practice, a WHERE JSON CONTAINS equals zero filter would silently drop Bruno and Chen.
- **Rule 4, the fourth argument switches equality to LIKE.** With search mode 1, DP dash percent is evaluated as a LIKE pattern, so DP dash 800 matches for Ada and Dara. With the default mode 0 those cells would be 0.

Engine remarks worth passing on: on the 2025 RTM build the target must be the native json type; an nvarchar target raises error 8116. A json-typed search value is also rejected with 8116, and a string that looks like an array is compared as one literal string. The function is version-gated, not compatibility-gated; it also runs at level 160.

Memory hook: "One means found, zero means looked and missed, NULL means could not even look. Match the SQL type, star the array, and mode one means LIKE."

## 9. Follow-up oral questions (optional)

1. "If CertLike dropped its fourth argument, what would Ada's cell become?" (0. Under equality, DP dash percent does not equal DP dash 800.)
2. "What would JSON CONTAINS of Profile, the string sql, at dollar dot skills star return for Chen?" (0. The skills key exists and the array is empty but the path resolves; sql is simply not there.)
3. "Which SQL type must the search value have to match a JSON true?" (bit.)

## 10. References

- JSON_CONTAINS (Transact-SQL): https://learn.microsoft.com/en-us/sql/t-sql/functions/json-contains-transact-sql
- JSON data type: https://learn.microsoft.com/en-us/sql/t-sql/data-types/json-data-type
- JSON path expressions: https://learn.microsoft.com/en-us/sql/relational-databases/json/json-path-expressions-sql-server
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
