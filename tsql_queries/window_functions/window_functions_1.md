# SQL Server question — Window Functions 1

## Statement

The city marathon uses a SQL Server database called `RaceDay` to run its timing system. As runners cross the finish line, the timing mats write one row per finisher. The script below is the complete history of the database, from creation onward.

```sql
CREATE DATABASE RaceDay;
GO
USE RaceDay;
GO
CREATE SCHEMA Timing;
GO
CREATE TABLE Timing.Finishers (
    BibNumber     INT           NOT NULL PRIMARY KEY,
    RunnerName    NVARCHAR(60)  NOT NULL,
    Category      VARCHAR(10)   NOT NULL
                  CHECK (Category IN ('Elite','Masters','Open')),
    FinishTime    TIME(0)       NOT NULL,
    CharityRaised DECIMAL(8,2)  NOT NULL
);
GO
INSERT INTO Timing.Finishers
    (BibNumber, RunnerName, Category, FinishTime, CharityRaised)
VALUES
    (101, N'Amara Diallo', 'Elite',   '02:06:05', 300.00),
    (118, N'Lucas Meyer',  'Elite',   '02:07:22', 200.00),
    (122, N'Priya Nair',   'Elite',   '02:08:30', 250.00),
    (130, N'Owen Carter',  'Elite',   '02:08:30', 100.00),
    (149, N'Tomas Silva',  'Elite',   '02:10:18',  50.00),
    (205, N'Kenji Sato',   'Elite',   '02:06:05', 150.00),
    (210, N'Grace Kim',    'Masters', '02:20:04', 120.00),
    (217, N'Ivan Petrov',  'Masters', '02:21:00', 200.00),
    (224, N'Nadia Farah',  'Masters', '02:22:11',  60.00),
    (233, N'Hana Yusuf',   'Masters', '02:21:00',  80.00);
GO
```

After the script runs, a single T-SQL query is executed against `RaceDay` and returns **exactly** the result set below: same column names, same values, same row order.

`Place` is the official finishing position inside each category. Note that the timing mats recorded exact ties: runners who cross together share the same official place. `CharityBanked` is what the charity dashboard displayed at the moment each runner crossed the line: the total money raised so far by that runner's category — and runners who cross together are credited together, so tied runners see the same figure.

```text
Category  BibNumber  RunnerName    FinishTime  Place  CharityBanked
--------  ---------  ------------  ----------  -----  -------------
Elite     101        Amara Diallo  02:06:05    1      450.00
Elite     205        Kenji Sato    02:06:05    1      450.00
Elite     118        Lucas Meyer   02:07:22    3      650.00
Elite     122        Priya Nair    02:08:30    4      1000.00
Elite     130        Owen Carter   02:08:30    4      1000.00
Elite     149        Tomas Silva   02:10:18    6      1050.00
Masters   210        Grace Kim     02:20:04    1      120.00
Masters   217        Ivan Petrov   02:21:00    2      400.00
Masters   233        Hana Yusuf    02:21:00    2      400.00
Masters   224        Nadia Farah   02:22:11    4      460.00
```

Write that query. Constraints:

1. The query must read `Timing.Finishers` exactly once (no self-joins, no correlated subqueries against the same table).
2. The query must not hard-code any runner data (no literals for names, bibs, times, places, or amounts).
3. The result must be deterministic: it must return the same row order on every execution, on any SQL Server instance.

## Correct Answer

### Reference query

```sql
SELECT
    f.Category,
    f.BibNumber,
    f.RunnerName,
    f.FinishTime,
    RANK() OVER (PARTITION BY f.Category
                 ORDER BY f.FinishTime)      AS Place,
    SUM(f.CharityRaised)
        OVER (PARTITION BY f.Category
              ORDER BY f.FinishTime)         AS CharityBanked
FROM Timing.Finishers AS f
ORDER BY f.Category, f.FinishTime, f.BibNumber;
```

Two window functions over the **same** window — `PARTITION BY Category ORDER BY FinishTime`, with ties on `FinishTime` deliberately left as ties — plus a presentation `ORDER BY` that adds `BibNumber` as a unique tiebreaker **only at the outermost level**.

### Equivalent correct alternatives

**A. Explicit default frame.** Spelling out the frame that `SUM ... OVER (... ORDER BY ...)` uses by default changes nothing:

```sql
SUM(f.CharityRaised)
    OVER (PARTITION BY f.Category
          ORDER BY f.FinishTime
          RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS CharityBanked
```

(`RANK` cannot take a frame at all — ranking functions accept no `ROWS`/`RANGE` clause.)

**B. Wrapped in a CTE.** Compute the window columns in a CTE and sort outside; window functions run over the CTE's full row set, so the result is identical:

```sql
WITH Board AS (
    SELECT
        f.Category,
        f.BibNumber,
        f.RunnerName,
        f.FinishTime,
        RANK() OVER (PARTITION BY f.Category
                     ORDER BY f.FinishTime)  AS Place,
        SUM(f.CharityRaised)
            OVER (PARTITION BY f.Category
                  ORDER BY f.FinishTime)     AS CharityBanked
    FROM Timing.Finishers AS f
)
SELECT Category, BibNumber, RunnerName, FinishTime, Place, CharityBanked
FROM Board
ORDER BY Category, FinishTime, BibNumber;
```

**C. Named window (SQL Server 2022+, compatibility level 160).** The `WINDOW` clause factors out the shared specification:

```sql
SELECT
    f.Category,
    f.BibNumber,
    f.RunnerName,
    f.FinishTime,
    RANK()               OVER W AS Place,
    SUM(f.CharityRaised) OVER W AS CharityBanked
FROM Timing.Finishers AS f
WINDOW W AS (PARTITION BY f.Category ORDER BY f.FinishTime)
ORDER BY f.Category, f.FinishTime, f.BibNumber;
```

All three return the same ten rows in the same order as the reference query.

## Explanation

### Step 0 — sort the raw data the way the windows will see it

The `INSERT` batch is deliberately out of order (Kenji Sato, one of the two Elite winners, is inserted last). Windows do not care about insertion order; they care about `PARTITION BY Category` and `ORDER BY FinishTime`. Rewriting the ten rows per category, sorted by finish time:

| Category | FinishTime | Bib | Runner | CharityRaised |
|---|---|---|---|---|
| Elite | 02:06:05 | 101 | Amara Diallo | 300.00 |
| Elite | 02:06:05 | 205 | Kenji Sato | 150.00 |
| Elite | 02:07:22 | 118 | Lucas Meyer | 200.00 |
| Elite | 02:08:30 | 122 | Priya Nair | 250.00 |
| Elite | 02:08:30 | 130 | Owen Carter | 100.00 |
| Elite | 02:10:18 | 149 | Tomas Silva | 50.00 |
| Masters | 02:20:04 | 210 | Grace Kim | 120.00 |
| Masters | 02:21:00 | 217 | Ivan Petrov | 200.00 |
| Masters | 02:21:00 | 233 | Hana Yusuf | 80.00 |
| Masters | 02:22:11 | 224 | Nadia Farah | 60.00 |

There are three exact ties: Elite `02:06:05` (bibs 101, 205), Elite `02:08:30` (bibs 122, 130), Masters `02:21:00` (bibs 217, 233). Everything difficult in this question flows from those three ties.

### Step 1 — `Place` must be `RANK`, not `ROW_NUMBER` and not `DENSE_RANK`

The expected `Place` column reads, per category:

- Elite: `1, 1, 3, 4, 4, 6`
- Masters: `1, 2, 2, 4`

Two properties identify the function:

1. **Tied rows share a value** (two 1s, two 4s, two 2s). `ROW_NUMBER()` never repeats a value inside a partition — it would emit `1, 2, 3, 4, 5, 6` for Elite (and which of Amara/Kenji gets 1 vs 2 would be nondeterministic, since nothing inside the window `ORDER BY` breaks the tie). Ruled out.
2. **Gaps follow the ties.** After the two Elite runners ranked 1, the next distinct time gets 3, not 2; after the two ranked 4, the last runner gets 6, not 5; after the two Masters ranked 2, Nadia Farah gets 4, not 3. `DENSE_RANK()` would compress those gaps and emit `1, 1, 2, 3, 3, 4` for Elite and `1, 2, 2, 3` for Masters. Ruled out.

Only `RANK()` produces "same rank for ties, then a jump equal to the number of tied rows": each row's rank is 1 + the number of rows strictly ahead of it in the partition.

Cell-by-cell:

| Row | Rows strictly faster in category | Place |
|---|---|---|
| 101 Amara | 0 | **1** |
| 205 Kenji | 0 (02:06:05 ties, does not precede) | **1** |
| 118 Lucas | 2 (bibs 101, 205) | **3** |
| 122 Priya | 3 (101, 205, 118) | **4** |
| 130 Owen | 3 (ties with 122) | **4** |
| 149 Tomas | 5 | **6** |
| 210 Grace | 0 | **1** |
| 217 Ivan | 1 (210) | **2** |
| 233 Hana | 1 (02:21:00 ties, does not precede) | **2** |
| 224 Nadia | 3 (210, 217, 233) | **4** |

Note also that `PARTITION BY Category` restarts the numbering: Grace Kim gets `1` even though five Elite runners finished before her. Without the partition, her place would be 7.

### Step 2 — `CharityBanked` needs the *default* frame, and the default is `RANGE`, not `ROWS`

Per Microsoft's documentation for the `OVER` clause: when an aggregate has an `ORDER BY` but no explicit `ROWS`/`RANGE`, the default frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, and a `RANGE ... CURRENT ROW` frame **includes every row whose `ORDER BY` value equals the current row's** — the current row's *peers*. So the running sum for a row is the sum over all rows of the partition whose `FinishTime` is less than **or equal to** the current row's, tied rows included, in both directions.

That is exactly the dashboard semantics in the statement: runners who cross together are credited together.

Cell-by-cell, Elite (category total 300 + 150 + 200 + 250 + 100 + 50 = 1050.00):

| Row | Frame = all Elite rows with FinishTime ≤ current | CharityBanked |
|---|---|---|
| 101 Amara (02:06:05) | {101, 205} → 300 + 150 | **450.00** |
| 205 Kenji (02:06:05) | {101, 205} — identical frame, peer of 101 | **450.00** |
| 118 Lucas (02:07:22) | {101, 205, 118} → 450 + 200 | **650.00** |
| 122 Priya (02:08:30) | {101, 205, 118, 122, 130} → 650 + 250 + 100 | **1000.00** |
| 130 Owen (02:08:30) | same five rows — peer of 122 | **1000.00** |
| 149 Tomas (02:10:18) | all six → 1000 + 50 | **1050.00** |

Masters (category total 120 + 200 + 80 + 60 = 460.00):

| Row | Frame | CharityBanked |
|---|---|---|
| 210 Grace (02:20:04) | {210} | **120.00** |
| 217 Ivan (02:21:00) | {210, 217, 233} → 120 + 200 + 80 | **400.00** |
| 233 Hana (02:21:00) | same three rows — peer of 217 | **400.00** |
| 224 Nadia (02:22:11) | all four → 400 + 60 | **460.00** |

The signature of `RANGE` is that **tied rows show the same running total**: 450.00/450.00, 1000.00/1000.00, 400.00/400.00. Each pair of tied rows *jumps* the running total by the sum of both peers at once (Kenji's frame already contains Amara, and vice versa).

### Why the naive alternatives fail

**`ROWS UNBOUNDED PRECEDING` (or `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`).** A `ROWS` frame is physical: it stops at the current row and does **not** pull in peers. The two Elite winners could then never both show 450.00 — one of them would show a partial sum (300.00 or 150.00, whichever the engine happens to place first, since `FinishTime` alone does not order them uniquely). The output would be wrong *and* nondeterministic. The grid's equal totals on tied rows are only reachable with the `RANGE` (default) frame.

**Adding the tiebreaker inside the window `ORDER BY`** — e.g. `SUM(...) OVER (PARTITION BY Category ORDER BY FinishTime, BibNumber)`. This looks harmless ("just making it deterministic") and is the subtlest wrong answer. With `BibNumber` in the window `ORDER BY`, no two rows are peers anymore, so even the default `RANGE` frame behaves row-by-row: Amara would show 300.00 and Kenji 450.00. It also breaks `Place`: `RANK() OVER (... ORDER BY FinishTime, BibNumber)` has no ties left and degenerates into `ROW_NUMBER` (`1, 2, 3, 4, 5, 6`). The unique tiebreaker belongs **only** in the outermost presentation `ORDER BY`, never in the window `ORDER BY` — the window must keep the ties, the presentation must remove them.

**Dropping `ORDER BY` from the `SUM`'s `OVER`.** With `PARTITION BY` alone the frame is the whole partition, so every Elite row would show 1050.00 and every Masters row 460.00 — a category total, not a running total.

**Dropping `PARTITION BY`.** Both columns would run across the whole field: Grace Kim would get place 7 and her `CharityBanked` would start from the Elite total (1050 + 120 = 1170.00), not 120.00.

### Step 3 — deterministic presentation order

The final `ORDER BY Category, FinishTime, BibNumber` sorts `Elite` before `Masters` (ascending `VARCHAR` order), then by finish time, and resolves the three ties by bib: 101 before 205, 122 before 130, 217 before 233 — matching the expected grid exactly. Without `BibNumber`, the relative order of each tied pair would be an implementation accident and constraint 3 would be violated. This is the mirror image of the previous trap: ties must survive *inside* `OVER (...)` and must be eliminated *outside* it.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

## DP-800 Exam Rule to Remember

A window `ORDER BY` and the presentation `ORDER BY` are two different instruments, and ties are treated oppositely in each:

```text
Inside OVER ( ORDER BY ... )   →  keep the ties: they define peers
Outside, final ORDER BY        →  kill the ties: add a unique key
```

Three consequences of a tie inside the window:

1. `ROW_NUMBER` splits it arbitrarily, `RANK` shares the value and leaves a gap (1, 1, 3), `DENSE_RANK` shares the value without a gap (1, 1, 2). Read the expected output's numbering pattern to pick the function.
2. An aggregate with `ORDER BY` but no frame defaults to `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, and `RANGE ... CURRENT ROW` includes all peer rows — tied rows get **equal** running totals. `ROWS` stops at the current row and gives tied rows different (and order-dependent) partial totals. Equal totals on tied rows in the expected output means the default `RANGE` frame; strictly increasing totals means `ROWS`.
3. Adding a tiebreaking column to the *window* `ORDER BY` silently destroys both behaviors above, because it removes the peers. Ranking functions accept no `ROWS`/`RANGE` clause at all, so for them the `ORDER BY` list is the only tie control there is.
