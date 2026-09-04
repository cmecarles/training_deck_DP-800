# SQL Server question — EDIT_DISTANCE 1

## Statement

The Hartfield Genealogy Society collects surname spellings that members transcribe from 19th-century parish registers. Volunteers reconcile each submitted spelling against the canonical spelling held in the society's registry. Because transcription errors are common (a dropped letter, a swapped pair, an over-enthusiastic use of capitals), the society upgraded to SQL Server 2025 to use the new fuzzy string-matching functions.

The following script is executed, exactly as shown, on a SQL Server 2025 (17.x) instance:

```sql
CREATE DATABASE KinRecords COLLATE Latin1_General_CI_AS;
GO
ALTER DATABASE KinRecords SET COMPATIBILITY_LEVEL = 170;
GO
USE KinRecords;
GO
ALTER DATABASE SCOPED CONFIGURATION SET PREVIEW_FEATURES = ON;
GO
CREATE SCHEMA Gen;
GO
CREATE TABLE Gen.SurnameLinks
(
    LinkID           int         NOT NULL PRIMARY KEY,
    SubmittedSurname varchar(40) NULL,
    RegistrySurname  varchar(40) NULL
);
GO
INSERT INTO Gen.SurnameLinks (LinkID, SubmittedSurname, RegistrySurname) VALUES
  (1, 'Whitfield',   'Whitfield'),
  (2, 'Smyth',       'Smith'),
  (3, 'Jonson',      'Johnson'),
  (4, 'Kowaslki',    'Kowalski'),
  (5, 'DELACROIX',   'Delacroix'),
  (6, 'Baumgartner', 'Bumgarner'),
  (7, 'Ashworth',    NULL);
GO
SELECT LinkID,
       SubmittedSurname,
       RegistrySurname,
       CASE WHEN SubmittedSurname = RegistrySurname
            THEN 'yes' ELSE 'no' END                       AS CollationSaysEqual,
       EDIT_DISTANCE(SubmittedSurname, RegistrySurname)    AS Dist
FROM Gen.SurnameLinks
ORDER BY LinkID;
```

Predict the **exact** result set of the final `SELECT`: every value of `CollationSaysEqual` and `Dist` for all rows, including any `NULL`s, in the returned order.

## Correct Answer

The query returns 7 rows (values copied from the actual output of SQL Server 2025 RTM, 17.0.1000.7):

| LinkID | SubmittedSurname | RegistrySurname | CollationSaysEqual | Dist |
|-------:|------------------|-----------------|--------------------|-----:|
| 1 | Whitfield | Whitfield | yes | 0 |
| 2 | Smyth | Smith | no | 1 |
| 3 | Jonson | Johnson | no | 1 |
| 4 | Kowaslki | Kowalski | no | **2** |
| 5 | DELACROIX | Delacroix | **yes** | **8** |
| 6 | Baumgartner | Bumgarner | no | 2 |
| 7 | Ashworth | NULL | **no** | NULL |

## Explanation

`EDIT_DISTANCE(s1, s2)` is new in SQL Server 2025 (17.x), returns **int**, and counts the minimum number of single-character **insertions, deletions, and substitutions** needed to turn one string into the other. One setup detail is a prerequisite, not decoration: on SQL Server 2025 RTM the function is gated behind `ALTER DATABASE SCOPED CONFIGURATION SET PREVIEW_FEATURES = ON` — without it the engine raises *Msg 195: 'EDIT_DISTANCE' is not a recognized built-in function name* even at compatibility level 170. (`COMPATIBILITY_LEVEL = 170` is the level the documentation ties the feature to, but on this RTM build the preview switch alone is decisive: engine-verified, with `PREVIEW_FEATURES = ON` the function also runs at compatibility level 160.)

Walk each row's edit operations:

- **Row 1 — Whitfield / Whitfield → 0.** Identical strings need zero operations; identical also under the collation, so `CollationSaysEqual` is `yes`.
- **Row 2 — Smyth → Smith → 1.** One substitution: `y` → `i` at position 3. All other characters align.
- **Row 3 — Jonson → Johnson → 1.** One insertion: add `h` after `Jo`. The length difference (6 vs 7) already forces at least one operation, and one suffices.
- **Row 4 — Kowaslki → Kowalski → 2, not 1.** The only defect is the adjacent pair `sl` swapped into `ls` — a transposition. The Microsoft Learn page says the function "implements the Damerau-Levenshtein algorithm", but the same page carries the note *"EDIT_DISTANCE currently doesn't support transpositions"*, and the engine confirms it: the swap is charged as **two substitutions** (`s`→`l`, `l`→`s`), i.e. plain **Levenshtein** behavior. A pure Damerau implementation would return 1. This is the subtle distractor of the question.
- **Row 5 — DELACROIX / Delacroix → CollationSaysEqual = 'yes', Dist = 8.** The `=` comparison uses the database collation `Latin1_General_CI_AS`, which is **c**ase-**i**nsensitive, so the strings compare equal. `EDIT_DISTANCE`, however, **ignores the collation and compares characters exactly**: verified empirically, `EDIT_DISTANCE('SMITH','smith')` returns 5 under a CI collation (and identically with an explicit `COLLATE Latin1_General_CS_AS`). Here `D` matches and each of the remaining 8 letters differs by case (`ELACROIX` vs `elacroix`), so 8 substitutions. A row the collation calls a duplicate can still be maximally distant for the fuzzy function — the trap most candidates fall into.
- **Row 6 — Baumgartner → Bumgarner → 2.** Two deletions: drop the `a` of `Ba` (giving `Bumgartner`) and drop the `t` (giving `Bumgarner`). Length 11 → 9, and every remaining character aligns, so the length difference of 2 is also the distance.
- **Row 7 — Ashworth / NULL → 'no', NULL.** If either input is `NULL`, `EDIT_DISTANCE` returns `NULL`. Note the `CASE` column: `'Ashworth' = NULL` evaluates to UNKNOWN, so the `ELSE` branch fires and prints `'no'` — the `CollationSaysEqual` column is `'no'`, not `NULL`.

`ORDER BY LinkID` makes the row order deterministic: 1 through 7.

Notes and equivalent alternatives:

- `EDIT_DISTANCE` is symmetric: `EDIT_DISTANCE(RegistrySurname, SubmittedSurname)` returns identical values.
- The `CASE` expression could equivalently be written `IIF(SubmittedSurname = RegistrySurname, 'yes', 'no')`.
- The function accepts an optional third argument, `maximum_distance`, as a short-circuit for filtering (`EDIT_DISTANCE(a, b, 2)`). Per the documentation, when the true distance exceeds the cap the function *may* return any value ≥ the cap (observed: `EDIT_DISTANCE('Johannessen','Jones', 2)` returns 2 while the true distance is 6), so its exact output is not something to hard-code in expected results — only the predicate `EDIT_DISTANCE(a, b, @n) <= @n` is reliable.
- The inputs cannot be `varchar(max)`/`nvarchar(max)`.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

## DP-800 Exam Rule to Remember

`EDIT_DISTANCE` (SQL Server 2025, documented for compat level 170, preview — gated by `PREVIEW_FEATURES = ON` on RTM):

```text
returns int = minimum # of insertions + deletions + substitutions
```

Three behaviors that override intuition:

1. **Collation does not apply.** Comparison is exact per character; a case-insensitive database still yields `EDIT_DISTANCE('SMITH','smith') = 5` even though `'SMITH' = 'smith'` is TRUE.
2. **A transposition costs 2**, not 1 — despite the "Damerau-Levenshtein" label, transpositions are currently unsupported (Levenshtein in practice).
3. **NULL in → NULL out** for the distance, but a `CASE ... = ... ELSE` around the same pair prints the `ELSE` value, because `x = NULL` is UNKNOWN, not an error.
