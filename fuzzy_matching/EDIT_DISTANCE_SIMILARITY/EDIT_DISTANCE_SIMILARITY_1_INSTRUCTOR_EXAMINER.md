# Instructor-Examiner guide — EDIT_DISTANCE_SIMILARITY 1

Companion to [EDIT_DISTANCE_SIMILARITY_1.md](EDIT_DISTANCE_SIMILARITY_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a predict-the-output question. Take it pair by pair: for each of the eight candidate pairs, ask for the score and whether the pair survives the threshold of 75. Then ask for the final row order. Spell product names character by character when asked, including digits and spaces; the exact characters matter. Do not hint that rows 3, 5, 6 and 8 are the traps.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Write advanced T-SQL code.
- Task bullet: Fuzzy string matching.
- What is tested: the `EDIT_DISTANCE_SIMILARITY` formula, normalisation by the longer string, rounding, the cost of a transposition, collation independence, and how a NULL score behaves in a WHERE clause.

## 2. Scenario to read aloud

**Piece 1, the story.** "ToolBarn is a hardware store chain. Every week it receives product feeds from suppliers and must deduplicate them against its own catalog. Product names arrive with small differences: a changed model number, swapped digits, inconsistent capitalization. So the data team uses the new SQL Server 2025 fuzzy matching to score every candidate pair, and keeps only pairs with a similarity of seventy five or more for human review."

**Piece 2, the database setup.** "The script creates a database called ToolBarn with the collation Latin1 underscore General underscore CI underscore AS, which is case insensitive. It sets compatibility level 170. It turns on the database scoped configuration PREVIEW underscore FEATURES. And it creates a schema called Cat."

**Piece 3, the table.** "One table, Cat dot FeedMatches. Three columns. MatchID, an integer, the primary key. FeedName, a varchar of sixty, nullable. And CatalogName, a varchar of sixty, nullable."

**Piece 4, the data, pairs 1 to 4.** "Eight pairs are inserted, feed name first, then catalog name. Pair 1: Claw Hammer and Claw Hammer, identical. Pair 2: Rasp space 200 m m, and Rasp space 250 m m. So the second digit changes from zero to five. Pair 3: Drill space 750 W, and Drill space 705 W. So the five and the zero are swapped. Pair 4: Adjustable Wrench space 25, and Adjustable Wrench space 26."

**Piece 5, the data, pairs 5 to 8.** "Pair 5: Rivets space 8, that is R I V E T S, space, digit 8; and Rivet space 6, R I V E T, space, digit 6. Pair 6: STAPLE GUN, all in capitals, and Staple Gun, with only the S and the G capitalised. Pair 7: Axe, A X E, and Bit, B I T. Pair 8: Pry Bar, and NULL."

**Piece 6, the query.** "The final SELECT returns MatchID, FeedName, CatalogName, and a column called Score, which is EDIT underscore DISTANCE underscore SIMILARITY open paren FeedName comma CatalogName close paren. The WHERE clause keeps rows where that same function is greater than or equal to 75. The ORDER BY is Score descending, then MatchID."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE ToolBarn COLLATE Latin1_General_CI_AS;
GO
ALTER DATABASE ToolBarn SET COMPATIBILITY_LEVEL = 170;
GO
USE ToolBarn;
GO
ALTER DATABASE SCOPED CONFIGURATION SET PREVIEW_FEATURES = ON;
GO
CREATE SCHEMA Cat;
GO
CREATE TABLE Cat.FeedMatches
(
    MatchID     int         NOT NULL PRIMARY KEY,
    FeedName    varchar(60) NULL,
    CatalogName varchar(60) NULL
);
GO
INSERT INTO Cat.FeedMatches (MatchID, FeedName, CatalogName) VALUES
  (1, 'Claw Hammer',          'Claw Hammer'),
  (2, 'Rasp 200mm',           'Rasp 250mm'),
  (3, 'Drill 750W',           'Drill 705W'),
  (4, 'Adjustable Wrench 25', 'Adjustable Wrench 26'),
  (5, 'Rivets 8',             'Rivet 6'),
  (6, 'STAPLE GUN',           'Staple Gun'),
  (7, 'Axe',                  'Bit'),
  (8, 'Pry Bar',              NULL);
GO
SELECT MatchID,
       FeedName,
       CatalogName,
       EDIT_DISTANCE_SIMILARITY(FeedName, CatalogName) AS Score
FROM Cat.FeedMatches
WHERE EDIT_DISTANCE_SIMILARITY(FeedName, CatalogName) >= 75
ORDER BY Score DESC, MatchID;
```

## 4. The question (ask exactly this)

"Predict the exact result set of the final SELECT: which of the eight candidate pairs survive the threshold, each pair's exact Score, and the row order. Let's go one pair at a time, starting with pair 1, Claw Hammer and Claw Hammer. What is the score, and does it survive?"

Then pairs 2 through 8 in order. After all eight: "Now tell me the returned rows in order, by MatchID."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

Five of the eight rows are returned, engine-verified on SQL Server 2025 RTM:

| MatchID | FeedName | CatalogName | Score |
|---|---|---|---|
| 1 | Claw Hammer | Claw Hammer | 100 |
| 4 | Adjustable Wrench 25 | Adjustable Wrench 26 | 95 |
| 2 | Rasp 200mm | Rasp 250mm | 90 |
| 3 | Drill 750W | Drill 705W | 80 |
| 5 | Rivets 8 | Rivet 6 | 75 |

Scores per pair, using similarity equals open paren 1 minus distance divided by the longer length close paren times 100, rounded to nearest:

- Pair 1: distance 0, lengths 11 and 11, score 100. Survives.
- Pair 2: one substitution, lengths 10 and 10, score 90. Survives.
- Pair 3: the swap 50 versus 05 costs two substitutions, lengths 10 and 10, score 80. Survives. Not 90.
- Pair 4: one substitution, lengths 20 and 20, score 95. Survives.
- Pair 5: delete the s, substitute 8 to 6, distance 2, lengths 8 and 7, normalised by 8, score 75. Exactly on the inclusive threshold. Survives.
- Pair 6: collation would call them equal, but the function is exact: S, the space and G match, seven characters differ by case, distance 7 over 10, score 30. Filtered out.
- Pair 7: all three characters differ, score 0. Filtered out.
- Pair 8: NULL input, score NULL, the predicate NULL greater than or equal to 75 is UNKNOWN, row silently removed. No error.

Order: Score descending gives 100, 95, 90, 80, 75, so MatchIDs 1, 4, 2, 3, 5.

## 6. Hint ladder (one hint per attempt, in order)

**Pair 1, Claw Hammer**
1. "Identical strings. What is the edit distance, and what score does a distance of zero give?"

**Pair 2, Rasp 200mm**
1. "Line the characters up. How many differ? And how long are the strings?"
2. "One edit over ten characters. Compute one minus one tenth, times one hundred."

**Pair 3, Drill 750W**
1. "The five and the zero are swapped. Which operations does the function count: insertions, deletions, substitutions. Is a swap one of them?"
2. "Transpositions are not supported. How many substitutions repair a swapped pair?"
3. "Two edits over ten characters. What score does that give?"

**Pair 4, Adjustable Wrench 25**
1. "Only the last digit differs. Count the characters, including spaces."
2. "One edit over twenty characters."

**Pair 5, Rivets 8**
1. "Two edits are needed. Name them: one letter to remove, one digit to change."
2. "The strings have different lengths, eight and seven. Which length does the formula divide by, the shorter or the longer?"
3. "Two over eight is a quarter. Now, is the threshold strictly greater than 75, or greater than or equal?"

**Pair 6, STAPLE GUN**
1. "The database collation is case insensitive. Does the fuzzy function use the collation, or does it compare characters exactly?"
2. "Compare character by character. Which characters are the same in both strings? Remember the space."
3. "Three match, seven differ, over ten characters. Does that pass 75?"

**Pair 7, Axe and Bit**
1. "Do any characters match at all? What is the distance, and what is the score?"

**Pair 8, Pry Bar and NULL**
1. "What does the function return when one input is NULL?"
2. "Now put that NULL into the WHERE clause. What is NULL greater than or equal to 75, and what does a WHERE do with that?"
3. "UNKNOWN is not true. Does the row appear, or is there an error?"

**Row order**
1. "Sort the surviving scores from high to low. Then map each score back to its MatchID."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "Pair 3 scores 90, a swap is one edit" | Assumes Damerau-Levenshtein transpositions | "Which three operations does EDIT_DISTANCE count? Check the note on transpositions in the docs." |
| "Pair 5 scores 71, it is filtered out" | Divides by the shorter length | "Which of the two lengths does the normalisation use, the greater or the smaller?" |
| "Pair 5 is filtered out because 75 is not above 75" | Reads the threshold as strict | "Read the operator in the WHERE clause again. Is it greater than, or greater than or equal?" |
| "Pair 6 scores 100, they are the same under CI" | Assumes the function respects collation | "Does the function consult the collation at all?" |
| "Pair 6 scores 20 or 40" | Miscounts matching characters | "Go character by character, including the space. Which ones are identical in both?" |
| "Pair 8 causes an error" | Thinks NULL in a comparison fails | "What does a WHERE clause do with an UNKNOWN result: error, or drop the row?" |
| "Pair 8 returns with a score of NULL" | Forgets the WHERE clause | "The function is also in the WHERE clause. Does NULL greater than or equal to 75 pass?" |
| "Pair 2 scores 89 or 91" | Rounding confusion | "One tenth of one hundred is exact. There is nothing to round here." |

## 8. Teaching notes (after the answer is complete or revealed)

Explain what `EDIT_DISTANCE_SIMILARITY` is:

- New in SQL Server 2025, preview gated on RTM by PREVIEW_FEATURES equals ON; without it, error 195. Returns an int from 0, no match, to 100, full match.
- The formula is a normalisation of the Levenshtein edit distance: one minus the distance divided by the length of the longer string, times one hundred, rounded to the nearest integer. Engine verified: a raw 66.67 returns 67, a raw 83.33 returns 83, an exact 87.5 rounds up to 88.
- Like EDIT_DISTANCE it counts insertions, deletions and substitutions only. Transpositions are not supported and cost two. Comparison is exact per character; the collation never applies. Any NULL input yields NULL.

Three ways a "duplicate" escapes a similarity threshold:

1. **Case differences are real edits.** STAPLE GUN and Staple Gun are equal under the CI collation but score 30. Real dedup pipelines normalise case, for example with UPPER, before scoring.
2. **A digit or letter swap costs two edits.** Drill 750W versus Drill 705W scores 80, not 90. One keystroke slip, two edit operations.
3. **NULL scores are UNKNOWN in WHERE.** Rows with a missing name vanish from the review queue instead of failing loudly.

And normalisation always divides by the longer string. Rivets 8 versus Rivet 6 is distance 2 over max of 8 and 7, exactly 75, which passes the inclusive threshold. Dividing by 7 would give 71 and wrongly drop the row.

Equivalent forms: the score can be rebuilt from EDIT_DISTANCE with CAST of ROUND of open paren 1.0 minus 1.0 times distance divided by GREATEST of the two LENs close paren times 100 as int. And a CROSS APPLY computing the score once, then filtering on it, returns the same rows.

Memory hook: "Divide by the longer string. A swap costs two. Case counts. NULL is quietly dropped by WHERE."

## 9. Follow-up oral questions (optional)

1. "How could the team stop pair 6 from being lost?" (Normalise case before scoring, for example UPPER on both names.)
2. "If pair 5 were Rivets 8 against Rivet 8, what would the score be?" (Distance 1 over 8, so 87.5, rounded to 88.)
3. "Why does pair 8 not raise an error?" (NULL greater than or equal to 75 is UNKNOWN; a WHERE clause just drops rows that are not true.)

## 10. References

- EDIT_DISTANCE_SIMILARITY: https://learn.microsoft.com/en-us/sql/t-sql/functions/edit-distance-similarity-transact-sql
- EDIT_DISTANCE: https://learn.microsoft.com/en-us/sql/t-sql/functions/edit-distance-transact-sql
- ALTER DATABASE SCOPED CONFIGURATION, PREVIEW_FEATURES: https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-database-scoped-configuration-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
