# Instructor-Examiner guide — JARO_WINKLER_DISTANCE 1

Companion to [JARO_WINKLER_DISTANCE_1.md](JARO_WINKLER_DISTANCE_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a predict-the-output question with exact decimals, which is hard to do by voice. Accept the values to four decimal places, but be generous: if the learner gets the structure right, that is the polarity of the function, the position of the NULL row, and the ranking of the pairs, and is within a rounding step on the decimals, count that part as right. The two key parts are the full row order, and which of pairs 2 and 3 is closer. Spell names letter by letter on request.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Write advanced T-SQL code.
- Task bullet: Fuzzy string matching.
- What is tested: that `JARO_WINKLER_DISTANCE` is a distance in zero to one where zero means identical, the Jaro formula and the Winkler prefix bonus, collation independence, NULL propagation, and where NULL sorts under ORDER BY ascending.

## 2. Scenario to read aloud

**Piece 1, the story.** "BankGuard's financial crime unit screens new customer names against a sanctions watchlist during customer onboarding. Exact matching misses trivial misspellings, so the unit scores every candidate pair with the new SQL Server 2025 Jaro Winkler function. That function favours names that agree at the beginning, where honest transcription errors are rare and fraudulent look alikes tend to differ least."

**Piece 2, the database setup.** "The script creates a database called BankGuard with the collation Latin1 underscore General underscore CI underscore AS, which is case insensitive. It sets compatibility level 170, turns on the database scoped configuration PREVIEW underscore FEATURES, and creates a schema called Kyc, K Y C."

**Piece 3, the table.** "One table, Kyc dot ScreeningPairs. Three columns. PairID, an integer, the primary key. CustomerName, a varchar of forty, nullable. And WatchlistName, a varchar of forty, nullable."

**Piece 4, the data.** "Seven pairs are inserted, customer name first, then watchlist name. All in capital letters unless I say otherwise. Pair 1: MARTHA and MARTHA, identical. Pair 2: MARTHA and MARHTA, M A R H T A, so the T and the H are swapped at the end. Pair 3: MARTHA and AMRTHA, A M R T H A, so the M and the A are swapped at the start. Pair 4: DWAYNE, D W A Y N E, and DUANE, D U A N E. Pair 5: DIXON, D I X O N, and DICKSONX, D I C K S O N X. Pair 6: Martha with a capital M and the rest lowercase, and martha all lowercase. Pair 7: MARTHA and NULL."

**Piece 5, the query.** "The SELECT returns PairID, the two names, and a column called JwDist, which is JARO underscore WINKLER underscore DISTANCE open paren CustomerName comma WatchlistName close paren, cast to decimal six comma four. The ORDER BY is JwDist ascending, then PairID."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE BankGuard COLLATE Latin1_General_CI_AS;
GO
ALTER DATABASE BankGuard SET COMPATIBILITY_LEVEL = 170;
GO
USE BankGuard;
GO
ALTER DATABASE SCOPED CONFIGURATION SET PREVIEW_FEATURES = ON;
GO
CREATE SCHEMA Kyc;
GO
CREATE TABLE Kyc.ScreeningPairs
(
    PairID        int         NOT NULL PRIMARY KEY,
    CustomerName  varchar(40) NULL,
    WatchlistName varchar(40) NULL
);
GO
INSERT INTO Kyc.ScreeningPairs (PairID, CustomerName, WatchlistName) VALUES
  (1, 'MARTHA', 'MARTHA'),
  (2, 'MARTHA', 'MARHTA'),
  (3, 'MARTHA', 'AMRTHA'),
  (4, 'DWAYNE', 'DUANE'),
  (5, 'DIXON',  'DICKSONX'),
  (6, 'Martha', 'martha'),
  (7, 'MARTHA', NULL);
GO
SELECT PairID,
       CustomerName,
       WatchlistName,
       CAST(JARO_WINKLER_DISTANCE(CustomerName, WatchlistName)
            AS decimal(6,4))                                AS JwDist
FROM Kyc.ScreeningPairs
ORDER BY JwDist ASC, PairID;
```

## 4. The question (ask exactly this)

"Predict the exact result set: every JwDist value to four decimal places, and the exact row order. Pay attention to which of pairs 2 and 3 scores as the closer match; both differ from MARTHA by the same single adjacent swap. Let's take it in parts. Part one: what does pair 1, MARTHA against MARTHA, return, and what does that tell you about the direction of the scale?"

Then: "Part two: pair 7, MARTHA against NULL. What value, and where does that row sort?"
Then: "Part three: pairs 2 and 3. Which is closer, and what are the two values?"
Then: "Part four: pair 6, Martha against martha."
Then: "Part five: pairs 4 and 5, DWAYNE against DUANE, and DIXON against DICKSONX."
Finally: "Now give me the full row order by PairID."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

Seven rows in this order, engine-verified on SQL Server 2025 RTM:

| PairID | CustomerName | WatchlistName | JwDist |
|---|---|---|---|
| 7 | MARTHA | NULL | NULL |
| 1 | MARTHA | MARTHA | 0.0000 |
| 2 | MARTHA | MARHTA | 0.0389 |
| 3 | MARTHA | AMRTHA | 0.0556 |
| 6 | Martha | martha | 0.1111 |
| 4 | DWAYNE | DUANE | 0.1600 |
| 5 | DIXON | DICKSONX | 0.1867 |

Details:

- The function returns a float distance in zero to one: 0 means identical, 1 means nothing in common. It is one minus the Jaro Winkler similarity.
- Pair 7: NULL in, NULL out. ORDER BY ascending puts NULL first, because SQL Server sorts NULL lower than every value.
- Pair 2: m equals 6, t equals 1, Jaro equals 17 over 18, about 0.9444. Prefix MAR, length 3, bonus 3 times 0.1 times 0.0556, so similarity 0.9611, distance 0.0389.
- Pair 3: same Jaro 17 over 18, but the first characters differ so the prefix length is 0, no bonus. Distance 1 over 18, 0.0556.
- Pair 6: M and m differ, m equals 5 of 6, t equals 0, Jaro 0.8889, prefix 0. Distance 0.1111. Not zero.
- Pair 4: m equals 4, t equals 0, Jaro 0.8222, prefix D, length 1, similarity 0.84. Distance 0.1600.
- Pair 5: window is 3, the two X characters are five positions apart and do not match, m equals 4, t equals 0, Jaro 0.7667, prefix DI, length 2, similarity 0.8133. Distance 0.1867.

## 6. Hint ladder (one hint per attempt, in order)

**Part one, pair 1 and polarity**
1. "Read the function name literally. Is it a similarity or a distance?"
2. "For identical strings, a distance is at its minimum. What number is that?"
3. "So a smaller JwDist means more alike. Keep that in mind for the ordering."

**Part two, pair 7 and the NULL**
1. "What does the function return when either input is NULL?"
2. "Now, with ORDER BY JwDist ascending, where does SQL Server place a NULL: before or after the numbers?"
3. "SQL Server treats NULL as lower than any value when sorting. So which row comes first?"

**Part three, pairs 2 and 3**
1. "Both have the same swap, so the Jaro part is identical: six matches, one transposition. What makes Jaro Winkler different from plain Jaro?"
2. "The Winkler bonus rewards a shared prefix, up to four characters. How many leading characters do MARTHA and MARHTA share? And MARTHA and AMRTHA?"
3. "Three shared leading letters versus zero. Which pair gets the bonus, and does a bonus make the distance smaller or larger?"
4. "Jaro is 17 over 18. Pair 3 is one minus that. Pair 2 adds a bonus of three times 0.1 times one minus Jaro to the similarity."

**Part four, pair 6**
1. "The collation is case insensitive. Does the function use the collation?"
2. "If capital M and lowercase m are different characters, how many of the six letters match?"
3. "Five of six match, no transpositions, and the first characters differ so no prefix bonus. Average the three fractions."

**Part five, pairs 4 and 5**
1. "For DWAYNE and DUANE, list the letters that appear in both, in order. How many?"
2. "For DIXON and DICKSONX, remember the matching window: half the longer length minus one. Are the two X characters close enough to match?"
3. "Both pairs share a prefix: D for pair 4, DI for pair 5. Both get a bonus. Which pair ends up more distant?"

**Full order**
1. "NULL first, then identical, then sort the distances from small to large."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "Pair 1 is 1.0, or 100" | Confuses distance with similarity | "Is this the DISTANCE function or the SIMILARITY function? What is a distance between identical things?" |
| "Pair 7 sorts last" | Assumes NULL sorts high | "Under ORDER BY ascending in SQL Server, is NULL treated as lower or higher than every value?" |
| "Pairs 2 and 3 tie" | Ignores the Winkler prefix bonus | "Where does the swap sit in each pair, at the start or at the end? Why does that matter for Jaro Winkler?" |
| "Pair 3 is closer than pair 2" | Bonus applied in the wrong direction | "The bonus increases similarity. Which pair has the shared prefix?" |
| "Pair 6 is 0.0000, they are equal under CI" | Assumes the function respects collation | "Does any of the 2025 fuzzy functions consult the collation?" |
| "Pair 5: the X characters match, m is 5" | Ignores the matching window | "What is the window for lengths 5 and 8? How far apart are the two X characters?" |
| "It fails, the function is not recognised" | Missed PREVIEW_FEATURES | "Which database scoped configuration was switched on in the setup?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain what `JARO_WINKLER_DISTANCE` is:

- New in SQL Server 2025, preview gated on RTM by PREVIEW_FEATURES equals ON; without it, error 195. Despite the name looking like a score, it returns a float distance in zero to one: one minus the Jaro Winkler similarity. Identical strings return 0.0, ABC versus XYZ returns 1.0. Its integer sibling JARO_WINKLER_SIMILARITY returns 0 to 100 where 100 is identical, the opposite polarity.
- NULL in, NULL out. Comparison is exact per character; the collation never applies, so Martha versus martha is 0.1111, not zero.

The two-stage formula:

- **Jaro.** Characters match when equal and within a window of half the longer length minus one positions. m is the number of matches, t is half the number of matched characters that are out of order. Jaro is the average of m over length one, m over length two, and m minus t over m. For MARTHA versus MARHTA: m equals 6, t equals 1, Jaro equals 17 over 18.
- **Winkler.** Add a bonus for the common prefix, length l capped at 4, with scaling factor 0.1: similarity equals Jaro plus l times 0.1 times one minus Jaro. MARHTA shares MAR, so l equals 3 and the distance drops to 0.0389. AMRTHA has l equals 0, so the distance stays at 1 over 18, 0.0556. Same Jaro, different Jaro Winkler, purely because of where the error sits. An error at the start of a name costs more than the same error at the end. That is exactly why the function suits name screening.

The order: ORDER BY JwDist ascending sorts pair 7's NULL first, because SQL Server treats NULL as lower than every non NULL value. Then 0.0000, 0.0389, 0.0556, 0.1111, 0.1600, 0.1867. The cast to decimal six comma four rounds away the float noise in the sixteenth digit, which is what makes the result deterministic.

Equivalent: JARO_WINKLER_SIMILARITY returns the ints 100, 96, 94, 84, 81, 89, NULL for pairs 1 to 7. ORDER BY that descending ranks the non NULL pairs identically, with the NULL row moving to the end.

Memory hook: "Distance: zero is identical. Winkler pays for a matching prefix. Case counts. NULL sorts first under ascending."

## 9. Follow-up oral questions (optional)

1. "What would JARO_WINKLER_SIMILARITY return for pair 2?" (96, an int; it is roughly 100 times one minus the distance.)
2. "What is the maximum prefix length the Winkler bonus rewards?" (Four characters, with scaling factor 0.1.)
3. "If the ORDER BY were descending, where would pair 7 go?" (Last; NULL is the lowest value, so descending puts it at the end.)

## 10. References

- JARO_WINKLER_DISTANCE: https://learn.microsoft.com/en-us/sql/t-sql/functions/jaro-winkler-distance-transact-sql
- JARO_WINKLER_SIMILARITY: https://learn.microsoft.com/en-us/sql/t-sql/functions/jaro-winkler-similarity-transact-sql
- ORDER BY clause, NULL ordering: https://learn.microsoft.com/en-us/sql/t-sql/queries/select-order-by-clause-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
