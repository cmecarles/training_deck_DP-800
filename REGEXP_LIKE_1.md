# SQL Server question — REGEXP_LIKE 1

## Statement

A national vehicle authority migrates its registration system to SQL Server 2025. A valid modern plate has the format *two uppercase letters, hyphen, four digits, hyphen, two uppercase letters* (for example `KL-4821-XT`). The legacy loader, however, only enforced a very loose `CHECK` constraint, so the imported data contains historical junk.

The complete session below runs top to bottom on SQL Server 2025 (17.x). Every `INSERT` succeeds (the loose `CHECK` only demands a run of 3 or 4 consecutive digits somewhere in the value).

```sql
CREATE DATABASE FleetReg;
GO
ALTER DATABASE FleetReg SET COMPATIBILITY_LEVEL = 170;
GO
USE FleetReg;
GO
CREATE SCHEMA Dmv;
GO
CREATE TABLE Dmv.Plates
(
    PlateID int          NOT NULL PRIMARY KEY,
    Plate   nvarchar(40) NOT NULL
        CONSTRAINT CK_Plates_HasDigits
        CHECK (REGEXP_LIKE(Plate, N'[0-9]{3,4}'))
);
GO
INSERT INTO Dmv.Plates (PlateID, Plate) VALUES
    (1, N'KL-4821-XT'),
    (2, N'kl-4821-xt'),
    (3, N'OLD KL-4821-XT'),
    (4, N'KL-482-XT'),
    (5, N'KLX-4821-XT'),
    (6, N'scrapped' + NCHAR(10) + N'QF-0042-ZP'),
    (7, N'KL-4821-XTA'),
    (8, N'MN 7301 PC');
GO
```

Note that the value of `PlateID` 6 contains an embedded line feed: it is the two-line string `scrapped` + LF + `QF-0042-ZP`.

A data-quality engineer then runs:

```sql
SELECT PlateID, Plate
FROM Dmv.Plates
WHERE REGEXP_LIKE(Plate, N'^[A-Z]{2}-[0-9]{4}-[A-Z]{2}$', 'im')
ORDER BY PlateID;
```

Which `PlateID` values does the query return?

### a.

`1, 2`

### b.

`1, 2, 6`

### c.

`1, 2, 3, 6`

### d.

`1, 2, 6, 7`

## Correct Answer

**b**

The query returns exactly:

```text
PlateID     Plate
----------- ----------------------------------------
          1 KL-4821-XT
          2 kl-4821-xt
          6 scrapped
QF-0042-ZP
```

(Row 6 prints on two lines because the stored value itself contains a line feed.)

This result was executed and captured on SQL Server 2025 (RTM, 17.0.1000.7); every value above is the engine's actual output.

## Explanation

`REGEXP_LIKE(string_expression, pattern_expression [, flags])` returns a Boolean and is usable wherever a predicate is allowed — `WHERE`, `HAVING`, `CHECK` constraints, `CASE WHEN`. It is the **only** `REGEXP_*` function that requires `COMPATIBILITY_LEVEL = 170`; that is why the script raises the level immediately after `CREATE DATABASE`.

The flags argument `'im'` combines:

- `i` — case-insensitive matching (default is `c`, case-**sensitive**);
- `m` — multi-line mode: `^` and `$` match at the beginning/end of each *line* (around every LF), in addition to the beginning/end of the whole string. The "last flag wins" rule applies only to *contradictory* flags (`i` vs `c`); `i` and `m` do not contradict, so both are active (`'im'` and `'mi'` were verified to return identical rows).

Walk the anchored pattern `^[A-Z]{2}-[0-9]{4}-[A-Z]{2}$` against each row (with `i`, read `[A-Z]` as "any letter"):

| PlateID | Value | Verdict | Why |
|---|---|---|---|
| 1 | `KL-4821-XT` | **match** | `^`KL`-`4821`-`XT`$` — exact shape. |
| 2 | `kl-4821-xt` | **match** | Only because of `i`; the default `c` would reject it. |
| 3 | `OLD KL-4821-XT` | no | `^` anchors at position 1: `O`,`L` consume `[A-Z]{2}`, then the pattern demands `-` but finds `D`. `{2}` is exact, so there is nothing to backtrack. `m` adds anchor points only around line feeds, and this value has none — `m` does **not** turn `^...$` into "contains". |
| 4 | `KL-482-XT` | no | `[0-9]{4}` needs four digits; `482-` fails at the fourth position. |
| 5 | `KLX-4821-XT` | no | After `^KL` the pattern demands `-` but finds `X`. (Unanchored, this value *would* match via the substring `LX-4821-XT` — see below.) |
| 6 | `scrapped`+LF+`QF-0042-ZP` | **match** | The trap. With `m`, `^` also matches just after the LF and `$` at end of string, so line 2, `QF-0042-ZP`, matches on its own. Without `m` the value is one 19-character string that starts with `s` — no match (verified: dropping `m` returns only `1, 2`). |
| 7 | `KL-4821-XTA` | no | `^KL-4821-` then `[A-Z]{2}` takes `XT`, but `$` finds `A` instead of end-of-string. Retrying `[A-Z]{2}` as `TA` puts the second hyphen check on `X` and fails. The pattern only accepts strings of exactly 10 characters; this one has 11. |
| 8 | `MN 7301 PC` | no | Separators are spaces, not hyphens. |

### Why option a is wrong

`1, 2` is what you get if you overlook the `m` flag (or believe `m` in `'im'` cancels `i` under the "last flag wins" rule — it does not; that rule only arbitrates `i` vs `c`). With `m` active, `^` and `$` re-anchor around the embedded LF in row 6, and its second line `QF-0042-ZP` is a full, uppercase, correctly shaped plate. Engine-verified: the flag string `'i'` alone returns `1, 2`; the flag string `'im'` returns `1, 2, 6`.

### Why option c is wrong

`c` adds row 3 on the theory that `m` "relaxes" the anchors so that any value *containing* a valid plate matches. `m` does nothing of the sort: it only creates additional anchor points immediately before and after line-feed characters. `OLD KL-4821-XT` contains no LF, so its only anchor points are the true start and end of the string, and the leading `OLD ` kills the match.

### Why option d is wrong

`d` adds row 7 on the misreading that `[A-Z]{2}` means "two **or more** letters" (that would be `{2,}`) or that `$` tolerates trailing characters. `{2}` is an exact quantifier and `$` demands end-of-string (or, with `m`, end-of-line). `KL-4821-XTA` has one letter too many and is rejected.

### The anchors lesson, demonstrated on the same table

The same pattern **without** anchors and with the default case-sensitive flag —

```sql
WHERE REGEXP_LIKE(Plate, N'[A-Z]{2}-[0-9]{4}-[A-Z]{2}', 'c')
```

— was executed against the same rows and returns `1, 3, 5, 6, 7`: every value that merely *contains* a plate-shaped substring, including the junk rows 3, 5 and 7. This is exactly the mistake the loose `CHECK` constraint made: an unanchored `REGEXP_LIKE` in a `CHECK` validates a substring, not the value. (The `CHECK` does work as far as it goes — inserting `N'NO-DIGITS-HERE'` fails with error 547 — it is just far too permissive.)

Also engine-verified: without `m`, `$` does **not** match before a trailing line feed — `REGEXP_LIKE(N'AB-1234-CD' + NCHAR(10), N'^...$', 'c')` is false in SQL Server 2025, so a value with a trailing LF fails a `$`-anchored case-sensitive check.

### Equivalent alternatives

- `[0-9]` ≡ `\d`: the pattern `N'^[A-Z]{2}-\d{4}-[A-Z]{2}$'` with `'im'` was executed and returns the same rows.
- Flag order is irrelevant for non-contradictory flags: `'mi'` ≡ `'im'` (verified).

## DP-800 Exam Rule to Remember

`REGEXP_LIKE(string, pattern [, flags])` — two mandatory arguments, flags third.

- **Compatibility level 170 is required for `REGEXP_LIKE` only**; the other `REGEXP_*` scalar functions (`REGEXP_SUBSTR`, `REGEXP_REPLACE`, `REGEXP_COUNT`, `REGEXP_INSTR`) work at any compatibility level.
- Flags: `c` case-sensitive (default), `i` case-insensitive, `m` multi-line anchors, `s` lets `.` match LF. Contradictory flags (`i`/`c`): the last one wins.
- **Validation needs anchors.** In a `WHERE` or a `CHECK` constraint, `pattern` means *contains* unless you pin it with `^` and `$`. Unanchored patterns accept garbage.
- `m` never weakens anchors; it only adds line-boundary anchor points at embedded line feeds — which can silently admit multi-line values whose *second* line looks valid.
