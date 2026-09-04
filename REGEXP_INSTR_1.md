# SQL Server question — REGEXP_INSTR 1

## Statement

A hosting company stores simplified web-server access-log lines in a SQL Server 2025 database and uses the new regular-expression functions to scan them.

The following script is run on a SQL Server 2025 (17.x) instance, from scratch:

```sql
CREATE DATABASE HostMon;
GO
ALTER DATABASE HostMon SET COMPATIBILITY_LEVEL = 170;
GO
USE HostMon;
GO
CREATE SCHEMA Logs;
GO
CREATE TABLE Logs.AccessLog
(
    LogID   int          NOT NULL PRIMARY KEY,
    LogLine varchar(100) NOT NULL
);
GO
INSERT INTO Logs.AccessLog (LogID, LogLine) VALUES
  (1, 'GET /api/v1/items HTTP/1.1 200'),
  (2, 'get /API/v1/ADMIN http/1.0 403'),
  (3, 'POST /login HTTP/1.1 302');
GO
```

Then the monitoring team runs this query:

```sql
SELECT
    LogID,
    REGEXP_INSTR(LogLine, '^[A-Z]+', 1, 1, 1)               AS MethodEnd,
    REGEXP_INSTR(LogLine, '/\w+', 1, 3)                     AS ThirdSeg,
    REGEXP_INSTR(LogLine, '/\w+', 6, 1)                     AS SegFrom6,
    REGEXP_INSTR(LogLine, 'http/1\.[01]', 1, 1, 1, 'i')     AS AfterProto,
    REGEXP_INSTR(LogLine, ' (\d{3})$', 1, 1, 0, 'c', 1)     AS StatusPos,
    REGEXP_INSTR(LogLine, 'admin', 1, 1, 0, 'ci')           AS AdminPos
FROM Logs.AccessLog
ORDER BY LogID;
```

Predict the **exact** result set: every integer in every cell, for all three rows.

The full argument list is `REGEXP_INSTR(string, pattern [, start [, occurrence [, return_option [, flags [, group ]]]]])`.

## Correct Answer

Output copied from the engine (SQL Server 2025 RTM, 17.0.1000.7):

| LogID | MethodEnd | ThirdSeg | SegFrom6 | AfterProto | StatusPos | AdminPos |
|-------|-----------|----------|----------|------------|-----------|----------|
| 1     | 4         | 12       | 9        | 27         | 28        | 0        |
| 2     | 0         | 12       | 9        | 27         | 28        | 13       |
| 3     | 5         | 0        | 6        | 21         | 22        | 0        |

## Explanation

First lay a 1-based character ruler over the three lines (positions are 1-based in every `REGEXP_*` function):

```text
          1         2         3
 123456789012345678901234567890
'GET /api/v1/items HTTP/1.1 200'   -- LogID 1
'get /API/v1/ADMIN http/1.0 403'   -- LogID 2
'POST /login HTTP/1.1 302'         -- LogID 3
```

### MethodEnd — `'^[A-Z]+', 1, 1, 1` (return_option = 1)

`return_option = 0` returns the **start** of the match; `return_option = 1` returns the position **after the last character** of the match — not the position of the last character.

- Row 1: `GET` occupies 1–3 → returns **4**, not 3.
- Row 2: `get` is lowercase and the default flag is `c` (case-sensitive), so `^[A-Z]+` finds no match → **0**. `REGEXP_INSTR` never returns NULL for a non-NULL input; no match is always `0`.
- Row 3: `POST` occupies 1–4 → **5**.

### ThirdSeg — `'/\w+', 1, 3` (occurrence = 3)

Occurrences are counted non-overlapping, left to right. Note that `/1` inside `HTTP/1.1` also matches `/\w+` — but it is the *fourth* occurrence in rows 1–2, so it never surfaces here.

- Row 1: `/api` (5), `/v1` (9), `/items` (12) → third occurrence starts at **12**.
- Row 2: `/API` (5), `/v1` (9), `/ADMIN` (12) → **12**.
- Row 3: only `/login` (6) and `/1` (17) exist. There is no third occurrence → **0**.

### SegFrom6 — `'/\w+', 6, 1` (start = 6)

`start` moves the beginning of the *scan*, but every returned position stays **absolute** within the original string. A match whose first character lies before `start` is simply invisible — it is not "partially" found.

- Rows 1–2: position 6 is the letter after `/a`/`/A`. The `/api` (or `/API`) match starting at 5 can no longer be seen, because from position 6 onward there is no leading `/` until position 9 → **9** (the trap answer is 5).
- Row 3: position 6 is exactly the `/` of `/login` → **6**.

### AfterProto — `'http/1\.[01]', 1, 1, 1, 'i'`

Case-insensitive flag `i`, and `return_option = 1` again:

- Row 1: `HTTP/1.1` occupies 19–26 → **27**.
- Row 2: `http/1.0` occupies 19–26 → **27**.
- Row 3: `HTTP/1.1` occupies 13–20 → **21**.

### StatusPos — `' (\d{3})$', 1, 1, 0, 'c', 1` (group = 1)

The seventh argument selects a **capture group**: the position returned is the start of what group 1 matched (the three digits), not the start of the whole match (which includes the leading space).

- Rows 1–2: the whole match ` 200` / ` 403` starts at 27, but group 1 starts at **28**.
- Row 3: the whole match ` 302` starts at 21, group 1 at **22**.

(If `group` were larger than the number of groups in the pattern, the function would return `0`, verified on the engine.)

### AdminPos — `'admin', 1, 1, 0, 'ci'` (contradictory flags)

When the flags string contains contradictory characters, SQL Server uses the **last** one. `'ci'` therefore means case-**insensitive** (`'ic'` would mean case-sensitive).

- Row 2: `ADMIN` starts at **13**.
- Rows 1 and 3 contain no `admin` in any case → **0**.

### Equivalent alternatives

- `StatusPos` could equally be written without the `group` argument as `REGEXP_INSTR(LogLine, '\d{3}$')` — anchoring directly on the digits returns the same 28 / 28 / 22.
- `AdminPos` is equivalent to `REGEXP_INSTR(LogLine, 'admin', 1, 1, 0, 'i')` — the leading `c` is dead weight once `i` follows it.
- The default argument values are `start = 1`, `occurrence = 1`, `return_option = 0`, `flags = 'c'`, `group = 0`, so `REGEXP_INSTR(LogLine, '/\w+')` ≡ `REGEXP_INSTR(LogLine, '/\w+', 1, 1, 0, 'c', 0)`.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

## DP-800 Exam Rule to Remember

`REGEXP_INSTR(string, pattern, start, occurrence, return_option, flags, group)`:

- All positions are **1-based** and always **absolute** in the original string, even when `start > 1`.
- `return_option 0` = position of the **first** character of the match; `return_option 1` = position **after the last** character (last char + 1). Any other value is an error.
- **No match → 0** (never NULL for non-NULL inputs). Also `0` when `start` is beyond the end of the string, and when `group` exceeds the number of capture groups. `start < 1` is an error.
- `start` hides any match that begins before it; `occurrence` then counts matches from `start` onward, non-overlapping.
- Flags are `i`, `m`, `s`, `c` (default `c`); with contradictory flags, the **last one wins** (`'ci'` → insensitive, `'ic'` → sensitive).
- `group = 0` (default) positions the whole match; `group = n` positions what the *n*-th parenthesized group captured.
