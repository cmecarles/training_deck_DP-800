# SQL Server question — EDIT_DISTANCE_SIMILARITY 1

## Statement

ToolBarn, a hardware-store chain, receives weekly product feeds from suppliers and must deduplicate them against its own catalog. Product names arrive with small differences — a changed model number, swapped digits, inconsistent capitalization — so the data team uses the new SQL Server 2025 fuzzy matching to score every candidate pair and keeps only pairs at or above a similarity threshold of 75 for human review.

The following script is executed, exactly as shown, on a SQL Server 2025 (17.x) instance:

```sql
CREATE DATABASE ToolBarn COLLATE Latin1_General_CI_AS;
GO
ALTER DATABASE ToolBarn SET COMPATIBILITY_LEVEL = 170;
GO
USE ToolBarn;
GO
ALTER DATABASE SCOPED CONFIGURATION SET PREVIEW_FEATURES = ON;
GO
CREATE SCHEMA Cat;
GO
CREATE TABLE Cat.FeedMatches
(
    MatchID     int         NOT NULL PRIMARY KEY,
    FeedName    varchar(60) NULL,
    CatalogName varchar(60) NULL
);
GO
INSERT INTO Cat.FeedMatches (MatchID, FeedName, CatalogName) VALUES
  (1, 'Claw Hammer',          'Claw Hammer'),
  (2, 'Rasp 200mm',           'Rasp 250mm'),
  (3, 'Drill 750W',           'Drill 705W'),
  (4, 'Adjustable Wrench 25', 'Adjustable Wrench 26'),
  (5, 'Rivets 8',             'Rivet 6'),
  (6, 'STAPLE GUN',           'Staple Gun'),
  (7, 'Axe',                  'Bit'),
  (8, 'Pry Bar',              NULL);
GO
SELECT MatchID,
       FeedName,
       CatalogName,
       EDIT_DISTANCE_SIMILARITY(FeedName, CatalogName) AS Score
FROM Cat.FeedMatches
WHERE EDIT_DISTANCE_SIMILARITY(FeedName, CatalogName) >= 75
ORDER BY Score DESC, MatchID;
```

Predict the **exact** result set of the final `SELECT`: which of the 8 candidate pairs survive the threshold, each pair's exact `Score`, and the row order.

## Correct Answer

The query returns exactly 5 of the 8 rows (values copied from the actual output of SQL Server 2025 RTM, 17.0.1000.7):

| MatchID | FeedName | CatalogName | Score |
|--------:|----------|-------------|------:|
| 1 | Claw Hammer | Claw Hammer | 100 |
| 4 | Adjustable Wrench 25 | Adjustable Wrench 26 | 95 |
| 2 | Rasp 200mm | Rasp 250mm | 90 |
| 3 | Drill 750W | Drill 705W | 80 |
| 5 | Rivets 8 | Rivet 6 | 75 |

Rows 6, 7, and 8 are filtered out.

## Explanation

`EDIT_DISTANCE_SIMILARITY(s1, s2)` (SQL Server 2025, documented for compat level 170; on RTM the operative gate is `ALTER DATABASE SCOPED CONFIGURATION SET PREVIEW_FEATURES = ON`, otherwise *Msg 195*) returns an **int from 0 (no match) to 100 (full match)**. Per Microsoft Learn, the score is a normalization of the Levenshtein edit distance:

```text
similarity = (1 − edit_distance / GREATEST(LEN(s1), LEN(s2))) × 100
```

rounded to the nearest integer (engine-verified: a raw 66.67 returns 67, a raw 83.33 returns 83, and an exact 87.5 rounds up to 88). Like `EDIT_DISTANCE`, it counts insertions, deletions, and substitutions only — the docs' "Damerau-Levenshtein" label notwithstanding, transpositions are currently unsupported and cost 2 — the comparison is exact per character (collation-independent), and any `NULL` input yields `NULL`.

Score every row (edit distances verified independently with `EDIT_DISTANCE` on the same engine):

- **Row 1 — 'Claw Hammer' / 'Claw Hammer':** distance 0, lengths 11/11 → (1 − 0/11) × 100 = **100**.
- **Row 2 — 'Rasp 200mm' / 'Rasp 250mm':** one substitution (`0`→`5`), lengths 10/10 → (1 − 1/10) × 100 = **90**.
- **Row 3 — 'Drill 750W' / 'Drill 705W':** `50` vs `05` is a transposition, charged as **two** substitutions, lengths 10/10 → (1 − 2/10) × 100 = **80**, not the 90 a Damerau assumption predicts. This is the subtle distractor: one keystroke slip, two edit operations.
- **Row 4 — 'Adjustable Wrench 25' / 'Adjustable Wrench 26':** one substitution, lengths 20/20 → (1 − 1/20) × 100 = **95**.
- **Row 5 — 'Rivets 8' / 'Rivet 6':** delete `s`, substitute `8`→`6`; distance 2, lengths 8/7, normalize by the **greater** length → (1 − 2/8) × 100 = **75**. Exactly at the threshold, and `>= 75` is inclusive, so it survives. Normalizing by the shorter length (7) would give 71 and wrongly drop the row.
- **Row 6 — 'STAPLE GUN' / 'Staple Gun': score 30, filtered out.** Under the case-insensitive database collation `'STAPLE GUN' = 'Staple Gun'` is TRUE — a classic `=`-based dedup would call these identical. But the function ignores collation: `S`, the space, and `G` match while the other 7 characters differ by case, so distance 7, lengths 10/10 → (1 − 7/10) × 100 = **30**. The pair most obviously "the same product" scores far below threshold — the question's main trap. Real dedup pipelines normalize case (e.g. `UPPER()`) before scoring.
- **Row 7 — 'Axe' / 'Bit':** all 3 characters differ → distance 3, lengths 3/3 → (1 − 3/3) × 100 = **0**.
- **Row 8 — 'Pry Bar' / NULL:** the function returns `NULL`, the predicate `NULL >= 75` evaluates to UNKNOWN, and the row is **silently removed** by the `WHERE` clause — no error, no row.

`ORDER BY Score DESC, MatchID` gives 100, 95, 90, 80, 75 → MatchIDs 1, 4, 2, 3, 5. There are no score ties, but the `MatchID` tiebreaker is what makes the order fully deterministic by contract.

Equivalent alternatives (all verified to return the same values on the same engine):

- The score can be derived from `EDIT_DISTANCE` explicitly; this returned identical values for all 8 rows:

  ```sql
  CAST(ROUND((1.0 - 1.0 * EDIT_DISTANCE(FeedName, CatalogName)
                    / GREATEST(LEN(FeedName), LEN(CatalogName))) * 100, 0) AS int)
  ```

- Because the raw query calls the function twice (SELECT list and `WHERE`), a `CROSS APPLY (SELECT EDIT_DISTANCE_SIMILARITY(FeedName, CatalogName) AS Score) AS x ... WHERE x.Score >= 75` computes it once and returns the same result set.

## DP-800 Exam Rule to Remember

`EDIT_DISTANCE_SIMILARITY` (SQL Server 2025, documented for compat 170, preview-gated on RTM) returns an **int 0–100**, defined as:

```text
(1 − Levenshtein_distance / length_of_the_LONGER_string) × 100, rounded to nearest
```

Remember the three ways a "duplicate" escapes a similarity threshold:

1. **Case differences are real edits** — the collation never applies, so a CI-equal pair can score 30.
2. **A digit/letter swap costs 2 edits** (no transposition support), halving the penalty-free margin.
3. **NULL scores are UNKNOWN in `WHERE`** — rows with missing names vanish from the review queue instead of failing loudly.

And normalization always divides by the **longer** string's length — the inclusive threshold row (distance 2 over max(8,7)) lands exactly on 75.
