# Instructor-Examiner guide — REGEXP_REPLACE 1

Companion to [REGEXP_REPLACE_1.md](REGEXP_REPLACE_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a five-part prediction question, R1 to R5, each returning one string. Take them one at a time. The learner must say the whole output string, in words; accept "open bracket QUOTE close bracket" for the literal `[QUOTE]` and "X X X X" for the masking token. Be ready to repeat the snippet text for a query as many times as needed, because the learner must count characters for R5.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked. For patterns, say "double quote", "dot star", "dot star question mark", "backslash 1", "backslash 2".

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Write advanced T-SQL code.
- Task bullet: Use built-in functions, including the SQL Server 2025 regular expression functions.
- What is tested: the argument order of `REGEXP_REPLACE`, its default occurrence of 0 meaning all, greedy versus lazy quantifiers, back-references in the replacement, and what the `start` argument does and does not do to the output.

## 2. Scenario to read aloud

**Piece 1, the story.** "A newspaper is digitizing its archive into SQL Server 2025 and builds a redaction pipeline with REGEXP underscore REPLACE. Before running it over millions of articles, the archivist tests it on three short snippets."

**Piece 2, the database and the table.** "The script creates a database called NewsArchive, sets compatibility level 170, and creates a schema called Press. One table, Press dot Snippets, with two columns. SnippetID, an integer, the primary key. And Body, an nvarchar of two hundred characters, not null."

**Piece 3, the data.** "Three rows.

- Snippet 1: He said, then in double quotes, off the record, then the words: and later, then in double quotes, print it all, then the word twice and a full stop. So the sentence is: He said quote off the record quote and later quote print it all quote twice period.
- Snippet 2: the byline: By Maria Torres and Omar Haddad. Capital B y, capital M aria, capital T orres, lowercase and, capital O mar, capital H addad.
- Snippet 3: Call 555 hyphen 0117 or 555 hyphen 0184 or 555 hyphen 0199 now, full stop. Three phone numbers, each three digits, hyphen, four digits."

**Piece 4, R1 and R2, on snippet 1.** "R1 replaces, in the Body of snippet 1, the pattern double quote, dot star, double quote, with the literal text open square bracket QUOTE close square bracket. Just three arguments: string, pattern, replacement. R2 is identical except the pattern is double quote, dot star question mark, double quote. The question mark after the star makes it lazy."

**Piece 5, R3, on snippet 2.** "R3 replaces, in snippet 2, the pattern: open paren, one capital letter, one or more lowercase letters, close paren, a space, open paren, one capital letter, one or more lowercase letters, close paren. Two capture groups separated by a space. The replacement is backslash 2, comma, space, backslash 1. Again only three arguments."

**Piece 6, R4 and R5, on snippet 3.** "R4 replaces, in snippet 3, the pattern: open paren, three digits, close paren, hyphen, open paren, four digits, close paren, with the replacement backslash 1, hyphen, capital X X X X. Then two more arguments: start 1, occurrence 2. R5 is the same call but with start 13 and occurrence 1."

**Piece 7, what is asked.** "Predict the exact string each of the five queries returns."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE NewsArchive;
GO
ALTER DATABASE NewsArchive SET COMPATIBILITY_LEVEL = 170;
GO
USE NewsArchive;
GO
CREATE SCHEMA Press;
GO
CREATE TABLE Press.Snippets
(
    SnippetID int           NOT NULL PRIMARY KEY,
    Body      nvarchar(200) NOT NULL
);
GO
INSERT INTO Press.Snippets (SnippetID, Body) VALUES
    (1, N'He said "off the record" and later "print it all" twice.'),
    (2, N'By Maria Torres and Omar Haddad'),
    (3, N'Call 555-0117 or 555-0184 or 555-0199 now.');
GO
-- R1: redact quoted speech (greedy)
SELECT REGEXP_REPLACE(Body, N'".*"', N'[QUOTE]') AS R1
FROM Press.Snippets WHERE SnippetID = 1;

-- R2: redact quoted speech (lazy)
SELECT REGEXP_REPLACE(Body, N'".*?"', N'[QUOTE]') AS R2
FROM Press.Snippets WHERE SnippetID = 1;

-- R3: flip bylines to "Surname, Forename"
SELECT REGEXP_REPLACE(Body, N'([A-Z][a-z]+) ([A-Z][a-z]+)', N'\2, \1') AS R3
FROM Press.Snippets WHERE SnippetID = 2;

-- R4: mask the second phone number only
SELECT REGEXP_REPLACE(Body, N'([0-9]{3})-([0-9]{4})', N'\1-XXXX', 1, 2) AS R4
FROM Press.Snippets WHERE SnippetID = 3;

-- R5: mask the first phone number found from character 13 onward
SELECT REGEXP_REPLACE(Body, N'([0-9]{3})-([0-9]{4})', N'\1-XXXX', 13, 1) AS R5
FROM Press.Snippets WHERE SnippetID = 3;
```

## 4. The question (ask exactly this)

"Predict the exact string each query returns. One at a time. R1, the greedy quote redaction on snippet 1: what string comes back?"

Then R2, R3, R4 and R5 in order, each time: "What exact string does it return?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

```text
R1: He said [QUOTE] twice.
R2: He said [QUOTE] and later [QUOTE] twice.
R3: Maria, By Torres and Haddad, Omar
R4: Call 555-0117 or 555-XXXX or 555-0199 now.
R5: Call 555-0117 or 555-XXXX or 555-0199 now.
```

All five engine-verified on SQL Server 2025 RTM.

- R1: greedy `.*` runs from the first double quote to the last double quote, one single match covering both quotations and the text between. One replacement.
- R2: lazy `.*?` stops at the nearest closing quote. Default occurrence 0 replaces all, so both quotations are replaced.
- R3: the first match is "By Maria", not "Maria Torres", because B plus y satisfies capital plus one-or-more lowercase. After that match is consumed, "Torres and" fails because "and" is lowercase. Then "Omar Haddad" matches. Both replaced, default occurrence 0.
- R4: occurrence 2 replaces only the second phone number. `\1` keeps the 555.
- R5: character 13 is the final 7 inside the first phone number. A match must begin at or after start, so 555-0117, which begins at character 6, is invisible. The first occurrence found is 555-0184. The prefix before character 13 is preserved verbatim. So R5 equals R4.

## 6. Hint ladder (one hint per attempt, in order)

**R1**
1. "Dot star is greedy. When it starts at the first double quote, how far does it run before it looks for a closing quote?"
2. "It runs to the end of the string and backtracks only as far as needed. Which double quote is the last one in the sentence?"
3. "So there is one match from the first quote to the last quote, including the words: and later. How much of the sentence survives?"

**R2**
1. "The question mark after the star makes it lazy. Where does the first match now stop?"
2. "The call has no occurrence argument. What is the default occurrence for REGEXP_REPLACE: one, or all?"
3. "With all occurrences, is the second quotation also replaced?"

**R3**
1. "Do not assume the first match is Maria Torres. Start at the first character of the string. Is capital B followed by lowercase y a valid group one?"
2. "Plus means one or more. One lowercase letter is enough. So group one is By and group two is Maria. Write the replacement: group two, comma, space, group one."
3. "Now scanning resumes after Maria. Torres is capitalized, but what is the word right after it? Does it start with a capital?"
4. "So Torres is left alone and stays where it is. Then Omar Haddad matches. Assemble the whole string."

**R4**
1. "The fourth argument is start, the fifth is occurrence. Which occurrence is targeted?"
2. "Only that one phone number is masked. What does backslash 1 keep?"

**R5**
1. "Count the characters of snippet 3 from the first C. Which character is number 13?"
2. "Character 13 sits inside the first phone number. A match must begin at or after start. Can the first phone number, which begins at character 6, be occurrence 1?"
3. "Does the start argument cut off the text before it, or is the prefix kept in the output?"
4. "So the first phone number found from character 13 is the second one in the sentence. Compare with R4."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| R1: "He said [QUOTE] and later [QUOTE] twice." | Thinks greedy `.*` stops at the nearest quote | "Greedy means longest. From the first quote, how far can dot star reach and still find a closing quote after it?" |
| R2: "He said [QUOTE] and later quote print it all quote twice." | Believes default occurrence is 1, as in REGEXP_SUBSTR | "Which function defaults occurrence to 1, and which defaults it to 0 meaning all? This one is REGEXP_REPLACE." |
| R3: "By Torres, Maria and Haddad, Omar" | Assumes the first name pair is Maria Torres | "Start matching at the very first character. Does By fit the shape capital plus one or more lowercase?" |
| R3: "Maria, By Torres and Haddad, Omar" but with Torres also flipped somehow | Forgets that a consumed match cannot be reused and that "and" is lowercase | "After By Maria is consumed, what word follows Torres, and does it start with a capital?" |
| R4: masks the first number | Reads the fourth argument as occurrence | "Name the arguments in order: string, pattern, replacement, then what, then what?" |
| R5: "555-XXXX or 555-0184 or 555-0199 now." with the prefix cut | Thinks start trims the output | "Does REGEXP_REPLACE ever return less than the full original string? What happens to the characters before start?" |
| R5: masks the first phone number | Thinks a match that straddles the start position counts | "Where must a match begin, relative to start? Where does the first phone number begin?" |

## 8. Teaching notes (after the answer is complete or revealed)

Signature first, because argument order is the whole trap:

- `REGEXP_REPLACE(string, pattern [, replacement [, start [, occurrence [, flags]]]])`. Defaults: replacement empty string, so matches are deleted; start 1; occurrence 0, which means replace all; flags `c`. Contrast with `REGEXP_SUBSTR`, whose occurrence defaults to 1. Memorize the asymmetry.

Then each query:

- **R1, greedy.** `".*"` finds the leftmost match starting at the first quote. `.*` consumes to the end and backtracks just enough for a final quote to succeed, which is the last quote in the string. One match from the first quote to the last, including the unquoted text in between. Result: He said [QUOTE] twice.
- **R2, lazy plus occurrence 0.** `.*?` extends one character at a time until the next quote closes it. First match: off the record in quotes. Default occurrence 0 continues scanning and also replaces print it all. Result: He said [QUOTE] and later [QUOTE] twice.
- **R3, capture groups.** The pattern is capital, one or more lowercase, space, capital, one or more lowercase. At position 1, By is a valid group one and Maria a valid group two, so the first match is By Maria, rewritten as Maria, By. Scanning resumes at Torres; the next word is lowercase and, so no match starts at Torres. Omar Haddad matches and becomes Haddad, Omar. Result: Maria, By Torres and Haddad, Omar. The intuitive answer is wrong twice: By is a perfectly valid forename to this pattern, and once By Maria is consumed, Torres has no capitalized neighbor.
- **R4, occurrence 2.** Matches are counted left to right; only the second is replaced. `\1` reinserts the captured 555. Result: Call 555-0117 or 555-XXXX or 555-0199 now.
- **R5, start 13.** Counting: C 1, a 2, l 3, l 4, space 5, 5 6, 5 7, 5 8, hyphen 9, 0 10, 1 11, 1 12, 7 13. Character 13 is inside the first phone number. Two engine-verified behaviors: the characters before start are not discarded, the output keeps its full prefix; and a match must begin at or after start, so 555-0117 can never be occurrence 1 of this search. The first occurrence found is 555-0184. R5 equals R4.

Engine-verified side notes:

- A back-reference to a nonexistent group is silently dropped, not an error. `\1-\3end` on a two-group pattern returns 555-end.
- Microsoft Learn documents `&` in the replacement as "insert the whole match", but the RTM build returns the literal `<&>`. To reuse the whole match, wrap the whole pattern in parentheses and use `\1`. Do not rely on `&`.
- If the pattern matches nothing, `REGEXP_REPLACE` returns the original string, unlike `REGEXP_SUBSTR`, which returns NULL.
- `[0-9]{3}` and `\d{3}` are equivalent; R4 rewritten with `\d` returns the identical string.

Memory hook: "Replace defaults to all, greedy grabs the longest, lazy grabs the shortest, and start moves the search window but never trims the output."

## 9. Follow-up oral questions (optional)

1. "What is the default occurrence for REGEXP_REPLACE, and what is it for REGEXP_SUBSTR?" (0, meaning all, for REGEXP_REPLACE. 1 for REGEXP_SUBSTR.)
2. "If R4's pattern matched nothing in the string, what would the function return?" (The original string unchanged. Never NULL.)
3. "Besides the lazy quantifier, name another pattern that would redact each quotation separately." (A negated class: double quote, open bracket caret double quote close bracket star, double quote.)

## 10. References

- REGEXP_REPLACE: https://learn.microsoft.com/en-us/sql/t-sql/functions/regexp-replace-transact-sql
- REGEXP_SUBSTR, for the occurrence default contrast: https://learn.microsoft.com/en-us/sql/t-sql/functions/regexp-substr-transact-sql
- Regular expressions in SQL Server 2025, overview: https://learn.microsoft.com/en-us/sql/relational-databases/regular-expressions/regular-expressions-overview
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
