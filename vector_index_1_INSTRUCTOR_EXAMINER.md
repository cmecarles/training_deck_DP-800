# Instructor-Examiner guide — Vector Index 1

Companion to [vector_index_1.md](vector_index_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is an eleven-statement walkthrough on SQL Server 2025 RTM, where the vector index is a preview feature. Take the statements strictly in order. S4 and S9 need arithmetic; let the learner work with fractions and accept answers to four decimals. If the learner mentions the Azure SQL Database behaviour, where the newer index supports DML, acknowledge it but say this question is about the SQL Server 2025 RTM build described in the scenario.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Implement AI capabilities in database solutions (25–30%).
- Skill: Design and implement intelligent search.
- Task bullet: Create and manage vector indexes.
- What is tested: vector storage size and the 1998-dimension cap, VECTORPROPERTY and VECTOR underscore NORMALIZE, cosine versus dot product on unit vectors, the requirements of CREATE VECTOR INDEX, the read-only table while the index exists, and the rule that VECTOR underscore SEARCH must use the index's metric.

## 2. Scenario to read aloud

**Piece 1, the story.** "An online gallery stores style embeddings of artworks in a SQL Server 2025 database called ArtLens. For this exercise the embeddings have only three dimensions. The three axes mean warm colours, seascape, and monochrome. The vector index and VECTOR underscore SEARCH are preview features on SQL Server 2025, so the database sets the scoped configuration PREVIEW underscore FEATURES to ON."

**Piece 2, the table.** "One table, in a schema called Gallery, named Artworks. ArtworkId, an integer, NOT NULL, with a clustered primary key constraint. Title, text up to eighty. StyleVector, of type VECTOR open paren three close paren, NOT NULL."

**Piece 3, the data.** "Five rows. Artwork 1, Harbour at Dawn, vector three, four, zero. Artwork 2, Red Square Study, vector one, zero, zero. Artwork 3, Blue Monochrome, vector zero, zero, two. Artwork 4, Harbour at Dawn, poster, vector six, eight, zero. Notice that is exactly double artwork 1. Artwork 5, Inverse Red, vector minus one, zero, zero."

**Piece 4, statements S1 to S3.** "S1 selects, for artwork 1, DATALENGTH of StyleVector as Bytes, VECTORPROPERTY of StyleVector with Dimensions as Dims, and VECTORPROPERTY with BaseType as BaseType. S2 creates a table Gallery dot HiRes with an integer primary key and a column Vec of type VECTOR open paren one thousand nine hundred ninety-nine close paren. S3 selects, for artwork 1, VECTOR underscore NORMALIZE of StyleVector with norm2, cast to VARCHAR eighty, as UnitVector."

**Piece 5, statement S4.** "S4 declares at q as a VECTOR three equal to one, zero, zero. Then, for every artwork ordered by ArtworkId, it selects two columns. CosDist: VECTOR underscore DISTANCE with metric cosine between StyleVector and at q, cast to four decimals. And DotOfUnit: VECTOR underscore DISTANCE with metric dot between the norm2-normalized StyleVector and the norm2-normalized at q, also cast to four decimals."

**Piece 6, statements S5 to S7.** "S5 creates a vector index named VX underscore Artworks underscore Style on Gallery dot Artworks, column StyleVector, WITH METRIC cosine and TYPE diskann. S6 inserts artwork 6, Green Field, vector zero, one, zero. S7 updates Artworks, setting Title to Harbour at Dusk where ArtworkId is 1. Only the title, not the vector."

**Piece 7, statements S8 and S9.** "S8 declares at q as one, zero, zero again and runs VECTOR underscore SEARCH with TABLE equals Gallery dot Artworks as a, COLUMN equals StyleVector, SIMILAR underscore TO equals at q, METRIC equals euclidean, TOP underscore N equals three, as r. It selects ArtworkId and r dot distance to four decimals, ordered by distance then ArtworkId. S9 is the same search but with METRIC equals cosine, and it also selects Title."

**Piece 8, statements S10 and S11.** "S10 drops the index VX underscore Artworks underscore Style on Gallery dot Artworks. S11, in its own batch, repeats the insert of artwork 6, Green Field, zero, one, zero."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE ArtLens;
GO
USE ArtLens;
GO
ALTER DATABASE SCOPED CONFIGURATION SET PREVIEW_FEATURES = ON;
GO
CREATE SCHEMA Gallery;
GO
CREATE TABLE Gallery.Artworks
(
    ArtworkId   INT          NOT NULL CONSTRAINT PK_Artworks PRIMARY KEY CLUSTERED,
    Title       NVARCHAR(80) NOT NULL,
    StyleVector VECTOR(3)    NOT NULL
);
GO
INSERT INTO Gallery.Artworks (ArtworkId, Title, StyleVector) VALUES
    (1, N'Harbour at Dawn',          '[3, 4, 0]'),
    (2, N'Red Square Study',         '[1, 0, 0]'),
    (3, N'Blue Monochrome',          '[0, 0, 2]'),
    (4, N'Harbour at Dawn (poster)', '[6, 8, 0]'),
    (5, N'Inverse Red',              '[-1, 0, 0]');
GO

-- S1
SELECT DATALENGTH(StyleVector)                   AS Bytes,
       VECTORPROPERTY(StyleVector, 'Dimensions') AS Dims,
       VECTORPROPERTY(StyleVector, 'BaseType')   AS BaseType
FROM Gallery.Artworks WHERE ArtworkId = 1;

-- S2
CREATE TABLE Gallery.HiRes (ArtworkId INT NOT NULL PRIMARY KEY, Vec VECTOR(1999) NULL);

-- S3
SELECT CAST(VECTOR_NORMALIZE(StyleVector, 'norm2') AS VARCHAR(80)) AS UnitVector
FROM Gallery.Artworks WHERE ArtworkId = 1;

-- S4
DECLARE @q VECTOR(3) = '[1, 0, 0]';
SELECT ArtworkId,
       CAST(VECTOR_DISTANCE('cosine', StyleVector, @q) AS DECIMAL(10,4)) AS CosDist,
       CAST(VECTOR_DISTANCE('dot', VECTOR_NORMALIZE(StyleVector, 'norm2'),
                                   VECTOR_NORMALIZE(@q, 'norm2')) AS DECIMAL(10,4)) AS DotOfUnit
FROM Gallery.Artworks ORDER BY ArtworkId;

-- S5
CREATE VECTOR INDEX VX_Artworks_Style ON Gallery.Artworks (StyleVector)
    WITH (METRIC = 'cosine', TYPE = 'diskann');

-- S6
INSERT INTO Gallery.Artworks (ArtworkId, Title, StyleVector) VALUES (6, N'Green Field', '[0, 1, 0]');

-- S7
UPDATE Gallery.Artworks SET Title = N'Harbour at Dusk' WHERE ArtworkId = 1;

-- S8
DECLARE @q VECTOR(3) = '[1, 0, 0]';
SELECT a.ArtworkId, CAST(r.distance AS DECIMAL(10,4)) AS Distance
FROM VECTOR_SEARCH(TABLE = Gallery.Artworks AS a, COLUMN = StyleVector,
                   SIMILAR_TO = @q, METRIC = 'euclidean', TOP_N = 3) AS r
ORDER BY r.distance, a.ArtworkId;

-- S9
DECLARE @q VECTOR(3) = '[1, 0, 0]';
SELECT a.ArtworkId, a.Title, CAST(r.distance AS DECIMAL(10,4)) AS Distance
FROM VECTOR_SEARCH(TABLE = Gallery.Artworks AS a, COLUMN = StyleVector,
                   SIMILAR_TO = @q, METRIC = 'cosine', TOP_N = 3) AS r
ORDER BY r.distance, a.ArtworkId;

-- S10
DROP INDEX VX_Artworks_Style ON Gallery.Artworks;

-- S11
INSERT INTO Gallery.Artworks (ArtworkId, Title, StyleVector) VALUES (6, N'Green Field', '[0, 1, 0]');
```

## 4. The question (ask exactly this)

"For each of the eleven statements, S1 to S11, tell me whether it succeeds or raises an error, and give the exact result of every statement that returns rows. One at a time, starting with S1."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

| Stmt | Outcome | Detail |
|---|---|---|
| S1 | Succeeds | Bytes = 20, Dims = 3, BaseType = float32 |
| S2 | Fails, Msg 2717 | The size (1999) given to the column 'Vec' exceeds the maximum allowed (1998) |
| S3 | Succeeds | [6.0000002e-001,8.0000001e-001,0.0000000e+000], that is (0.6, 0.8, 0) in float32 |
| S4 | Succeeds | Table below |
| S5 | Succeeds | Index created; the engine prints a warning that the join order has been enforced because a local join hint is used; sys dot indexes shows type underscore desc VECTOR |
| S6 | Fails, Msg 42231 | Data modification statement failed because table 'Artworks' has a vector index on it |
| S7 | Fails, Msg 42231 | Same message, although only Title is updated |
| S8 | Fails, Msg 42227 | Cannot find a vector index with metric 'euclidean' on column 'StyleVector' |
| S9 | Succeeds | Table below |
| S10 | Succeeds | Index dropped |
| S11 | Succeeds | (1 rows affected); the table is writable again |

S4:

| ArtworkId | CosDist | DotOfUnit |
|---|---|---|
| 1 | 0.4000 | −0.6000 |
| 2 | 0.0000 | −1.0000 |
| 3 | 1.0000 | 0.0000 |
| 4 | 0.4000 | −0.6000 |
| 5 | 2.0000 | 1.0000 |

S9:

| ArtworkId | Title | Distance |
|---|---|---|
| 2 | Red Square Study | 0.0000 |
| 1 | Harbour at Dawn | 0.4000 |
| 4 | Harbour at Dawn (poster) | 0.4000 |

## 6. Hint ladder (one hint per attempt, in order)

**S1**
1. "A vector stores float32 elements. How many bytes per element, and is there a header?"
2. "Four bytes per dimension plus an eight-byte header. Three dimensions. And BaseType is the only base type supported today."

**S2**
1. "There is a maximum number of dimensions for a VECTOR column. It is tied to the 8000-byte row limit. Work it backwards: 8000 minus 8, divided by 4."
2. "1998 is the cap. Is 1999 above it? What kind of error is a bad column size: parse time or run time?"

**S3**
1. "Norm2 divides by the Euclidean length. What is the length of three, four, zero?"
2. "Divide each element by five. Remember the output is float32, so 0.6 prints with a tiny rounding tail."

**S4**
1. "Cosine distance is one minus the cosine of the angle. The query vector is one, zero, zero. For each artwork, what is the cosine of the angle with the x axis?"
2. "Artwork 4 is double artwork 1. Does magnitude change the angle?"
3. "Dot distance returns the negative of the dot product. On unit vectors, the dot product is the cosine itself. So DotOfUnit equals CosDist minus one."

**S5**
1. "List the requirements: preview features on, a clustered primary key on a single four-byte INT, one index per column, a valid metric. Does Artworks meet them?"

**S6 and S7**
1. "On this build, what happens to a table once it has a vector index? Think about DML."
2. "The first-version index makes the table read-only. Does that depend on which column the UPDATE touches?"

**S8**
1. "Which metric was the index created with? Which metric does S8 ask for?"
2. "VECTOR underscore SEARCH looks for an index with the same metric. Is there a silent fallback to an exact scan on this build?"

**S9**
1. "Now the metrics match. Take your S4 cosine distances and pick the three smallest."
2. "Two of them tie at 0.4. The ORDER BY has a tiebreaker. Which one comes first?"

**S10 and S11**
1. "Dropping a vector index works like dropping any index. After it is gone, is the table writable again? And S11 is in its own batch."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "S1 Bytes is 12" | Forgets the header | "Three float32 values are twelve bytes. Is there anything else stored with a vector?" |
| "S2 succeeds, vectors can be up to 2000 or more" | Does not know the 1998 cap | "Compute the storage for 1999 dimensions with four bytes each plus eight. Does it fit in a row?" |
| "S3 returns exactly 0.6, 0.8, 0" | Correct in value, but the printed form differs | "Right values. The engine prints float32 in scientific notation, so accept the tiny tail." |
| "S4 artwork 4 has distance 0.2 or something smaller than artwork 1" | Thinks magnitude affects cosine | "Cosine divides magnitude out. Six, eight, zero and three, four, zero point the same way." |
| "S4 DotOfUnit is positive 0.6 for artwork 1" | Forgets dot distance is the negative dot product | "VECTOR underscore DISTANCE with dot returns minus the dot product. Flip the sign." |
| "S5 fails, only five rows" | Applies the Azure minimum of 100 rows | "That minimum belongs to the newer Azure index. On this RTM build the index builds on five rows." |
| "S7 succeeds because it does not touch the vector" | Thinks the lock is per column | "The rule on this build is per table, not per column. What does the error message say about the table?" |
| "S8 falls back to an exact scan" | Expects a graceful fallback | "On this build there is no fallback. What does the engine do when no index with that metric exists?" |
| "S9 returns 2, 4, 1" | Ignores the tiebreaker | "Artworks 1 and 4 tie at 0.4. Read the ORDER BY once more." |
| "S11 fails, same as S6" | Forgets the index was dropped | "What did S10 do? And S11 is a separate batch, compiled after the drop." |

## 8. Teaching notes (after the answer is complete or revealed)

Walk the statements by topic:

- **Storage and limits, S1 and S2.** A vector of n dimensions is stored as four bytes per dimension plus an eight-byte header. Three dimensions give twenty bytes; 1536 gives 6152; 1998 gives exactly 8000, the row size limit, which is why 1998 is the cap. VECTOR underscore 1999 is rejected at parse time with error 2717. A model that emits 3072 values must be shortened with its dimensions parameter before storing. VECTORPROPERTY returns Dimensions and BaseType, and BaseType is float32, the only base type today; any other property name returns NULL.
- **Normalization and metrics, S3 and S4.** VECTOR underscore NORMALIZE with norm2 divides by the Euclidean length: three, four, zero over five is 0.6, 0.8, 0; the printed 6.0000002e-001 is the float32 representation of 0.6. Other norms are norm1 and norminf; an unknown name fails with Msg 42210. Cosine distance is one minus cosine theta: 0.4, 0, 1, 0.4, 2. Dot distance is the negative dot product; on unit vectors the dot product is cosine theta, so DotOfUnit equals CosDist minus one: minus 0.6, minus 1, 0, minus 0.6, 1. The two columns differ by a constant, so they give the same ranking. That is why embeddings are often stored pre-normalized: the cheaper dot metric becomes equivalent to cosine. Raw un-normalized dot would rank artwork 4 at minus six above artwork 2 at minus one; magnitude leaks in. Euclidean on unit vectors is also rank-equivalent.
- **CREATE VECTOR INDEX, S5.** Syntax: CREATE VECTOR INDEX name ON table, vector column, WITH METRIC cosine, dot or euclidean, optional TYPE diskann, optional MAXDOP. Requirements on this build: PREVIEW underscore FEATURES ON, else Msg 343; a clustered primary key on a single four-byte INT column, else Msg 42217; one vector index per column, else Msg 42230; QUOTED underscore IDENTIFIER ON, else Msg 1934. sys dot vector underscore indexes exposes the type, metric and build parameters. The Azure SQL Database latest index requires at least 100 non-NULL vectors; this RTM build built on five.
- **Read-only table, S6 and S7.** With the first-version index in place, every DML statement fails with 42231, INSERT, DELETE and UPDATE, even an UPDATE that does not touch the vector column. TRUNCATE fails with 42232. To change data you drop the index, modify, and rebuild, so vector indexes suit read-mostly tables loaded in bulk. Azure SQL Database's latest index lifts this with full DML support; the older ALLOW underscore STALE underscore VECTOR underscore INDEX configuration is not available on this build.
- **Search, S8 and S9.** VECTOR underscore SEARCH looks for an index on the column with the same metric. Only a cosine index exists, so the euclidean search fails with 42227; no silent fallback. With cosine it returns the table's columns plus distance: artworks 2, 1 and 4, with the ArtworkId tiebreaker putting 1 before 4. On five vectors the approximate result equals the exact one; at scale ANN may miss a true neighbour, the recall trade-off. TOP underscore N is the syntax this build accepts; the newer TOP n WITH APPROXIMATE form fails here with Msg 102.
- **Drop and write, S10 and S11.** DROP INDEX removes it like any index; the next batch inserts artwork 6. After the drop, VECTOR underscore SEARCH fails with 42227 again, no index no ANN, while ORDER BY VECTOR underscore DISTANCE keeps working because exact search never uses an index. If the DROP and the INSERT were in the same batch the INSERT would still fail with 42231, because the batch was compiled while the index existed.

Memory hook: "Four n plus eight bytes, 1998 max. Unit vectors make dot equal cosine. Index with the metric you will search with. On RTM, index on means read-only."

## 9. Follow-up oral questions (optional)

1. "What is the storage size of a VECTOR 1536 column value?" (6152 bytes.)
2. "After S10, does ORDER BY VECTOR underscore DISTANCE still work?" (Yes. Exact search never uses the index. VECTOR underscore SEARCH does not, because there is no index.)
3. "Why would you choose the dot metric over cosine?" (When the vectors are pre-normalized, dot gives the same ranking and avoids the per-row division, so it is cheaper.)

## 10. References

- Vector data type: https://learn.microsoft.com/en-us/sql/t-sql/data-types/vector-data-type
- VECTOR_DISTANCE: https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-distance-transact-sql
- VECTOR_NORMALIZE: https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-normalize-transact-sql
- VECTORPROPERTY: https://learn.microsoft.com/en-us/sql/t-sql/functions/vectorproperty-transact-sql
- CREATE VECTOR INDEX: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-vector-index-transact-sql
- VECTOR_SEARCH: https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-search-transact-sql
- Vector indexes overview: https://learn.microsoft.com/en-us/sql/relational-databases/vectors/vectors-sql-server
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
