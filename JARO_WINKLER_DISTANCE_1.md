# SQL Server question — JARO_WINKLER_DISTANCE 1

## Statement

BankGuard's financial-crime unit screens new customer names against a sanctions watchlist as part of KYC onboarding. Exact matching misses trivial misspellings, so the unit scores every candidate pair with the new SQL Server 2025 Jaro-Winkler function, which favors names that agree at the **beginning** — where honest transcription errors are rare and fraudulent look-alikes tend to differ least.

The following script is executed, exactly as shown, on a SQL Server 2025 (17.x) instance:

```sql
CREATE DATABASE BankGuard COLLATE Latin1_General_CI_AS;
GO
ALTER DATABASE BankGuard SET COMPATIBILITY_LEVEL = 170;
GO
USE BankGuard;
GO
ALTER DATABASE SCOPED CONFIGURATION SET PREVIEW_FEATURES = ON;
GO
CREATE SCHEMA Kyc;
GO
CREATE TABLE Kyc.ScreeningPairs
(
    PairID        int         NOT NULL PRIMARY KEY,
    CustomerName  varchar(40) NULL,
    WatchlistName varchar(40) NULL
);
GO
INSERT INTO Kyc.ScreeningPairs (PairID, CustomerName, WatchlistName) VALUES
  (1, 'MARTHA', 'MARTHA'),
  (2, 'MARTHA', 'MARHTA'),
  (3, 'MARTHA', 'AMRTHA'),
  (4, 'DWAYNE', 'DUANE'),
  (5, 'DIXON',  'DICKSONX'),
  (6, 'Martha', 'martha'),
  (7, 'MARTHA', NULL);
GO
SELECT PairID,
       CustomerName,
       WatchlistName,
       CAST(JARO_WINKLER_DISTANCE(CustomerName, WatchlistName)
            AS decimal(6,4))                                AS JwDist
FROM Kyc.ScreeningPairs
ORDER BY JwDist ASC, PairID;
```

Predict the **exact** result set: every `JwDist` value to four decimal places, and the exact row order. Pay attention to which of pairs 2 and 3 scores as the closer match — both differ from `MARTHA` by the same single adjacent swap.

## Correct Answer

The query returns 7 rows in this order (values copied from the actual output of SQL Server 2025 RTM, 17.0.1000.7):

| PairID | CustomerName | WatchlistName | JwDist |
|-------:|--------------|---------------|-------:|
| 7 | MARTHA | NULL | NULL |
| 1 | MARTHA | MARTHA | 0.0000 |
| 2 | MARTHA | MARHTA | 0.0389 |
| 3 | MARTHA | AMRTHA | 0.0556 |
| 6 | Martha | martha | 0.1111 |
| 4 | DWAYNE | DUANE | 0.1600 |
| 5 | DIXON | DICKSONX | 0.1867 |

## Explanation

Despite its name, `JARO_WINKLER_DISTANCE` (SQL Server 2025, documented for compat level 170; on RTM the operative gate is `ALTER DATABASE SCOPED CONFIGURATION SET PREVIEW_FEATURES = ON`, otherwise *Msg 195*) returns a **float distance in [0, 1]**:

```text
JARO_WINKLER_DISTANCE = 1 − Jaro-Winkler similarity
0.0 = identical        1.0 = nothing in common
```

Engine-verified: identical strings return `0.0`; `'ABC'` vs `'XYZ'` returns `1.0`. So *smaller means more alike* — the opposite polarity of its integer sibling `JARO_WINKLER_SIMILARITY` (0–100, 100 = identical). `NULL` in → `NULL` out. Like the other 2025 fuzzy functions the comparison is exact per character: the collation never applies.

**Row order first.** `ORDER BY JwDist ASC` sorts pair 7's `NULL` **first**: SQL Server treats `NULL` as lower than every non-NULL value when ordering. Then the six numeric distances ascend.

**The Jaro machinery, worked for pair 2 — 'MARTHA' vs 'MARHTA'.** Jaro similarity is a three-fraction average over the match count *m* and transposition count *t*:

```text
sim_jaro = ( m/|s1| + m/|s2| + (m − t)/m ) / 3
```

Characters "match" when equal and within a window of `max(|s1|,|s2|)/2 − 1 = 2` positions. Every letter of `MARTHA` finds its counterpart in `MARHTA`, so **m = 6**. Reading the matched characters in order gives `MARTHA` vs `MARHTA`: positions 4 and 5 disagree (`TH` vs `HT`), and t is half the number of disagreeing positions, so **t = 1**. Therefore:

```text
sim_jaro = ( 6/6 + 6/6 + (6−1)/6 ) / 3 = (1 + 1 + 5/6) / 3 = 17/18 = 0.944444…
```

Winkler then rewards the **common prefix** (length ℓ, capped at 4) with scaling factor p = 0.1. `MARTHA`/`MARHTA` share `MAR`, so ℓ = 3:

```text
sim_jw   = sim_jaro + ℓ · p · (1 − sim_jaro)
         = 0.944444 + 3 × 0.1 × 0.055556 = 0.961111
distance = 1 − 0.961111 = 0.038889
```

The engine's raw float is `3.8888888888888862E-2` = 0.0388889…, and `CAST(... AS decimal(6,4))` rounds it to **0.0389** — the hand derivation reconciles exactly.

**Pair 3 — 'MARTHA' vs 'AMRTHA' — isolates the Winkler bonus.** Same defect type (one adjacent swap), and the Jaro part is *identical*: m = 6, t = 1, sim_jaro = 17/18. But the swap sits at the **front**, so the first characters differ, ℓ = 0, and no bonus applies:

```text
distance = 1 − 17/18 = 1/18 = 0.055556  →  0.0556   (raw 5.5555555555555469E-2)
```

Two pairs with the same Jaro score rank differently on Jaro-Winkler purely because of *where* the error sits — the subtle distractor is expecting rows 2 and 3 to tie. A swap at the end is cheaper than a swap at the start; that is exactly the "prefers strings that match from the beginning" behavior the docs describe.

**Pair 6 — 'Martha' vs 'martha' — case sensitivity.** Under the CI collation `'Martha' = 'martha'` is TRUE, but the function sees `M` ≠ `m`: m = 5 of 6, t = 0, sim_jaro = (5/6 + 5/6 + 5/5)/3 = 0.888889; first characters differ so ℓ = 0 → distance 0.111111 → **0.1111** (raw `0.11111111111111105`). Not 0.0.

**Pair 4 — 'DWAYNE' vs 'DUANE'.** Matches: `D`,`A`,`N`,`E` → m = 4, t = 0; sim_jaro = (4/6 + 4/5 + 4/4)/3 = 0.822222; shared prefix `D`, ℓ = 1 → sim_jw = 0.822222 + 0.1 × 0.177778 = 0.84 → distance **0.1600** (raw `0.15999999999999992`).

**Pair 5 — 'DIXON' vs 'DICKSONX'.** Window = max(5,8)/2 − 1 = 3; `D`,`I`,`O`,`N` match but the two `X`s sit 5 positions apart — outside the window — so m = 4, t = 0; sim_jaro = (4/5 + 4/8 + 4/4)/3 = 0.766667; prefix `DI`, ℓ = 2 → sim_jw = 0.813333 → distance **0.1867** (raw `0.18666666666666676`).

The `decimal(6,4)` cast rounds the raw floats (whose binary noise appears only around the 16th significant digit), which is what makes the displayed values stable and the question deterministic.

Equivalent alternative (verified on the same engine): `JARO_WINKLER_SIMILARITY(CustomerName, WatchlistName)` returns the **int** scores 100, 96, 94, 84, 81, 89, NULL for pairs 1–7 — i.e. `ROUND(100 × (1 − distance))` — so `ORDER BY JARO_WINKLER_SIMILARITY(...) DESC` ranks the non-NULL pairs identically (with the NULL row moving to the end under `DESC`).

## DP-800 Exam Rule to Remember

`JARO_WINKLER_DISTANCE` (SQL Server 2025, documented for compat 170, preview-gated on RTM) returns a **float distance in [0, 1] where 0 = identical** — read the name literally: it is a *distance*, while `JARO_WINKLER_SIMILARITY` is its 0–100 integer mirror. Keep the two-stage formula straight:

```text
Jaro    = average of ( m/len1 , m/len2 , (m−t)/m )
Winkler = Jaro + ℓ·0.1·(1−Jaro)      ℓ = shared prefix, max 4
```

The Winkler term means errors at the **start** of a name cost more than the same error at the end (`MARHTA` 0.0389 vs `AMRTHA` 0.0556 from the same Jaro 17/18) — well-suited to name screening, where surnames' leading letters are the trustworthy part. And as with all the 2025 fuzzy functions: case differences count (collation is ignored), and `NULL` propagates — which under `ORDER BY ... ASC` puts the unmatched row **first**, not last.
