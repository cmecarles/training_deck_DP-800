# SQL Server question — Window Functions 2

## Statement

A solar cooperative stores the monthly energy readings of its rooftop panels in a SQL Server database called `SunMeter`. A reading is `NULL` when the panel's meter was offline for that month. The script below is the complete history of the database.

```sql
CREATE DATABASE SunMeter;
GO
USE SunMeter;
GO
CREATE SCHEMA Solar;
GO
CREATE TABLE Solar.Readings
(
    PanelId   CHAR(2)      NOT NULL,
    ReadMonth DATE         NOT NULL,
    kWh       DECIMAL(6,1) NULL,
    CONSTRAINT PK_Readings PRIMARY KEY (PanelId, ReadMonth)
);
GO
INSERT INTO Solar.Readings (PanelId, ReadMonth, kWh) VALUES
    ('P1', '2026-01-01', 410.0),
    ('P1', '2026-02-01', NULL),
    ('P1', '2026-03-01', 380.0),
    ('P1', '2026-04-01', 395.0),
    ('P1', '2026-05-01', NULL),
    ('P1', '2026-06-01', NULL),
    ('P1', '2026-07-01', 450.0),
    ('P2', '2026-01-01', NULL),
    ('P2', '2026-02-01', 380.0),
    ('P2', '2026-03-01', 420.0),
    ('P2', '2026-04-01', NULL),
    ('P2', '2026-05-01', 405.0);
GO
```

Two queries are then executed. **Q1** walks each panel's timeline:

```sql
-- Q1
SELECT
    r.PanelId,
    r.ReadMonth,
    r.kWh,
    LAG(r.kWh, 1, 0)   OVER (PARTITION BY r.PanelId ORDER BY r.ReadMonth) AS PrevKwh,
    LEAD(r.kWh, 2, -1) OVER (PARTITION BY r.PanelId ORDER BY r.ReadMonth) AS TwoAhead,
    FIRST_VALUE(r.kWh) OVER (PARTITION BY r.PanelId ORDER BY r.ReadMonth) AS FirstKwh,
    LAST_VALUE(r.kWh)  OVER (PARTITION BY r.PanelId ORDER BY r.ReadMonth) AS LastDefault,
    LAST_VALUE(r.kWh)  OVER (PARTITION BY r.PanelId ORDER BY r.ReadMonth
                             ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING) AS LastFixed,
    LAST_VALUE(r.kWh) IGNORE NULLS
                       OVER (PARTITION BY r.PanelId ORDER BY r.ReadMonth) AS Carried
FROM Solar.Readings AS r
ORDER BY r.PanelId, r.ReadMonth;
```

**Q2** ranks the months that actually have a reading, across both panels:

```sql
-- Q2
SELECT
    r.PanelId,
    r.ReadMonth,
    r.kWh,
    NTILE(3) OVER (ORDER BY r.kWh DESC, r.PanelId, r.ReadMonth)        AS Tercile,
    CAST(PERCENT_RANK() OVER (ORDER BY r.kWh) AS DECIMAL(5,4))         AS PctRank,
    CAST(CUME_DIST()    OVER (ORDER BY r.kWh) AS DECIMAL(5,4))         AS CumeDist
FROM Solar.Readings AS r
WHERE r.kWh IS NOT NULL
ORDER BY r.kWh, r.PanelId, r.ReadMonth;
```

Give the **exact** result set of Q1 (12 rows) and of Q2 (7 rows): every value, including every `NULL`, in the row order produced.

## Correct Answer

**Q1** (engine output; `DECIMAL(6,1)` values):

```text
PanelId  ReadMonth   kWh    PrevKwh  TwoAhead  FirstKwh  LastDefault  LastFixed  Carried
-------  ----------  -----  -------  --------  --------  -----------  ---------  -------
P1       2026-01-01  410.0    0.0     380.0     410.0      410.0        450.0     410.0
P1       2026-02-01  NULL   410.0     395.0     410.0      NULL         450.0     410.0
P1       2026-03-01  380.0  NULL      NULL      410.0      380.0        450.0     380.0
P1       2026-04-01  395.0  380.0     NULL      410.0      395.0        450.0     395.0
P1       2026-05-01  NULL   395.0     450.0     410.0      NULL         450.0     395.0
P1       2026-06-01  NULL   NULL       -1.0     410.0      NULL         450.0     395.0
P1       2026-07-01  450.0  NULL       -1.0     410.0      450.0        450.0     450.0
P2       2026-01-01  NULL     0.0     420.0     NULL       NULL         405.0     NULL
P2       2026-02-01  380.0  NULL      NULL      NULL       380.0        405.0     380.0
P2       2026-03-01  420.0  380.0     405.0     NULL       420.0        405.0     420.0
P2       2026-04-01  NULL   420.0      -1.0     NULL       NULL         405.0     420.0
P2       2026-05-01  405.0  NULL       -1.0     NULL       405.0        405.0     405.0
```

**Q2**:

```text
PanelId  ReadMonth   kWh    Tercile  PctRank  CumeDist
-------  ----------  -----  -------  -------  --------
P1       2026-03-01  380.0  3        0.0000   0.2857
P2       2026-02-01  380.0  3        0.0000   0.2857
P1       2026-04-01  395.0  2        0.3333   0.4286
P2       2026-05-01  405.0  2        0.5000   0.5714
P1       2026-01-01  410.0  1        0.6667   0.7143
P2       2026-03-01  420.0  1        0.8333   0.8571
P1       2026-07-01  450.0  1        1.0000   1.0000
```

## Explanation

Every window in Q1 is `PARTITION BY PanelId ORDER BY ReadMonth`, so each panel is processed alone, in calendar order. `ReadMonth` is unique inside a panel (it is part of the primary key), so there are **no peers** in Q1 — which is exactly why the `LAST_VALUE` trap shows up in its purest form.

### `PrevKwh` — `LAG(kWh, 1, 0)`: the default only covers "no such row", not "the row is NULL"

`LAG(expr, offset, default)` returns the value of `expr` from the row `offset` positions *before* the current one. The third argument is used **only when that row does not exist** (the offset falls before the start of the partition). It is *not* a `COALESCE`:

- P1 January and P2 January have no previous row → `0.0` (the default, converted to `DECIMAL(6,1)`).
- P1 March's previous row is P1 February, which **exists** but holds `NULL` → `LAG` returns that `NULL`, not `0.0`. Same for P1 June, P1 July, P2 February and P2 May.

So the `PrevKwh` column has both `0.0` (no previous month) and `NULL` (previous month offline), and they mean different things. To skip offline months you would write `LAG(kWh, 1, 0) IGNORE NULLS`, which returns the last **non-null** previous value (verified: P1 July would then show `395.0`, P1 March `410.0`).

### `TwoAhead` — `LEAD(kWh, 2, -1)`: same rule, looking forward

The offset can be any positive integer; `2` means "two rows later". The last two rows of each partition have no row two positions ahead, so they get the default `-1.0` (P1 June/July, P2 April/May). Every other `NULL` in the column (P1 March, P1 April, P2 February) is a real `NULL` read from an existing offline row.

### `FirstKwh` — `FIRST_VALUE` with the default frame

With `ORDER BY` and no explicit frame, the frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. For `FIRST_VALUE` that frame always starts at the first row of the partition, so the default frame is harmless: P1 gets `410.0` on every row. P2's first row is offline, so `FIRST_VALUE` faithfully returns `NULL` for all five P2 rows — a `NULL` is a value like any other unless you say `IGNORE NULLS` (verified: `FIRST_VALUE(kWh) IGNORE NULLS` gives `380.0` for P2 from February onward, and still `NULL` for P2 January, whose frame contains no non-null row yet).

### `LastDefault` — the `LAST_VALUE` trap

The same default frame ends at `CURRENT ROW`. The *last* row of a frame that ends at the current row **is the current row**, so `LAST_VALUE(kWh) OVER (PARTITION BY PanelId ORDER BY ReadMonth)` simply echoes `kWh`: `410.0, NULL, 380.0, 395.0, NULL, NULL, 450.0` for P1. It never reaches the "last reading of the panel" that the column name suggests. (Had `ReadMonth` contained ties, the `RANGE` frame would extend to the current row's peers and `LAST_VALUE` would return an arbitrary peer's value — worse, not better.)

### `LastFixed` — the frame that makes `LAST_VALUE` useful

`ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING` extends the frame to the end of the partition, so the last row of the frame is the last row of the panel: `450.0` (P1 July) on every P1 row, `405.0` (P2 May) on every P2 row. `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` would give the same answer.

Two related compile-time rules were verified on this build:

- Omitting `ORDER BY` from `FIRST_VALUE`/`LAST_VALUE`/`LAG`/`LEAD` is an error: `Msg 4112 — The function 'LAST_VALUE' must have an OVER clause with ORDER BY.` (identical text for `LAG`).
- Adding a frame to `LAG`/`LEAD` (or to any ranking/distribution function) is an error: `Msg 10752 — The function 'LAG' may not have a window frame.` (`'PERCENT_RANK'` produces the same message). Only aggregates and `FIRST_VALUE`/`LAST_VALUE` accept `ROWS`/`RANGE`.

### `Carried` — `LAST_VALUE ... IGNORE NULLS` with the default frame is "carry the last reading forward"

Here the default frame (`UNBOUNDED PRECEDING` to `CURRENT ROW`) is exactly what you want: the last non-null value **up to and including the current row**. P1: `410.0` carries through February; `380.0`, `395.0` are their own; `395.0` carries through May and June; July is `450.0`. P2 January has no non-null value in its frame yet, so `Carried` is `NULL`. `IGNORE NULLS`/`RESPECT NULLS` (the default) exist since SQL Server 2022 for `FIRST_VALUE`, `LAST_VALUE`, `LAG` and `LEAD` only — no other window function accepts them.

### Q2 — `NTILE(3)` with 7 rows

`NTILE(n)` splits the ordered partition into `n` groups. When the row count is not divisible by `n`, the **first** groups get one extra row: 7 rows into 3 terciles gives sizes 3, 2, 2. Ordered by `kWh DESC`: `450, 420, 410` → 1; `405, 395` → 2; `380, 380` → 3. The window `ORDER BY` deliberately adds `PanelId, ReadMonth` after `kWh DESC` because the two `380.0` readings tie — `NTILE` never shares a value between tied rows, so without a unique ordering the assignment inside a tie would be nondeterministic (here both 380s land in tercile 3 either way, but in general a tie can straddle a bucket boundary).

### Q2 — `PERCENT_RANK` versus `CUME_DIST`

Both are computed over `ORDER BY kWh` alone, so the two `380.0` rows are peers and must receive identical values. With `n = 7`:

- `PERCENT_RANK() = (RANK() − 1) / (n − 1)`: the lowest value(s) always get **0**, the highest always gets **1**. The two 380s have `RANK 1` → `0/6 = 0.0000`; 395 has `RANK 3` (rank skips because of the tie) → `2/6 = 0.3333`; 405 → `3/6 = 0.5000`; 410 → `4/6 = 0.6667`; 420 → `5/6 = 0.8333`; 450 → `6/6 = 1.0000`.
- `CUME_DIST() = (rows with value ≤ current) / n`: the two 380s → `2/7 = 0.2857`; 395 → `3/7 = 0.4286`; 405 → `4/7 = 0.5714`; 410 → `5/7 = 0.7143`; 420 → `6/7 = 0.8571`; 450 → `7/7 = 1.0000`.

The tell-tale differences: `PERCENT_RANK` starts at 0 and is *never* 0 for `CUME_DIST`; `CUME_DIST` counts the current row and its peers (2/7 for the tied pair), `PERCENT_RANK` counts rows strictly below. Both return `FLOAT` — the `CAST(... AS DECIMAL(5,4))` in Q2 only fixes the display.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

## DP-800 Exam Rule to Remember

```text
LAG/LEAD(expr, offset, default)   default fires only when the offset row DOES NOT EXIST;
                                  an existing NULL row returns NULL (use IGNORE NULLS to skip it)
FIRST_VALUE / LAST_VALUE          default frame = RANGE UNBOUNDED PRECEDING .. CURRENT ROW
                                  -> LAST_VALUE == current row (or a peer). To get the real last row:
                                     ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
LAST_VALUE(x) IGNORE NULLS (default frame) = carry last known value forward  (SQL Server 2022+)
NTILE(n)                          n buckets; the FIRST buckets take the remainder (7 rows / 3 -> 3,2,2);
                                  ties are split arbitrarily -> make the window ORDER BY unique
PERCENT_RANK = (rank-1)/(n-1)     starts at 0, ends at 1, counts rows strictly below
CUME_DIST    = (rows <= me)/n     never 0, ends at 1, counts me and my peers
```

Two compile-time gates: `LAG`, `LEAD`, `FIRST_VALUE`, `LAST_VALUE` and every ranking/distribution function **require** `ORDER BY` in `OVER` (Msg 4112), and only aggregates plus `FIRST_VALUE`/`LAST_VALUE` may carry a `ROWS`/`RANGE` frame (Msg 10752 for the rest). When an expected output shows `LAST_VALUE` equal to the current row, the author forgot the frame — that is the trap, not a bug in the engine.
