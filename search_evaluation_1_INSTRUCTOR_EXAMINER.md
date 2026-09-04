# Instructor-Examiner guide — Search Evaluation 1

Companion to [search_evaluation_1.md](search_evaluation_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.
7. **This is a multiple-choice question whose options are tables of numbers.** Read all four options, pieces 8 to 11, before taking an answer. Each option is three rows, hybrid, keyword, vector, with four numbers each. Read them slowly, row by row, and repeat any row on request. Take one letter as the answer.
8. The learner will need to do arithmetic in their head. Allow time. If they ask, re-read the two ranked lists in piece 4 and the relevant set in piece 3 as often as needed. Do not do the arithmetic for them unless a hint says so.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Implement AI-powered search solutions.
- Task bullet: Evaluate keyword, vector and hybrid search using precision, recall, reciprocal rank and nDCG at k, and fuse rankings with reciprocal rank fusion.
- What is tested: computing precision@k, recall@k, reciprocal rank and nDCG@k from ranked lists and relevance judgements, and understanding how weighted RRF orders a hybrid list.

## 2. Scenario to read aloud

**Piece 1, the story.** "A software vendor's help centre runs on a SQL Server 2025 database named DocForge. The team is deciding between keyword search with CONTAINSTABLE, vector search with VECTOR underscore DISTANCE, and a hybrid of both fused with weighted reciprocal rank fusion. They want a quantitative answer rather than a debate. One test query, reset two-factor authentication, was run through both engines. The top five results of each engine were stored in a table, together with the relevance judgements a support lead wrote for that query. Only judged documents appear, and three of them are relevant."

**Piece 2, the documents.** "A schema Eval. Table Eval dot Docs has DocId, an integer primary key, and Title, NVARCHAR one hundred. Nine documents. One, Install the desktop client. Two, Password policy and expiry. Three, Reset a lost authenticator app. Four, Enable single sign-on. Five, Recover access when your phone is lost. Six, Release notes 2026 dot 3. Seven, Configure email notifications. Eight, Admin guide, MFA re-enrollment. Nine, Biometric login on mobile."

**Piece 3, the judgements.** "Table Eval dot Judgments has QueryId, DocId, and IsRelevant, a bit. Primary key on QueryId and DocId. For query 1, five judgements. Documents 3, 5 and 8 are relevant. Documents 2 and 9 are judged not relevant. So the relevant set is three, five, eight."

**Piece 4, the rankings.** "Table Eval dot Rankings has QueryId, Method, a VARCHAR ten, Rnk, an integer rank, and DocId. Primary key on QueryId, Method and Rnk. For query 1 there are two lists of five. The keyword list, ranks one to five: document 2, then 3, then 8, then 5, then 1. The vector list, ranks one to five: document 5, then 9, then 3, then 4, then 8. Let me repeat. Keyword: two, three, eight, five, one. Vector: five, nine, three, four, eight."

**Piece 5, the hybrid.** "The hybrid ranking is built with weighted reciprocal rank fusion. The constant k is sixty. The keyword weight is one point zero, the vector weight is two point zero. For each document, the score is the sum over the lists it appears in of the weight divided by sixty plus its rank in that list. A document absent from a list contributes zero from that list. The hybrid rank orders by score descending, ties broken by DocId."

**Piece 6, the query, summarised.** "One query with common table expressions. Fused computes the hybrid score per DocId. AllRanks unions the keyword and vector rows with the hybrid rows numbered by ROW underscore NUMBER over score descending. Rel is the set of relevant DocIds. Scored left-joins AllRanks to Rel and flags each row IsRel as one or zero. The final select groups by Method and computes four measures, all cast to DECIMAL six comma four. I can read any line on request."

**Piece 7, the four measures.** "PrecisionAt3 is the count of relevant rows with rank at most three, divided by three. RecallAt3 is that same count divided by the total number of relevant documents, which is three. ReciprocalRank is one divided by the smallest rank at which a relevant document appears, in the full list of five. nDCGAt3 is the sum, over ranks one to three, of IsRel divided by log base two of rank plus one, divided by the ideal value, which is the same sum with all three positions relevant. The output is ordered by Method, so hybrid, then keyword, then vector."

**Piece 8, option a.** "Option a. Hybrid: precision one, recall one, reciprocal rank one, nDCG one. Keyword: precision zero point six six six seven, recall zero point six six six seven, reciprocal rank zero point five, nDCG zero point five three zero seven. Vector: precision zero point six six six seven, recall zero point six six six seven, reciprocal rank one, nDCG zero point seven zero three nine."

**Piece 9, option b.** "Option b. Hybrid: precision one, recall one, reciprocal rank zero point six one one one, nDCG one. Keyword: precision zero point six six six seven, recall zero point six six six seven, reciprocal rank zero point three six one one, nDCG zero point five three zero seven. Vector: precision zero point six six six seven, recall zero point six six six seven, reciprocal rank zero point five one one one, nDCG zero point seven zero three nine."

**Piece 10, option c.** "Option c. Hybrid: precision zero point six six six seven, recall zero point six six six seven, reciprocal rank one, nDCG zero point seven zero three nine. Keyword: precision zero point six six six seven, recall zero point six six six seven, reciprocal rank zero point five, nDCG zero point five three zero seven. Vector: precision zero point six six six seven, recall zero point six six six seven, reciprocal rank one, nDCG zero point seven zero three nine. So in option c the hybrid row equals the vector row."

**Piece 11, option d.** "Option d. Hybrid: precision zero point six, recall one, reciprocal rank one, nDCG one. Keyword: precision zero point six, recall one, reciprocal rank zero point five, nDCG zero point five three zero seven. Vector: precision zero point six, recall one, reciprocal rank one, nDCG zero point seven zero three nine."

## 3. Setup script (reference only; do not read verbatim unless asked)

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

## 4. The question (ask exactly this)

"What does the query return? Option a, b, c or d?"

- **a.** hybrid 1.0000, 1.0000, 1.0000, 1.0000 · keyword 0.6667, 0.6667, 0.5000, 0.5307 · vector 0.6667, 0.6667, 1.0000, 0.7039
- **b.** hybrid 1.0000, 1.0000, 0.6111, 1.0000 · keyword 0.6667, 0.6667, 0.3611, 0.5307 · vector 0.6667, 0.6667, 0.5111, 0.7039
- **c.** hybrid 0.6667, 0.6667, 1.0000, 0.7039 · keyword 0.6667, 0.6667, 0.5000, 0.5307 · vector 0.6667, 0.6667, 1.0000, 0.7039
- **d.** hybrid 0.6000, 1.0000, 1.0000, 1.0000 · keyword 0.6000, 1.0000, 0.5000, 0.5307 · vector 0.6000, 1.0000, 1.0000, 0.7039

(Columns in order: PrecisionAt3, RecallAt3, ReciprocalRank, nDCGAt3.)

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

Hybrid list by weighted RRF, k = 60:

| DocId | keyword rank | vector rank | 1.0/(60+kw) | 2.0/(60+vec) | Score | hybrid rank |
|---|---|---|---|---|---|---|
| 5 | 4 | 1 | 0.015625 | 0.032787 | 0.048412 | 1 |
| 3 | 2 | 3 | 0.016129 | 0.031746 | 0.047875 | 2 |
| 8 | 3 | 5 | 0.015873 | 0.030769 | 0.046642 | 3 |
| 9 | none | 2 | 0 | 0.032258 | 0.032258 | 4 |
| 4 | none | 4 | 0 | 0.031250 | 0.031250 | 5 |
| 2 | 1 | none | 0.016393 | 0 | 0.016393 | 6 |
| 1 | 5 | none | 0.015385 | 0 | 0.015385 | 7 |

Scores at k = 3, relevant set {3, 5, 8}, IDCG@3 = 1 + 0.6309 + 0.5 = 2.1309:

| Method | top 3 | P@3 | R@3 | RR | DCG@3 | nDCG@3 |
|---|---|---|---|---|---|---|
| keyword | 2 no, 3 yes, 8 yes | 0.6667 | 0.6667 | first relevant at rank 2, 0.5000 | 0.6309 + 0.5 = 1.1309 | 0.5307 |
| vector | 5 yes, 9 no, 3 yes | 0.6667 | 0.6667 | rank 1, 1.0000 | 1 + 0.5 = 1.5000 | 0.7039 |
| hybrid | 5, 3, 8, all yes | 1.0000 | 1.0000 | 1.0000 | 2.1309 | 1.0000 |

Why the wrong options are wrong:

- **b.** Its ReciprocalRank averages 1 over rank across all relevant documents in the list (keyword: (1/2 + 1/3 + 1/4) / 3 = 0.3611). Reciprocal rank uses only the first relevant hit, as the query's 1.0 divided by MIN of the relevant rank does.
- **c.** Copies the vector row into hybrid, assuming the 2.0 weight makes the vector list dominate and the hybrid top 3 is {5, 9, 3}. With k = 60 every contribution sits between 1/65 and 2/61, so a document in both lists collects about three units and a vector-only document about two. Document 8 at 0.0466 stays above document 9 at 0.0323.
- **d.** Rank-based numbers are right, but precision is computed over five returned rows and recall over the whole list (3/5 = 0.6 and 3/3 = 1.0). The query filters Rnk <= 3. Precision@3 means the top three only.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with the two lists you already have, keyword and vector. Take the top three of each. How many of those three are in the relevant set three, five, eight? That gives you precision at 3 and recall at 3 for both engines. Notice they come out the same."
2. "Now reciprocal rank. It is one divided by the rank of the first relevant document. In the keyword list the first relevant document is at which rank? In the vector list? Only the first one counts, not an average over all of them."
3. "That already rules out one option, the one where reciprocal rank is 0.3611 and 0.5111. And check the option where precision is 0.6: how many positions would you have to count for three relevant out of 0.6? Is that k equals 3?"
4. "Two options remain, and they differ only in the hybrid row. Build the hybrid list. Which documents appear in both lists? Documents 5, 3 and 8. Each of those collects one over sixty-plus-rank from keyword plus two over sixty-plus-rank from vector."
5. "Compare document 8, in both lists at ranks 3 and 5, with document 9, only in the vector list at rank 2. Roughly: document 8 gets one sixty-third plus two sixty-fifths. Document 9 gets two sixty-seconds. Which is bigger?"
6. "So the hybrid top three is the three documents present in both lists. Are all three of them relevant? What are precision, recall, reciprocal rank and nDCG when the top three are all relevant?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "b, reciprocal rank should average over all the relevant hits" | Confuses reciprocal rank with an average-precision-like value | "Look at the query. It divides one by the MIN of the relevant ranks. How many ranks does MIN use?" |
| "c, the vector weight is double so the hybrid follows the vector list" | Thinks weight decides membership | "With k equals sixty, how much does one list contribute at most? Compare a document in two lists with a document in one." |
| "d, precision is three out of five" | Ignores the k cut-off | "The metric is precision at 3. Which rows does the query keep before summing?" |
| "keyword and vector must differ in precision because one is clearly better" | Expects set metrics to capture order | "Count the relevant documents in each top three. Then ask which metrics see the order and which do not." |
| "hybrid should have document 2 first because it is first on keywords" | Forgets a single-list document gets one contribution only | "Document 2 is absent from the vector list. What does it contribute from that list?" |
| "nDCG for keyword should be 0.6667 like precision" | Does not apply the log discount | "Position two is discounted by log base two of three, position three by log base two of four. Sum those and divide by the ideal." |

## 8. Teaching notes (after the answer is complete or revealed)

Explain how search quality is measured:

- **A labelled test set.** For each query, a list of documents judged relevant. The Learn module defines precision, of the results returned, what percentage are relevant; recall, of all relevant documents, what percentage did the search find; and mean reciprocal rank, how high does the first relevant result appear. It recommends running full-text only, vector only and hybrid over the same test set and comparing. nDCG adds position-discounted credit.
- **Weighted RRF.** The score is the sum over lists of weight divided by k plus rank. A document missing from a list contributes zero. The module shows the weighted form with COALESCE of 1.0 over k plus keyword rank plus 2.0 over k plus vector rank, to weight vector search twice. With k equals sixty, documents in both lists, 5, 3 and 8, beat every single-list document, including document 2, first on keywords, and document 9, second on vectors. Weights shift scores; list membership decides the order. The unweighted fusion gives the same top three.
- **The metrics at k equals 3.** Keyword: 2, 3, 8, two relevant, precision and recall two thirds, first relevant at rank 2 so reciprocal rank 0.5, DCG 0.6309 plus 0.5 equals 1.1309, over IDCG 2.1309 gives 0.5307. Vector: 5, 9, 3, two relevant, same precision and recall, first relevant at rank 1 so reciprocal rank 1, DCG 1 plus 0.5 equals 1.5, nDCG 0.7039. Hybrid: 5, 3, 8, all relevant, every metric 1.
- **The lesson in the keyword versus vector pair.** Same precision and recall, different reciprocal rank and nDCG. The two engines found the same number of relevant documents but ordered them differently. Set-based metrics are blind to order; rank-based metrics are not.
- **Reading the result.** Hybrid wins on every metric for this query, matching the module's expectation that hybrid improves recall with minimal precision loss. One query is an anecdote; run a few hundred labelled queries and average: MRR, mean precision and recall at k, mean nDCG at k. Two knobs show up in the numbers: the candidate count fed into RRF, the module suggests starting at fifty, and the constant k, sixty is standard, smaller amplifies top ranks. Filters on product, language or tenant belong before fusion, because filtering after RRF might eliminate highly ranked results.

Memory hook: "Precision and recall count the top k. Reciprocal rank looks at the first hit only. nDCG discounts by position. RRF: in two lists beats in one."

## 9. Follow-up oral questions (optional)

1. "If the query were scored at k equals 5 instead of 3, what would recall be for all three methods?" (One for all three; every list contains 3, 5 and 8 within five positions. That is why the cut-off matters.)
2. "What is the difference between reciprocal rank and mean reciprocal rank?" (Reciprocal rank is one over the first relevant rank for one query. MRR is the mean of that over all queries in the test set.)
3. "Why should a tenant or language filter be applied before RRF rather than after?" (Filtering after fusion can remove highly ranked results and leave fewer than the requested top N; filtering before lets each engine return N valid candidates.)

## 10. References

- Build hybrid search with full-text and vector search in SQL (Learn module, evaluation and RRF): https://learn.microsoft.com/en-us/training/modules/build-hybrid-search-sql/
- Hybrid search in SQL Server and Azure SQL: https://learn.microsoft.com/en-us/sql/relational-databases/vectors/hybrid-search
- Vector search and VECTOR_DISTANCE: https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-distance-transact-sql
- CONTAINSTABLE: https://learn.microsoft.com/en-us/sql/relational-databases/system-functions/containstable-transact-sql
- LOG function: https://learn.microsoft.com/en-us/sql/t-sql/functions/log-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
