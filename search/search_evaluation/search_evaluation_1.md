# SQL Server question — Search Evaluation 1

## Statement

A software vendor's help centre runs on a SQL Server 2025 database named `DocForge`. The team is deciding between **keyword** search (`CONTAINSTABLE`), **vector** search (`VECTOR_DISTANCE`) and a **hybrid** of both fused with weighted reciprocal rank fusion, and wants a quantitative answer rather than a debate.

For the evaluation, one test query — *"reset two-factor authentication"* — was run through both engines. The top five results of each engine were materialised into a table, together with the **relevance judgements** a support lead wrote for that query (only judged documents appear; three of them are relevant):

```sql
CREATE DATABASE DocForge;
GO
USE DocForge;
GO
CREATE SCHEMA Eval;
GO
CREATE TABLE Eval.Docs
(
    DocId INT           NOT NULL PRIMARY KEY,
    Title NVARCHAR(100) NOT NULL
);
INSERT INTO Eval.Docs (DocId, Title) VALUES
    (1, N'Install the desktop client'),
    (2, N'Password policy and expiry'),
    (3, N'Reset a lost authenticator app'),
    (4, N'Enable single sign-on'),
    (5, N'Recover access when your phone is lost'),
    (6, N'Release notes 2026.3'),
    (7, N'Configure email notifications'),
    (8, N'Admin guide: MFA re-enrollment'),
    (9, N'Biometric login on mobile');
GO
CREATE TABLE Eval.Judgments
(
    QueryId    INT NOT NULL,
    DocId      INT NOT NULL REFERENCES Eval.Docs (DocId),
    IsRelevant BIT NOT NULL,
    PRIMARY KEY (QueryId, DocId)
);
INSERT INTO Eval.Judgments (QueryId, DocId, IsRelevant) VALUES
    (1, 3, 1), (1, 5, 1), (1, 8, 1),
    (1, 2, 0), (1, 9, 0);
GO
CREATE TABLE Eval.Rankings
(
    QueryId INT         NOT NULL,
    Method  VARCHAR(10) NOT NULL,
    Rnk     INT         NOT NULL,
    DocId   INT         NOT NULL REFERENCES Eval.Docs (DocId),
    PRIMARY KEY (QueryId, Method, Rnk)
);
INSERT INTO Eval.Rankings (QueryId, Method, Rnk, DocId) VALUES
    (1, 'keyword', 1, 2), (1, 'keyword', 2, 3), (1, 'keyword', 3, 8), (1, 'keyword', 4, 5), (1, 'keyword', 5, 1),
    (1, 'vector',  1, 5), (1, 'vector',  2, 9), (1, 'vector',  3, 3), (1, 'vector',  4, 4), (1, 'vector',  5, 8);
GO
```

The hybrid ranking is built with weighted RRF (`k = 60`, keyword weight 1.0, vector weight 2.0), and the three methods are scored at **k = 3** with precision@3, recall@3, reciprocal rank (of the first relevant document) and nDCG@3 with binary gains:

```sql
WITH Fused AS
(
    SELECT DocId,
           SUM(CASE Method WHEN 'keyword' THEN 1.0 ELSE 2.0 END / (60 + Rnk)) AS Score
    FROM Eval.Rankings WHERE QueryId = 1 GROUP BY DocId
),
AllRanks AS
(
    SELECT Method, Rnk, DocId FROM Eval.Rankings WHERE QueryId = 1
    UNION ALL
    SELECT 'hybrid', ROW_NUMBER() OVER (ORDER BY Score DESC, DocId), DocId FROM Fused
),
Rel AS
(
    SELECT DocId FROM Eval.Judgments WHERE QueryId = 1 AND IsRelevant = 1
),
Scored AS
(
    SELECT a.Method, a.Rnk, a.DocId,
           CASE WHEN r.DocId IS NULL THEN 0 ELSE 1 END AS IsRel
    FROM AllRanks AS a
    LEFT JOIN Rel AS r ON r.DocId = a.DocId
)
SELECT Method,
       CAST(SUM(CASE WHEN Rnk <= 3 THEN IsRel END) / 3.0 AS DECIMAL(6,4))                               AS PrecisionAt3,
       CAST(SUM(CASE WHEN Rnk <= 3 THEN IsRel END) * 1.0 / (SELECT COUNT(*) FROM Rel) AS DECIMAL(6,4))   AS RecallAt3,
       CAST(1.0 / MIN(CASE WHEN IsRel = 1 THEN Rnk END) AS DECIMAL(6,4))                                 AS ReciprocalRank,
       CAST(SUM(CASE WHEN Rnk <= 3 THEN IsRel / LOG(Rnk + 1, 2) END)
            / (SELECT SUM(1.0 / LOG(n + 1, 2)) FROM (VALUES (1), (2), (3)) AS v(n)) AS DECIMAL(6,4))    AS nDCGAt3
FROM Scored
GROUP BY Method
ORDER BY Method;
```

What does the query return?

### a.

| Method | PrecisionAt3 | RecallAt3 | ReciprocalRank | nDCGAt3 |
|---|---|---|---|---|
| hybrid | 1.0000 | 1.0000 | 1.0000 | 1.0000 |
| keyword | 0.6667 | 0.6667 | 0.5000 | 0.5307 |
| vector | 0.6667 | 0.6667 | 1.0000 | 0.7039 |

### b.

| Method | PrecisionAt3 | RecallAt3 | ReciprocalRank | nDCGAt3 |
|---|---|---|---|---|
| hybrid | 1.0000 | 1.0000 | 0.6111 | 1.0000 |
| keyword | 0.6667 | 0.6667 | 0.3611 | 0.5307 |
| vector | 0.6667 | 0.6667 | 0.5111 | 0.7039 |

### c.

| Method | PrecisionAt3 | RecallAt3 | ReciprocalRank | nDCGAt3 |
|---|---|---|---|---|
| hybrid | 0.6667 | 0.6667 | 1.0000 | 0.7039 |
| keyword | 0.6667 | 0.6667 | 0.5000 | 0.5307 |
| vector | 0.6667 | 0.6667 | 1.0000 | 0.7039 |

### d.

| Method | PrecisionAt3 | RecallAt3 | ReciprocalRank | nDCGAt3 |
|---|---|---|---|---|
| hybrid | 0.6000 | 1.0000 | 1.0000 | 1.0000 |
| keyword | 0.6000 | 1.0000 | 0.5000 | 0.5307 |
| vector | 0.6000 | 1.0000 | 1.0000 | 0.7039 |

## Correct Answer

**a**

## Explanation

Search quality is measured against a **labelled test set**: for each query, a list of documents judged relevant. The Learn module defines the three classic metrics — *precision* ("Of the results returned, what percentage are relevant?"), *recall* ("Of all relevant documents in your database, what percentage did the search find?") and *Mean Reciprocal Rank* ("How high does the first relevant result appear? MRR rewards searches that put the best result at the top") — and recommends running "each approach (full-text only, vector only, hybrid)" over the same test set and comparing. nDCG adds position-discounted credit. All numbers below are the engine's output.

### Step 1 — build the hybrid list with weighted RRF

RRF sums `w / (k + rank)` per list; a document missing from a list contributes 0. The module shows the weighted variant literally — `COALESCE(1.0 / (@rrfK + ks.keyword_rank), 0.0) + COALESCE(2.0 / (@rrfK + vs.vector_rank), 0.0)` to "weight vector search 2x more than keyword search". With `k = 60`:

| DocId | keyword rank | vector rank | 1.0/(60+kw) | 2.0/(60+vec) | Score | hybrid rank |
|---|---|---|---|---|---|---|
| 5 | 4 | 1 | 0.015625 | 0.032787 | 0.048412 | 1 |
| 3 | 2 | 3 | 0.016129 | 0.031746 | 0.047875 | 2 |
| 8 | 3 | 5 | 0.015873 | 0.030769 | 0.046642 | 3 |
| 9 | — | 2 | 0 | 0.032258 | 0.032258 | 4 |
| 4 | — | 4 | 0 | 0.031250 | 0.031250 | 5 |
| 2 | 1 | — | 0.016393 | 0 | 0.016393 | 6 |
| 1 | 5 | — | 0.015385 | 0 | 0.015385 | 7 |

Documents present in **both** lists (5, 3, 8) beat every single-list document, including document 2, which is first on keywords, and document 9, which is second on vectors. That is the behaviour RRF is designed for, and it is why the hybrid top 3 is exactly the relevant set {5, 3, 8}.

### Step 2 — score the three top-3 lists

Relevant set R = {3, 5, 8}, |R| = 3. IDCG@3 (three relevant documents in the best possible order) = 1/log2(2) + 1/log2(3) + 1/log2(4) = 1 + 0.6309 + 0.5 = **2.1309**.

| Method | top 3 (relevant?) | P@3 | R@3 | RR | DCG@3 | nDCG@3 |
|---|---|---|---|---|---|---|
| keyword | 2 (no), 3 (yes), 8 (yes) | 2/3 = 0.6667 | 2/3 = 0.6667 | first relevant at rank 2 → 0.5000 | 0.6309 + 0.5 = 1.1309 | 1.1309 / 2.1309 = 0.5307 |
| vector | 5 (yes), 9 (no), 3 (yes) | 0.6667 | 0.6667 | rank 1 → 1.0000 | 1 + 0.5 = 1.5000 | 0.7039 |
| hybrid | 5, 3, 8 (all yes) | 1.0000 | 1.0000 | 1.0000 | 2.1309 | 1.0000 |

That is option a. The instructive part is the keyword-versus-vector row pair: both have the **same** precision@3 and recall@3 (two relevant documents each), yet reciprocal rank and nDCG separate them, because the vector engine put a relevant document first and the keyword engine put an irrelevant one first. Set-based metrics are blind to order; rank-based metrics are not.

### Why option b is wrong

Its `ReciprocalRank` column averages `1/rank` over **all** relevant documents in the full list (keyword: (1/2 + 1/3 + 1/4)/3 = 0.3611; vector: (1/1 + 1/3 + 1/5)/3 = 0.5111; hybrid: (1 + 1/2 + 1/3)/3 = 0.6111). That is a different quantity (an average-precision-like number), not reciprocal rank, which uses **only the first** relevant hit — the query's `1.0 / MIN(CASE WHEN IsRel = 1 THEN Rnk END)`. MRR is then the mean of that value across queries in the test set.

### Why option c is wrong

Option c makes the hybrid row a copy of the vector row, reasoning that a 2.0 weight lets the vector list "dominate" so the hybrid top 3 is {5, 9, 3}. It does not: with `k = 60` every contribution sits between 1/65 and 2/61, so a document that appears in **both** lists collects roughly three units (1 + 2) while a vector-only document collects two — document 8 (0.0466) stays above document 9 (0.0323). Weighting shifts scores; list membership decides the order. (Verified: the **unweighted** fusion gives the same top 3 — 5 at 0.032018, 3 at 0.032002, 8 at 0.031258.)

### Why option d is wrong

This is the subtle distractor: every rank-based number is right. But it computes precision over the **five** returned rows and recall over the whole list (keyword: 3/5 = 0.6, 3/3 = 1.0). The `@k` cut-off is the point of the metrics: the query filters `Rnk <= 3`, and "precision@3" means the top three positions only. Evaluating the whole candidate list also hides the difference between engines (all three reach recall 1.0 at k = 5), which is exactly the signal the team needs.

### Reading the result

Hybrid wins on every metric for this query, which matches the module's expectation that "hybrid search typically improves recall compared to either approach alone, often with minimal precision loss". One query is an anecdote, not an evaluation: the same query runs over a few hundred labelled queries and the metrics are averaged (MRR, mean P@k / R@k, mean nDCG@k). Two tuning knobs then show up in the numbers: the candidate count `@topN` fed into RRF (the module suggests starting at 50 — too small a cut and the fusion never sees a document that only one list ranks low) and the constant `k` (60 is the standard; a smaller `k` amplifies top ranks). Filters (product, language, tenant) belong **before** fusion: "filtering after RRF might eliminate highly ranked results".

Verified against SQL Server 2025 (RTM 17.0.1000.7); every value above is the engine's literal output.

## DP-800 Exam Rule to Remember

```text
precision@k = relevant in top k / k              recall@k = relevant in top k / |all relevant|
RR          = 1 / rank of the FIRST relevant hit  (MRR = mean of RR over the query set)
DCG@k       = sum(rel_i / log2(i + 1)),  nDCG@k = DCG@k / IDCG@k   (1.0 = perfect order)
weighted RRF(d) = sum_i  w_i / (k + rank_i(d)),   k = 60, absent from a list -> 0
```

- Same P@k and R@k, different RR/nDCG → the engines differ in **ordering**, not in what they found.
- Weights change scores, not membership: at k = 60 a document in two lists beats a document in one.
- Compute on a labelled test set, at a fixed k, averaged over many queries; filter before fusion; tune `@topN` and `k` by re-running the harness, not by intuition.
