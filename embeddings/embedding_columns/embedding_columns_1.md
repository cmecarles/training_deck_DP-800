# SQL Server question — Embedding Columns 1

## Statement

`GearHub` is the product catalog database of an outdoor-equipment retailer (Azure SQL Database). Two tables matter for the new semantic search:

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

Requirements for the search feature:

1. A query such as *"waterproof jacket for alpine hiking under 150"* must return products by **what they are** (name, category, descriptions) and by **how customers describe them** (review text). The price limit must be applied **exactly** — a 151.00 product must never be returned for "under 150".
2. Embeddings are billed per call. An embedding may be regenerated **only when the text it represents actually changed**; a stock update or a price change must not trigger a re-embed.
3. Long descriptions exceed what one embedding can represent well; retrieval must be able to point to the relevant passage.

Which design correctly decides what goes into the embedding text and what stays outside it?

### a.

One embedding per product row, built from every column so nothing is lost:

```sql
CONCAT_WS(N' ', ProductId, Sku, Name, CategoryName, ShortDescription, LongDescription,
          Price, IsActive, StockQty, CreatedAt, ModifiedAt)
```

Re-embed a row whenever `RowVersion` changes.

### b.

Build a labelled text from the semantic columns only, one embedding per **chunk** of it, and one embedding per **review**; keep everything else as filter columns; detect changes with a hash of the embedded text:

```sql
-- product text (chunked into Catalog.ProductChunks with a ProductId FK + Ordinal)
CONCAT_WS(N' | ', N'Product: ' + Name, N'Category: ' + CategoryName,
          ShortDescription, LongDescription)
-- review text: one embedding per Catalog.Reviews row (ProductId FK)
-- change detection: ContentHash = HASHBYTES('SHA2_256', <that text>), re-embed only when it differs
-- Price, IsActive, StockQty: WHERE Price <= 150 AND IsActive = 1 AND StockQty > 0 on the search query
```

### c.

Embed only the short, cheap columns and let a full-text index handle the rest:

```sql
CONCAT_WS(N' ', Sku, Name, CategoryName)    -- one embedding per product
```

`LongDescription` and `ReviewText` are covered by `CONTAINS(...)`; re-embed when `Name` or `CategoryName` changes.

### d.

One embedding per product from the descriptive columns plus the numeric facts expressed as sentences, so the model can reason about them:

```sql
CONCAT_WS(N' ', Name, CategoryName, ShortDescription, LongDescription,
          N'Price: ' + CAST(Price AS NVARCHAR(20)),
          N'Average rating: ' + CAST((SELECT AVG(Rating * 1.0) FROM Catalog.Reviews r
                                       WHERE r.ProductId = p.ProductId) AS NVARCHAR(10)))
```

Re-embed a row whenever `ModifiedAt` is later than the embedding's timestamp.

## Correct Answer

**b**

## Explanation

Deciding which columns to embed is a classification exercise: a column goes into the embedding text only if its **words carry meaning** that a buyer's natural-language query could match. Everything else is **metadata** that belongs in `WHERE` clauses and joins.

| Column | Semantic? | Role |
|---|---|---|
| `Name`, `CategoryName`, `ShortDescription`, `LongDescription`, `ReviewText` | yes | embedding text |
| `Sku`, `ProductId`, `ReviewId` | no (opaque tokens) | keys / exact-match lookups |
| `Price`, `Rating`, `StockQty`, `IsActive` | no (numbers, flags) | exact filters and sorts |
| `CreatedAt`, `ModifiedAt`, `RowVersion` | no | change bookkeeping, never embedded |

### Why option b is correct

- **Semantic columns only, with labels.** `Product: Stratos 3L Shell | Category: Jackets | ...` gives the model the words a buyer would use and a small amount of structure. Labels are cheap and help the model separate "the name" from "the category" when both are short. (The concatenation and the hash were executed on SQL Server 2025 as a sanity check: `CONCAT_WS(N' | ', N'Title: ' + N'Trail Runner 3', N'Category: ' + N'Footwear', N'Description: ' + N'Lightweight shoe for rocky trails')` returns `Title: Trail Runner 3 | Category: Footwear | Description: Lightweight shoe for rocky trails`, and `HASHBYTES('SHA2_256', ...)` of the same three values returns the 32-byte hash `0x5EC18ABF…D8E92C91`.)
- **Filters stay filters.** "Under 150" is an *inequality*; an embedding cannot express "≤ 150" reliably. `WHERE Price <= 150 AND IsActive = 1 AND StockQty > 0` is exact, index-friendly, and — with a vector index — can be applied as a pre/iterative filter so the vector search only ranks eligible rows.
- **One embedding per chunk, one per review.** A 6,000-word description is far past the point where a single vector represents it well, and the requirement asks to point at the relevant passage: a `Catalog.ProductChunks` table (`ProductId` FK, `Ordinal`, `ChunkText`, `Embedding`) gives one vector per passage. Reviews are separate rows written by different people; embedding each `ReviewText` row on its own keeps "how customers describe it" searchable and lets a hit be traced to the review. Both chunk and review hits join back to `ProductId`.
- **Hash-based change detection.** `ContentHash = HASHBYTES('SHA2_256', <embedding text>)` changes **if and only if** the embedded text changes. A stock update, a price change, or a `ModifiedAt` bump leaves the hash unchanged, so no call is billed. Compare hashes in the maintenance job and re-embed only the rows (or chunks) whose hash differs.

### Why option a is wrong

Concatenating every column pollutes the vector with tokens that have no meaning to a query: the surrogate key `ProductId`, the SKU code, a `bit`, a stock count, two timestamps. Those tokens dilute the semantic signal, and worse, they make the text **volatile**: `StockQty` and `ModifiedAt` change many times a day, so the text (and `RowVersion`, which changes on *every* update of the row) changes constantly. Requirement 2 is violated: the job would re-embed rows whose meaning never changed. Embedding `Price` also does not deliver the exact "under 150" guarantee. And one vector per product cannot satisfy requirement 3 for long descriptions.

### Why option c is wrong

It embeds the wrong side of the data. `Sku` is an opaque code (`JK-STR3L-BLU`) and adds noise; `Name` and `CategoryName` alone (`Stratos 3L Shell`, `Jackets`) do not say *waterproof*, *alpine*, or *hiking* — those words live in the descriptions and reviews, which option c leaves out of the embeddings. Full-text `CONTAINS` on the descriptions is a keyword search: it finds "waterproof" only if that exact word (or an inflectional form) appears, not "keeps you dry in a storm". Requirement 1 asks for semantic retrieval over the descriptive text, so the descriptive text must be embedded. (Full-text is a fine *complement* in a hybrid design, not a replacement for embedding the semantic columns.)

### Why option d is wrong

This is the subtle distractor: the descriptive columns are right, and "Price: 149.99" as a sentence *feels* like it gives the model the number. It fails on two points:

- **Numbers as text do not give ordering semantics.** An embedding of "Price: 149.99" is close to "Price: 151.00" and to "Price: 14.99" in ways that have nothing to do with `<= 150`. A 151.00 jacket can and will surface for "under 150"; only a `WHERE Price <= 150` predicate is exact. Similarly the average rating changes with every new review, forcing a re-embed of the whole product text for a value that should be a filter/sort column.
- **`ModifiedAt` is the wrong change signal.** The column is bumped by *every* update, including the many daily `StockQty` changes, so the row would be re-embedded although its text is unchanged. Change detection must be tied to the embedded **content** (a hash of the text), not to a row-level timestamp.

Verified against SQL Server 2025 (RTM 17.0.1000.7) for the `CONCAT_WS` / `HASHBYTES` snippet quoted above; the design comparison itself is conceptual.

## DP-800 Exam Rule to Remember

```text
INTO the embedding text        OUT of it (metadata)
------------------------       ------------------------------------------
title / name                   surrogate keys, GUIDs, SKUs, codes
category / tag names           prices, quantities, ratings (filter / sort)
descriptions, body, reviews    booleans / status flags (filter)
                               timestamps, rowversion (change bookkeeping)
```

Three rules that follow:

1. **Label and concatenate** the semantic columns (`Product: … | Category: … | …`); keep exact predicates (`Price <= 150`, `IsActive = 1`) in the `WHERE` clause.
2. **Granularity follows the text**: one embedding per row for short texts; one per chunk (child table with FK + ordinal) for long texts; one per review/comment row when many people write about one product.
3. **Re-embed on content change only**: store `ContentHash = HASHBYTES('SHA2_256', embedded_text)` and compare hashes; never key re-embedding on `ModifiedAt` or `ROWVERSION`, which move for non-semantic updates.
