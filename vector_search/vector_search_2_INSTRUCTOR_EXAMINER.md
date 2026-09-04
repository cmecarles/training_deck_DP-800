# Instructor-Examiner guide — Vector Search 2

Companion to [vector_search_2.md](vector_search_2.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

**Specific to this question.** This is an eleven-statement prediction question. Take the statements one at a time, S1 to S11. The learner may compute distances by hand; allow time and offer to repeat the vectors. Accept distances rounded to four decimals. The vectors were chosen so that every distance is exact or a simple square root.

## 1. Exam skill covered

- Functional group: Implement AI capabilities in database solutions (25–30%).
- Skill: Design and implement intelligent search.
- Task bullet: Implement vector search.
- What is tested: how a JSON array string becomes a `vector(n)`, the dimension-mismatch and invalid-JSON errors, the three `VECTOR_DISTANCE` metrics including the negative dot product, `VECTOR_NORM`, storage size, what a vector column can and cannot do (no equality, no ORDER BY), and the NULL trap in an exact TOP k search.

## 2. Scenario to read aloud

**Piece 1, the story.** "A herbal-remedies knowledge base runs on SQL Server 2025, in a database called HerbIndex. Each remedy has a tiny effect-profile embedding with four dimensions. The four axes are, in order: calming, digestive, immune, and stimulant. The numbers are small integers on purpose, so you can compute every distance in your head."

**Piece 2, the table.** "There is one schema, Herbal, and one table, Herbal dot Remedies. Three columns. RemedyId, an integer, the clustered primary key. Name, text up to sixty characters. And Effect, of type VECTOR open paren 4 close paren, which allows NULL."

**Piece 3, the data.** "Six rows. Remedy 1, Chamomile, vector two, zero, zero, zero. Remedy 2, Peppermint, vector zero, three, four, zero. Remedy 3, Elderberry, vector three, zero, four, zero. Remedy 4, Valerian, vector one, one, one, one. Remedy 5, Guarana, vector minus two, zero, zero, zero. And remedy 6, Placebo, whose Effect is NULL."

**Piece 4, the query vector.** "A user asks for something purely calming. The toy embedding of that request is q equals one, zero, zero, zero. Whenever a statement uses a variable at q, it is declared as VECTOR of 4 with that value, in the same batch."

**Piece 5, statements S1 to S3.** "S1 inserts remedy 7, Ginger, with the string open bracket one, two, three close bracket. Only three numbers. S2 inserts remedy 7, Ginger, with the string one comma two comma three comma four, with no square brackets at all. S3 selects, for remedy 2, DATALENGTH of Effect, and Effect cast to VARCHAR of 100."

**Piece 6, statements S4 and S5.** "S4 declares at q and returns, for every remedy ordered by RemedyId, three columns: VECTOR_DISTANCE with metric cosine between Effect and at q, the same with metric euclidean, and the same with metric dot. Each is cast to DECIMAL ten comma four. S5 returns, for every remedy, VECTOR_NORM of Effect with norm2, with norm1, and with norminf."

**Piece 7, statements S6 to S8.** "S6 declares at q and calls VECTOR_DISTANCE with the metric string manhattan, on remedy 1. S7 calls VECTOR_DISTANCE with metric cosine, Effect, and, as the third argument, the plain string literal open bracket one, zero, zero, zero close bracket, no variable, no cast, on remedy 1. S8 selects RemedyId where Effect equals CAST of the string two, zero, zero, zero as VECTOR of 4."

**Piece 8, statements S9 to S11.** "S9 declares at q and selects TOP 3 RemedyId, Name, and the cosine distance to at q cast to DECIMAL, from the whole table, ordered by that distance ascending, then RemedyId ascending. S10 is the same query with one addition: WHERE Effect IS NOT NULL. S11 declares at q and selects TOP 2 RemedyId, Name, and the dot distance to at q, WHERE Effect IS NOT NULL, ordered by distance ascending then RemedyId."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE HerbIndex;
GO
USE HerbIndex;
GO
CREATE SCHEMA Herbal;
GO
CREATE TABLE Herbal.Remedies
(
    RemedyId INT          NOT NULL CONSTRAINT PK_Remedies PRIMARY KEY CLUSTERED,
    Name     NVARCHAR(60) NOT NULL,
    Effect   VECTOR(4)    NULL
);
GO
INSERT INTO Herbal.Remedies (RemedyId, Name, Effect) VALUES
    (1, N'Chamomile',  '[2, 0, 0, 0]'),
    (2, N'Peppermint', '[0, 3, 4, 0]'),
    (3, N'Elderberry', '[3, 0, 4, 0]'),
    (4, N'Valerian',   '[1, 1, 1, 1]'),
    (5, N'Guarana',    '[-2, 0, 0, 0]'),
    (6, N'Placebo',    NULL);
GO
-- S1
INSERT INTO Herbal.Remedies (RemedyId, Name, Effect) VALUES (7, N'Ginger', '[1, 2, 3]');
-- S2
INSERT INTO Herbal.Remedies (RemedyId, Name, Effect) VALUES (7, N'Ginger', '1, 2, 3, 4');
-- S3
SELECT DATALENGTH(Effect) AS Bytes, CAST(Effect AS VARCHAR(100)) AS EffectText
FROM Herbal.Remedies WHERE RemedyId = 2;
-- S4
DECLARE @q VECTOR(4) = '[1, 0, 0, 0]';
SELECT RemedyId,
       CAST(VECTOR_DISTANCE('cosine',    Effect, @q) AS DECIMAL(10,4)) AS Cos,
       CAST(VECTOR_DISTANCE('euclidean', Effect, @q) AS DECIMAL(10,4)) AS Euc,
       CAST(VECTOR_DISTANCE('dot',       Effect, @q) AS DECIMAL(10,4)) AS Dot
FROM Herbal.Remedies ORDER BY RemedyId;
-- S5
SELECT RemedyId,
       VECTOR_NORM(Effect, 'norm2')   AS L2,
       VECTOR_NORM(Effect, 'norm1')   AS L1,
       VECTOR_NORM(Effect, 'norminf') AS LInf
FROM Herbal.Remedies ORDER BY RemedyId;
-- S6
DECLARE @q VECTOR(4) = '[1, 0, 0, 0]';
SELECT VECTOR_DISTANCE('manhattan', Effect, @q) AS D FROM Herbal.Remedies WHERE RemedyId = 1;
-- S7
SELECT VECTOR_DISTANCE('cosine', Effect, '[1, 0, 0, 0]') AS D FROM Herbal.Remedies WHERE RemedyId = 1;
-- S8
SELECT RemedyId FROM Herbal.Remedies WHERE Effect = CAST('[2, 0, 0, 0]' AS VECTOR(4));
-- S9
DECLARE @q VECTOR(4) = '[1, 0, 0, 0]';
SELECT TOP (3) RemedyId, Name, CAST(VECTOR_DISTANCE('cosine', Effect, @q) AS DECIMAL(10,4)) AS D
FROM Herbal.Remedies ORDER BY D ASC, RemedyId ASC;
-- S10
DECLARE @q VECTOR(4) = '[1, 0, 0, 0]';
SELECT TOP (3) RemedyId, Name, CAST(VECTOR_DISTANCE('cosine', Effect, @q) AS DECIMAL(10,4)) AS D
FROM Herbal.Remedies WHERE Effect IS NOT NULL ORDER BY D ASC, RemedyId ASC;
-- S11
DECLARE @q VECTOR(4) = '[1, 0, 0, 0]';
SELECT TOP (2) RemedyId, Name, CAST(VECTOR_DISTANCE('dot', Effect, @q) AS DECIMAL(10,4)) AS D
FROM Herbal.Remedies WHERE Effect IS NOT NULL ORDER BY D ASC, RemedyId ASC;
```

## 4. The question (ask exactly this)

"For each of the eleven statements, S1 to S11, tell me whether it succeeds or raises an error, and if it errors, which error number. For the ones that return rows, give me the exact result: the values, and the row order where it matters. Let's go one at a time, starting with S1."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

| Stmt | Outcome | Detail |
|---|---|---|
| S1 | Fails, error 42204 | The vector dimensions 4 and 3 do not match |
| S2 | Fails, error 13670 | Input JSON is not a valid Vector : 'Malformed JSON' |
| S3 | Succeeds | Bytes 24; EffectText is the JSON array in scientific notation: [0.0000000e+000,3.0000000e+000,4.0000000e+000,0.0000000e+000] |
| S4 | Succeeds | Six rows, see below; Placebo gives NULL in all three columns |
| S5 | Succeeds | Six rows, see below; Placebo gives NULL |
| S6 | Fails, error 42201 | The requested distance metric 'manhattan' is not supported by vector_distance |
| S7 | Fails, error 8116 | Argument data type varchar is invalid for argument 3 of vector_distance function; a string literal is not implicitly converted inside the function |
| S8 | Fails, error 8117 | Operand data type vector is invalid for equal to operator |
| S9 | Succeeds | Rows: 6 Placebo NULL; 1 Chamomile 0; 3 Elderberry 0.4. The NULL row sorts first |
| S10 | Succeeds | Rows: 1 Chamomile 0; 3 Elderberry 0.4; 4 Valerian 0.5 |
| S11 | Succeeds | Rows: 3 Elderberry minus 3; 1 Chamomile minus 2 |

S4 values (cosine, euclidean, dot):

| RemedyId | Cos | Euc | Dot |
|---|---|---|---|
| 1 Chamomile | 0 | 1 | minus 2 |
| 2 Peppermint | 1 | 5.0990 (square root of 26) | 0 |
| 3 Elderberry | 0.4 | 4.4721 (square root of 20) | minus 3 |
| 4 Valerian | 0.5 | 1.7321 (square root of 3) | minus 1 |
| 5 Guarana | 2 | 3 | 2 |
| 6 Placebo | NULL | NULL | NULL |

S5 values (norm2, norm1, norminf):

| RemedyId | L2 | L1 | LInf |
|---|---|---|---|
| 1 | 2 | 2 | 2 |
| 2 | 5 | 7 | 4 |
| 3 | 5 | 7 | 4 |
| 4 | 2 | 4 | 1 |
| 5 | 2 | 2 | 2 |
| 6 | NULL | NULL | NULL |

## 6. Hint ladder (one hint per attempt, in order)

**S1**
1. "Count the numbers in the string, and compare with the column definition."
2. "The column is VECTOR of 4. Is a three-element array padded, truncated, or rejected?"
3. "It is rejected. The error message names both sizes, declared first, supplied second."

**S2**
1. "Look at the shape of the string. Is it a JSON array?"
2. "Without square brackets it is not valid JSON. The engine reports the reason inside the message."

**S3**
1. "Storage is a fixed number of bytes per dimension plus a header. Four bytes per element, eight for the header."
2. "For the text: the engine renders the vector as a JSON array, but each float is printed in scientific notation, like three point zero e plus zero zero zero."

**S4**
1. "With q equals one, zero, zero, zero, the dot product of any vector with q is just its first component, and the length of q is one."
2. "Cosine distance is one minus the cosine of the angle. Cosine of the angle is dot product divided by the two lengths. Elderberry is three, zero, four, zero: its length is five."
3. "Euclidean distance is the length of the difference vector. Subtract q from each vector, then take the square root of the sum of squares."
4. "The dot metric returns the negative dot product. So Chamomile, with dot product two, shows minus two."
5. "What does any metric return when one side is NULL?"

**S5**
1. "norm2 is the Euclidean length. norm1 is the sum of absolute values. norminf is the largest absolute value."
2. "Peppermint is zero, three, four, zero: length five, sum seven, max four. Now do the rest."
3. "Guarana has a negative component. Norms use absolute values."

**S6**
1. "How many metric names does VECTOR_DISTANCE accept? Name them."
2. "Manhattan is not one of them. Is that a warning, a NULL, or an error?"

**S7**
1. "Compare S7 with S4. What is different about the third argument?"
2. "In S4 the query vector was a declared VECTOR variable. In S7 it is a string literal. Does the function convert a string for you?"
3. "Implicit conversion from string to vector happens on assignment, INSERT or DECLARE. Inside this function the argument must already be a vector."

**S8**
1. "Which operators does the vector type support? Think about equality, less-than, ORDER BY."
2. "None of them. Only IS NULL and IS NOT NULL. So what happens to an equals sign?"

**S9**
1. "Six rows are ranked, and one of them has a NULL Effect. What is its distance?"
2. "Where does NULL sort in an ascending ORDER BY in SQL Server?"
3. "NULL sorts first. So the NULL row is inside the TOP 3, and one real neighbour is pushed out."

**S10**
1. "Now the NULL row is filtered out. Take the three smallest cosine distances from S4."

**S11**
1. "Use the dot column of S4. Smallest first."
2. "Elderberry has minus three, Chamomile minus two. Which is smaller? Does that match the direction of q?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "S1 succeeds and the fourth element becomes zero" | Thinks the engine pads short arrays | "Is the dimension count negotiable, or is it a contract? Check what the message says." |
| "S2 succeeds; the engine parses the numbers" | Thinks any comma list is a vector | "What format does the vector type accept as text? Is a bare list valid JSON?" |
| "S3 returns 16 bytes" | Forgets the header | "Four times four is sixteen. Is there anything stored besides the elements?" |
| "S3 prints [0, 3, 4, 0]" | Expects the input formatting back | "The engine does not keep your string. It renders floats. How does it print three point zero?" |
| "Dot for Chamomile is plus 2" | Forgets the negation | "Smaller must mean closer for all three metrics. What sign makes that work?" |
| "Euclidean for Elderberry is smaller than for Valerian" | Confuses angle with distance | "Elderberry is long, Valerian is short. Euclidean cares about length. Recompute the difference vectors." |
| "S6 returns NULL" | Assumes unknown metric is ignored | "Unknown metric names are not tolerated silently. What kind of outcome is that?" |
| "S7 works, the string is converted" | Assumes implicit conversion everywhere | "Where did implicit conversion work earlier: on INSERT and DECLARE. Is a function argument the same thing?" |
| "S7 fails with 42204" | Right idea about strictness, wrong layer | "Does the engine even get as far as counting elements? What is the type of that argument?" |
| "S8 returns 1" | Thinks vectors are comparable | "What did the docs say about equality, uniqueness and ordering of vector values?" |
| "S9 returns Chamomile, Elderberry, Valerian" | Forgets NULL sorts first | "You have six rows, not five. Where does the sixth one land in ascending order?" |
| "S11 returns Chamomile first" | Uses cosine intuition for dot | "Read the dot values off S4. Which one is the smallest number?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the conversion contract first:

- **vector(n) is exactly n float32 values.** Text goes in as a JSON array; a wrong element count raises error 42204 with both sizes in the message, declared first, supplied second. That is S1. Anything that is not a plain JSON array of numbers, such as no brackets, a null element, a string element, an object, an empty array, or a nested array, raises error 13670 with the reason quoted. That is S2.
- **Storage is four bytes per dimension plus eight bytes**, so vector of 4 is 24 bytes and vector of 1998 is exactly 8000 bytes, which is why 1998 is the cap. Cast to text, the engine prints scientific notation with no spaces. That is S3.

Then the three metrics, from S4:

- Cosine is one minus the cosine of the angle: length is divided out, so Chamomile at twice the length of q is still distance zero.
- Euclidean is the length of the difference: short vectors look close, so Valerian beats Elderberry.
- Dot is the negative dot product, so smaller still means closer. Un-normalized, it rewards long vectors, which is why Elderberry beats Chamomile in S11. Use dot only on pre-normalized vectors, where it ranks like cosine at lower cost.
- Any NULL operand gives NULL.

Then the vocabulary and typing rules:

- Metric names are exactly cosine, euclidean, dot; anything else is error 42201. That is S6. VECTOR_NORM and VECTOR_NORMALIZE use a different vocabulary: norm1, norm2, norminf. That is S5.
- Implicit string-to-vector conversion happens on assignment only. Inside VECTOR_DISTANCE a string literal is error 8116; cast it or use a variable. That is S7.
- Vectors cannot be compared or sorted: equals is error 8117, ORDER BY on the column is error 42213. Only IS NULL works. That is S8.

Finally the search pattern:

- Exact k-nearest-neighbour search is SELECT TOP k, ORDER BY VECTOR_DISTANCE. It never uses a vector index. NULL sorts first in ascending order, so a nullable vector column needs WHERE column IS NOT NULL, or the NULL rows steal places in the top k. That is S9 versus S10.

Memory hook: "Four floats plus eight bytes. Three metrics, dot is negative. Strings convert on assignment, not inside the function. Filter the NULLs before you rank."

## 9. Follow-up oral questions (optional)

1. "How would you fix S7 without declaring a variable?" (Wrap the literal in CAST of the string AS VECTOR of 4; the result is zero.)
2. "What does VECTOR_NORMALIZE with norm2 return for Peppermint, zero, three, four, zero?" (Zero, zero point six, zero point eight, zero; printed in float32 as 6.0000002e-001 and 8.0000001e-001.)
3. "If every stored vector were first normalized to unit length, would the dot ranking in S11 still put Elderberry first?" (No. On unit vectors the dot distance equals cosine distance minus one, so Chamomile, at cosine zero, comes first.)

## 10. References

- vector data type, conversions and limitations: https://learn.microsoft.com/en-us/sql/t-sql/data-types/vector-data-type
- VECTOR_DISTANCE and the three metrics: https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-distance-transact-sql
- VECTOR_NORM: https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-norm-transact-sql
- VECTOR_NORMALIZE: https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-normalize-transact-sql
- Vector search and vector indexes overview: https://learn.microsoft.com/en-us/sql/sql-server/ai/vectors
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
