# Instructor-Examiner guide — Search Strategy 1

Companion to [search_strategy_1.md](search_strategy_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a, b, c and d. Each option makes four choices: a technique for Search 1, for Search 2, for Search 3, and an evaluation method. Read the three search experiences, the evaluation requirement and all four options before taking an answer. This is a conceptual question; nothing was executed against an engine. The T-SQL in section 3 is an illustration from the explanation; read a line only on request.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Implement AI capabilities in database solutions (25–30%).
- Skill: Design and implement intelligent search.
- Task bullet: Choose between full-text, semantic vector and hybrid search, and evaluate vector and hybrid search.
- What is tested: matching each search requirement to the right technique, and knowing that an approximate vector index is evaluated by recall at k against the exact result, plus latency and build cost, with exact search preferred for small filtered candidate sets.

## 2. Scenario to read aloud

**Piece 1, the story.** "PartsPro is the catalog database of an industrial-parts distributor on Azure SQL Database. The catalog table holds about twelve million parts. The team must deliver three search experiences and must also decide how to evaluate the search."

**Piece 2, the table.** "The table is Catalog dot Parts. Six columns. PartId, an integer, primary key. PartNumber, varchar thirty, not null and unique, with values such as B R G dash six two zero five dash two R S. Name, nvarchar one hundred fifty. Description, nvarchar max, written in English, Spanish and German. Category, nvarchar sixty. And Embedding, a vector column of dimension fifteen thirty six, not null, produced by the text-embedding-3-large model with dimensions set to fifteen thirty six."

**Piece 3, Search 1.** "Search one is the technician lookup. Technicians type exact part numbers, such as BRG dash six two zero five dash two R S. They type prefixes, such as BRG dash six two followed by an asterisk. And they type boolean expressions such as bearing AND NOT sealed. Results must be exact: a part number that is one character off must not match."

**Piece 4, Search 2.** "Search two is the describe-the-problem search. Buyers type natural language, often paraphrased, sometimes in Spanish or German. For example: something to stop a conveyor belt slipping in a damp warehouse. That should find anti-slip belt lagging, even though no word of the query appears in the description."

**Piece 5, Search 3 and the evaluation.** "Search three is the main catalog search box. One box that must give keyword precision, so an exact model name typed by an expert ranks first, and semantic recall, so paraphrases still find the right parts, over all twelve million rows, with sub-second latency. And the evaluation: because the twelve-million-row semantic search will use an approximate vector index, the team must be able to say how much accuracy the approximation costs and when to bypass it."

**Piece 6, option a.** "Option a. Search one: a full-text index on PartNumber, Name and Description, queried with CONTAINS, using prefix terms and AND and AND NOT. Search two: a DiskANN vector index on Embedding, queried with VECTOR underscore SEARCH using cosine. Search three: hybrid. Run the full-text ranking with CONTAINSTABLE or FREETEXTTABLE and the vector ranking, then fuse them with reciprocal rank fusion. Evaluation: on a sample of a few hundred real queries, compute recall at ten of VECTOR underscore SEARCH against the exact TOP ten ORDER BY VECTOR underscore DISTANCE result, record latency and index build time, and use exact search instead of the index whenever a WHERE filter on category or supplier leaves at most a few tens of thousands of candidate rows."

**Piece 7, option b.** "Option b. Use vector search for all three. Embed part numbers, names and descriptions together so that the model understands codes as well as prose. One DiskANN index on Embedding serves every search box. Evaluation: compute the average cosine distance of the rows each query returns. The smaller the average, the better the search."

**Piece 8, option c.** "Option c. Use full-text search for all three. CONTAINS with FORMSOF THESAURUS for synonyms, and per-language thesaurus files for Spanish and German, so that paraphrases and translations match. Hybrid search is unnecessary because FREETEXT already matches meaning. Evaluation: count how many queries return at least one row."

**Piece 9, option d.** "Option d. Search one: full-text with CONTAINS. Searches two and three: the DiskANN vector index alone. The argument is that an expert's exact model name is text too, so its embedding will rank the right part first. Evaluation: validate the index by verifying, for every production query, that VECTOR underscore SEARCH returns the same ten rows as an exact VECTOR underscore DISTANCE scan of all twelve million rows. If any query differs, the index is faulty and must be rebuilt."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE TABLE Catalog.Parts
(
    PartId      INT            NOT NULL PRIMARY KEY,
    PartNumber  VARCHAR(30)    NOT NULL UNIQUE,   -- e.g. 'BRG-6205-2RS'
    Name        NVARCHAR(150)  NOT NULL,
    Description NVARCHAR(MAX)  NOT NULL,          -- English, Spanish, German
    Category    NVARCHAR(60)   NOT NULL,
    Embedding   VECTOR(1536)   NOT NULL           -- text-embedding-3-large, dimensions = 1536
);
```

Search 1, full-text:

```sql
SELECT PartId, PartNumber, Name
FROM Catalog.Parts
WHERE CONTAINS((PartNumber, Name, Description), '"BRG-62*" OR (bearing AND NOT sealed)');
```

Search 2, approximate vector search:

```sql
DECLARE @qv VECTOR(1536) = AI_GENERATE_EMBEDDINGS(N'something to stop a conveyor belt slipping in a damp warehouse'
                                                  USE MODEL PartsEmbedder);
SELECT TOP (10) WITH APPROXIMATE p.PartId, p.Name, r.distance
FROM VECTOR_SEARCH(TABLE = Catalog.Parts AS p, COLUMN = Embedding,
                   SIMILAR_TO = @qv, METRIC = 'cosine') AS r
ORDER BY r.distance;          -- WITH APPROXIMATE requires ORDER BY distance ASC and nothing else
```

Search 3, hybrid with reciprocal rank fusion, k = 60:

```sql
WITH Kw AS
(
    SELECT [KEY] AS PartId, ROW_NUMBER() OVER (ORDER BY [RANK] DESC, [KEY]) AS Rnk
    FROM CONTAINSTABLE(Catalog.Parts, (Name, Description), 'conveyor AND belt', 50)
),
Vec AS
(
    -- window functions cannot sit next to TOP ... WITH APPROXIMATE: run the ANN search in a subquery
    SELECT a.PartId, ROW_NUMBER() OVER (ORDER BY a.distance, a.PartId) AS Rnk
    FROM (SELECT TOP (50) WITH APPROXIMATE p.PartId, r.distance
          FROM VECTOR_SEARCH(TABLE = Catalog.Parts AS p, COLUMN = Embedding,
                             SIMILAR_TO = @qv, METRIC = 'cosine') AS r
          ORDER BY r.distance) AS a
)
SELECT TOP (10) COALESCE(k.PartId, v.PartId) AS PartId,
       ISNULL(1.0e0 / (60 + k.Rnk), 0) + ISNULL(1.0e0 / (60 + v.Rnk), 0) AS RrfScore
FROM Kw AS k
FULL OUTER JOIN Vec AS v ON v.PartId = k.PartId
ORDER BY RrfScore DESC, PartId;
```

Evaluation, recall at 10 for one query:

```sql
WITH Exact AS
(
    SELECT TOP (10) PartId
    FROM Catalog.Parts
    ORDER BY VECTOR_DISTANCE('cosine', Embedding, @qv), PartId      -- exact kNN, never uses the index
),
Approx AS
(
    SELECT TOP (10) WITH APPROXIMATE p.PartId
    FROM VECTOR_SEARCH(TABLE = Catalog.Parts AS p, COLUMN = Embedding,
                       SIMILAR_TO = @qv, METRIC = 'cosine') AS r
    ORDER BY r.distance                                                  -- ANN via the DiskANN index
)
SELECT COUNT(*) / 10.0 AS RecallAt10
FROM Exact AS e
WHERE EXISTS (SELECT 1 FROM Approx AS a WHERE a.PartId = e.PartId);
```

## 4. The question (ask exactly this)

"Which strategy is correct? Option a, option b, option c, or option d?"

Options in full:

- **a.** Search 1 full-text with CONTAINS; Search 2 DiskANN vector index with VECTOR_SEARCH, cosine; Search 3 hybrid, full-text ranking plus vector ranking fused with reciprocal rank fusion; evaluation by recall@10 of VECTOR_SEARCH against exact ORDER BY VECTOR_DISTANCE on a sample of real queries, plus latency and build time, using exact search when a WHERE filter leaves at most a few tens of thousands of rows.
- **b.** Vector search for all three, embedding codes and prose together, one DiskANN index; evaluation by the average cosine distance of the returned rows.
- **c.** Full-text for all three, with FORMSOF(THESAURUS) and per-language thesaurus files; no hybrid; evaluation by counting queries that return at least one row.
- **d.** Search 1 full-text; Searches 2 and 3 the DiskANN index alone; evaluation by checking every production query against an exact 12-million-row scan and rebuilding the index on any difference.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

- Search 1: full-text is token-exact. `"BRG-62*"` in double quotes is a prefix term, `bearing AND NOT sealed` is boolean logic. One character off does not match.
- Search 2: third-generation OpenAI embeddings handle multilingual paraphrase; on 12 million rows an exhaustive VECTOR_DISTANCE scan is too slow, so a DiskANN index with VECTOR_SEARCH, queried as `SELECT TOP (n) WITH APPROXIMATE ... ORDER BY distance`, is right. The older TOP_N argument is deprecated, rejected by current-version indexes with Msg 42274.
- Search 3: hybrid. Full-text alone misses paraphrases; vector alone ranks an exact model name only approximately first. Reciprocal rank fusion, score equals the sum of one over k plus rank with k equal to 60, merges the two lists without comparing score scales.
- Evaluation: recall at k is the proportion of approximate neighbours that match the exact neighbours; perfect is 1. Measure it on a sample, record latency and index build time, and use exact search when filters leave roughly 50,000 or fewer candidate vectors, where recall is 1 by definition.

Why the others are wrong, one line each:

- **b.** An embedding model treats a part number as an opaque token, so a one-character-off code will match, and AND NOT cannot be expressed as nearest neighbour. Average cosine distance measures closeness of whatever came back, not whether the right rows came back.
- **c.** Full-text is lexical. A thesaurus maps listed words to listed synonyms; it cannot map a paraphrase to anti-slip lagging or translate languages. FREETEXT matches inflections and thesaurus forms of the same words, not meaning. Counting non-empty results measures nothing about relevance.
- **d.** Search 1 and 2 are right, but Search 3 fails: an exact model name is ranked first deterministically by full-text, while its embedding competes with similar products and may land fifth. The evaluation is unworkable: checking every production query against the exact scan means running the exact search anyway, and any recall below 1 is the accepted ANN trade-off, not a fault.

## 6. Hint ladder (one hint per attempt, in order)

1. "Ask what question each technique answers. Full-text asks which rows contain these tokens. Vector search asks which rows mean something similar. Which of the three searches is about tokens, and which is about meaning?"
2. "Search one says a part number one character off must not match, and it needs AND NOT. Can an embedding model tell two codes apart by one character, and can it express NOT?"
3. "Search two has Spanish and German queries with no word overlap. Can a thesaurus file translate a paraphrase it was never told about?"
4. "Option b sends codes and booleans through an embedding model, and grades the index by average distance. Neither part works. That eliminates b."
5. "Option c relies on FREETEXT and thesaurus files to understand meaning across languages. Full-text is lexical. That eliminates c."
6. "You are down to a and d. Both agree on searches one and two. Look at search three, the single box that needs keyword precision and semantic recall, and at how each option evaluates the approximate index. Does one of them run the exact scan for every production query?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "b, embeddings understand everything including codes" | Believes an embedding preserves identifier exactness | "How different are the vectors of BRG-6205-2RS and BRG-6205-2RZ? And how would you say AND NOT sealed as a nearest-neighbour query?" |
| "c, FREETEXT matches meaning, the docs say so" | Overreads the FREETEXT documentation | "Meaning in what sense? Inflections and listed synonyms of the same words, or a paraphrase in another language?" |
| "d, the vector index will rank the exact name first anyway" | Ignores that ANN ranking is approximate and semantic | "Is a vector ranking deterministic for an exact token? What does full-text do with that same token?" |
| "d, verifying every query against the exact scan is the safest evaluation" | Misunderstands ANN as a bug when recall is below 1 | "If you run the exact scan for every query, what is the index for? And is recall below one a fault or a trade-off?" |
| "a, but average distance would also work as a metric" | Confuses closeness with correctness | "If the index misses the true nearest rows and returns slightly farther ones, does the average distance look bad?" |
| "a, but exact search on twelve million rows is always too slow" | Forgets the filtered-candidate guidance | "What if a WHERE on category leaves thirty thousand rows? What recall does exact search have then?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the three techniques and what each is blind to:

- Full-text, CONTAINS and CONTAINSTABLE, answers which rows contain these tokens. Strong at exact codes, prefixes in double quotes with the asterisk inside the quotes, phrases, AND, OR, AND NOT, NEAR, and inflections through FORMSOF INFLECTIONAL. Blind to meaning, other languages and paraphrase. That is Search 1.
- Vector, VECTOR_DISTANCE for exact and VECTOR_SEARCH with a DiskANN index for approximate, answers which rows mean something similar. Strong at paraphrase, intent and multilingual retrieval. Blind to exact codes, because a part number is an opaque token to an embedding model. That is Search 2. The current query form is SELECT TOP n WITH APPROXIMATE, ORDER BY distance ascending and nothing else. Vector indexes are still preview; the current-version index exists in Azure SQL Database and SQL database in Fabric, while SQL Server 2025 builds the earlier version queried with TOP_N.
- Hybrid runs both and fuses the rankings with reciprocal rank fusion: each list contributes one over k plus rank, k equal to 60, and the sums are ordered descending. Keyword precision and semantic recall in one box. That is Search 3, and why option d fails.

Then the evaluation:

- Recall at k is the size of the intersection of the ANN top k and the exact top k, divided by k. Measure it on a sample of real queries against ORDER BY VECTOR_DISTANCE, which never uses the index. Perfect is 1. Record latency, for example with SET STATISTICS TIME ON or sys.dm_exec_query_stats, and the elapsed time and CPU of CREATE VECTOR INDEX. ANN trades some recall for speed; that is expected, not a fault.
- Exact search is fine for small sets, roughly 50,000 candidate vectors or fewer after WHERE filtering, and has recall 1 by definition. So bypass the index when filters are selective.
- Maintenance: current-version DiskANN indexes accept INSERT, UPDATE, DELETE and MERGE and are maintained in the background, visible in sys.dm_db_vector_indexes; recreate after a near-complete re-embedding because recall degrades. Earlier-version indexes made the table read-only unless ALLOW_STALE_VECTOR_INDEX was set.

Memory hook: "Tokens: full-text. Meaning: vector. Both in one box: hybrid with RRF. Grade ANN by recall against exact, and go exact when the filter is small."

## 9. Follow-up oral questions (optional)

1. "How is reciprocal rank fusion computed for one row that appears in both lists?" (One over sixty plus its keyword rank, plus one over sixty plus its vector rank; rows in only one list get one term.)
2. "Roughly how many candidate vectors can an exact VECTOR_DISTANCE search handle comfortably, according to the guidance?" (About 50,000 or fewer after filtering.)
3. "Which clause must accompany SELECT TOP n WITH APPROXIMATE, and what happens if you use the old TOP_N argument on a current-version index?" (ORDER BY distance ascending; TOP_N is deprecated and rejected with message 42274.)

## 10. References

- Vector search and vector indexes in SQL: https://learn.microsoft.com/en-us/sql/relational-databases/vectors/vectors-sql-server
- Create and use vector indexes, DiskANN: https://learn.microsoft.com/en-us/sql/relational-databases/vectors/vector-indexes
- VECTOR_SEARCH: https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-search-transact-sql
- VECTOR_DISTANCE: https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-distance-transact-sql
- CONTAINS, prefix terms and boolean logic: https://learn.microsoft.com/en-us/sql/t-sql/queries/contains-transact-sql
- CONTAINSTABLE: https://learn.microsoft.com/en-us/sql/relational-databases/system-functions/containstable-transact-sql
- Hybrid search sample, reciprocal rank fusion: https://github.com/Azure-Samples/azure-sql-db-vector-search
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
