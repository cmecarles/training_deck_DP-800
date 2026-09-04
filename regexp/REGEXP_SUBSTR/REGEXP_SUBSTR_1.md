# SQL Server question — REGEXP_SUBSTR 1

## Statement

A biology laboratory stores free-text instrument log lines in SQL Server 2025. Official sample codes have the shape *two uppercase letters, hyphen, four digits* (for example `MK-0042`); technicians, however, sometimes type codes in lowercase, and some log lines contain no code at all.

The complete session below runs top to bottom on SQL Server 2025 (17.x):

```sql
CREATE DATABASE BioLab;
GO
ALTER DATABASE BioLab SET COMPATIBILITY_LEVEL = 170;
GO
USE BioLab;
GO
CREATE SCHEMA Lab;
GO
CREATE TABLE Lab.SampleLog
(
    SampleID int          NOT NULL PRIMARY KEY,
    LogLine  nvarchar(80) NOT NULL
);
GO
INSERT INTO Lab.SampleLog (SampleID, LogLine) VALUES
    (1, N'run1: MK-0042 rerun MK-0107 fail mk-0999'),
    (2, N'baseline only: mk-3005'),
    (3, N'no samples logged');
GO
SELECT
    SampleID,
    REGEXP_SUBSTR(LogLine, N'[A-Z]{2}-[0-9]{4}', 1, 2)                AS C1,
    REGEXP_SUBSTR(LogLine, N'[A-Z]{2}-[0-9]{4}', 8, 1, 'i')           AS C2,
    REGEXP_SUBSTR(LogLine, N'([A-Za-z]{2})-([0-9]{4})', 1, 1, 'c', 2) AS C3,
    REGEXP_SUBSTR(LogLine, N'[A-Z]{2}-[0-9]{4}', 1, 3, 'i')           AS C4
FROM Lab.SampleLog
ORDER BY SampleID;
```

Predict the **exact** result grid, writing NULL wherever the function returns NULL.

## Correct Answer

```text
SampleID C1      C2      C3   C4
-------- ------- ------- ---- -------
1        MK-0107 MK-0107 0042 mk-0999
2        NULL    mk-3005 3005 NULL
3        NULL    NULL    NULL NULL
```

This grid was executed and captured on SQL Server 2025 (RTM, 17.0.1000.7); every value, including every NULL, is the engine's actual output.

## Explanation

Signature — the argument order is the whole question:

```text
REGEXP_SUBSTR(string_expression, pattern_expression
              [, start [, occurrence [, flags [, group]]]])
```

Defaults: `start` 1, `occurrence` 1 (note: `REGEXP_REPLACE` defaults its occurrence to 0 = all — an asymmetry the exam loves), flags `c` (case-sensitive), `group` 0 (return the whole match). **If no qualifying match exists, the function returns NULL** — it never errors out for "not found".

### C1 — `(pattern, 1, 2)`: second occurrence, case-sensitive

The third argument is `start`, the fourth is `occurrence`. So C1 asks for the **2nd** case-sensitive occurrence starting the search at character 1.

- Row 1 `run1: MK-0042 rerun MK-0107 fail mk-0999`: case-sensitive occurrences of `[A-Z]{2}-[0-9]{4}` are `MK-0042` and `MK-0107`; `mk-0999` is lowercase and does not count. 2nd occurrence → **`MK-0107`**.
- Row 2 `baseline only: mk-3005`: the only code is lowercase; with flag defaulting to `c` there are zero occurrences → **NULL**.
- Row 3: no code shape at all → **NULL**.

### C2 — `(pattern, 8, 1, 'i')`: the start-position trap

`start = 8` means a match must **begin at or after character 8**; anything starting earlier is invisible even if it extends past position 8.

Row 1, character by character: `r(1) u(2) n(3) 1(4) :(5) ␠(6) M(7) K(8) ...` — `MK-0042` begins at character **7**. Since 7 < 8, that whole occurrence is out of the window: the engine cannot start a match at 8 (`K-0042…` fails `[A-Z]{2}-`). The first occurrence beginning at ≥ 8 is `MK-0107` (character 21) → **`MK-0107`** — identical to C1's answer, but for a completely different reason. Whoever thinks `start` merely "trims the output" or that a match *straddling* position 8 counts would answer `MK-0042`.

Row 2: `b(1) a(2) s(3) e(4) l(5) i(6) n(7) e(8) ...` — the search window opens at the second `e` of `baseline`; `mk-3005` begins at character 16, and flag `i` makes the lowercase code match → **`mk-3005`**.

Row 3: no match anywhere → **NULL**.

(Doc- and engine-consistent: a `start` greater than the string length also yields NULL, and `start < 1` raises an error.)

### C3 — `(pattern, 1, 1, 'c', 2)`: return capture group 2

The sixth argument, `group`, selects which parenthesized subexpression to return; 0 (default) is the whole match. Groups are numbered by the order of their left parentheses: group 1 = `([A-Za-z]{2})`, group 2 = `([0-9]{4})`.

- Row 1: first occurrence (the pattern's own class `[A-Za-z]` accepts either case, and flag `c` is irrelevant to it) is `MK-0042`; group 2 → **`0042`**.
- Row 2: first occurrence is `mk-3005` — matched *despite* the `c` flag because the character class itself allows lowercase; group 2 → **`3005`**. (Flags govern how classes/literals compare; a class that already lists `a-z` needs no `i`.)
- Row 3: no match → **NULL**. When the pattern does not match, the group argument is moot — there is no group to extract.

### C4 — `(pattern, 1, 3, 'i')`: occurrence overrun returns NULL

Case-insensitive occurrences:

- Row 1: `MK-0042`, `MK-0107`, `mk-0999` — the 3rd exists → **`mk-0999`**, returned exactly as stored (flags affect matching, never the returned text's case).
- Row 2: only one occurrence (`mk-3005`); asking for the 3rd overruns the count → **NULL**, silently — no error, no fallback to the last occurrence.
- Row 3: zero occurrences → **NULL**.

### The three roads to NULL

Row 3 shows road one (pattern never matches). C1/C4 on row 2 show road two (occurrence exceeds the number of matches). Road three is `start` beyond the last possible match position. All three produce plain NULL — which means downstream code must `IS NULL`-check extracted codes, and a `WHERE REGEXP_SUBSTR(...) = 'MK-0042'` silently drops non-matching rows (NULL is not equal to anything) rather than failing.

### Equivalent alternatives

- `[0-9]{4}` ≡ `\d{4}`: `REGEXP_SUBSTR(LogLine, N'[A-Z]{2}-\d{4}', 1, 2)` was executed against row 1 and returns the same `MK-0107`.
- C1 could be written with the flags spelled explicitly: `(LogLine, pattern, 1, 2, 'c', 0)` — `c` and group 0 are the defaults.
- Contrast function: to *test* rather than extract, `REGEXP_LIKE` is the predicate form — but remember it alone requires compatibility level 170, while `REGEXP_SUBSTR` works at any compatibility level.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

## DP-800 Exam Rule to Remember

`REGEXP_SUBSTR(string, pattern [, start [, occurrence [, flags [, group]]]])`

Memorize the positional order **start → occurrence → flags → group** (mnemonic: "**S**amples **O**f **F**ield **G**enetics"). The classic exam traps:

- Reading the 3rd argument as occurrence: it is `start`. The 4th is occurrence.
- `start` = where a match may **begin**; an occurrence that starts before `start` is skipped entirely, even if it overlaps the window.
- Default occurrence is **1** here, but **0 (= all)** in `REGEXP_REPLACE`.
- `group` (6th) returns one capture group; 0 = whole match; group numbers follow left parentheses.
- **NULL, never an error**, when: the pattern has no match, the requested occurrence does not exist, or `start` lies past any match. (`REGEXP_REPLACE` in the same situation returns the original string instead.)
- Flags change how the pattern *compares* (`i`, `m`, `s`, default `c`), never the case of what is *returned*, and a character class like `[A-Za-z]` bypasses case sensitivity on its own.
