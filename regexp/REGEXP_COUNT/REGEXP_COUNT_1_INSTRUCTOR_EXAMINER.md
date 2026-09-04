# Instructor-Examiner guide — REGEXP_COUNT 1

Companion to [REGEXP_COUNT_1.md](REGEXP_COUNT_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a predict-the-numbers question: a grid of four rows by five count columns, twenty cells. Take it one column at a time across the four posts. The learner will need exact character positions and string lengths; spell the post bodies slowly, and repeat the positions from piece 3 as often as needed. Be strict: the answer is a number in every cell, and zero is different from NULL.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Write advanced T-SQL code.
- Task bullet: Use built-in functions, including the SQL Server 2025 regular-expression functions.
- What is tested: non-overlapping counting in `REGEXP_COUNT`, the one-based `start` argument, the `i` flag, and how a pattern that can match the empty string yields length plus one.

## 2. Scenario to read aloud

**Piece 1, the story.** "A social-media analytics startup ingests raw post bodies into a SQL Server 2025 database. It computes hashtag metrics with the new regular-expression function REGEXP underscore COUNT."

**Piece 2, the table.** "The database is BuzzBoard, at compatibility level one hundred seventy. There is a schema Social and one table, Social dot Posts, with two columns. PostID, an integer primary key. And Body, varchar two hundred, not null."

**Piece 3, the four posts.** "Four posts are inserted. I will read each body and its length.

- Post 1: Loving the, space, hash SummerSale, space, hash summersale in all lower case, space, vibes exclamation mark, space, hash Sale2026. That is fifty-one characters. The three hash signs are at positions twelve, twenty-four and forty-three.
- Post 2: no tags here, comma, just vibes. Twenty-four characters. No hash sign at all.
- Post 3: the word hahahaha, that is ha four times, space, hash capital L O L, space, hash lower case l o l, space, hash capital L, lower o, capital L. Twenty-three characters. The hash signs are at positions ten, fifteen and twenty.
- Post 4: hash a, space, hash b, hash c, hash hash d. Ten characters. Hash signs at positions one, four, six, eight and nine. Positions eight and nine are two hashes in a row, then the letter d at position ten."

**Piece 4, the query.** "The query selects PostID and five REGEXP COUNT columns, ordered by PostID. The full argument list is string, pattern, optional start, optional flags. The five columns are:

- Tags: REGEXP COUNT of Body with the pattern hash, backslash w, plus. That is a hash followed by one or more word characters.
- TagsFrom13: the same pattern, with a third argument, start equal to thirteen.
- SaleMentions: the pattern is the literal word sale, start one, and the flag i.
- Laughs: the pattern is the literal haha, no other arguments.
- XRuns: the pattern is the letter x followed by a star, meaning zero or more x characters. No other arguments."

**Piece 5, what is asked.** "You will be asked for the exact count in every one of the twenty cells. I can repeat any post body, position or length on request."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE BuzzBoard;
GO
ALTER DATABASE BuzzBoard SET COMPATIBILITY_LEVEL = 170;
GO
USE BuzzBoard;
GO
CREATE SCHEMA Social;
GO
CREATE TABLE Social.Posts
(
    PostID int          NOT NULL PRIMARY KEY,
    Body   varchar(200) NOT NULL
);
GO
INSERT INTO Social.Posts (PostID, Body) VALUES
  (1, 'Loving the #SummerSale #summersale vibes! #Sale2026'),
  (2, 'no tags here, just vibes'),
  (3, 'hahahaha #LOL #lol #LoL'),
  (4, '#a #b#c##d');
GO
SELECT
    PostID,
    REGEXP_COUNT(Body, '#\w+')            AS Tags,
    REGEXP_COUNT(Body, '#\w+', 13)        AS TagsFrom13,
    REGEXP_COUNT(Body, 'sale', 1, 'i')    AS SaleMentions,
    REGEXP_COUNT(Body, 'haha')            AS Laughs,
    REGEXP_COUNT(Body, 'x*')              AS XRuns
FROM Social.Posts
ORDER BY PostID;
```

Position map:

```text
Post 1: 'Loving the #SummerSale #summersale vibes! #Sale2026'   # at 12, 24, 43   LEN = 51
Post 2: 'no tags here, just vibes'                              no #              LEN = 24
Post 3: 'hahahaha #LOL #lol #LoL'                               # at 10, 15, 20   LEN = 23
Post 4: '#a #b#c##d'                                            # at 1, 4, 6, 8, 9   LEN = 10
```

## 4. The question (ask exactly this)

"Predict the exact count in every cell, for all four rows. Let's go one column at a time. First, Tags: give me the count for post 1, post 2, post 3 and post 4."

Then TagsFrom13, then SaleMentions, then Laughs, then XRuns, each for the four posts.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

Output from SQL Server 2025 RTM:

| PostID | Tags | TagsFrom13 | SaleMentions | Laughs | XRuns |
|---|---|---|---|---|---|
| 1 | 3 | 2 | 3 | 0 | 52 |
| 2 | 0 | 0 | 0 | 0 | 25 |
| 3 | 3 | 2 | 0 | 2 | 24 |
| 4 | 4 | 0 | 0 | 0 | 11 |

Reasons:

- Tags: post 1 has three hashtags. Post 2 none. Post 3 three. Post 4: hash a, hash b, hash c, then the hash at position 8 is followed by another hash and fails, but the hash at position 9 followed by d matches, so 4.
- TagsFrom13: scanning starts at character 13; a match must begin at or after 13 and still needs its own hash. Post 1: the hash of SummerSale is at 12, too early, leaving 2. Post 2: 0. Post 3: position 13 is the last L of hash LOL, so that tag is lost, leaving 2. Post 4: the body is 10 characters, start is past the end, result 0, not NULL and not an error.
- SaleMentions: flag i, case-insensitive, and sale may sit inside a longer word. Post 1: SummerSale, summersale, Sale2026, so 3. Others 0.
- Laughs: non-overlapping. In hahahaha the matches are positions 1 to 4 and 5 to 8, so 2. The overlapping one at 3 to 6 is never counted. Others 0.
- XRuns: no body contains an x, so every match is zero-length, and REGEXP_COUNT counts one empty match before each character plus one at the end: LEN plus 1. That is 52, 25, 24, 11.

## 6. Hint ladder (one hint per attempt, in order)

**Tags**
1. "The pattern needs a hash followed by at least one word character. Count the hashes that have a letter or digit right after them."
2. "For post 4, look at the double hash. The first hash of the pair is followed by a hash. Does that one match? Does the second one?"
3. "A doubled hash costs one failed candidate, not the whole tag."

**TagsFrom13**
1. "The third argument is where the scan begins. Any match that starts before that position is invisible. Is start one-based?"
2. "For post 1, where is the first hash? Is twelve at or after thirteen?"
3. "For post 3, what character sits at position thirteen? Can a tag whose hash is behind you still match without its hash?"
4. "For post 4, the body is ten characters and start is thirteen. Is that an error, a NULL, or a number?"

**SaleMentions**
1. "What does the flag i change?"
2. "Does sale have to be a whole word, or can it be inside SummerSale?"
3. "Count the occurrences of the letters s a l e in post 1, in any case."

**Laughs**
1. "Post 3 starts with ha repeated four times. After a match is found, where does the scan resume: at the next character, or after the end of the match?"
2. "Matches do not overlap. Positions one to four are a match. Where does the next attempt begin?"

**XRuns**
1. "There is no letter x anywhere. Can x star still match something?"
2. "x star matches zero x characters, which is an empty match. Where can an empty match occur in a string?"
3. "One empty match before each character, plus one at the very end. Add those up for each body length."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| Post 4 Tags is 3 | Thinks the double hash kills the last tag | "There are two hashes side by side. Try each one separately as a starting point." |
| Post 1 TagsFrom13 is 3 | Thinks the tail of a tag counts, or that start is zero-based | "Where exactly is the first hash, and where does the scan begin?" |
| Post 4 TagsFrom13 is NULL or an error | Expects start past the end to fail | "Does a start beyond the string raise an error, or just find nothing?" |
| Post 1 SaleMentions is 1 or 2 | Forgets the i flag, or requires whole words | "With flag i, does capital S A L E count? And does sale need to stand alone?" |
| Post 3 Laughs is 3 | Counts overlapping matches | "After the first haha is consumed, which character does the scan look at next?" |
| XRuns is 0 everywhere | Assumes no x means no match | "Can a star pattern match an empty string?" |
| XRuns equals the length | Forgets the final empty match at the end | "Is there a match position after the last character?" |
| A NULL somewhere | Confuses no match with NULL input | "The bodies are not NULL. What does REGEXP COUNT return when nothing matches?" |

## 8. Teaching notes (after the answer is complete or revealed)

REGEXP COUNT of string, pattern, start, flags. The rules:

- **Non-overlapping.** Matches are counted left to right, and the scan resumes after the last character of each match. That is why hahahaha gives 2, not 3.
- **start is one-based.** Matches beginning before start are ignored entirely; a partial tail cannot match unless it satisfies the whole pattern, including the hash. Start past the end returns 0. Start less than 1 raises an error.
- **No match gives 0. NULL input gives NULL.** Keep those apart. An empty string with a real pattern gives 0.
- **Empty matches count.** A pattern that can match the empty string, such as x star, backslash d star, or dot star question mark, counts LEN plus 1 zero-length matches on a string with no real match: one per character position plus one at the end. That is why production filters should use x plus, not x star.
- **Flags** are i, m, s and c, with c the default. With contradictory flags the last one wins. The inline option open paren question mark i close paren also works and is equivalent to the i flag.
- **Signature.** Unlike REGEXP INSTR, there is no occurrence, return option, or group argument; the list stops at flags.

Equivalent forms: backslash w is letters, digits and underscore, so the Tags pattern equals hash followed by the bracket class A to Z, a to z, 0 to 9, underscore, plus. TagsFrom13 equals running the plain pattern over SUBSTRING of Body from 13, because REGEXP COUNT returns no positions, so shifting the string changes nothing. SaleMentions equals counting sale on LOWER of Body.

Memory hook: "Count never overlaps, start is one-based, star counts the empties: length plus one."

## 9. Follow-up oral questions (optional)

1. "What would REGEXP COUNT of Body with the pattern x plus return for post 1?" (0. x plus needs at least one x, and there is none.)
2. "What does REGEXP COUNT return when the string argument is NULL?" (NULL, not 0.)
3. "What would Laughs be for post 3 if the pattern were ha instead of haha?" (4. The four non-overlapping ha pairs in hahahaha.)

## 10. References

- REGEXP_COUNT (Transact-SQL): https://learn.microsoft.com/en-us/sql/t-sql/functions/regexp-count-transact-sql
- Regular expressions in SQL Server, overview and flags: https://learn.microsoft.com/en-us/sql/relational-databases/regular-expressions/overview
- REGEXP_INSTR (Transact-SQL), for the signature contrast: https://learn.microsoft.com/en-us/sql/t-sql/functions/regexp-instr-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
