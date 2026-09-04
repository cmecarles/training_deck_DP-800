# SQL Server question — REGEXP_COUNT 1

## Statement

A social-media analytics startup ingests raw post bodies into a SQL Server 2025 database and computes hashtag metrics with the new regular-expression functions.

The following script is run on a SQL Server 2025 (17.x) instance, from scratch:

```sql
CREATE DATABASE BuzzBoard;
GO
ALTER DATABASE BuzzBoard SET COMPATIBILITY_LEVEL = 170;
GO
USE BuzzBoard;
GO
CREATE SCHEMA Social;
GO
CREATE TABLE Social.Posts
(
    PostID int          NOT NULL PRIMARY KEY,
    Body   varchar(200) NOT NULL
);
GO
INSERT INTO Social.Posts (PostID, Body) VALUES
  (1, 'Loving the #SummerSale #summersale vibes! #Sale2026'),
  (2, 'no tags here, just vibes'),
  (3, 'hahahaha #LOL #lol #LoL'),
  (4, '#a #b#c##d');
GO
```

Then the analytics team runs this query:

```sql
SELECT
    PostID,
    REGEXP_COUNT(Body, '#\w+')            AS Tags,
    REGEXP_COUNT(Body, '#\w+', 13)        AS TagsFrom13,
    REGEXP_COUNT(Body, 'sale', 1, 'i')    AS SaleMentions,
    REGEXP_COUNT(Body, 'haha')            AS Laughs,
    REGEXP_COUNT(Body, 'x*')              AS XRuns
FROM Social.Posts
ORDER BY PostID;
```

Predict the **exact** count in every cell, for all four rows.

The full argument list is `REGEXP_COUNT(string, pattern [, start [, flags ]])`.

## Correct Answer

Output copied from the engine (SQL Server 2025 RTM, 17.0.1000.7):

| PostID | Tags | TagsFrom13 | SaleMentions | Laughs | XRuns |
|--------|------|------------|--------------|--------|-------|
| 1      | 3    | 2          | 3            | 0      | 52    |
| 2      | 0    | 0          | 0            | 0      | 25    |
| 3      | 3    | 2          | 0            | 2      | 24    |
| 4      | 4    | 0          | 0            | 0      | 11    |

## Explanation

The body lengths (needed for `XRuns`) are 51, 24, 23, and 10 characters. The hashtag `#` positions:

```text
Post 1: 'Loving the #SummerSale #summersale vibes! #Sale2026'
                    ^12        ^24                ^43            (LEN = 51)
Post 3: 'hahahaha #LOL #lol #LoL'
                  ^10  ^15  ^20                                  (LEN = 23)
Post 4: '#a #b#c##d'
          ^1 ^4 ^6^8,9                                           (LEN = 10)
```

### Tags — `'#\w+'`

- Post 1: `#SummerSale`, `#summersale`, `#Sale2026` → **3**.
- Post 2: no `#` at all → **0**.
- Post 3: `#LOL`, `#lol`, `#LoL` → **3**.
- Post 4: `#a` (1), `#b` (4), `#c` (6); at position 8 the `#` is followed by another `#` (not a `\w`), so it fails, but the `#d` starting at position 9 matches → **4**. A doubled `##` costs one candidate, not the whole tag.

### TagsFrom13 — `'#\w+', 13` (start = 13)

The scan begins at character 13; any match starting earlier is invisible. Matching still needs the literal `#` — the "tail" of a hashtag whose `#` lies before position 13 does not count.

- Post 1: the `#` of `#SummerSale` is at 12, one character too early. Only `#summersale` (24) and `#Sale2026` (43) are counted → **2** (the trap answer is 3).
- Post 2: no hashtags anywhere → **0**.
- Post 3: position 13 is the final `L` of `#LOL`, so `#LOL` is lost; `#lol` (15) and `#LoL` (20) remain → **2**.
- Post 4: the body is only 10 characters long, and when `start` exceeds the string length the function returns **0** — not NULL, not an error.

### SaleMentions — `'sale', 1, 'i'`

Flag `i` makes the search case-insensitive, and `sale` may appear inside a longer word:

- Post 1: `#Summer**Sale**`, `#summer**sale**`, `#**Sale**2026` → **3**.
- Posts 2–4: no `sale` in any case → **0**.

### Laughs — `'haha'` (non-overlapping counting)

Matches are counted **non-overlapping**: after a match is found, the scan resumes at the character *after* that match.

- Post 3: in `hahahaha` the matches are positions 1–4 and 5–8 → **2**. The overlapping occurrence at 3–6 is never counted (the trap answer is 3).
- Posts 1, 2, 4 → **0**.

### XRuns — `'x*'` (a pattern that matches the empty string)

None of the bodies contains an `x`, so every match of `x*` is **zero-length** — and `REGEXP_COUNT` counts one empty match at every position where a match attempt can start: before each character *and* once at the end of the string. That is `LEN(Body) + 1`:

- Post 1: 51 + 1 = **52**
- Post 2: 24 + 1 = **25**
- Post 3: 23 + 1 = **24**
- Post 4: 10 + 1 = **11**

A pattern that can match empty therefore never returns 0 for a non-NULL string — the classic reason production filters should use `x+`, not `x*`.

### Related edge behaviors (engine-verified)

- `REGEXP_COUNT(NULL, 'a')` → **NULL** (NULL in, NULL out — unlike no-match, which is 0).
- `REGEXP_COUNT('', 'a')` → **0**.
- `start < 1` raises an error; `start` past the end returns 0.
- Flags are `i`, `m`, `s`, `c` (default `c`); with contradictory flags the last one wins.

### Equivalent alternatives

- `SaleMentions` ≡ `REGEXP_COUNT(Body, '(?i)sale')` — the RE2-style inline option `(?i)` works too (engine-verified: also returns 3 on post 1) — and ≡ `REGEXP_COUNT(LOWER(Body), 'sale')`.
- `Tags` ≡ `REGEXP_COUNT(Body, '#[A-Za-z0-9_]+')` — `\w` is letters, digits, and underscore.
- `TagsFrom13` ≡ `REGEXP_COUNT(SUBSTRING(Body, 13, LEN(Body)), '#\w+')` — with `REGEXP_COUNT` (unlike `REGEXP_INSTR`) no absolute positions are returned, so shifting the string changes nothing.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

## DP-800 Exam Rule to Remember

`REGEXP_COUNT(string, pattern, start, flags)`:

- Counts **non-overlapping** matches, left to right; the scan resumes after each match's last character.
- `start` is 1-based; matches beginning before `start` are ignored entirely (a partial "tail" cannot match unless it satisfies the whole pattern). `start` past the end → **0**; `start < 1` → error.
- **No match → 0; NULL input → NULL.** Keep those two apart.
- A pattern that can match the empty string (`x*`, `\d*`, `.*?`) counts **LEN(string) + 1** zero-length matches on a string with no real match — one per character position plus one at the end.
- Flags `i`, `m`, `s`, `c` (default `c`); last contradictory flag wins.
- Unlike `REGEXP_INSTR`, there is **no** `occurrence`, `return_option`, or `group` argument — the signature stops at `flags`.
