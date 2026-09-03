# Instructor-Examiner guide — REGEXP_MATCHES 1

Companion to [REGEXP_MATCHES_1.md](REGEXP_MATCHES_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a prediction question with several parts: the row count, the rows for each reagent, and the JSON in the substring_matches column. Take it one reagent at a time, and within each reagent, one match at a time. Ask for the JSON only after the learner has the positions and match values right; accept a spoken description of the JSON such as "two objects, value H start 1 length 1, value 2 start 2 length 1".

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked. For the pattern, say "open square bracket capital A to capital Z close square bracket" and "backslash d".

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Implement programmability objects and query data.
- Task bullet: Use built-in functions, including the SQL Server 2025 regular expression table-valued functions.
- What is tested: the exact output columns of `REGEXP_MATCHES`, how `end_position` is counted, what `substring_matches` contains, what an empty capture looks like, and how `CROSS APPLY` treats an outer row with no matches.

## 2. Scenario to read aloud

**Piece 1, the story.** "A laboratory-supplies company stores chemical formulas in a SQL Server 2025 database. They want to break every formula into its element symbols and atom counts using the new table-valued function REGEXP underscore MATCHES."

**Piece 2, the database and the table.** "The script creates a database called ChemStock, sets its compatibility level to 170, and creates a schema called Chem. There is one table, Chem dot Reagents, with two columns. ReagentID, an integer, the primary key. And Formula, a varchar of thirty characters, not null."

**Piece 3, the data.** "Four rows are inserted.

- Reagent 1, formula capital H, 2, capital S, capital O, 4. Sulfuric acid, H2SO4.
- Reagent 2, formula capital N, lowercase a, capital H, capital C, capital O, 3. Sodium bicarbonate, NaHCO3.
- Reagent 3, formula capital F, lowercase e, 2, capital O, 3. Iron oxide, Fe2O3.
- Reagent 4, formula lowercase a, lowercase q. Just the two lowercase letters a q."

**Piece 4, the pattern.** "The pattern has two capture groups. Group one is: open paren, a character class of one uppercase letter A to Z, followed by a character class of one lowercase letter a to z with a question mark, meaning optional, close paren. So group one is one capital letter optionally followed by one lowercase letter. Group two is: open paren, backslash d, star, close paren. So group two is zero or more digits. The whole pattern, in words, is: an element symbol followed by an optional atom count."

**Piece 5, the query.** "The query selects from Chem dot Reagents aliased r, CROSS APPLY REGEXP underscore MATCHES of r dot Formula and that pattern, aliased m. It returns six columns: r dot ReagentID, m dot match underscore id, m dot start underscore position, m dot end underscore position, m dot match underscore value, and m dot substring underscore matches. It orders by ReagentID and then match_id."

**Piece 6, what is asked.** "You have to predict the exact returned table. How many rows, and for every row, the six column values, including the full JSON in substring_matches."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE ChemStock;
GO
ALTER DATABASE ChemStock SET COMPATIBILITY_LEVEL = 170;
GO
USE ChemStock;
GO
CREATE SCHEMA Chem;
GO
CREATE TABLE Chem.Reagents
(
    ReagentID int         NOT NULL PRIMARY KEY,
    Formula   varchar(30) NOT NULL
);
GO
INSERT INTO Chem.Reagents (ReagentID, Formula) VALUES
  (1, 'H2SO4'),
  (2, 'NaHCO3'),
  (3, 'Fe2O3'),
  (4, 'aq');
GO
SELECT
    r.ReagentID,
    m.match_id,
    m.start_position,
    m.end_position,
    m.match_value,
    m.substring_matches
FROM Chem.Reagents AS r
CROSS APPLY REGEXP_MATCHES(r.Formula, '([A-Z][a-z]?)(\d*)') AS m
ORDER BY r.ReagentID, m.match_id;
```

## 4. The question (ask exactly this)

"Predict the exact returned table. Let's go one part at a time.

Part 1: how many rows does the query return in total?

Part 2: for reagent 1, H2SO4, tell me each match: its match_id, start_position, end_position and match_value.

Part 3: the same for reagent 2, NaHCO3.

Part 4: the same for reagent 3, Fe2O3.

Part 5: what comes back for reagent 4, aq?

Part 6: describe the substring_matches JSON for the match S of reagent 1, and for the match Na of reagent 2."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Nine rows.** Engine output on SQL Server 2025 RTM:

| ReagentID | match_id | start_position | end_position | match_value | substring_matches |
|---|---|---|---|---|---|
| 1 | 1 | 1 | 2 | H2 | `[{"value":"H","start":1,"length":1},{"value":"2","start":2,"length":1}]` |
| 1 | 2 | 3 | 3 | S | `[{"value":"S","start":3,"length":1},{"value":"","start":4,"length":0}]` |
| 1 | 3 | 4 | 5 | O4 | `[{"value":"O","start":4,"length":1},{"value":"4","start":5,"length":1}]` |
| 2 | 1 | 1 | 2 | Na | `[{"value":"Na","start":1,"length":2},{"value":"","start":3,"length":0}]` |
| 2 | 2 | 3 | 3 | H | `[{"value":"H","start":3,"length":1},{"value":"","start":4,"length":0}]` |
| 2 | 3 | 4 | 4 | C | `[{"value":"C","start":4,"length":1},{"value":"","start":5,"length":0}]` |
| 2 | 4 | 5 | 6 | O3 | `[{"value":"O","start":5,"length":1},{"value":"3","start":6,"length":1}]` |
| 3 | 1 | 1 | 3 | Fe2 | `[{"value":"Fe","start":1,"length":2},{"value":"2","start":3,"length":1}]` |
| 3 | 2 | 4 | 5 | O3 | `[{"value":"O","start":4,"length":1},{"value":"3","start":5,"length":1}]` |

Part answers:

- Part 1: 9 rows. Three for reagent 1, four for reagent 2, two for reagent 3, zero for reagent 4.
- Part 2: H2 at 1 to 2; S at 3 to 3; O4 at 4 to 5.
- Part 3: Na at 1 to 2; H at 3 to 3; C at 4 to 4; O3 at 5 to 6.
- Part 4: Fe2 at 1 to 3; O3 at 4 to 5.
- Part 5: no rows at all. The pattern needs an uppercase letter, aq has none, the function returns an empty table, and CROSS APPLY drops the outer row. With OUTER APPLY the row would survive as 4 followed by five NULLs.
- Part 6: for S of reagent 1, a JSON array with two objects: value S, start 3, length 1; and value empty string, start 4, length 0. For Na of reagent 2: value Na, start 1, length 2; and value empty string, start 3, length 0. The empty capture is an empty string, not a JSON null, and its start still points one past the symbol.

Key facts: `end_position` is the position of the last matched character, inclusive. `match_id` restarts at 1 for every outer row. `substring_matches` holds only the capture groups, one element per group, in order; the whole match is not element zero.

## 6. Hint ladder (one hint per attempt, in order)

**Part 1, the row count**
1. "One row per non-overlapping match, per reagent. Count the element symbols in each formula."
2. "Group one is a capital letter with an optional lowercase letter. Does aq contain a capital letter?"
3. "If a reagent has no match, the function returns an empty table. What does CROSS APPLY do with an outer row whose applied table is empty?"
4. "Three plus four plus two plus zero."

**Part 2, reagent 1**
1. "Start at the first character, H. Group two is zero or more digits. Does it take the 2?"
2. "After H2 comes S. Are there digits after S? Group two can match zero digits, so the match is just S. What are its start and end positions?"
3. "end_position is the position of the last character of the match, inclusive. Not one past it. So H2 ends at 2, not 3."

**Part 3, reagent 2**
1. "Capital N, then lowercase a. The lowercase class is optional but greedy. Does it take the a?"
2. "So the first symbol is Na, two characters, positions 1 to 2, and there are no digits after it. Then H, C, and O3."

**Part 4, reagent 3**
1. "Capital F, lowercase e, then a digit. One match, three characters. Then O3."

**Part 5, reagent 4**
1. "Is there an uppercase letter anywhere in aq?"
2. "Zero matches means an empty table. Does CROSS APPLY keep or drop the outer row when the inner table is empty?"

**Part 6, the JSON**
1. "The array has one element per capture group. How many groups are in the pattern?"
2. "Each element has three keys: value, start and length. Where does the full match go? Not into the array; it has its own column."
3. "For S, group two matched zero digits at position 4. Is a participating group that captured nothing a null, or an empty string with length zero?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "Ten rows, reagent 4 gives one row of NULLs" | Confuses CROSS APPLY with OUTER APPLY | "Which APPLY keeps outer rows with no inner rows? Is that the one in the query?" |
| "H2 is at positions 1 to 3" | Thinks end_position is one past the match, like REGEXP_INSTR with return_option 1 | "How many characters does H2 have, and where does the second one sit?" |
| "NaHCO3 splits as N, a, H, ..." | Forgets the optional lowercase class is greedy and consumes the a | "The lowercase class has a question mark. Given the chance, does the regex engine take the lowercase letter or leave it?" |
| "match_id keeps counting across reagents, 1 to 9" | Thinks match_id is global | "The function is invoked once per outer row. Where does the counter start on each invocation?" |
| "substring_matches starts with the full match as element zero" | Assumes group 0 appears in the array | "The full match already has its own column. What does the array hold, and how many elements?" |
| "For S, the second element is null" | Confuses an empty capture with a non-participating group | "Group two is star quantified. Did it participate and match zero characters, or did it not participate at all? Those produce different JSON." |
| "Reagent 4 raises an error" | Thinks no match is an error | "Regex functions do not error on no match. What table does a TVF return when nothing matches?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the function's shape:

- `REGEXP_MATCHES(string, pattern [, flags])` is one of the two regex table-valued functions in SQL Server 2025. The other is `REGEXP_SPLIT_TO_TABLE`. It returns one row per non-overlapping match, and always these columns: `match_id`, a bigint sequence number starting at 1 per invocation; `start_position`, int, 1-based; `end_position`, int, 1-based and inclusive, the last matched character; `match_value`, same type as the input; and `substring_matches`, a JSON array describing the capture groups.
- There is no start or occurrence argument. Unlike `REGEXP_INSTR` and `REGEXP_SUBSTR`, the TVF always scans the whole string and returns all matches. Flags, `i`, `m`, `s`, `c`, are its only option.
- Microsoft Learn states the TVFs require compatibility level 170, or the `ALLOW_BUILTIN_TVF_IN_ALL_COMPAT_LEVELS` database-scoped configuration, a preview option not on the RTM build. For the exam, answer with the documented 170 requirement. Engine-verified: at level 160 the TVF raises "Invalid object name REGEXP_MATCHES".

Then the pattern walk:

- `H2SO4`: H plus 2 at 1 to 2, S alone at 3, O plus 4 at 4 to 5. Three matches.
- `NaHCO3`: Na at 1 to 2 because the greedy optional lowercase class consumes the a, then H at 3, C at 4, O3 at 5 to 6. Four matches.
- `Fe2O3`: Fe2 at 1 to 3, O3 at 4 to 5. Two matches.
- `aq`: no uppercase letter, no match, empty table, CROSS APPLY eliminates the outer row. Reagent 4 silently disappears. OUTER APPLY would keep it with five NULLs.

Then the JSON traps:

- Only the capture groups appear in the array, one element per parenthesized group, in order. The full match is not element zero; it lives in `match_value`. Only when the pattern has no groups at all does the engine put the whole match into `substring_matches` as its single element.
- A group that participates but captures nothing returns an empty string with length 0 and a start that points to where the empty capture occurred. For S of H2SO4 that is start 4, one past the symbol.
- A group that does not participate at all returns JSON nulls. Rewriting the count group as `(\d+)?` instead of `(\d*)` changes exactly that: for S, the second element becomes value null, start null, length null. Empty capture and absent capture are different, and the JSON records the difference.
- `match_id` restarts at 1 for each CROSS APPLY invocation, so it is unique only per outer row. The ORDER BY on ReagentID then match_id makes the output deterministic.
- `end_position` is the last matched character. `REGEXP_INSTR` with return_option 1 would return the position after the match. Two adjacent functions, two conventions.

Memory hook: "One row per match, groups only in the JSON, end is inclusive, no match means no row, and CROSS APPLY drops what OUTER APPLY keeps."

## 9. Follow-up oral questions (optional)

1. "How would you keep reagent 4 in the output?" (Replace CROSS APPLY with OUTER APPLY. The row comes back as 4 with NULL in the five function columns.)
2. "If the pattern were rewritten with the count group as open paren backslash d plus close paren question mark, what changes for the match S?" (The second JSON element becomes nulls for value, start and length, because the group did not participate at all.)
3. "How would you pull the element symbol out of the JSON into a relational column?" (JSON_VALUE of substring_matches with the path dollar, open bracket 0, close bracket, dot value.)

## 10. References

- REGEXP_MATCHES: https://learn.microsoft.com/en-us/sql/t-sql/functions/regexp-matches-transact-sql
- Regular expressions in SQL Server 2025, overview: https://learn.microsoft.com/en-us/sql/relational-databases/regular-expressions/regular-expressions-overview
- APPLY operator, CROSS APPLY and OUTER APPLY: https://learn.microsoft.com/en-us/sql/t-sql/queries/from-transact-sql
- JSON_VALUE: https://learn.microsoft.com/en-us/sql/t-sql/functions/json-value-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
