# Instructor-Examiner guide — OPENJSON 1

Companion to [OPENJSON_1.md](OPENJSON_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a predict-the-output question with three queries, A, B and C. Take them one at a time. For query A ask for the six rows in order, each as key, value and type code. For query B go column by column, asking for EGLL then KSEA. For query C ask whether it returns rows or fails, and with what message. Be strict about NULL versus the text null, and about type codes.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Implement programmability objects and data types.
- Task bullet: Work with JSON data using the native JSON functions.
- What is tested: the default `key`, `value`, `type` schema of `OPENJSON` and its type codes, the `WITH` clause with and without `AS JSON`, case-sensitive matching of column names to keys when no path is given, and lax versus strict path failure.

## 2. Scenario to read aloud

**Piece 1, the story.** "A meteorological network ingests raw weather-station observation feeds. Each station's latest observation arrives as one JSON document and is stored as text. Three queries shred those documents with OPENJSON."

**Piece 2, the table.** "The database is SkyWatch on SQL Server 2025, at compatibility level one hundred seventy. There is a schema Wx and one table, Wx dot Stations, with three columns. StationID, an identity integer primary key. StationCode, a four-character code, not null and unique. And Obs, an NVARCHAR MAX column, not null, with a CHECK constraint that ISJSON of Obs equals one."

**Piece 3, station KSEA.** "Two stations are inserted. The first is KSEA, that is K S E A, station ID one. Its document has six keys in this order. temp underscore c, the number twelve point five. wind, an object with dir equal to the string NW and kts equal to the number eighteen. gusts, an array of three numbers: twenty-two, twenty-seven, thirty-one. qc underscore pass, the boolean true. obs underscore time, the string zero six colon zero zero Z. And ceiling underscore ft, a JSON null."

**Piece 4, station EGLL.** "The second is EGLL, that is E G L L, station ID two. temp c, the number minus three. wind, an object with dir N and kts five. gusts, an empty array. qc pass, the boolean false. obs time, the string zero six colon one zero Z. And ceiling ft, the number one thousand two hundred."

**Piece 5, query A.** "Query A uses the default schema. It selects key, value and type from Stations, CROSS APPLY OPENJSON of Obs, with no WITH clause. It filters where StationCode is KSEA, and orders by key."

**Piece 6, query B.** "Query B uses an explicit WITH clause over both stations, ordered by StationCode. It selects StationCode and six columns from the WITH clause. I will read the six column definitions.

- TempC, decimal four comma one, path dollar dot temp c.
- WindKts, int, path strict dollar dot wind dot kts.
- Gusts, nvarchar max, path dollar dot gusts, followed by AS JSON.
- GustsScalar, nvarchar forty, path dollar dot gusts, with no AS JSON.
- CeilingFt, int, path dollar dot ceiling ft.
- QcPass, bit, with no path at all. Note the spelling: capital Q, c, capital P, a, s, s, with no underscore."

**Piece 7, query C.** "Query C selects StationCode and one WITH-clause column, RunwayVis, int, path strict dollar dot runway underscore vis. It filters where StationCode is KSEA. Neither document contains a runway vis key."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE SkyWatch;
GO
ALTER DATABASE SkyWatch SET COMPATIBILITY_LEVEL = 170;
GO
USE SkyWatch;
GO
CREATE SCHEMA Wx;
GO
CREATE TABLE Wx.Stations
(
    StationID   INT IDENTITY(1,1) PRIMARY KEY,
    StationCode CHAR(4)       NOT NULL UNIQUE,
    Obs         NVARCHAR(MAX) NOT NULL CHECK (ISJSON(Obs) = 1)
);
GO
INSERT INTO Wx.Stations (StationCode, Obs) VALUES
 ('KSEA', N'{"temp_c":12.5,"wind":{"dir":"NW","kts":18},"gusts":[22,27,31],"qc_pass":true,"obs_time":"06:00Z","ceiling_ft":null}'),
 ('EGLL', N'{"temp_c":-3,"wind":{"dir":"N","kts":5},"gusts":[],"qc_pass":false,"obs_time":"06:10Z","ceiling_ft":1200}');
GO
-- Query A
SELECT j.[key], j.[value], j.[type]
FROM Wx.Stations AS s
CROSS APPLY OPENJSON(s.Obs) AS j
WHERE s.StationCode = 'KSEA'
ORDER BY j.[key];
-- Query B
SELECT
    s.StationCode,
    j.TempC,
    j.WindKts,
    j.Gusts,
    j.GustsScalar,
    j.CeilingFt,
    j.QcPass
FROM Wx.Stations AS s
CROSS APPLY OPENJSON(s.Obs)
WITH (
    TempC       DECIMAL(4,1)  '$.temp_c',
    WindKts     INT           'strict $.wind.kts',
    Gusts       NVARCHAR(MAX) '$.gusts' AS JSON,
    GustsScalar NVARCHAR(40)  '$.gusts',
    CeilingFt   INT           '$.ceiling_ft',
    QcPass      BIT
) AS j
ORDER BY s.StationCode;
-- Query C
SELECT s.StationCode, j.RunwayVis
FROM Wx.Stations AS s
CROSS APPLY OPENJSON(s.Obs)
WITH (
    RunwayVis INT 'strict $.runway_vis'
) AS j
WHERE s.StationCode = 'KSEA';
```

## 4. The question (ask exactly this)

"Predict the exact result of each of the three queries: every column value of every row, including NULLs, or the exact failure if a query cannot complete. Let's start with query A. How many rows, and for each row in order, the key, the value, and the type code."

Then: "Query B. Two rows, EGLL then KSEA. Let's go column by column: TempC, WindKts, Gusts, GustsScalar, CeilingFt, QcPass."

Then: "Query C. Does it return rows? If so, what. If not, what happens?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Query A, six rows** (verified on SQL Server 2025 RTM):

| key | value | type |
|---|---|---|
| ceiling_ft | NULL | 0 |
| gusts | [22,27,31] | 4 |
| obs_time | 06:00Z | 1 |
| qc_pass | true | 3 |
| temp_c | 12.5 | 2 |
| wind | {"dir":"NW","kts":18} | 5 |

Type codes: 0 null, 1 string, 2 number, 3 true or false, 4 array, 5 object. The JSON null becomes SQL NULL with type 0. The boolean is the text true. Nested values are not expanded. Rows are in key order because of the ORDER BY, so ceiling_ft comes first.

**Query B, two rows:**

| StationCode | TempC | WindKts | Gusts | GustsScalar | CeilingFt | QcPass |
|---|---|---|---|---|---|---|
| EGLL | -3.0 | 5 | [] | NULL | 1200 | NULL |
| KSEA | 12.5 | 18 | [22,27,31] | NULL | NULL | NULL |

- TempC: lax cast to decimal, minus three point zero and twelve point five.
- WindKts: strict path that resolves in both documents, 5 and 18. Strict is not an automatic error.
- Gusts with AS JSON: the fragment, an empty array as two characters for EGLL, not NULL.
- GustsScalar without AS JSON: JSON_VALUE semantics on an array, NULL for both rows.
- CeilingFt: 1200 for EGLL; KSEA has JSON null, which is SQL NULL.
- QcPass with no path: column name matched to key case-sensitively with BIN2. QcPass is not qc_pass, so NULL for both rows.

**Query C, fails at run time, no rows:**

```text
Msg 13608, Level 16, State 6
Property cannot be found on the specified JSON path.
```

The WHERE filter does not help; the strict mapping fails while the applied rowset is evaluated, and one failing row aborts the statement.

## 6. Hint ladder (one hint per attempt, in order)

**Query A, row count and columns**
1. "With no WITH clause, what three columns does OPENJSON return, and one row per what?"
2. "Only first-level keys become rows. How many first-level keys does the KSEA document have? Does wind dot kts get its own row?"
3. "There is an ORDER BY on key. Which key sorts first alphabetically?"

**Query A, values and type codes**
1. "The type codes run from zero to five. Zero is null. Can you list the rest in order: string, number, boolean, array, object?"
2. "ceiling ft holds a JSON null. Does the value column show the text null, or a SQL NULL? And what type code?"
3. "qc pass holds true. An nvarchar column cannot hold a boolean. What text appears?"
4. "For wind and gusts, is the value expanded or shown as the raw fragment?"

**Query B, TempC and WindKts**
1. "TempC is a lax mapping with a cast to decimal four comma one. What does minus three become with one decimal place?"
2. "WindKts is strict. Does strict fail by itself, or only when the path does not resolve? Do both documents have wind dot kts?"

**Query B, Gusts and GustsScalar**
1. "Same path, one with AS JSON and one without. Which JSON function does each one behave like?"
2. "Without AS JSON the mapping wants a scalar. The path lands on an array. Lax mode. What comes back?"
3. "With AS JSON, EGLL's gusts is an empty array. Is that NULL or the two characters open bracket close bracket?"

**Query B, CeilingFt**
1. "EGLL has a number. KSEA has a JSON null. What does each become?"

**Query B, QcPass**
1. "This column has no path. How does OPENJSON decide which key to read?"
2. "It matches the column name against the key. Is that comparison case-sensitive? Is QcPass spelled the same as qc underscore pass?"
3. "The names do not match. Lax mode. What is the column value?"

**Query C**
1. "The path is strict and the key runway vis does not exist. What does strict do on a miss?"
2. "It raises an error. Does the WHERE clause on KSEA prevent that, and does the query return any rows?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| Query A has eight or more rows | Expects nested keys to be expanded | "Does the default schema descend into wind, or return it whole?" |
| ceiling ft value is the text null | Confuses JSON null with a string | "How does a JSON null surface in a relational column?" |
| qc pass has type 1 | Treats the boolean as a string | "Which type code is reserved for true and false?" |
| Rows in document order for query A | Ignores the ORDER BY | "Is there an ORDER BY on the query? On which column?" |
| WindKts errors because it is strict | Thinks strict always fails | "Strict only changes what happens on a miss. Does wind dot kts exist in both documents?" |
| GustsScalar shows the array text | Forgets AS JSON is required for fragments | "Without AS JSON, which function's semantics apply? Can it return an array?" |
| Gusts for EGLL is NULL | Treats an empty array as missing | "Does the gusts key exist for EGLL? What is its raw fragment?" |
| QcPass is 1 and 0 | Assumes case-insensitive name matching | "Compare the column name with the key letter by letter, including the underscore." |
| Query C returns one row with NULL | Forgets the strict prefix | "Read the path again. What is the first word, and what does it do on a miss?" |

## 8. Teaching notes (after the answer is complete or revealed)

Default-schema OPENJSON returns key, value and type for first-level properties only. Key is nvarchar 4000 in a BIN2 collation, value is nvarchar max with the value rendered as text, and type is a small integer code. Microsoft Learn documents the type column as int, but the engine's actual metadata on 2025 RTM is tinyint. The codes are: 0 null, 1 string, 2 number, 3 true or false, 4 array, 5 object. A JSON null becomes SQL NULL with type 0; the key exists and the type column records that. A boolean becomes the literal text true or false, type 3. Nested objects and arrays come back as the raw fragment, types 5 and 4, never expanded. Row order follows ORDER BY; without it rows come back in document order, and relying on that is unsafe.

In a WITH clause, four rules:

- **No AS JSON means JSON VALUE semantics.** A scalar is expected. An object or array gives NULL in lax mode and an error in strict mode. That is GustsScalar, the classic "my column is always NULL" bug.
- **AS JSON means JSON QUERY semantics.** The fragment is returned and the type must be NVARCHAR MAX. An empty array is two characters, not NULL. That is Gusts.
- **No path means the column name is matched to the key, case-sensitively, with BIN2.** QcPass does not equal qc underscore pass, so the column is NULL on both rows even though both documents carry the boolean. Writing the column as qc underscore pass, or adding the path dollar dot qc pass, would return 1 and 0.
- **Lax is the default; a miss gives NULL. Strict on a miss raises error 13608 and aborts the whole statement.** WindKts shows that strict is harmless when the path resolves. Query C shows the failure, and the WHERE filter does not save it.

CROSS APPLY runs OPENJSON once per table row; a single-object document yields one row per application, so query B joins to two rows. OPENJSON needs compatibility level 130 or higher, or the ALLOW_BUILTIN_TVF_IN_ALL_COMPAT_LEVELS scoped configuration. The queries behave the same if Obs is the native json type.

Memory hook: "Zero null, one string, two number, three bool, four array, five object. No AS JSON, no fragment. No path, exact name. Strict miss, whole statement dies."

## 9. Follow-up oral questions (optional)

1. "How would you fix the QcPass column so it returns 1 and 0?" (Name the column qc_pass, or add the explicit path dollar dot qc_pass.)
2. "If query C used a lax path instead of strict, what would it return?" (One row, KSEA, with RunwayVis NULL.)
3. "What type code would query A show for the EGLL ceiling ft key?" (2, number, because EGLL's value is 1200.)

## 10. References

- OPENJSON (Transact-SQL), default schema and type codes: https://learn.microsoft.com/en-us/sql/t-sql/functions/openjson-transact-sql
- Use OPENJSON with an explicit schema, including AS JSON and column-name matching: https://learn.microsoft.com/en-us/sql/relational-databases/json/use-openjson-with-an-explicit-schema-sql-server
- JSON path expressions, lax and strict modes: https://learn.microsoft.com/en-us/sql/relational-databases/json/json-path-expressions-sql-server
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
