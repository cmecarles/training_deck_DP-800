# Instructor-Examiner guide — Sequences 1

Companion to [sequences_1.md](sequences_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a reverse-engineering question: the learner is shown a result and must say which query produced it. Take it in two parts: first which table, then which FOR XML clauses. The learner may want to replay the twenty inserts on paper; give them time and repeat piece 5 as often as needed. Several equivalent queries are accepted; check section 5 before judging.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Design and implement database objects.
- Task bullet: Create and use sequences.
- What is tested: that a SEQUENCE is a shared, table-independent counter, that NEXT VALUE FOR in a DEFAULT draws a value on every insert into any table that uses it, and which FOR XML options produce element-centric output with a root element and the xsi namespace declaration.

## 2. Scenario to read aloud

**Piece 1, the story.** "You run a script in a new database called SEQUENCE underscore EXERCISE. It creates four sequences, five tables that use those sequences as column defaults, and then does twenty inserts. Afterwards someone runs a query and shows you the result. Your job is to name that query."

**Piece 2, the sequences.** "Four sequences, all in dbo, all BIGINT, all start with one and increment by one. Their names are SEQ underscore AB, SEQ underscore ABC, SEQ underscore ABCD, and SEQ underscore ABCDE."

**Piece 3, the tables, part one.** "Five tables. Every table has an Id column, integer, IDENTITY starting at one, primary key, and a Step column, TINYINT, NOT NULL. Then come BIGINT NOT NULL columns whose DEFAULT is NEXT VALUE FOR one of the sequences. TABLE underscore A has four such columns: AB from SEQ underscore AB, ABC from SEQ underscore ABC, ABCD from SEQ underscore ABCD, and ABCDE from SEQ underscore ABCDE. TABLE underscore B is identical to TABLE underscore A: the same four sequence columns."

**Piece 4, the tables, part two.** "TABLE underscore C has three sequence columns: ABC, ABCD and ABCDE. TABLE underscore D has two: ABCD and ABCDE. TABLE underscore E has just one: ABCDE. Notice the pattern. The letters in the sequence name tell you which tables use it. SEQ underscore ABCDE is used by all five tables. SEQ underscore AB only by tables A and B."

**Piece 5, the twenty inserts.** "Now twenty inserts, in order, each one supplying only the Step column. So every other column comes from IDENTITY or from a sequence default. Step 1 goes into A. Step 2 into B. Step 3 into C. Step 4 into D. Step 5 into E. Step 6 into E again. Step 7 into D. Step 8 into C. Step 9 into B. Step 10 into A. Step 11 into A. Step 12 into B. Step 13 into C. Step 14 into D. Step 15 into E. Step 16 into E. Step 17 into D. Step 18 into C. Step 19 into B. And step 20 into A. So the pattern is A B C D E, E D C B A, A B C D E, E D C B A."

**Piece 6, the result, shape.** "The result is an XML document. At the top there is an XML declaration, version one point zero, encoding utf-8. The root element is called Data. The root carries one namespace declaration: xmlns colon xsi, pointing at the XML Schema instance URL, w3 dot org, 2001, XMLSchema-instance. Inside Data there are four elements called Row. Inside each Row, every column is its own child element: Id, Step, AB, ABC, ABCD, ABCDE. No attributes anywhere, apart from the namespace on the root."

**Piece 7, the result, values.** "Row one: Id 1, Step 1, AB 1, ABC 1, ABCD 1, ABCDE 1. Row two: Id 2, Step 10, AB 4, ABC 6, ABCD 8, ABCDE 10. Row three: Id 3, Step 11, AB 5, ABC 7, ABCD 9, ABCDE 11. Row four: Id 4, Step 20, AB 8, ABC 12, ABCD 16, ABCDE 20."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE SEQUENCE_EXERCISE;
GO

USE SEQUENCE_EXERCISE;
GO

CREATE SEQUENCE dbo.SEQ_AB     AS BIGINT START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE dbo.SEQ_ABC    AS BIGINT START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE dbo.SEQ_ABCD   AS BIGINT START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE dbo.SEQ_ABCDE  AS BIGINT START WITH 1 INCREMENT BY 1;
GO

CREATE TABLE dbo.TABLE_A(
Id    INT IDENTITY(1,1) PRIMARY KEY,
Step  TINYINT NOT NULL,
AB    BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_AB),
ABC   BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABC),
ABCD  BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABCD),
ABCDE BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABCDE)
);

CREATE TABLE dbo.TABLE_B(
Id    INT IDENTITY(1,1) PRIMARY KEY,
Step  TINYINT NOT NULL,
AB    BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_AB),
ABC   BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABC),
ABCD  BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABCD),
ABCDE BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABCDE)
);

CREATE TABLE dbo.TABLE_C(
Id    INT IDENTITY(1,1) PRIMARY KEY,
Step  TINYINT NOT NULL,
ABC   BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABC),
ABCD  BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABCD),
ABCDE BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABCDE)
);

CREATE TABLE dbo.TABLE_D(
Id    INT IDENTITY(1,1) PRIMARY KEY,
Step  TINYINT NOT NULL,
ABCD  BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABCD),
ABCDE BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABCDE)
);

CREATE TABLE dbo.TABLE_E(
Id    INT IDENTITY(1,1) PRIMARY KEY,
Step  TINYINT NOT NULL,
ABCDE BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABCDE)
);
GO

INSERT INTO dbo.TABLE_A (Step) VALUES (1);
INSERT INTO dbo.TABLE_B (Step) VALUES (2);
INSERT INTO dbo.TABLE_C (Step) VALUES (3);
INSERT INTO dbo.TABLE_D (Step) VALUES (4);
INSERT INTO dbo.TABLE_E (Step) VALUES (5);
INSERT INTO dbo.TABLE_E (Step) VALUES (6);
INSERT INTO dbo.TABLE_D (Step) VALUES (7);
INSERT INTO dbo.TABLE_C (Step) VALUES (8);
INSERT INTO dbo.TABLE_B (Step) VALUES (9);
INSERT INTO dbo.TABLE_A (Step) VALUES (10);
INSERT INTO dbo.TABLE_A (Step) VALUES (11);
INSERT INTO dbo.TABLE_B (Step) VALUES (12);
INSERT INTO dbo.TABLE_C (Step) VALUES (13);
INSERT INTO dbo.TABLE_D (Step) VALUES (14);
INSERT INTO dbo.TABLE_E (Step) VALUES (15);
INSERT INTO dbo.TABLE_E (Step) VALUES (16);
INSERT INTO dbo.TABLE_D (Step) VALUES (17);
INSERT INTO dbo.TABLE_C (Step) VALUES (18);
INSERT INTO dbo.TABLE_B (Step) VALUES (19);
INSERT INTO dbo.TABLE_A (Step) VALUES (20);
GO
```

The result shown to the learner:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Data xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <Row>
    <Id>1</Id>
    <Step>1</Step>
    <AB>1</AB>
    <ABC>1</ABC>
    <ABCD>1</ABCD>
    <ABCDE>1</ABCDE>
  </Row>
  <Row>
    <Id>2</Id>
    <Step>10</Step>
    <AB>4</AB>
    <ABC>6</ABC>
    <ABCD>8</ABCD>
    <ABCDE>10</ABCDE>
  </Row>
  <Row>
    <Id>3</Id>
    <Step>11</Step>
    <AB>5</AB>
    <ABC>7</ABC>
    <ABCD>9</ABCD>
    <ABCDE>11</ABCDE>
  </Row>
  <Row>
    <Id>4</Id>
    <Step>20</Step>
    <AB>8</AB>
    <ABC>12</ABC>
    <ABCD>16</ABCD>
    <ABCDE>20</ABCDE>
  </Row>
</Data>
```

## 4. The question (ask exactly this)

"Which query produced that result? Let's take it in two parts. Part one: which table was queried, and why? Part two: which FOR XML clauses were used to get that exact shape, with the Row elements, the Data root, the columns as child elements, and the xsi namespace on the root?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Part one: TABLE underscore A.**

Each insert draws a new value from every sequence its table has a default for. So SEQ underscore AB counts inserts into A and B, SEQ underscore ABC counts inserts into A, B and C, SEQ underscore ABCD counts A, B, C and D, and SEQ underscore ABCDE counts every insert. Replaying the twenty inserts:

| Step | Table | AB | ABC | ABCD | ABCDE |
|---|---|---|---|---|---|
| 1 | A | 1 | 1 | 1 | 1 |
| 2 | B | 2 | 2 | 2 | 2 |
| 3 | C | — | 3 | 3 | 3 |
| 4 | D | — | — | 4 | 4 |
| 5 | E | — | — | — | 5 |
| 6 | E | — | — | — | 6 |
| 7 | D | — | — | 5 | 7 |
| 8 | C | — | 4 | 6 | 8 |
| 9 | B | 3 | 5 | 7 | 9 |
| 10 | A | 4 | 6 | 8 | 10 |
| 11 | A | 5 | 7 | 9 | 11 |
| 12 | B | 6 | 8 | 10 | 12 |
| 13 | C | — | 9 | 11 | 13 |
| 14 | D | — | — | 12 | 14 |
| 15 | E | — | — | — | 15 |
| 16 | E | — | — | — | 16 |
| 17 | D | — | — | 13 | 17 |
| 18 | C | — | 10 | 14 | 18 |
| 19 | B | 7 | 11 | 15 | 19 |
| 20 | A | 8 | 12 | 16 | 20 |

TABLE underscore A received steps 1, 10, 11 and 20, giving exactly the four rows shown. Only A and B have all four sequence columns, and B's Step values would be 2, 9, 12, 19. The Step column pins it to A. Giveaway: in row four, ABCDE equals Step, because SEQ underscore ABCDE fires on every one of the twenty inserts.

**Part two: `FOR XML PATH('Row'), ROOT('Data'), ELEMENTS XSINIL`.**

```sql
SELECT Id, Step, AB, ABC, ABCD, ABCDE
FROM dbo.TABLE_A
ORDER BY Id
FOR XML PATH('Row'), ROOT('Data'), ELEMENTS XSINIL;
```

- PATH with Row: each row becomes a Row element, columns become child elements.
- ROOT with Data: the Data wrapper.
- ELEMENTS XSINIL: element-centric output plus the xmlns colon xsi declaration on the root. FOR XML adds that namespace only when XSINIL is specified.
- The XML prolog is added by the client tool, not by SQL Server.

Accepted equivalents: `SELECT *` (column order matches). `FOR XML RAW('Row'), ROOT('Data'), ELEMENTS XSINIL`. `FROM dbo.TABLE_A AS Row ... FOR XML AUTO, ROOT('Data'), ELEMENTS XSINIL`. ORDER BY Step or any of the sequence columns instead of Id, since all increase together within TABLE underscore A.

## 6. Hint ladder (one hint per attempt, in order)

**Part one, which table**
1. "Look at the columns in the result. Which tables even have all four sequence columns AB, ABC, ABCD and ABCDE?"
2. "That leaves two candidates. Now look at the Step values in the result: 1, 10, 11, 20. Which table received those inserts?"
3. "Check with the sequences. SEQ underscore ABCDE is used by every table, so it goes up by one on every insert. In row four, ABCDE is 20 and Step is 20. Is that consistent with your table?"
4. "Replay SEQ underscore AB. It only moves when A or B gets an insert. Steps 1 and 2 give values 1 and 2. Steps 9 and 10 give 3 and 4. So at step 10 the AB value is 4. Does your table's second row show AB equals 4?"

**Part two, the XML shape**
1. "A plain SELECT returns a grid, not XML. Which clause turns a result set into XML?"
2. "The columns are child elements, not attributes, and each row is wrapped in an element called Row. Which FOR XML mode lets you name the row element, and which keyword makes columns into elements?"
3. "There is a wrapper element called Data around everything. Which FOR XML option adds a root element with a given name?"
4. "Look at the root once more. It has an xmlns colon xsi namespace declaration. FOR XML only emits that when one particular keyword is present, the one that makes NULLs explicit. Which keyword?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "TABLE underscore B, because it also has four sequence columns" | Did not check the Step column | "Both A and B have those columns. Now which steps were inserted into B? Compare with the Step values in the result." |
| "The sequence values should be 1, 2, 3, 4 per table" | Thinks a sequence is per table like IDENTITY | "A sequence is a separate object. Who else draws from SEQ underscore AB besides TABLE underscore A?" |
| "AB should equal Id" | Confuses IDENTITY with the shared sequence | "Id is IDENTITY, private to the table. AB comes from a sequence shared with TABLE underscore B. Do they advance at the same rate?" |
| "SELECT star from TABLE underscore A ORDER BY Id, that is all" | Forgets the output is XML | "That query returns a grid. What produced the angle brackets?" |
| "FOR XML AUTO" | Not wrong, but incomplete | "AUTO can work if the row element is named Row. How do you get that name, and what about the Data root and the namespace?" |
| "FOR XML PATH with Row and ROOT with Data, done" | Missed the xsi namespace | "That gives Row elements and a Data root. Now look at the root element once more. Where does the xmlns colon xsi come from?" |
| "FOR XML RAW with ELEMENTS" | Missed the element name, the root and XSINIL | "RAW by default names the row element row, lowercase. And there is a root and a namespace to account for." |
| "The XML declaration at the top comes from SQL Server" | Does not know clients add the prolog | "SQL Server does not emit an XML prolog. Who does?" |

## 8. Teaching notes (after the answer is complete or revealed)

Two lessons: **sequences are shared counters**, and **FOR XML options are recoverable from the output shape**.

- **Sequences versus IDENTITY.** IDENTITY belongs to one table. A SEQUENCE is a standalone schema object; any table, default, or statement can call NEXT VALUE FOR on it, and every call advances the same counter. Here the names document the consumers: SEQ underscore AB advances on inserts into A or B, SEQ underscore ABCDE on every insert. That is why TABLE underscore A's four rows show AB 1, 4, 5, 8 rather than 1, 2, 3, 4. Sequences are the tool when several tables must share one numbering, or when you need the number before the insert, or when you want to control cache and cycle behaviour.
- **NEXT VALUE FOR in a DEFAULT.** The default fires once per inserted row, in the order of the inserts. The value is consumed even if the transaction rolls back, so sequences can have gaps.
- **Reading the XML shape back.** Element-centric with a named row element means PATH with that name, or RAW with that name plus ELEMENTS, or AUTO with a table alias of that name plus ELEMENTS. A wrapper element means ROOT with that name. And the xmlns colon xsi declaration on the root only appears when XSINIL is specified: XSINIL exists to mark NULL columns as xsi colon nil equals true, so the engine has to declare the xsi prefix. No NULLs appear in this data, but the declaration is still emitted. The XML prolog, version and encoding, is not part of the FOR XML output; the client adds it when saving the result.
- **Row order.** ORDER BY Id, or by Step, or by any sequence column, all give the same order in TABLE underscore A, because they all increase together.

Memory hook: "IDENTITY is per table, SEQUENCE is per database. And xsi on the root means XSINIL was in the query."

## 9. Follow-up oral questions (optional)

1. "If the same query were run against TABLE underscore B, what Step values would appear?" (2, 9, 12, 19.)
2. "What would the query output if you dropped XSINIL?" (Same elements and root, but no xmlns colon xsi declaration on the root.)
3. "What is the last value SEQ underscore ABCD reached after the twenty inserts?" (16; it advances on inserts into A, B, C and D, which is sixteen of the twenty.)

## 10. References

- CREATE SEQUENCE: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-sequence-transact-sql
- NEXT VALUE FOR: https://learn.microsoft.com/en-us/sql/t-sql/functions/next-value-for-transact-sql
- Sequence numbers overview: https://learn.microsoft.com/en-us/sql/relational-databases/sequence-numbers/sequence-numbers
- FOR XML PATH mode: https://learn.microsoft.com/en-us/sql/relational-databases/xml/use-path-mode-with-for-xml
- FOR XML, ROOT, ELEMENTS and XSINIL options: https://learn.microsoft.com/en-us/sql/relational-databases/xml/basic-syntax-of-the-for-xml-clause
- FOR XML RAW mode: https://learn.microsoft.com/en-us/sql/relational-databases/xml/use-raw-mode-with-for-xml
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
