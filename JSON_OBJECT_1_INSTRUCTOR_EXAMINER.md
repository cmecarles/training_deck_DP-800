# Instructor-Examiner guide — JSON_OBJECT 1

Companion to [JSON_OBJECT_1.md](JSON_OBJECT_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is an exact-output question with three rows. Take one row at a time, and within a row go key by key: device, room, path, net, levels, online. The learner speaks JSON, so accept spoken forms such as "quote path quote colon null" and "backslash quote". Be strict about which keys are present, where null appears, whether the boolean is bare, and how quotes and backslashes are escaped. Row 3 has the backslash trap; leave it for last.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Implement programmability objects and data types.
- Task bullet: Work with JSON data using the native JSON functions.
- What is tested: the `NULL ON NULL` default of `JSON_OBJECT`, what `ABSENT ON NULL` does to a key, `NULL ON NULL` on a nested `JSON_ARRAY`, `bit` rendered as a JSON boolean, and escaping of double quotes and backslashes.

## 2. Scenario to read aloud

**Piece 1, the story.** "A smart-home platform stores the configuration of every device in a mesh network. A REST endpoint must return one JSON configuration document per device. The document is built directly in T-SQL with the JSON underscore OBJECT constructor, on SQL Server 2025."

**Piece 2, the table.** "The database is HomeMesh, at compatibility level one hundred seventy. There is a schema Iot and one table, Iot dot Devices, with nine columns. DeviceId, an integer primary key. DeviceName, text up to sixty characters, not null. Room, text up to forty, nullable. StreamPath, text up to eighty, nullable. IpAddress, varchar fifteen, not null. Gateway, varchar fifteen, nullable. Brightness, integer, nullable. Volume, integer, nullable. And IsOnline, a bit, not null."

**Piece 3, device 1.** "Three devices are inserted. Device 1 is Hall Thermostat. Room is Hallway. StreamPath is null. IP address ten dot zero dot zero dot seven. Gateway ten dot zero dot zero dot one. Brightness eighty. Volume null. IsOnline one."

**Piece 4, device 2.** "Device 2 has the name Kids, space, then the word Play wrapped in double quotes, space, Lamp. So the name contains two double-quote characters. Room is null. StreamPath is null. IP address ten dot zero dot zero dot nine. Gateway is null. Brightness is null. Volume thirty-five. IsOnline zero."

**Piece 5, device 3.** "Device 3 is Garage Cam. Room is Garage. StreamPath is a UNC path: two backslashes, hub zero one, backslash, streams, backslash, cam two. Four backslashes in total, because T-SQL string literals do not treat backslash as an escape, so the two leading backslashes are really stored. IP address ten dot zero dot zero dot twelve. Gateway ten dot zero dot zero dot one. Brightness null. Volume null. IsOnline one."

**Piece 6, the query.** "One query selects DeviceId and a column ConfigJson, ordered by DeviceId. ConfigJson is an outer JSON OBJECT with six keys in this order. Key device, from DeviceName. Key room, from Room. Key path, from StreamPath. Key net, whose value is a nested JSON OBJECT with keys ip from IpAddress and gw from Gateway, followed by the clause ABSENT ON NULL. Key levels, whose value is a JSON ARRAY of Brightness and Volume, followed by the clause NULL ON NULL. And key online, from IsOnline. The outer JSON OBJECT has no null clause of its own."

**Piece 7, what is asked.** "You will be asked for the exact ConfigJson string for each of the three rows. Every character matters: which keys appear, where the word null appears, how the boolean is written, and how quotes and backslashes are escaped. I can read any line on request."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE HomeMesh;
GO
ALTER DATABASE HomeMesh SET COMPATIBILITY_LEVEL = 170;
GO
USE HomeMesh;
GO
CREATE SCHEMA Iot;
GO
CREATE TABLE Iot.Devices (
    DeviceId   INT           PRIMARY KEY,
    DeviceName NVARCHAR(60)  NOT NULL,
    Room       NVARCHAR(40)  NULL,
    StreamPath NVARCHAR(80)  NULL,
    IpAddress  VARCHAR(15)   NOT NULL,
    Gateway    VARCHAR(15)   NULL,
    Brightness INT           NULL,
    Volume     INT           NULL,
    IsOnline   BIT           NOT NULL
);
GO
INSERT INTO Iot.Devices (DeviceId, DeviceName, Room, StreamPath, IpAddress, Gateway, Brightness, Volume, IsOnline) VALUES
  (1, N'Hall Thermostat',  N'Hallway', NULL,                    '10.0.0.7',  '10.0.0.1', 80,   NULL, 1),
  (2, N'Kids "Play" Lamp', NULL,       NULL,                    '10.0.0.9',  NULL,       NULL, 35,   0),
  (3, N'Garage Cam',       N'Garage',  N'\\hub01\streams\cam2', '10.0.0.12', '10.0.0.1', NULL, NULL, 1);
GO
SELECT d.DeviceId,
       JSON_OBJECT(
           'device': d.DeviceName,
           'room':   d.Room,
           'path':   d.StreamPath,
           'net':    JSON_OBJECT('ip': d.IpAddress, 'gw': d.Gateway ABSENT ON NULL),
           'levels': JSON_ARRAY(d.Brightness, d.Volume NULL ON NULL),
           'online': d.IsOnline
       ) AS ConfigJson
FROM Iot.Devices AS d
ORDER BY d.DeviceId;
```

## 4. The question (ask exactly this)

"Write the exact ConfigJson string returned for each of the three rows. Every character, including quoting, escaping, key order, and how every SQL NULL is, or is not, represented. Let's go one row at a time. Start with DeviceId 1, key by key: device, room, path, net, levels, online."

Then DeviceId 2, then DeviceId 3.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

DeviceId 1:

```json
{"device":"Hall Thermostat","room":"Hallway","path":null,"net":{"ip":"10.0.0.7","gw":"10.0.0.1"},"levels":[80,null],"online":true}
```

DeviceId 2:

```json
{"device":"Kids \"Play\" Lamp","room":null,"path":null,"net":{"ip":"10.0.0.9"},"levels":[null,35],"online":false}
```

DeviceId 3:

```json
{"device":"Garage Cam","room":"Garage","path":"\\\\hub01\\streams\\cam2","net":{"ip":"10.0.0.12","gw":"10.0.0.1"},"levels":[null,null],"online":true}
```

Points to check:

- Outer object: null values keep their key, so path colon null in row 1, room and path colon null in row 2.
- Nested net object: in row 2 the gw key is missing entirely; only ip remains.
- levels is always two elements: 80 null, null 35, null null.
- online is bare true or false, never 1, 0 or a quoted string.
- Row 2 device: each embedded double quote becomes backslash quote.
- Row 3 path: each stored backslash becomes two, so four stored backslashes render as eight. Two before hub01 become four; one before streams becomes two; one before cam2 becomes two.
- Keys in call order, no whitespace anywhere.

## 6. Hint ladder (one hint per attempt, in order)

**Row 1 and row 2, the null keys room and path**
1. "The outer JSON OBJECT has no null clause. Which null behaviour does JSON OBJECT use by default?"
2. "JSON OBJECT and JSON ARRAY have opposite defaults. The object is the one that keeps something. What does it keep?"
3. "The key stays. What value is written after the colon?"

**Row 2, the net object**
1. "The nested net object says ABSENT ON NULL. Gateway is null for device 2. What happens to the gw pair?"
2. "ABSENT means the whole key-value pair is gone. How many keys does net have in row 2?"

**levels, any row**
1. "The array says NULL ON NULL. Does a null Brightness or Volume keep its position?"
2. "So the array always has two elements. Which positions are null in this row?"

**online, any row**
1. "IsOnline is a bit. Under the FOR JSON type rules, is a bit a number, a string, or a boolean?"
2. "A boolean in JSON is written as a bare word. Which word for one, and which for zero?"

**Row 2, the device name**
1. "The stored name contains double quotes. Can a raw double quote appear inside a JSON string?"
2. "Each embedded double quote needs a backslash in front of it. How many escaped quotes are there?"

**Row 3, the path**
1. "First, how many backslashes are actually stored? Remember that T-SQL does not treat backslash as an escape character."
2. "Four are stored. In JSON, what must be done to every backslash inside a string?"
3. "Each stored backslash becomes two. So how many backslashes are in the rendered path, and how are they grouped?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| Row 1 without the path key | Applies the JSON_ARRAY default to JSON_OBJECT | "Which constructor has ABSENT ON NULL as its default? Is it the object?" |
| Row 2 net as ip plus gw colon null | Ignores the nested ABSENT ON NULL | "Read the clause at the end of the net constructor. What does absent mean for a key?" |
| Row 2 net key missing, or net colon null | Thinks a nested constructor can be NULL | "Does a JSON OBJECT call ever return SQL NULL, or always a string?" |
| levels as a one-element array | Forgets the explicit NULL ON NULL on the array | "The array has its own clause. Does it drop nulls or keep them?" |
| online colon 1, or online colon quote true quote | Does not know bit maps to boolean | "How does the bit type render under the FOR JSON conversion rules?" |
| Row 3 path with four backslashes | Forgets JSON escapes backslashes | "Is a backslash a plain character inside a JSON string, or must it be escaped?" |
| Row 3 path with two backslashes before hub01 | Thinks T-SQL collapsed the literal's double backslash | "Does a T-SQL string literal treat backslash as an escape character? So how many are stored?" |
| Keys in alphabetical order | Assumes keys are sorted | "In what order does JSON OBJECT emit keys?" |

## 8. Teaching notes (after the answer is complete or revealed)

Five independent rules combine to produce these strings:

- **Rule 1, JSON OBJECT defaults to NULL ON NULL.** With no clause on the outer object, every SQL NULL becomes a JSON null with its key kept. That is path colon null in row 1, and room and path colon null in row 2. This is the exact opposite of JSON ARRAY and JSON ARRAYAGG, whose default is ABSENT ON NULL. It is the most confused fact about the constructor family.
- **Rule 2, ABSENT ON NULL removes the whole key.** The nested net object overrides the default. In row 2, Gateway is null, so gw disappears and net has one key. Note what does not happen: the nested JSON OBJECT itself never returns SQL NULL, so the outer net key can never be nulled or removed by the outer object's clause. A null clause only governs the arguments inside its own constructor. Also, ISNULL on Gateway is not a substitute for ABSENT ON NULL; no substitution value removes a key.
- **Rule 3, JSON ARRAY with NULL ON NULL keeps positions.** levels is always two elements. Without the clause the default ABSENT ON NULL would give 80 alone, 35 alone, and an empty array, with a different length per row.
- **Rule 4, bit renders as a bare true or false.** JSON OBJECT uses the FOR JSON type rules. Never 1, 0, or a quoted word.
- **Rule 5, escaping.** Inside string values a double quote becomes backslash quote, and a backslash becomes a double backslash. T-SQL itself never escapes backslashes, so the literal with two leading backslashes stores two. JSON then doubles all four to eight.

Keys appear in the order of the call, with no whitespace. The result type is nvarchar max; adding RETURNING json returns the native json type with the same text.

Memory hook: "Object keeps the key, array drops the element. Bit is a bare boolean. T-SQL stores backslashes, JSON doubles them."

## 9. Follow-up oral questions (optional)

1. "If the levels array lost its NULL ON NULL clause, what would row 3's levels be?" (An empty array, because JSON ARRAY defaults to ABSENT ON NULL.)
2. "If the outer object were given ABSENT ON NULL, which keys would disappear in row 2?" (room and path. net, levels and online stay, because nested constructors and the bit are never NULL.)
3. "How many backslash characters does the stored StreamPath of device 3 contain, and how many does the JSON output contain?" (Four stored, eight in the output.)

## 10. References

- JSON_OBJECT (Transact-SQL), including the NULL ON NULL default: https://learn.microsoft.com/en-us/sql/t-sql/functions/json-object-transact-sql
- JSON_ARRAY (Transact-SQL): https://learn.microsoft.com/en-us/sql/t-sql/functions/json-array-transact-sql
- FOR JSON type conversion and escaping rules: https://learn.microsoft.com/en-us/sql/relational-databases/json/how-for-json-converts-sql-server-data-types-to-json-data-types-sql-server
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
