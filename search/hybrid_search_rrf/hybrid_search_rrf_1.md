# SQL Server question — Hybrid Search RRF 1

## Statement

A router manufacturer's support site runs on a SQL Server 2025 database named `NetHelp`. Its search box is **hybrid**: a keyword ranking and a vector ranking are computed for the same help articles and fused with **reciprocal rank fusion (RRF)**.

For this exercise the embeddings are 3-dimensional, and the keyword ranking has been materialized into a table: `Help.KeywordHits` holds, for the query *"router password"*, the `KEY` / `RANK` pairs that `CONTAINSTABLE` returned (only matching articles appear; a higher `Score` means a better keyword match).

```sql
CREATE DATABASE NetHelp;
GO
USE NetHelp;
GO
CREATE SCHEMA Help;
GO
CREATE TABLE Help.Articles
(
    ArticleId INT           NOT NULL PRIMARY KEY,
    Title     NVARCHAR(100) NOT NULL,
    TopicVec  VECTOR(3)     NOT NULL
);
GO
INSERT INTO Help.Articles (ArticleId, Title, TopicVec) VALUES
    (1, N'Reset your router admin password',      '[4, 3, 0]'),
    (2, N'Router firmware update guide',          '[0, 1, 0]'),
    (3, N'Forgot the Wi-Fi password? Recover it', '[3, 4, 0]'),
    (4, N'Regain access to the admin console',    '[1, 0, 0]'),
    (5, N'Slow Wi-Fi troubleshooting',            '[0, 0, 1]'),
    (6, N'Change the guest network key',          '[1, 2, 0]');
GO
CREATE TABLE Help.KeywordHits
(
    ArticleId INT NOT NULL PRIMARY KEY REFERENCES Help.Articles (ArticleId),
    Score     INT NOT NULL
);
GO
INSERT INTO Help.KeywordHits (ArticleId, Score) VALUES
    (3, 64), (5, 48), (1, 35), (2, 22);
GO
```

The query embedding is `[1, 0, 0]`. The fusion query keeps the **top 4** of each ranking and uses `k = 60`:

```sql
DECLARE @q VECTOR(3) = '[1, 0, 0]';

WITH Keyword AS
(
    SELECT ArticleId,
           ROW_NUMBER() OVER (ORDER BY Score DESC, ArticleId) AS Rnk
    FROM Help.KeywordHits
),
Vector AS
(
    SELECT TOP (4) ArticleId,
           ROW_NUMBER() OVER (ORDER BY VECTOR_DISTANCE('cosine', TopicVec, @q), ArticleId) AS Rnk
    FROM Help.Articles
    ORDER BY Rnk
)
SELECT TOP (3)
       COALESCE(k.ArticleId, v.ArticleId) AS ArticleId,
       CAST(ISNULL(1.0e0 / (60 + k.Rnk), 0) + ISNULL(1.0e0 / (60 + v.Rnk), 0) AS DECIMAL(10,4)) AS RrfScore
FROM Keyword AS k
FULL OUTER JOIN Vector AS v ON v.ArticleId = k.ArticleId
ORDER BY ISNULL(1.0e0 / (60 + k.Rnk), 0) + ISNULL(1.0e0 / (60 + v.Rnk), 0) DESC,
         ArticleId;
```

What does the query return?

### a.

| ArticleId | RrfScore |
|---|---|
| 3 | 0.0323 |
| 1 | 0.0320 |
| 4 | 0.0164 |

### b.

| ArticleId | RrfScore |
|---|---|
| 3 | 1.3333 |
| 4 | 1.0000 |
| 1 | 0.8333 |

### c.

| ArticleId | RrfScore |
|---|---|
| 3 | 0.0323 |
| 1 | 0.0320 |
| 5 | 0.0161 |

### d.

| ArticleId | RrfScore |
|---|---|
| 4 | 0.0164 |
| 1 | 0.0320 |
| 3 | 0.0323 |

## Correct Answer

**a**

## Explanation

Reciprocal rank fusion merges rankings **by position, not by score**: each list contributes `1 / (k + rank)` for every document it contains, contributions are summed per document, and documents absent from a list get 0 from that list. `k = 60` is the conventional constant; it damps the gap between rank 1 and rank 2 so that appearing in *both* lists outweighs a single top position.

### Step 1 — the keyword ranking

`ROW_NUMBER() OVER (ORDER BY Score DESC, ArticleId)` turns `CONTAINSTABLE`'s `RANK` values into positions:

| Article | Score | keyword rank |
|---|---|---|
| 3 | 64 | 1 |
| 5 | 48 | 2 |
| 1 | 35 | 3 |
| 2 | 22 | 4 |

Articles 4 and 6 have no keyword hit (the query words are not in their titles), so they are absent from this list.

### Step 2 — the vector ranking

Cosine distance to q = (1, 0, 0), computed by the engine (`CAST(... AS DECIMAL(10,4))`):

| Article | vector | cosine distance | vector rank |
|---|---|---|---|
| 4 | (1, 0, 0) | 0.0000 | 1 |
| 1 | (4, 3, 0) | 0.2000 | 2 |
| 3 | (3, 4, 0) | 0.4000 | 3 |
| 6 | (1, 2, 0) | 0.5528 | 4 |
| 2 | (0, 1, 0) | 1.0000 | — (cut by `TOP (4)`) |
| 5 | (0, 0, 1) | 1.0000 | — |

The `ArticleId` tiebreaker inside `ROW_NUMBER` matters only for exact distance ties (2 and 5 here, both outside the top 4); with real embeddings a deterministic tiebreaker is what makes the ranking reproducible.

### Step 3 — fuse

`FULL OUTER JOIN` keeps every article that appears in **either** list; `ISNULL(1.0e0 / (60 + Rnk), 0)` supplies the 0 contribution for a missing side. Engine output for all six articles (six-decimal view, then rounded to four):

| Article | kw rank | vec rank | 1/(60+kw) | 1/(60+vec) | sum | RrfScore |
|---|---|---|---|---|---|---|
| 3 | 1 | 3 | 0.016393 | 0.015873 | 0.032266 | **0.0323** |
| 1 | 3 | 2 | 0.015873 | 0.016129 | 0.032002 | **0.0320** |
| 4 | — | 1 | 0 | 0.016393 | 0.016393 | **0.0164** |
| 5 | 2 | — | 0.016129 | 0 | 0.016129 | 0.0161 |
| 2 | 4 | — | 0.015625 | 0 | 0.015625 | 0.0156 |
| 6 | — | 4 | 0 | 0.015625 | 0.015625 | 0.0156 |

`TOP (3)` ordered by the sum descending gives **3, 1, 4** — option a. Two things RRF does here that a single ranking cannot:

- Article 1 is only 3rd on keywords and 2nd on vectors, yet it beats every article that tops one list, because it is present in both.
- Article 4 (*Regain access to the admin console*) contains none of the query words, so keyword search alone would never show it; its vector rank 1 alone (0.0164) is enough to enter the fused top 3 ahead of article 5, which is 2nd on keywords but semantically far from the query.

### Why option b is wrong

Those are the numbers you get when **`k` is omitted** (`1/rank` instead of `1/(60 + rank)`): 3 → 1/1 + 1/3 = 1.3333, 4 → 1/1 = 1.0, 1 → 1/3 + 1/2 = 0.8333 (verified by running that variant). Without `k`, a single rank-1 position dominates everything, so article 4 jumps above article 1 even though article 1 is well placed in both lists. The constant is not decoration; it is what makes agreement between rankings count.

### Why option c is wrong

This is the subtle distractor — the first two rows are right. It is the output of a **`LEFT JOIN` from the keyword list** (or an `INNER JOIN` would give only two rows): any article present only in the vector list is dropped, so article 4 disappears and article 5 (keyword rank 2, no vector hit) slides into third place with 0.0161 (verified). "Documents missing from one list contribute 0 from that list" requires a `FULL OUTER JOIN` (or `UNION ALL` + `GROUP BY`); a one-sided join silently throws away the other search's exclusive hits — exactly the semantic recall hybrid search exists to add.

### Why option d is wrong

Same three articles, sorted **ascending**. Vector search trains you to sort distances ascending, but an RRF score is a *relevance* score: higher is better, so the `ORDER BY ... DESC` in the query is essential and the correct order is 3, 1, 4. Reading a fused score like a distance inverts the result.

### Equivalent formulation

`UNION ALL` + `GROUP BY` produces the identical top 3 (verified: 3 → 0.0323, 1 → 0.0320, 4 → 0.0164) and generalizes naturally to three or more rankings:

```sql
WITH Keyword AS (...), Vector AS (...),
U AS (SELECT ArticleId, Rnk FROM Keyword UNION ALL SELECT ArticleId, Rnk FROM Vector)
SELECT TOP (3) ArticleId, CAST(SUM(1.0e0 / (60 + Rnk)) AS DECIMAL(10,4)) AS RrfScore
FROM U
GROUP BY ArticleId
ORDER BY SUM(1.0e0 / (60 + Rnk)) DESC, ArticleId;
```

Two implementation traps, both verified: writing `1 / (60 + Rnk)` with integer operands performs **integer division** and every score becomes 0 (the query then returns articles 1, 2, 3 purely by the tiebreaker) — use `1.0e0` or `1.0`; and the vector CTE must be **cut to a top-N before ranking** (here `TOP (4)`), otherwise every row in the table gets a rank and a tiny positive contribution, diluting the fusion. In production the keyword list comes straight from `CONTAINSTABLE(Help.Articles, Title, 'router AND password')` (`KEY`, `RANK`) — the `KeywordHits` table stands in for it here because the lab instance has no full-text component — and the vector list from `VECTOR_SEARCH` or `ORDER BY VECTOR_DISTANCE`.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every value above is the engine's literal output.

## DP-800 Exam Rule to Remember

```text
RRF(d) = Σ over rankings  1 / (k + rank_i(d))      k = 60, missing from a list → contributes 0
  1. rank each list with ROW_NUMBER() (keyword: RANK DESC; vector: distance ASC), cut each to top-N
  2. combine with FULL OUTER JOIN + ISNULL(...,0)   — or UNION ALL + GROUP BY + SUM
  3. ORDER BY fused score DESC (higher = better), then a deterministic tiebreaker
```

Checklist of the classic mistakes: integer division (`1/(60+r)` → 0), dropping `k`, one-sided (`LEFT`/`INNER`) join, and sorting the fused score ascending. A document ranked well in *both* lists should beat a document that is first in only one — if your output says otherwise, one of those four is wrong.
