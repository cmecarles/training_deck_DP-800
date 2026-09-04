# Instructor-Examiner guide — Embedding Columns 1

Companion to [embedding_columns_1.md](embedding_columns_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice design question with four options, a to d. There is no result set to predict; the learner must judge each design against three requirements. Read all three requirements and all four options before taking an answer. When the learner picks a letter, ask them to say which requirement each of the other options fails.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Implement AI capabilities in database solutions (25–30%).
- Skill: Design and implement models and embeddings.
- Task bullet: Identify which columns to embed for semantic search.
- What is tested: separating semantic text from metadata, keeping exact predicates in the WHERE clause, choosing the granularity of embeddings, and tying re-embedding to content change rather than row timestamps.

## 2. Scenario to read aloud

**Piece 1, the story.** "GearHub is the product catalog database of an outdoor equipment retailer, on Azure SQL Database. The retailer is adding semantic search. Two tables matter: Products and Reviews, both in a schema called Catalog."

**Piece 2, the Products table.** "Catalog dot Products has twelve columns. ProductId, an integer identity, the primary key. Sku, a char of twelve, unique, a code like J K dash S T R 3 L dash B L U. Name, an NVARCHAR of 120, for example Stratos 3L Shell. CategoryName, an NVARCHAR of sixty, for example Jackets. ShortDescription, an NVARCHAR of four hundred. LongDescription, an NVARCHAR MAX, between two hundred and six thousand words. Price, a decimal nine comma two. IsActive, a bit. StockQty, an integer, updated many times a day. CreatedAt and ModifiedAt, both DATETIME2 zero; ModifiedAt is bumped by every UPDATE, including stock changes. And RowVersion, a ROWVERSION column."

**Piece 3, the Reviews table.** "Catalog dot Reviews has four columns. ReviewId, an integer identity, the primary key. ProductId, an integer, foreign key to Products. Rating, a tinyint. And ReviewText, an NVARCHAR of two thousand."

**Piece 4, requirement 1.** "Three requirements. Requirement 1. A query such as: waterproof jacket for alpine hiking under 150, must return products by what they are, meaning name, category and descriptions, and by how customers describe them, meaning review text. The price limit must be applied exactly. A product priced 151 must never be returned for under 150."

**Piece 5, requirement 2.** "Requirement 2. Embeddings are billed per call. An embedding may be regenerated only when the text it represents actually changed. A stock update or a price change must not trigger a re-embed."

**Piece 6, requirement 3.** "Requirement 3. Long descriptions exceed what one embedding can represent well. Retrieval must be able to point to the relevant passage."

**Piece 7, option a.** "Option a. One embedding per product row, built from every column so nothing is lost. The text is CONCAT underscore WS with a space separator over ProductId, Sku, Name, CategoryName, ShortDescription, LongDescription, Price, IsActive, StockQty, CreatedAt and ModifiedAt. Re-embed a row whenever RowVersion changes."

**Piece 8, option b.** "Option b. Build a labelled text from the semantic columns only. The text is CONCAT underscore WS with a pipe separator over: the string Product colon plus Name; the string Category colon plus CategoryName; ShortDescription; and LongDescription. That text is chunked into a child table, Catalog dot ProductChunks, with a ProductId foreign key and an Ordinal, and there is one embedding per chunk. Separately, one embedding per review row in Catalog dot Reviews. Change detection uses a ContentHash column, HASHBYTES SHA2 underscore 256 of the embedded text, and re-embeds only when the hash differs. Price, IsActive and StockQty stay outside as filter columns: WHERE Price less than or equal to 150 AND IsActive equals 1 AND StockQty greater than 0 on the search query."

**Piece 9, option c.** "Option c. Embed only the short, cheap columns and let a full-text index handle the rest. The text is CONCAT underscore WS over Sku, Name and CategoryName, one embedding per product. LongDescription and ReviewText are covered by CONTAINS. Re-embed when Name or CategoryName changes."

**Piece 10, option d.** "Option d. One embedding per product from the descriptive columns plus the numeric facts expressed as sentences, so the model can reason about them. The text is CONCAT underscore WS over Name, CategoryName, ShortDescription, LongDescription, then the string Price colon plus the price cast to text, then the string Average rating colon plus the average of Rating from Reviews for that product, cast to text. Re-embed a row whenever ModifiedAt is later than the embedding's timestamp."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE SCHEMA Catalog;
GO
CREATE TABLE Catalog.Products
(
    ProductId        INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    Sku              CHAR(12)          NOT NULL UNIQUE,      -- e.g. 'JK-STR3L-BLU'
    Name             NVARCHAR(120)     NOT NULL,             -- e.g. 'Stratos 3L Shell'
    CategoryName     NVARCHAR(60)      NOT NULL,             -- e.g. 'Jackets'
    ShortDescription NVARCHAR(400)     NOT NULL,
    LongDescription  NVARCHAR(MAX)     NOT NULL,             -- 200 - 6,000 words
    Price            DECIMAL(9,2)      NOT NULL,
    IsActive         BIT               NOT NULL,
    StockQty         INT               NOT NULL,             -- updated many times a day
    CreatedAt        DATETIME2(0)      NOT NULL,
    ModifiedAt       DATETIME2(0)      NOT NULL,             -- bumped by every UPDATE, including stock changes
    RowVersion       ROWVERSION        NOT NULL
);
GO
CREATE TABLE Catalog.Reviews
(
    ReviewId   INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    ProductId  INT               NOT NULL REFERENCES Catalog.Products (ProductId),
    Rating     TINYINT           NOT NULL,
    ReviewText NVARCHAR(2000)    NOT NULL
);
GO
```

Option a:

```sql
CONCAT_WS(N' ', ProductId, Sku, Name, CategoryName, ShortDescription, LongDescription,
          Price, IsActive, StockQty, CreatedAt, ModifiedAt)
-- re-embed whenever RowVersion changes
```

Option b:

```sql
-- product text (chunked into Catalog.ProductChunks with a ProductId FK + Ordinal)
CONCAT_WS(N' | ', N'Product: ' + Name, N'Category: ' + CategoryName,
          ShortDescription, LongDescription)
-- review text: one embedding per Catalog.Reviews row (ProductId FK)
-- change detection: ContentHash = HASHBYTES('SHA2_256', <that text>), re-embed only when it differs
-- Price, IsActive, StockQty: WHERE Price <= 150 AND IsActive = 1 AND StockQty > 0 on the search query
```

Option c:

```sql
CONCAT_WS(N' ', Sku, Name, CategoryName)    -- one embedding per product
-- LongDescription and ReviewText covered by CONTAINS(...); re-embed when Name or CategoryName changes
```

Option d:

```sql
CONCAT_WS(N' ', Name, CategoryName, ShortDescription, LongDescription,
          N'Price: ' + CAST(Price AS NVARCHAR(20)),
          N'Average rating: ' + CAST((SELECT AVG(Rating * 1.0) FROM Catalog.Reviews r
                                       WHERE r.ProductId = p.ProductId) AS NVARCHAR(10)))
-- re-embed whenever ModifiedAt is later than the embedding's timestamp
```

## 4. The question (ask exactly this)

"Which design correctly decides what goes into the embedding text and what stays outside it? Option a, every column in one embedding per product, re-embed on RowVersion. Option b, labelled semantic columns, one embedding per chunk and one per review, hash-based change detection, price and flags as WHERE filters. Option c, embed only Sku, Name and Category, full-text CONTAINS for the rest. Option d, descriptive columns plus price and average rating as sentences, re-embed on ModifiedAt. Give me one letter, and tell me which requirement each of the others fails."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct: b.** Only the columns whose words carry meaning go into the text: Name, CategoryName, ShortDescription, LongDescription, and ReviewText separately. Price, IsActive and StockQty stay as exact WHERE predicates, so "under 150" is exact (requirement 1). A SHA2_256 hash of the embedded text changes if and only if the text changes, so stock and price updates never trigger a call (requirement 2). One embedding per chunk in a child table with ProductId and Ordinal, and one per review, lets a hit point at the passage or review (requirement 3).

- **a is wrong.** Keys, SKU, a bit, a stock count and timestamps pollute the vector, and they make the text volatile. RowVersion changes on every update, so stock changes trigger re-embeds; fails requirement 2. Price in the text does not give an exact "under 150"; one vector per product fails requirement 3.
- **c is wrong.** Sku is noise; Name and CategoryName alone do not say waterproof, alpine or hiking. Those words live in the descriptions and reviews, which are left out of the embeddings. CONTAINS is keyword search, not semantic; fails requirement 1.
- **d is wrong.** Numbers as text carry no ordering semantics; "Price: 151.00" can surface for "under 150", so requirement 1 fails. Average rating changes with every review and ModifiedAt is bumped by every stock update, so requirement 2 fails. One vector per product also fails requirement 3.

## 6. Hint ladder (one hint per attempt, in order)

1. "Sort the columns into two piles: words a buyer might type, and everything else. Which pile is the embedding text, and where does the other pile go?"
2. "Requirement 1 says a 151 product must never come back for under 150. Can a vector similarity express less than or equal to? Which options put the price inside the embedding text?"
3. "Requirement 2 says a stock update must not trigger a re-embed. Which signals move on every update of the row? RowVersion? ModifiedAt? What signal moves only when the embedded text moves?"
4. "Requirement 3 says retrieval must point to the relevant passage in a six-thousand-word description. Which option has more than one embedding per product?"
5. "One option embeds too little and hopes full-text search covers the rest. Does CONTAINS find a description that says keeps you dry in a storm when the query says waterproof?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, more columns means more information for the model" | Confuses data with signal | "What does a surrogate key or a stock count mean to a buyer's query? And how often does that text change?" |
| "d, saying the price as a sentence lets the model reason about it" | Expects ordering semantics from text similarity | "Is the embedding of Price 149.99 reliably farther from Price 151 than from Price 14.99? How do you guarantee under 150 exactly?" |
| "d, ModifiedAt is the right change signal" | Ties re-embedding to row change instead of content change | "The scenario says ModifiedAt is bumped by every stock update. Did the text change?" |
| "c, full-text search already handles descriptions" | Treats keyword search as semantic search | "Does CONTAINS match meaning or exact words and their inflections?" |
| "b, but the SKU should be embedded too so people can search by code" | Wants exact-match lookup inside the vector | "Is a SKU a word with meaning, or an opaque token? Where do exact-match lookups belong?" |
| "b is overkill, one embedding per product is enough" | Ignores requirement 3 | "How does one vector per product point at a passage inside six thousand words?" |

## 8. Teaching notes (after the answer is complete or revealed)

Deciding which columns to embed is a classification exercise. A column goes into the embedding text only if its words carry meaning a natural-language query could match. Everything else is metadata for WHERE clauses and joins.

- Semantic: Name, CategoryName, ShortDescription, LongDescription, ReviewText. Into the embedding text.
- Opaque tokens: Sku, ProductId, ReviewId. Keys and exact-match lookups.
- Numbers and flags: Price, Rating, StockQty, IsActive. Exact filters and sorts.
- Bookkeeping: CreatedAt, ModifiedAt, RowVersion. Never embedded.

Why b works:

- **Semantic columns only, with labels.** "Product: Stratos 3L Shell | Category: Jackets | ..." gives the model the buyer's words plus a little structure. Labels are cheap and help separate the name from the category when both are short. The CONCAT_WS and HASHBYTES snippet was executed on SQL Server 2025 as a sanity check.
- **Filters stay filters.** "Under 150" is an inequality; an embedding cannot express less than or equal reliably. WHERE Price <= 150 AND IsActive = 1 AND StockQty > 0 is exact and index friendly, and with a vector index it can be applied as a pre-filter so the vector search only ranks eligible rows.
- **One embedding per chunk, one per review.** A 6,000-word description is far past what one vector represents well. A ProductChunks table with ProductId, Ordinal, ChunkText and Embedding gives one vector per passage. Reviews are separate rows by different people; one embedding per ReviewText keeps "how customers describe it" searchable and traceable. Both join back to ProductId.
- **Hash-based change detection.** ContentHash = HASHBYTES('SHA2_256', text) changes if and only if the text changes. Stock, price and ModifiedAt bumps leave it unchanged, so nothing is billed. The maintenance job compares hashes and re-embeds only differing rows or chunks.

Why the others fail: a pollutes the vector with keys, codes, flags and timestamps, and RowVersion moves on every update. c embeds the wrong side of the data and relies on keyword search for the words that matter. d is the subtle one: the descriptive columns are right, but numbers as text do not give ordering, the average rating changes with every review, and ModifiedAt is bumped by stock updates.

Memory hook: "Words in, numbers out. Filters in the WHERE. One vector per passage. Re-embed on the hash, never on the timestamp."

## 9. Follow-up oral questions (optional)

1. "Where does a search by exact SKU belong in this design?" (In a WHERE clause or an index seek on the Sku column, not in the embedding.)
2. "Why is a hash of the text a better change signal than ROWVERSION?" (ROWVERSION changes on every update of the row; the hash changes only when the embedded text changes.)
3. "Is full-text search useless here?" (No. It is a fine complement in a hybrid design; it just cannot replace embedding the semantic columns.)

## 10. References

- Vector search and embeddings in SQL Server and Azure SQL: https://learn.microsoft.com/en-us/sql/relational-databases/vectors/vectors-sql-server
- Vector data type: https://learn.microsoft.com/en-us/sql/t-sql/data-types/vector-data-type
- AI_GENERATE_CHUNKS: https://learn.microsoft.com/en-us/sql/t-sql/functions/ai-generate-chunks-transact-sql
- AI_GENERATE_EMBEDDINGS: https://learn.microsoft.com/en-us/sql/t-sql/functions/ai-generate-embeddings-transact-sql
- HASHBYTES: https://learn.microsoft.com/en-us/sql/t-sql/functions/hashbytes-transact-sql
- CONCAT_WS: https://learn.microsoft.com/en-us/sql/t-sql/functions/concat-ws-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
