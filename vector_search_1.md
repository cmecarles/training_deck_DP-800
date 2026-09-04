# SQL Server question — Vector Search 1

## Statement

A music-streaming service stores song-lyrics theme embeddings in Azure SQL Database so that listeners can search for songs by theme ("songs about the freedom of the open road") instead of by keyword.

For this exercise the embedding model produces tiny 3-dimensional vectors. The three dimensions encode, in order, the strength of three lyric themes: *(open road / freedom, heartbreak, nostalgia)*.

The database is built from scratch by the following complete script:

```sql
CREATE DATABASE TuneFinder;
GO
USE TuneFinder;
GO
CREATE SCHEMA Music;
GO
CREATE TABLE Music.Songs
(
    SongId      INT IDENTITY(1,1) PRIMARY KEY,
    Title       NVARCHAR(100) NOT NULL,
    ThemeVector VECTOR(3)     NOT NULL
);
GO
INSERT INTO Music.Songs (Title, ThemeVector) VALUES
    (N'Open Road Anthem',        '[1, 0, 0]'),
    (N'Miles and Regrets',       '[4, 3, 0]'),
    (N'Tears on the Interstate', '[3, 4, 0]'),
    (N'Midnight Heartache',      '[0, 1, 0]'),
    (N'Never Leaving Home',      '[-2, 0, 0]');
GO
```

A user searches for "songs about the freedom of the open road". The (toy) embedding of that search phrase is `[1, 0, 0]`.

Two candidate queries are run:

```sql
-- Query 1
DECLARE @q VECTOR(3) = '[1, 0, 0]';

SELECT TOP (3)
    s.Title,
    VECTOR_DISTANCE('cosine', s.ThemeVector, @q) AS Distance
FROM Music.Songs AS s
ORDER BY Distance ASC, s.SongId ASC;
```

```sql
-- Query 2
DECLARE @q VECTOR(3) = '[1, 0, 0]';

SELECT TOP (3)
    s.Title,
    VECTOR_DISTANCE('euclidean', s.ThemeVector, @q) AS Distance
FROM Music.Songs AS s
ORDER BY Distance ASC, s.SongId ASC;
```

Answer all three parts:

### A.

Predict the exact result of **Query 1**: the three titles, in order, with the mathematically exact distance of each.

### B.

Predict the exact result of **Query 2**: the three titles, in order, with the mathematically exact distance of each.

### C.

A teammate proposes speeding up Query 1 by creating a DiskANN vector index on `ThemeVector` with `CREATE VECTOR INDEX` and rewriting the query to use the `VECTOR_SEARCH` function with `METRIC = 'cosine'`. **True or false:** the rewritten query is guaranteed to return exactly the same three rows as Query 1.

## Correct Answer

**A. Query 1 (cosine):**

| # | Title | Distance |
|---|---|---|
| 1 | Open Road Anthem | 0.0 |
| 2 | Miles and Regrets | 0.2 |
| 3 | Tears on the Interstate | 0.4 |

**B. Query 2 (euclidean):**

| # | Title | Distance |
|---|---|---|
| 1 | Open Road Anthem | 0.0 |
| 2 | Midnight Heartache | √2 ≈ 1.4142 |
| 3 | Never Leaving Home | 3.0 |

The two queries agree only on the first row. The two "road" songs that cosine ranks 2nd and 3rd are the two **farthest** songs by euclidean distance.

(The engine returns **float** values, so the displayed decimals may show binary rounding in the last digits — e.g. `0.2` may print as `0.20000000298…`. The ranking, which is what the question asks for, is unaffected.)

**C. False.** `VECTOR_SEARCH` over a vector index performs **approximate** nearest-neighbor (ANN) search and may miss true neighbors (recall < 1). `ORDER BY VECTOR_DISTANCE(...)` is always an **exact** (ENN/kNN) search and never uses a vector index, even if one exists.

## Explanation

### Step 0 — decode the table

`SongId` is `IDENTITY(1,1)`, so the five songs are numbered 1–5 in insert order:

| SongId | Title | ThemeVector |
|---|---|---|
| 1 | Open Road Anthem | (1, 0, 0) |
| 2 | Miles and Regrets | (4, 3, 0) |
| 3 | Tears on the Interstate | (3, 4, 0) |
| 4 | Midnight Heartache | (0, 1, 0) |
| 5 | Never Leaving Home | (−2, 0, 0) |

The query vector is **q = (1, 0, 0)**, with norm ‖q‖ = 1.

The vectors were chosen so that every quantity below is exact: (4, 3, 0) and (3, 4, 0) are 3-4-5 triangles (‖v‖ = 5), and every dot product with q is just the first component.

### Step 1 — dot products and norms

| Song | v | v · q | ‖v‖ |
|---|---|---|---|
| 1 | (1, 0, 0) | 1 | 1 |
| 2 | (4, 3, 0) | 4 | √(16+9+0) = √25 = **5** |
| 3 | (3, 4, 0) | 3 | √(9+16+0) = √25 = **5** |
| 4 | (0, 1, 0) | 0 | 1 |
| 5 | (−2, 0, 0) | −2 | √4 = **2** |

### Step 2 — cosine distance (Query 1)

`VECTOR_DISTANCE('cosine', v, q)` returns the **cosine distance**:

```text
cosine distance = 1 − cosine similarity = 1 − (v · q) / (‖v‖ · ‖q‖)
```

Its range is [0, 2]: 0 for identical direction, 1 for orthogonal, 2 for opposite direction. **It depends only on the angle between the vectors — magnitude is divided out.**

| Song | cos similarity | cosine distance |
|---|---|---|
| 1 Open Road Anthem | 1 / (1·1) = 1 | 1 − 1 = **0** |
| 2 Miles and Regrets | 4 / (5·1) = 0.8 | 1 − 0.8 = **0.2** |
| 3 Tears on the Interstate | 3 / (5·1) = 0.6 | 1 − 0.6 = **0.4** |
| 4 Midnight Heartache | 0 / (1·1) = 0 | 1 − 0 = **1** |
| 5 Never Leaving Home | −2 / (2·1) = −1 | 1 − (−1) = **2** |

Ascending order: 0 < 0.2 < 0.4 < 1 < 2, no ties (the `SongId` tiebreaker in the `ORDER BY` never fires). `TOP (3)` keeps songs 1, 2, 3.

This matches the semantics: songs 2 and 3 both mix "road" and "heartbreak", but song 2 leans more toward "road" (4 vs 3 on the first axis), so it is angularly closer to the pure "road" query. The large magnitudes (norm 5) are irrelevant to cosine.

### Step 3 — euclidean distance (Query 2)

`VECTOR_DISTANCE('euclidean', v, q)` returns ‖v − q‖, the L2 norm of the difference. **It depends on magnitude as well as direction.**

| Song | v − q | euclidean distance |
|---|---|---|
| 1 Open Road Anthem | (0, 0, 0) | **0** |
| 2 Miles and Regrets | (3, 3, 0) | √(9+9) = √18 = 3√2 ≈ **4.2426** |
| 3 Tears on the Interstate | (2, 4, 0) | √(4+16) = √20 = 2√5 ≈ **4.4721** |
| 4 Midnight Heartache | (−1, 1, 0) | √(1+1) = √2 ≈ **1.4142** |
| 5 Never Leaving Home | (−3, 0, 0) | √9 = **3** |

Ascending order: 0 < √2 < 3 < √18 < √20. `TOP (3)` keeps songs 1, 4, 5.

### The trap — why the two rankings disagree almost everywhere

Full rankings side by side:

| Rank | cosine | euclidean |
|---|---|---|
| 1 | Open Road Anthem (0) | Open Road Anthem (0) |
| 2 | Miles and Regrets (0.2) | Midnight Heartache (√2) |
| 3 | Tears on the Interstate (0.4) | Never Leaving Home (3) |
| 4 | Midnight Heartache (1) | Miles and Regrets (√18) |
| 5 | Never Leaving Home (2) | Tears on the Interstate (√20) |

- **Cosine ignores magnitude.** (4, 3, 0) is a long vector, but after dividing by its norm only the angle survives, and its angle to (1, 0, 0) is small.
- **Euclidean punishes magnitude.** (4, 3, 0) and (3, 4, 0) sit far from the short query vector purely because they are long, so the two most road-themed songs drop to the bottom. Meanwhile (0, 1, 0) — a song with *zero* road content, orthogonal to the query — becomes the 2nd nearest neighbor simply because it is short. Even (−2, 0, 0), whose theme is *diametrically opposed* to the query (cosine distance 2, the maximum), beats both road songs.
- With embeddings, vector length usually reflects things like text length or theme intensity, not meaning — which is why **cosine is the default choice for semantic similarity**, and why euclidean ranking is only equivalent to cosine ranking when all vectors are first normalized to unit length. For unit vectors, ‖a − b‖² = 2 − 2·cos(a, b), a monotonic relationship, so after `VECTOR_NORMALIZE(v, 'norm2')` on every vector the euclidean `TOP (3)` would return the same rows as Query 1.
- For completeness, the third supported metric, `'dot'`, returns the **negative** dot product (−1, −4, −3, 0, 2 for songs 1–5), so it would rank *Miles and Regrets* (−4) **above** the exactly-matching *Open Road Anthem* (−1): the raw dot product rewards magnitude. Three metrics, three different winners of rank 2 — the metric string is never a cosmetic choice.

### Part C — ENN vs ANN

`VECTOR_DISTANCE` is documented as **always exact**: it never uses a vector index, even if one exists. `SELECT TOP (N) … ORDER BY VECTOR_DISTANCE(…)` is an exact k-nearest-neighbor (kNN / ENN) scan: every row's distance is computed, so the true top 3 is guaranteed.

`VECTOR_SEARCH` is the opposite by design: used with a vector index (DiskANN-based, created with `CREATE VECTOR INDEX`), it performs **approximate** nearest-neighbor (ANN) search. (If no compatible vector index exists, the engine raises a warning and falls back to an exact kNN scan.) That fallback is the behaviour of the current Azure SQL Database / Fabric SQL database vector-index version; on SQL Server 2025 RTM, where only the earlier index version exists, `VECTOR_SEARCH` without a matching index fails with Msg 42227 instead, as [vector_index_1.md](vector_index_1.md) shows. ANN trades recall for speed: it navigates the DiskANN graph instead of scanning all vectors, and may return a neighbor set that misses some true nearest neighbors (recall < 1). Therefore it is *not guaranteed* to return the same rows — the statement is **false**. (On 5 rows it would almost certainly match in practice — in fact, current vector indexes require at least 100 rows before they can even be created, so the toy table could not build the index at all — but the question asks about a *guarantee*; exact search is the recommendation for small filtered sets — Microsoft's guidance is roughly under 50,000 vectors — while ANN pays off at scale. Note also that the approximate vector index and `VECTOR_SEARCH` are in preview.)

### Equivalent alternative formulations of Query 1

All of the following return the same rows in the same order:

```sql
-- Repeat the expression in ORDER BY instead of using the column alias
SELECT TOP (3)
    s.Title,
    VECTOR_DISTANCE('cosine', s.ThemeVector, @q) AS Distance
FROM Music.Songs AS s
ORDER BY VECTOR_DISTANCE('cosine', s.ThemeVector, @q) ASC, s.SongId ASC;
```

```sql
-- Inline the query vector with an explicit CAST instead of a variable
SELECT TOP (3)
    s.Title,
    VECTOR_DISTANCE('cosine', s.ThemeVector,
                    CAST('[1, 0, 0]' AS VECTOR(3))) AS Distance
FROM Music.Songs AS s
ORDER BY Distance ASC, s.SongId ASC;
```

```sql
-- Build the query vector from JSON_ARRAY (implicit conversion on assignment)
DECLARE @q VECTOR(3) = JSON_ARRAY(1, 0, 0);
SELECT TOP (3)
    s.Title,
    VECTOR_DISTANCE('cosine', s.ThemeVector, @q) AS Distance
FROM Music.Songs AS s
ORDER BY Distance ASC, s.SongId ASC;
```

Swapping the last two arguments — `VECTOR_DISTANCE('cosine', @q, s.ThemeVector)` — is also equivalent: cosine and euclidean distances are symmetric. Omitting the `s.SongId` tiebreaker changes nothing *here* because all five distances are distinct, but with real embeddings a deterministic tiebreaker is the only way to make the row order fully reproducible.

Conceptual question (Azure / tooling); not executed against an engine.

## DP-800 Exam Rule to Remember

The `vector` type is declared as `VECTOR(n)` (float32 elements by default, max 1998 dimensions) and its literals are JSON arrays in a string: `'[1, 0, 0]'`. Inspect a vector with `VECTORPROPERTY(v, 'Dimensions')` / `VECTORPROPERTY(v, 'BaseType')`; rescale to unit length with `VECTOR_NORMALIZE(v, 'norm2')`.

`VECTOR_DISTANCE(metric, v1, v2)` supports exactly three metrics, and the metric decides the ranking:

```text
'cosine'    = 1 − cos(angle)      range [0, 2]   angle only — magnitude divided out
'euclidean' = ‖v1 − v2‖           range [0, ∞)   magnitude matters
'dot'       = −(v1 · v2)          range (−∞, ∞)  magnitude rewarded
```

Smaller is always more similar — including `'dot'`, because it is the *negative* dot product.

And keep the two search modes straight:

```text
ORDER BY VECTOR_DISTANCE(...)   → exact  (ENN / kNN)  — never uses a vector index
VECTOR_SEARCH(... vector index) → approximate (ANN)   — DiskANN, recall may be < 1
```

If the question says "guaranteed exact results", the answer uses `VECTOR_DISTANCE`; if it says "large table, fastest similar-item lookup, small accuracy trade-off acceptable", the answer is `CREATE VECTOR INDEX` + `VECTOR_SEARCH`.
