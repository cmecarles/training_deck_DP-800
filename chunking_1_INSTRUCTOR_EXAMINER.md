# Instructor-Examiner guide — Chunking 1

Companion to [chunking_1.md](chunking_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a three-part prediction question, A, B and C. Part A asks for a full result set; take it one row at a time, and let the learner count characters aloud. Spell the sentence on request, letter by letter, so the learner can count positions. Part C is a short one; do not let the learner overthink it.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Implement AI-enabled data solutions.
- Task bullet: Implement chunking strategies for text data.
- What is tested: the contract of AI_GENERATE_CHUNKS, how overlap is computed as a percentage of chunk size, what a NULL source returns, and the child-table pattern for storing chunks.

## 2. Scenario to read aloud

**Piece 1, the story.** "A wind-farm operator keeps turbine maintenance manuals in a SQL Server 2025 database called TurbineDocs. Manual pages are long, so before generating embeddings the team splits each page into overlapping fixed-size chunks with the AI underscore GENERATE underscore CHUNKS table-valued function, and stores the chunks in a child table. For this exercise the pages are tiny and the chunk size is only twenty characters, so you can predict the result by hand."

**Piece 2, the parent table.** "The database is at compatibility level 170, the default on SQL Server 2025. There is a schema called Manuals. The first table is Manuals dot Pages, with two columns. PageId, an integer, the primary key. And Body, an NVARCHAR MAX, which allows null."

**Piece 3, the data.** "Three pages are inserted. Page 1 has the body: Wind turbines convert kinetic energy into electricity, with a full stop at the end. That is fifty-four characters in total, counting spaces and the full stop. Page 2 has a NULL body. Page 3 has the body: Short page, with a full stop. That is eleven characters."

**Piece 4, the child table.** "The second table is Manuals dot PageChunks. ChunkId, an integer identity, the primary key. PageId, an integer, not null, with a foreign key to Pages. Ordinal, an integer, not null. ChunkText, NVARCHAR MAX, not null. Embedding, a VECTOR of 1536, which allows null. And a unique constraint on the pair PageId and Ordinal."

**Piece 5, the query for part A.** "Part A is a SELECT. It selects p dot PageId, then c dot chunk underscore order, c dot chunk underscore offset, c dot chunk underscore length and c dot chunk. FROM Manuals dot Pages as p, CROSS APPLY AI underscore GENERATE underscore CHUNKS with source equals p dot Body, chunk underscore type equals FIXED, chunk underscore size equals 20, overlap equals 25, aliased as c. ORDER BY PageId and then chunk underscore order."

**Piece 6, the insert for part B.** "Part B is an INSERT into Manuals dot PageChunks, columns PageId, Ordinal and ChunkText. The SELECT is the same CROSS APPLY as part A, same parameters, but it projects only p dot PageId, c dot chunk underscore order and c dot chunk. No ORDER BY."

**Piece 7, the query for part C.** "Part C is a SELECT of chunk underscore order, chunk underscore offset and chunk, directly from AI underscore GENERATE underscore CHUNKS. The source is the literal sentence from page 1, Wind turbines convert kinetic energy into electricity. Chunk type FIXED, chunk size 20, and this time overlap equals 60."

**Piece 8, the sentence, for counting.** "If you need to count positions, here is the page 1 text again, word by word: Wind, space, turbines, space, convert, space, kinetic, space, energy, space, into, space, electricity, full stop. Word lengths are four, eight, seven, seven, six, four, eleven, plus six spaces and the full stop. Fifty-four characters."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE TurbineDocs;        -- compatibility level 170 (default on SQL Server 2025)
GO
USE TurbineDocs;
GO
CREATE SCHEMA Manuals;
GO
CREATE TABLE Manuals.Pages
(
    PageId INT           NOT NULL PRIMARY KEY,
    Body   NVARCHAR(MAX) NULL
);
GO
INSERT INTO Manuals.Pages (PageId, Body) VALUES
    (1, N'Wind turbines convert kinetic energy into electricity.'),   -- 54 characters
    (2, NULL),
    (3, N'Short page.');                                              -- 11 characters
GO
CREATE TABLE Manuals.PageChunks
(
    ChunkId   INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    PageId    INT NOT NULL REFERENCES Manuals.Pages (PageId),
    Ordinal   INT NOT NULL,
    ChunkText NVARCHAR(MAX) NOT NULL,
    Embedding VECTOR(1536) NULL,
    CONSTRAINT UQ_PageChunks UNIQUE (PageId, Ordinal)
);
GO
```

```sql
-- Part A
SELECT p.PageId, c.chunk_order, c.chunk_offset, c.chunk_length, c.chunk
FROM Manuals.Pages AS p
CROSS APPLY AI_GENERATE_CHUNKS(source = p.Body, chunk_type = FIXED, chunk_size = 20, overlap = 25) AS c
ORDER BY p.PageId, c.chunk_order;

-- Part B
INSERT INTO Manuals.PageChunks (PageId, Ordinal, ChunkText)
SELECT p.PageId, c.chunk_order, c.chunk
FROM Manuals.Pages AS p
CROSS APPLY AI_GENERATE_CHUNKS(source = p.Body, chunk_type = FIXED, chunk_size = 20, overlap = 25) AS c;

-- Part C
SELECT c.chunk_order, c.chunk_offset, c.chunk
FROM AI_GENERATE_CHUNKS(source = N'Wind turbines convert kinetic energy into electricity.',
                        chunk_type = FIXED, chunk_size = 20, overlap = 60) AS c;
```

## 4. The question (ask exactly this)

One part at a time.

**Part A.** "Predict the exact result set of the part A query: every row, every column, in order. For each row give me PageId, chunk order, chunk offset, chunk length and the chunk text. Start with the first row."

**Part B.** "How many rows does the part B INSERT put into Manuals dot PageChunks, and does it succeed?"

**Part C.** "What happens when the overlap is raised to 60, as in the part C query?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Part A.** Five rows. Overlap 25 percent of 20 is 5 characters, so each chunk starts 15 characters after the previous one.

| PageId | chunk_order | chunk_offset | chunk_length | chunk |
|---|---|---|---|---|
| 1 | 1 | 1 | 20 | `Wind turbines conver` |
| 1 | 2 | 16 | 20 | `onvert kinetic energ` |
| 1 | 3 | 31 | 20 | `energy into electric` |
| 1 | 4 | 46 | 9 | `ctricity.` |
| 3 | 1 | 1 | 11 | `Short page.` |

Page 2, NULL body, produces no row at all. CROSS APPLY drops it.

**Part B.** It succeeds and inserts 5 rows, one per chunk above. Ordinal receives chunk_order, which restarts at 1 per page, so the UNIQUE (PageId, Ordinal) constraint is satisfied.

**Part C.** It fails before producing any row:

```text
Msg 43201, Level 16, State 1
The value 60 is invalid for overlap. Value must be between 0 and 50.
```

## 6. Hint ladder (one hint per attempt, in order)

**Part A**
1. "Overlap is not a number of characters. It is a percentage of something. Of what?"
2. "Twenty-five percent of twenty characters is how many characters? So how far apart do consecutive chunks start?"
3. "Chunk 1 covers positions 1 to 20. If the next chunk starts 15 later, where does chunk 2 start, and where does chunk 3 start?"
4. "The page is 54 characters. When you reach the last chunk, how many characters are left after position 46? That is its chunk length."
5. "Now page 2. Its body is NULL. Does the function return a row with NULL columns, or nothing at all? And what does CROSS APPLY do with a parent row when the function returns nothing?"
6. "Page 3 is only 11 characters, shorter than the chunk size. How many chunks does that produce, and what is its length?"

**Part B**
1. "The INSERT uses the same CROSS APPLY as part A. How many rows did part A produce?"
2. "The Ordinal column gets chunk underscore order. Does chunk underscore order restart per page or count globally? Check that against the unique constraint on PageId and Ordinal."
3. "Nothing in the insert violates a constraint, and ChunkText is never NULL. So does it succeed?"

**Part C**
1. "What is the allowed range for the overlap parameter? Think about what would happen if the overlap were more than half the chunk."
2. "It is not a warning and it is not a strange result set. The engine validates the parameter before chunking. What kind of outcome is that?"
3. "It is an error. The number is in the 43 thousands, and the message says the value must be between two numbers."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "Chunks start every 20 characters: offsets 1, 21, 41" | Ignores overlap | "The query says overlap 25. What does that do to the start of chunk 2?" |
| "Chunks start 25 characters apart" or "overlap 25 characters" | Reads overlap as characters, not a percentage | "Overlap is a percentage. Of which parameter?" |
| "Three chunks for page 1" | Correct step but forgot the tail | "After chunk 3 ends at position 50, are there characters left in a 54-character page?" |
| "Page 2 returns one row with NULLs" | Confuses CROSS APPLY with OUTER APPLY | "Which APPLY keeps the parent row when the function returns nothing? Which one is in the query?" |
| "Page 2 raises an error because source is NULL" | Assumes NULL input errors | "A NULL source is not an error. What does the function return for it?" |
| "Part B fails on the unique constraint because chunk order 1 appears twice" | Thinks the constraint is on Ordinal alone | "The unique constraint is on which columns together? Is PageId 1, Ordinal 1 the same as PageId 3, Ordinal 1?" |
| "Part C returns chunks with 12 characters of overlap" | Does not know the 0 to 50 limit | "Is 60 inside the allowed range for overlap?" |

## 8. Teaching notes (after the answer is complete or revealed)

The function's contract:

- AI_GENERATE_CHUNKS takes source, chunk_type, chunk_size, overlap and optionally enable_chunk_set_id. It is a table-valued function available from compatibility level 170. On SQL Server 2025 RTM it runs without PREVIEW_FEATURES.
- chunk_type accepts only FIXED. SENTENCE or PARAGRAPH is a syntax error, message 102. Fixed chunking is purely character based; it cuts through words, as the split in "conver" and "t" shows.
- chunk_size is in characters and must be greater than zero. Zero gives error 43202.
- overlap is a percentage of chunk_size, an integer from 0 to 50, default 0. Outside the range gives error 43201. The percentage is applied with integer truncation: overlap 7 on a chunk of 20 is 1 character; overlap 10 is 2 characters.
- Output columns: chunk, chunk_order (bigint, 1-based per source), chunk_offset (bigint, 1-based character position), chunk_length (int), and chunk_set_id when enabled.

The arithmetic: next offset equals previous offset plus chunk_size minus trunc(chunk_size times overlap over 100). With 20 and 25 that is plus 15: offsets 1, 16, 31, 46. The last chunk is short, 54 minus 46 plus 1 equals 9 characters. Without overlap the same page gives 3 chunks at 1, 21, 41, with lengths 20, 20, 14. Overlap buys continuity across a cut at the cost of a few extra chunks.

NULL source returns zero rows. CROSS APPLY then drops the parent row; OUTER APPLY would keep it with NULL chunk columns. Nothing errors. The page is simply absent.

The child-table pattern: ChunkId primary key, PageId foreign key, Ordinal equal to chunk_order and unique per parent, ChunkText, optionally chunk_offset and chunk_length for highlighting, and the vector column that gets embedded later, one embedding per chunk, never per parent document.

Why chunk at all: embedding models accept a bounded input, about 8,192 tokens for current Azure OpenAI embedding models, so long pages must be split; and one vector per passage ranks the relevant passage highly instead of averaging a whole page.

Memory hook: "Overlap is a percent, zero to fifty. Next offset equals chunk size minus overlap characters. NULL in, zero rows out, and CROSS APPLY drops the parent."

## 9. Follow-up oral questions (optional)

1. "If the query used OUTER APPLY instead of CROSS APPLY, what would page 2 look like in the result?" (One row for page 2 with NULL in all the chunk columns.)
2. "With chunk size 20 and overlap 10, how far apart do chunks start?" (18 characters: 10 percent of 20 is 2, and 20 minus 2 is 18. Offsets 1, 19, 37.)
3. "What happens if you pass chunk_type = PARAGRAPH?" (Syntax error, message 102. Only FIXED is accepted.)

## 10. References

- AI_GENERATE_CHUNKS: https://learn.microsoft.com/en-us/sql/t-sql/functions/ai-generate-chunks-transact-sql
- AI_GENERATE_EMBEDDINGS: https://learn.microsoft.com/en-us/sql/t-sql/functions/ai-generate-embeddings-transact-sql
- FROM clause, CROSS APPLY and OUTER APPLY: https://learn.microsoft.com/en-us/sql/t-sql/queries/from-transact-sql
- Vector data type: https://learn.microsoft.com/en-us/sql/t-sql/data-types/vector-data-type
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
