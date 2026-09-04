# Instructor-Examiner guide — Full-Text Search 2

Companion to [full_text_search_2.md](full_text_search_2.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**This question.** It is a multiple-choice question with four options, a to d, and only one is correct. Each option is a table of six results, Q1 to Q6. The best way to run it is query by query: ask the learner what each query returns, confirm right or wrong per query, and only at the end ask which option matches. Read all six queries and all four options before taking a final answer. This is a conceptual question taken from the documentation; the lab instance has no full-text component, so the learner is not expected to have run it.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Implement search capabilities.
- Task bullet: Implement full-text search, including inflectional and thesaurus forms, FREETEXT and the LANGUAGE argument.
- What is tested: that a simple CONTAINS term is exact, that FORMSOF INFLECTIONAL uses the language stemmer including irregular forms, that a thesaurus expansion set keeps every synonym while a replacement set drops the original pattern, that FREETEXT stems and applies the thesaurus implicitly, and that LANGUAGE zero x zero selects the neutral language with no stemmer.

## 2. Scenario to read aloud

**Piece 1, the story.** "A travel publisher keeps its guidebook paragraphs in a SQL Server 2025 database named AtlasGuides. The instance has the Full-Text and Semantic Extractions for Search feature installed; FULLTEXTSERVICEPROPERTY of IsFullTextInstalled returns one. The server option transform noise words keeps its default value of zero."

**Piece 2, the table.** "One table, Trips dot Guides. Three columns. GuideId, an integer, the primary key, with the constraint name PK underscore Guides. Title, text up to eighty characters. And Body, NVARCHAR MAX. Five rows are inserted. I will read the Body of each row carefully, because the exact words matter."

**Piece 3, the data.** "Row one, title Alpine ridge, body: Hikers ride the cable car up and then walk the ridge trail. Row two, title Valley railway, body: We rode the narrow-gauge railway through the valley. Row three, title Canal city, body: Riding a bicycle along the canal is the best way to see the city. Row four, title Harbour hop, body: The ferry crossing takes forty minutes; a bike rental is nearby. Row five, title Island loop, body: Cycling routes cover the whole island."

**Piece 4, the full-text index.** "A full-text catalog GuideCatalog is created as the default. Then a full-text index on Trips dot Guides, on the Body column with LANGUAGE one thousand thirty-three, which is English. KEY INDEX PK underscore Guides. WITH CHANGE underscore TRACKING AUTO and STOPLIST SYSTEM. Population has completed."

**Piece 5, the thesaurus.** "A sysadmin edits the English thesaurus file, tsenu dot xml, in the instance's FTDATA folder. They remove the comment markers around the shipped sample and save the file as Unicode with a byte-order mark. The file now has diacritics underscore sensitive set to zero. It has one expansion set with three sub entries: bicycle, bike, and cycling. And one replacement set with pattern ferry and substitute boat. Then they run EXECUTE sys dot sp underscore fulltext underscore load underscore thesaurus underscore file with argument one thousand thirty-three."

**Piece 6, the six queries.** "Six queries then run, each selecting GuideId from Trips dot Guides. Q1: WHERE CONTAINS of Body, and the string ride. Q2: CONTAINS of Body, FORMSOF INFLECTIONAL, ride. Q3: CONTAINS of Body, FORMSOF THESAURUS, bicycle. Q4: CONTAINS of Body, FORMSOF THESAURUS, ferry. Q5: FREETEXT of Body, riding. Q6: CONTAINS of Body, FORMSOF INFLECTIONAL, ride, with a third argument, LANGUAGE zero x zero."

**Piece 7, the options.** "The question asks which option lists the GuideId values each query returns. Option a: Q1 returns one. Q2 returns one, two, three. Q3 returns three, four, five. Q4 returns no rows. Q5 returns one, two, three. Q6 returns one. Option b: Q1 one, two, three. Q2 one, two, three. Q3 three, four, five. Q4 four. Q5 one, two, three. Q6 one, two, three. Option c: Q1 one. Q2 one, three. Q3 three. Q4 four. Q5 one, two, three. Q6 one. Option d: Q1 one. Q2 one, two, three. Q3 three, four, five. Q4 four. Q5 one. Q6 one."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE AtlasGuides;
GO
USE AtlasGuides;
GO
CREATE SCHEMA Trips;
GO
CREATE TABLE Trips.Guides
(
    GuideId INT           NOT NULL CONSTRAINT PK_Guides PRIMARY KEY,
    Title   NVARCHAR(80)  NOT NULL,
    Body    NVARCHAR(MAX) NOT NULL
);
INSERT INTO Trips.Guides (GuideId, Title, Body) VALUES
    (1, N'Alpine ridge',   N'Hikers ride the cable car up and then walk the ridge trail.'),
    (2, N'Valley railway', N'We rode the narrow-gauge railway through the valley.'),
    (3, N'Canal city',     N'Riding a bicycle along the canal is the best way to see the city.'),
    (4, N'Harbour hop',    N'The ferry crossing takes forty minutes; a bike rental is nearby.'),
    (5, N'Island loop',    N'Cycling routes cover the whole island.');
GO
CREATE FULLTEXT CATALOG GuideCatalog AS DEFAULT;
GO
CREATE FULLTEXT INDEX ON Trips.Guides (Body LANGUAGE 1033)
    KEY INDEX PK_Guides
    WITH (CHANGE_TRACKING = AUTO, STOPLIST = SYSTEM);
GO
```

Thesaurus file `tsenu.xml`:

```xml
<XML ID="Microsoft Search Thesaurus">
    <thesaurus xmlns="x-schema:tsSchema.xml">
        <diacritics_sensitive>0</diacritics_sensitive>
        <expansion>
            <sub>bicycle</sub>
            <sub>bike</sub>
            <sub>cycling</sub>
        </expansion>
        <replacement>
            <pat>ferry</pat>
            <sub>boat</sub>
        </replacement>
    </thesaurus>
</XML>
```

```sql
EXECUTE sys.sp_fulltext_load_thesaurus_file 1033;
```

```sql
-- Q1
SELECT GuideId FROM Trips.Guides WHERE CONTAINS(Body, 'ride');
-- Q2
SELECT GuideId FROM Trips.Guides WHERE CONTAINS(Body, 'FORMSOF(INFLECTIONAL, ride)');
-- Q3
SELECT GuideId FROM Trips.Guides WHERE CONTAINS(Body, 'FORMSOF(THESAURUS, bicycle)');
-- Q4
SELECT GuideId FROM Trips.Guides WHERE CONTAINS(Body, 'FORMSOF(THESAURUS, ferry)');
-- Q5
SELECT GuideId FROM Trips.Guides WHERE FREETEXT(Body, 'riding');
-- Q6
SELECT GuideId FROM Trips.Guides WHERE CONTAINS(Body, 'FORMSOF(INFLECTIONAL, ride)', LANGUAGE 0x0);
```

| Query | a | b | c | d |
|---|---|---|---|---|
| Q1 | 1 | 1, 2, 3 | 1 | 1 |
| Q2 | 1, 2, 3 | 1, 2, 3 | 1, 3 | 1, 2, 3 |
| Q3 | 3, 4, 5 | 3, 4, 5 | 3 | 3, 4, 5 |
| Q4 | (no rows) | 4 | 4 | 4 |
| Q5 | 1, 2, 3 | 1, 2, 3 | 1, 2, 3 | 1 |
| Q6 | 1 | 1, 2, 3 | 1 | 1 |

## 4. The question (ask exactly this)

"Let's take the queries one at a time. Which GuideId values does Q1 return?" Then Q2 to Q6 in turn.

After all six: "Which option lists the GuideId values that each query returns? Option a, option b, option c, or option d?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

| Query | Result | Why |
|---|---|---|
| Q1 | 1 | A simple term is an exact token match, case-insensitive. Only row 1 has the token ride; rode and Riding are different tokens. |
| Q2 | 1, 2, 3 | FORMSOF INFLECTIONAL uses the English stemmer. ride, rode and riding are inflectional forms, irregular past tense included. |
| Q3 | 3, 4, 5 | bicycle is a sub in an expansion set, so the query becomes bicycle OR bike OR cycling. Row 3 bicycle, row 4 bike, row 5 Cycling. |
| Q4 | no rows | ferry is the pattern of a replacement set, so the query searches for boat only. No row contains boat; row 4 with ferry is not returned. |
| Q5 | 1, 2, 3 | FREETEXT is wordbroken, stemmed and passed through the thesaurus. riding stems to the same forms as Q2. |
| Q6 | 1 | LANGUAGE zero x zero selects the neutral language, which has no stemmer, so INFLECTIONAL degrades to the literal token. |

- **b is wrong:** it stems the plain simple term in Q1, reads Q4's replacement set as an expansion set, and ignores LANGUAGE zero x zero in Q6.
- **c is wrong:** it assumes the stemmer handles only regular suffixes, dropping rode in Q2; it assumes the thesaurus is not in effect in Q3 even though it was explicitly loaded and would load lazily anyway; and it repeats the replacement-versus-expansion confusion in Q4.
- **d is wrong:** Q1, Q2, Q3 and Q6 are right. Q4 is wrong because a replacement set removes the pattern and substitutes the sub entries. Q5 is wrong because FREETEXT is never an exact match; it is the loosest predicate.

## 6. Hint ladder (one hint per attempt, in order)

**Q1**
1. "CONTAINS with a bare word is called a simple term. Does a simple term apply any stemming?"
2. "Which single row contains the exact token ride, spelled r i d e?"

**Q2**
1. "INFLECTIONAL asks the stemmer of the column's language for all forms of the word. Is the past tense a form?"
2. "The documentation's example is drive, drives, drove, driving, driven. Apply that to ride, rode, riding."

**Q3**
1. "In the thesaurus file, is bicycle inside an expansion element or a replacement element?"
2. "In an expansion set, a match for any synonym is expanded to include every other synonym. Which three words are you now searching for, and which rows contain them?"

**Q4**
1. "In the thesaurus file, is ferry a pat or a sub? Which element is it in?"
2. "A replacement set replaces the pattern with the substitution. After replacement, which word is the query really looking for? Does any row contain it?"
3. "The documentation's example is Win8 replaced by Windows Server 2012: results containing Win8 are not returned."

**Q5**
1. "FREETEXT does three things to its string automatically. Wordbreaking is one. What are the other two?"
2. "riding is stemmed. Which query earlier in the list produced the same set of forms?"

**Q6**
1. "The third argument, LANGUAGE zero x zero, is the neutral language. Does the neutral language have a stemmer?"
2. "Without a stemmer, what does FORMSOF INFLECTIONAL of ride fall back to? Compare with Q1."

**Final option**
1. "Only one option has no rows for Q4. Which one?"
2. "Two options differ only on Q4 and Q5. Recall what FREETEXT does with riding."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "Q1 returns one, two, three; full-text search is smart about word forms" | Thinks CONTAINS always stems | "Which predicates stem? Is a bare word in CONTAINS one of them?" |
| "Q2 returns one and three; rode is not a suffix form" | Thinks the stemmer is suffix-only | "Look at the documentation's example verb drive. Does it list drove?" |
| "Q3 returns only three; the thesaurus file needs a restart" | Does not know files load lazily and were loaded explicitly | "What did the sysadmin execute after saving the file? And what happens the first time a thesaurus query runs even without that?" |
| "Q4 returns four; the query looks for ferry or boat" | Confuses replacement with expansion | "Which element type keeps the original word, and which one removes it?" |
| "Q5 returns one; FREETEXT matches the literal word" | Thinks FREETEXT is stricter than CONTAINS | "Is FREETEXT the strictest or the loosest full-text predicate?" |
| "Q6 returns one, two, three; the language argument only changes the word breaker" | Does not know the neutral language has no stemmer | "What language resources does zero x zero select, and which resource is missing from them?" |

## 8. Teaching notes (after the answer is complete or revealed)

Three mechanisms widen a full-text query beyond the literal token, and a plain CONTAINS term uses none of them:

- **The language stemmer.** FORMSOF INFLECTIONAL asks the stemmer of the query language for every inflectional form, including irregular ones such as rode. FREETEXT stems implicitly. A simple term in CONTAINS is exact, case-insensitive, no stemming.
- **The thesaurus.** One file per language, ts plus a three-letter code plus dot xml, tsenu dot xml for English, plus the global tsglobal dot xml, in the FTDATA folder. Files ship essentially empty with the sample commented out; only sysadmins edit them; save as Unicode with a byte-order mark. An expansion set lists sub entries, and a match on any of them expands to all of them, both kept. A replacement set has a pat and sub entries: the pattern is replaced by the substitutes and no longer matches on its own. sp underscore fulltext underscore load underscore thesaurus underscore file with the LCID parses the file into tempdb and recompiles the queries that use it; without the call, the file loads on first use. FORMSOF THESAURUS uses it explicitly, FREETEXT implicitly. Azure SQL Database exposes no file system, so the thesaurus is a SQL Server feature.
- **The language.** LANGUAGE may be an alias string, an integer LCID such as one thousand thirty-three, or a hex value such as zero x four zero nine. It governs word breaking, stemming, thesaurus and stopwords. When omitted, the column's full-text language is used, which is why Body LANGUAGE one thousand thirty-three in the index matters. Zero x zero is the neutral language, which has no stemmer, so INFLECTIONAL degrades to the literal token.
- **FREETEXT.** The loosest predicate: wordbroken, stemmed, passed through the thesaurus, any term matches, and AND or NOT are treated as stopwords, not operators. FREETEXTTABLE and CONTAINSTABLE return KEY and RANK; ISABOUT with WEIGHT changes RANK only, never the row set of CONTAINS.
- **Stoplists.** STOPLIST SYSTEM strips stopwords from index and queries, keeping positions so phrase and NEAR distances remain correct. With transform noise words at zero, a Boolean condition containing a stopword returns zero rows with a warning.

Memory hook: "Simple term is exact. INFLECTIONAL stems. Expansion keeps, replacement swaps. FREETEXT does it all. Neutral language, no stemmer."

## 9. Follow-up oral questions (optional)

1. "If the replacement set had a pat of ferry and no sub at all, what would Q4 search for?" (Nothing; the term is simply removed from the query.)
2. "Why does CONTAINS of Body, canal AND the, return zero rows with a warning on this instance?" (the is a stopword; with transform noise words at zero a Boolean condition containing a stopword returns zero rows and raises a warning.)
3. "Does ISABOUT with WEIGHT change which rows CONTAINS returns?" (No. WEIGHT affects RANK in CONTAINSTABLE only, never the row set of CONTAINS.)

## 10. References

- CONTAINS: https://learn.microsoft.com/en-us/sql/t-sql/queries/contains-transact-sql
- FREETEXT: https://learn.microsoft.com/en-us/sql/t-sql/queries/freetext-transact-sql
- Configure and manage thesaurus files for full-text search: https://learn.microsoft.com/en-us/sql/relational-databases/search/configure-and-manage-thesaurus-files-for-full-text-search
- sys.sp_fulltext_load_thesaurus_file: https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sys-sp-fulltext-load-thesaurus-file-transact-sql
- Configure and manage stopwords and stoplists: https://learn.microsoft.com/en-us/sql/relational-databases/search/configure-and-manage-stopwords-and-stoplists-for-full-text-search
- Query with full-text search: https://learn.microsoft.com/en-us/sql/relational-databases/search/query-with-full-text-search
- CREATE FULLTEXT INDEX: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-fulltext-index-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
