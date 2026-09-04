# SQL Server question — REGEXP_SPLIT_TO_TABLE 1

## Statement

`LensFolio` is the catalog database of a photography portfolio site running on SQL Server 2025. For years, photographers typed keyword tags into a single free-text column, and the delimiters are a mess: some used commas, some used semicolons, some left stray spaces around the delimiter, some left the field empty-handed in other ways (double delimiters, trailing delimiters), and one photo has no tags at all.

The data team wants to shred the tag strings into one row per keyword using the new regex table-valued function `REGEXP_SPLIT_TO_TABLE`, and to compare it against the older `STRING_SPLIT`.

The following session is executed exactly as shown, from database creation onward:

```sql
CREATE DATABASE LensFolio;
GO
ALTER DATABASE LensFolio SET COMPATIBILITY_LEVEL = 170;
GO
USE LensFolio;
GO
CREATE SCHEMA Photo;
GO
CREATE TABLE Photo.Portfolio
(
    PhotoID  INT           NOT NULL PRIMARY KEY,
    Title    NVARCHAR(60)  NOT NULL,
    Tags     NVARCHAR(200) NULL
);
GO
INSERT INTO Photo.Portfolio (PhotoID, Title, Tags) VALUES
  (1, N'Harbor Dusk',  N'longexposure,harbor,,dusk'),
  (2, N'Alpine Ridge', N'alpine; ridge ;sunrise;'),
  (3, N'Street Solo',  N'monochrome'),
  (4, N'Untagged',     NULL);
GO
```

Then two queries are run.

**Query 1:**

```sql
SELECT p.PhotoID,
       s.ordinal,
       s.value AS Keyword
FROM Photo.Portfolio AS p
CROSS APPLY REGEXP_SPLIT_TO_TABLE(p.Tags, N'\s*[,;]\s*') AS s
ORDER BY p.PhotoID, s.ordinal;
```

**Query 2:**

```sql
SELECT value, ordinal
FROM STRING_SPLIT(N'alpine; ridge ;sunrise;', N';', 1)
ORDER BY ordinal;
```

Predict the **exact** result set of each query: every row, every column value, every ordinal, in order. Be explicit about:

1. how many rows each query returns;
2. which cells are the empty string `''` and which are `NULL`;
3. whether any `PhotoID` from `Photo.Portfolio` is missing from Query 1, and why;
4. whether the keyword spelled `ridge` comes back identically from both queries.

## Correct Answer

**Query 1 returns exactly 9 rows** (output captured from a SQL Server 2025 RTM instance):

| PhotoID | ordinal | Keyword        |
|---------|---------|----------------|
| 1       | 1       | `longexposure` |
| 1       | 2       | `harbor`       |
| 1       | 3       | `` (empty string) |
| 1       | 4       | `dusk`         |
| 2       | 1       | `alpine`       |
| 2       | 2       | `ridge`        |
| 2       | 3       | `sunrise`      |
| 2       | 4       | `` (empty string) |
| 3       | 1       | `monochrome`   |

- The two empty cells are **empty strings, not NULLs** (`LEN(Keyword) = 0`).
- **PhotoID 4 is missing entirely.** `Tags` is `NULL`, `REGEXP_SPLIT_TO_TABLE(NULL, ...)` returns an **empty table** (zero rows, not one NULL row), and `CROSS APPLY` against an empty table eliminates the outer row.
- Photo 2's keywords come back **without** the surrounding spaces (`ridge`, not `· ridge ·`), because `\s*` on both sides of `[,;]` makes the whitespace part of the delimiter match that gets consumed.

**Query 2 returns exactly 4 rows:**

| value      | ordinal |
|------------|---------|
| `alpine`   | 1       |
| `· ridge ·` (i.e. `' ridge '`, 7 characters, spaces intact) | 2 |
| `sunrise`  | 3       |
| `` (empty string) | 4 |

`STRING_SPLIT` splits on the literal single character `;` only, so the spaces around `ridge` survive into the value (`DATALENGTH` = 14 bytes = 7 nvarchar characters). So no — `ridge` is **not** identical across the two queries: Query 1 yields `N'ridge'`, Query 2 yields `N' ridge '`.

## Explanation

### The output shape of REGEXP_SPLIT_TO_TABLE

`REGEXP_SPLIT_TO_TABLE (string_expression, pattern_expression [, flags])` is one of the regex table-valued functions introduced in SQL Server 2025 (17.x). Per Microsoft Learn and confirmed on the engine, it returns a **two-column table**, always — there is no opt-in flag:

| Column name | Data type | Meaning |
|-------------|-----------|---------|
| `value`     | same type as `string_expression` (here **nvarchar**) | the substring between delimiter matches; the whole input if the pattern never matches |
| `ordinal`   | **bigint** | 1-based position of the substring in the input |

The column names are lowercase `value` and `ordinal`. Selecting `s.position` or expecting a single-column table are both wrong.

### Walking Query 1 photo by photo

The pattern `\s*[,;]\s*` means: one comma **or** one semicolon, with any surrounding run of whitespace absorbed into the delimiter.

- **Photo 1 — `longexposure,harbor,,dusk`.** Three delimiter matches produce four substrings. Between the adjacent commas there is nothing, so ordinal 3 is the **empty string**. Adjacent delimiters are *not* collapsed into one — each `[,;]` match is a separate split point (the two commas cannot be consumed by a single match of this pattern, since the character class matches exactly one character). Result: `longexposure` (1), `harbor` (2), `''` (3), `dusk` (4).
- **Photo 2 — `alpine; ridge ;sunrise;`.** The first delimiter match is `; ` (semicolon plus the following space), the second is ` ;` (space plus semicolon), so the stored values are trimmed to `alpine`, `ridge`, `sunrise`. The trailing semicolon is the third delimiter match, sitting at the very end of the string, and what follows it is the empty string: a trailing delimiter always yields a trailing empty element, here at ordinal 4. (Symmetrically, a leading delimiter would yield a leading empty element at ordinal 1.)
- **Photo 3 — `monochrome`.** The pattern never matches. Per the documentation, "if there's no match to the pattern, the function returns the string": one row, the whole input, ordinal 1. The function never returns zero rows for a non-NULL input.
- **Photo 4 — `NULL`.** A NULL input produces an **empty table**. Combined with `CROSS APPLY` (which behaves like an inner join between each outer row and its table-valued result), photo 4 vanishes from the output. To keep it as a single row with `ordinal = NULL, Keyword = NULL`, use `OUTER APPLY` instead — verified on the engine.

Total: 4 + 4 + 1 + 0 = **9 rows**, deterministically ordered by the explicit `ORDER BY p.PhotoID, s.ordinal`.

### Query 2 and the STRING_SPLIT contrast

`STRING_SPLIT (string, separator [, enable_ordinal])` differs from `REGEXP_SPLIT_TO_TABLE` on every axis this question probes:

1. **The separator is a literal single character** — `nvarchar(1)`. It cannot be a character class, cannot absorb whitespace, and cannot handle "comma or semicolon" in one call. That is why ordinal 2 comes back as `' ridge '` with its spaces.
2. **The `ordinal` column is opt-in.** Without the third argument (`1`), `STRING_SPLIT` returns a *single* column `value`; passing `enable_ordinal = 1` (SQL Server 2022 and later; it must be a constant) adds the **bigint** `ordinal` column. `REGEXP_SPLIT_TO_TABLE` always returns both columns.
3. **Order is not guaranteed** from `STRING_SPLIT` without `ORDER BY` — the docs say output rows "might be in any order". The explicit `ORDER BY ordinal` here makes the result deterministic.
4. Behaviors that **agree**: both return empty-string rows for consecutive delimiters and for a trailing delimiter (hence the 4th row, `''`), and both return an empty table for a NULL input.
5. **Compatibility level:** `STRING_SPLIT` needs at least 130, `REGEXP_SPLIT_TO_TABLE` needs **170** (the SQL Server 2025 default). At a lower level the engine simply cannot find the regex function ("invalid object name"), unless the database-scoped configuration `ALLOW_BUILTIN_TVF_IN_ALL_COMPAT_LEVELS` is enabled — a preview option (shipped in SQL Server 2025 CU5 and Azure SQL, not present on the RTM build) whose documented scope covers both functions. The `ALTER DATABASE ... SET COMPATIBILITY_LEVEL = 170` line in the script is what guarantees Query 1 compiles.

The optional third argument of `REGEXP_SPLIT_TO_TABLE` is `flags` (`i` case-insensitive, `m` multi-line, `s` dot-matches-newline, `c` case-sensitive, default `c`) — not an ordinal switch.

### Equivalent alternatives (engine-verified)

- `CROSS APPLY REGEXP_SPLIT_TO_TABLE(p.Tags, N'[,;]') AS s` with `TRIM(s.value) AS Keyword` returns the identical 9 rows for this data set (the delimiter no longer eats the spaces, so `TRIM` removes them from the values instead). It is only equivalent because no keyword here has *interior* edge whitespace that must survive.
- Replacing `CROSS APPLY` with `OUTER APPLY` is **not** equivalent: it returns 10 rows, adding `(4, NULL, NULL)`.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

## DP-800 Exam Rule to Remember

`REGEXP_SPLIT_TO_TABLE` (SQL Server 2025, Azure SQL) is a table-valued function that always returns exactly two columns:

```text
value   (same type as the input string)
ordinal (bigint, 1-based)
```

Memorize the splitting edge cases as a quartet:

- adjacent delimiters  → an empty-string element between them
- trailing/leading delimiter → an empty-string element at the end/start
- no pattern match → one row containing the whole string
- NULL input → zero rows (so `CROSS APPLY` drops the outer row; `OUTER APPLY` keeps it)

And the contrast table against `STRING_SPLIT`:

| | STRING_SPLIT | REGEXP_SPLIT_TO_TABLE |
|---|---|---|
| Delimiter | one literal character | full regular expression |
| `ordinal` column | opt-in (`enable_ordinal = 1`) | always present |
| Third argument | `enable_ordinal` | regex `flags` (`i`, `m`, `s`, `c`) |
| Minimum compat level | 130 | **170** |

Both compatibility gates can be bypassed with the `ALLOW_BUILTIN_TVF_IN_ALL_COMPAT_LEVELS` database-scoped configuration (in preview; available from SQL Server 2025 CU5 and in Azure SQL, not on the RTM build).
