# Instructor-Examiner guide — REGEXP_LIKE 1

Companion to [REGEXP_LIKE_1.md](REGEXP_LIKE_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Read all four options before taking an answer. Take a single letter as the answer. If the learner wants to reason row by row first, let them, but do not confirm individual rows until they commit to a letter.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked. For the regular expression, say "caret" for `^`, "dollar" for `$`, "hyphen" for `-`, and "open square bracket, capital A to capital Z, close square bracket" for `[A-Z]`.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Implement programmability objects and query data.
- Task bullet: Use built-in functions, including the SQL Server 2025 regular expression functions.
- What is tested: the flags argument of `REGEXP_LIKE`, what the `m` flag does to the `^` and `$` anchors, what the `i` flag does to a character class, and that exact quantifiers such as `{2}` and `{4}` never stretch.

## 2. Scenario to read aloud

**Piece 1, the story.** "A national vehicle authority is migrating its registration system to SQL Server 2025. A valid modern number plate has this shape: two uppercase letters, a hyphen, four digits, a hyphen, two uppercase letters. For example, K L, hyphen, 4 8 2 1, hyphen, X T. The old loader only enforced a very loose check constraint, so the imported data contains historical junk."

**Piece 2, the database and the table.** "The script creates a database called FleetReg and immediately sets its compatibility level to 170. Then it creates a schema called Dmv, and one table, Dmv dot Plates. The table has two columns. PlateID, an integer, the primary key. And Plate, an nvarchar of forty characters, not null. Plate has a check constraint called CK underscore Plates underscore HasDigits. The constraint is REGEXP underscore LIKE of Plate against the pattern open square bracket zero to nine close square bracket, curly brace three comma four. In words: the value must contain a run of three or four consecutive digits somewhere. That is all the constraint checks. Every insert that follows succeeds."

**Piece 3, the data.** "Eight rows are inserted. I will read each plate carefully.

- Row 1: capital K L, hyphen, 4 8 2 1, hyphen, capital X T. The perfect plate.
- Row 2: the same plate but all lowercase: lowercase k l, hyphen, 4 8 2 1, hyphen, lowercase x t.
- Row 3: the word OLD in capitals, a space, then K L, hyphen, 4 8 2 1, hyphen, X T. So a valid plate with the prefix OLD and a space in front.
- Row 4: K L, hyphen, 4 8 2, hyphen, X T. Only three digits in the middle.
- Row 5: K L X, hyphen, 4 8 2 1, hyphen, X T. Three letters at the start instead of two.
- Row 6: this is a two-line value. The word scrapped in lowercase, then an embedded line feed character, then on the second line Q F, hyphen, 0 0 4 2, hyphen, Z P. The line feed is stored inside the string.
- Row 7: K L, hyphen, 4 8 2 1, hyphen, X T A. Three letters at the end instead of two.
- Row 8: M N, space, 7 3 0 1, space, P C. Spaces instead of hyphens."

**Piece 4, the query.** "A data-quality engineer then runs a SELECT of PlateID and Plate from Dmv dot Plates, with this WHERE clause: REGEXP underscore LIKE of Plate, against the pattern caret, open bracket capital A to capital Z close bracket, curly brace 2, hyphen, open bracket zero to nine close bracket, curly brace 4, hyphen, open bracket capital A to capital Z close bracket, curly brace 2, dollar. And a third argument, the flags string, which is the two letters i and m, in that order. The query orders by PlateID. So the pattern is: start of line, exactly two capital letters, hyphen, exactly four digits, hyphen, exactly two capital letters, end of line. With flags i m."

**Piece 5, the options.** "The question asks which PlateID values the query returns. Here are the four options.

- Option a: rows 1 and 2.
- Option b: rows 1, 2 and 6.
- Option c: rows 1, 2, 3 and 6.
- Option d: rows 1, 2, 6 and 7."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE FleetReg;
GO
ALTER DATABASE FleetReg SET COMPATIBILITY_LEVEL = 170;
GO
USE FleetReg;
GO
CREATE SCHEMA Dmv;
GO
CREATE TABLE Dmv.Plates
(
    PlateID int          NOT NULL PRIMARY KEY,
    Plate   nvarchar(40) NOT NULL
        CONSTRAINT CK_Plates_HasDigits
        CHECK (REGEXP_LIKE(Plate, N'[0-9]{3,4}'))
);
GO
INSERT INTO Dmv.Plates (PlateID, Plate) VALUES
    (1, N'KL-4821-XT'),
    (2, N'kl-4821-xt'),
    (3, N'OLD KL-4821-XT'),
    (4, N'KL-482-XT'),
    (5, N'KLX-4821-XT'),
    (6, N'scrapped' + NCHAR(10) + N'QF-0042-ZP'),
    (7, N'KL-4821-XTA'),
    (8, N'MN 7301 PC');
GO
SELECT PlateID, Plate
FROM Dmv.Plates
WHERE REGEXP_LIKE(Plate, N'^[A-Z]{2}-[0-9]{4}-[A-Z]{2}$', 'im')
ORDER BY PlateID;
```

## 4. The question (ask exactly this)

"Which PlateID values does the query return? Option a: 1 and 2. Option b: 1, 2 and 6. Option c: 1, 2, 3 and 6. Option d: 1, 2, 6 and 7. Which letter?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: b.** The query returns PlateID 1, 2 and 6. Row 6 prints on two lines because the stored value contains a line feed.

Row by row, with the `i` flag turning `[A-Z]` into "any letter":

| PlateID | Value | Match? | Why |
|---|---|---|---|
| 1 | KL-4821-XT | yes | Exact shape |
| 2 | kl-4821-xt | yes | Only because of `i`; the default `c` would reject it |
| 3 | OLD KL-4821-XT | no | `^` anchors at position 1; O, L consume `[A-Z]{2}`, then the pattern wants a hyphen and finds D. No line feed, so `m` adds no extra anchor |
| 4 | KL-482-XT | no | `[0-9]{4}` needs four digits |
| 5 | KLX-4821-XT | no | After K L the pattern wants a hyphen and finds X |
| 6 | scrapped + LF + QF-0042-ZP | yes | With `m`, `^` also matches right after the line feed, so the second line matches on its own. Without `m` this row would not match |
| 7 | KL-4821-XTA | no | `{2}` is exact and `$` demands end of line; the value has eleven characters, the pattern accepts only ten |
| 8 | MN 7301 PC | no | Spaces, not hyphens |

Why each wrong option is wrong:

- **a (1, 2)** ignores the `m` flag, or wrongly believes that `m` in `'im'` cancels `i` under the "last flag wins" rule. That rule only arbitrates contradictory flags, `i` versus `c`. `i` and `m` do not contradict, so both are active. Engine-verified: flags `'i'` alone return 1 and 2; flags `'im'` return 1, 2 and 6.
- **c (1, 2, 3, 6)** believes `m` relaxes the anchors so that any value containing a valid plate matches. It does not. `m` only adds anchor points immediately before and after line feed characters. Row 3 has no line feed, so its only anchors are the true start and end, and the leading "OLD " kills the match.
- **d (1, 2, 6, 7)** reads `[A-Z]{2}` as "two or more letters", which would be `{2,}`, or thinks `$` tolerates trailing characters. `{2}` is exact and `$` demands end of string or, with `m`, end of line. Row 7 has one letter too many.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with the two rows everyone agrees on. Rows 1 and 2 are the same plate in different case. Which flag makes row 2 match?"
2. "Now look at the second flag, m. It changes what the caret and the dollar sign mean. Where in the string can the caret match when m is on?"
3. "The m flag only creates extra anchor points next to line feed characters. Which single row contains a line feed?"
4. "Consider row 3, the one starting with OLD. It has no line feed. So where is its only start-of-string anchor, and what are its first two characters?"
5. "Consider row 7, the one ending in X T A. The pattern says exactly two letters, then end of line. Is a third letter allowed before the end?"
6. "So which rows match: the two plain plates, plus the row whose second line is a perfect plate. Nothing else. Which option lists exactly those?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, rows 1 and 2" | Ignores the `m` flag, or thinks the last flag cancels the earlier one | "You have handled the i flag. What does the m flag add, and is there any row where it can make a difference?" |
| "The m cancels the i because the last flag wins" | Misapplies the last-flag-wins rule to non-contradictory flags | "Last flag wins is about flags that contradict each other. Do i and m contradict each other?" |
| "c, row 3 matches because with m the anchors are relaxed" | Thinks `m` turns anchored patterns into "contains" | "The m flag adds anchor points only in one specific place. Where? Does row 3 have that character?" |
| "d, row 7 matches because it has at least two letters" | Reads `{2}` as "two or more" | "Curly brace 2 with no comma. Is that a minimum or an exact count? And what does the dollar sign require right after?" |
| "Row 6 cannot match because it starts with the word scrapped" | Forgets that `m` makes `^` match after the line feed | "Without the m flag you would be right. What does the m flag do to the caret at the start of the second line?" |
| "Row 5 matches because it contains a valid plate" | Forgets the pattern is anchored with `^` | "The pattern begins with a caret. After the first two letters K L, what does the pattern demand next, and what does row 5 have there?" |
| "Row 4 matches, three digits is close enough" | Ignores the exact quantifier `{4}` | "How many digits does curly brace 4 require? Count the digits in row 4." |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the function first:

- `REGEXP_LIKE(string, pattern [, flags])` returns a Boolean and can be used wherever a predicate is allowed: a WHERE clause, a HAVING clause, a CHECK constraint, a CASE WHEN. It is the only REGEXP function that requires compatibility level 170. That is why the script raises the level right after CREATE DATABASE. The other scalar regex functions, REGEXP_SUBSTR, REGEXP_REPLACE, REGEXP_COUNT and REGEXP_INSTR, work at any compatibility level.

Then the flags:

- `c` is the default and means case-sensitive. `i` means case-insensitive. `m` means multi-line: `^` and `$` match at the beginning and end of every line, around each line feed, as well as at the beginning and end of the whole string. `s` lets the dot match a line feed.
- "Last flag wins" applies only to contradictory flags, `i` versus `c`. `i` and `m` do not contradict, so `'im'` and `'mi'` both activate both. Engine-verified to return identical rows.

Then the two traps:

- **Row 6, the trap that makes b right.** With `m`, the caret matches just after the embedded line feed and the dollar matches at end of string, so the second line, QF-0042-ZP, matches on its own. Without `m` the value is a single nineteen-character string starting with lowercase s, and it does not match. Multi-line mode can silently admit multi-line values whose second line looks valid.
- **Row 3, the trap that makes c wrong.** `m` never weakens anchors. It only adds anchor points at line feeds. A value with no line feed still has to match from its true first character. OLD KL-4821-XT fails at the third character, D, where the pattern wants a hyphen.

Then the exact quantifiers:

- `{2}` and `{4}` are exact. `{2,}` would mean two or more. Row 7 has eleven characters, the pattern accepts exactly ten, so `$` fails. Row 4 has three digits where four are required.

And the anchors lesson from the CHECK constraint:

- The same pattern without anchors and with the default `c` flag, `[A-Z]{2}-[0-9]{4}-[A-Z]{2}`, returns rows 1, 3, 5, 6 and 7: every value that merely contains a plate-shaped substring. That is exactly the mistake the loose CHECK constraint made. In a WHERE or a CHECK, a pattern means "contains" unless you pin it with `^` and `$`. Unanchored patterns accept garbage.
- Engine-verified side note: without `m`, `$` does not match before a trailing line feed. A value with a trailing line feed fails a `$`-anchored case-sensitive check.

Memory hook: "i ignores case, m adds anchors at line feeds, and curly braces without a comma are exact. Validation needs caret and dollar."

## 9. Follow-up oral questions (optional)

1. "If the flags argument were just 'i', without m, which rows would the query return?" (Rows 1 and 2 only. Row 6 needs the m flag.)
2. "Which REGEXP function is the only one that requires compatibility level 170?" (REGEXP_LIKE. REGEXP_SUBSTR, REGEXP_REPLACE, REGEXP_COUNT and REGEXP_INSTR work at any level.)
3. "If the pattern lost its caret and dollar anchors and used the default case-sensitive flag, which rows would come back?" (1, 3, 5, 6 and 7: every value that contains a plate-shaped substring.)

## 10. References

- REGEXP_LIKE, including the flags argument: https://learn.microsoft.com/en-us/sql/t-sql/functions/regexp-like-transact-sql
- Regular expressions in SQL Server 2025, overview and flags: https://learn.microsoft.com/en-us/sql/relational-databases/regular-expressions/regular-expressions-overview
- ALTER DATABASE compatibility level: https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-database-transact-sql-compatibility-level
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
