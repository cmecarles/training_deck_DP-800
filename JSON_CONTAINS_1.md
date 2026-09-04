# SQL Server question — JSON_CONTAINS 1

## Statement

A tech job board stores each candidate's profile as a native `json` document and matches candidates against vacancies with the `JSON_CONTAINS` function, introduced in SQL Server 2025 (17.x).

The following script is run on a SQL Server 2025 instance, in this exact order, with no errors:

```sql
CREATE DATABASE TalentPool;
GO
ALTER DATABASE TalentPool SET COMPATIBILITY_LEVEL = 170;
GO
USE TalentPool;
GO
CREATE SCHEMA Jobs;
GO
CREATE TABLE Jobs.Candidates
(
    CandidateID INT IDENTITY(1,1) PRIMARY KEY,
    FullName    NVARCHAR(80) NOT NULL,
    Profile     JSON         NOT NULL
);
GO
INSERT INTO Jobs.Candidates (FullName, Profile) VALUES
 (N'Ada Lindqvist', N'{"skills":["sql","python","azure"],"years":7,"remote":true,"certs":[{"id":"DP-800","active":true}]}'),
 (N'Bruno Ferrer',  N'{"skills":["java","sql"],"years":12,"remote":false}'),
 (N'Chen Wei',      N'{"skills":[],"years":3,"remote":true,"certs":[]}'),
 (N'Dara Okonkwo',  N'{"skills":["python","spark"],"years":"7","remote":true,"certs":[{"id":"DP-800","active":false}]}');
GO
```

Read the four profiles carefully before continuing:

- Bruno has **no `certs` key** at all.
- Chen's `skills` and `certs` are **empty arrays**.
- Dara's `years` is the **JSON string** `"7"`, not the number `7`.

Then a recruiter runs this matching query:

```sql
SELECT
    c.CandidateID,
    JSON_CONTAINS(c.Profile, 'sql', '$.skills[*]')       AS HasSql,
    JSON_CONTAINS(c.Profile, 'sql', '$.skills')          AS HasSqlNoWild,
    JSON_CONTAINS(c.Profile, 7,   '$.years')             AS Yrs7Int,
    JSON_CONTAINS(c.Profile, '7', '$.years')             AS Yrs7Txt,
    JSON_CONTAINS(c.Profile, CAST(1 AS bit), '$.remote') AS RemoteBit,
    JSON_CONTAINS(c.Profile, 1, '$.remote')              AS RemoteInt,
    JSON_CONTAINS(c.Profile, 'DP-%', '$.certs[*].id', 1) AS CertLike
FROM Jobs.Candidates AS c
ORDER BY c.CandidateID;
```

Which result set does the query return?

### a.

| CandidateID | HasSql | HasSqlNoWild | Yrs7Int | Yrs7Txt | RemoteBit | RemoteInt | CertLike |
|---|---|---|---|---|---|---|---|
| 1 | 1 | 0 | 1 | 1 | 1 | 1 | 1 |
| 2 | 1 | 0 | 0 | 0 | 0 | 0 | NULL |
| 3 | 0 | 0 | 0 | 0 | 1 | 1 | NULL |
| 4 | 0 | 0 | 1 | 1 | 1 | 1 | 1 |

### b.

| CandidateID | HasSql | HasSqlNoWild | Yrs7Int | Yrs7Txt | RemoteBit | RemoteInt | CertLike |
|---|---|---|---|---|---|---|---|
| 1 | 1 | 0 | 1 | 0 | 1 | 0 | 1 |
| 2 | 1 | 0 | 0 | 0 | 0 | 0 | NULL |
| 3 | 0 | 0 | 0 | 0 | 1 | 0 | NULL |
| 4 | 0 | 0 | 0 | 1 | 1 | 0 | 1 |

### c.

| CandidateID | HasSql | HasSqlNoWild | Yrs7Int | Yrs7Txt | RemoteBit | RemoteInt | CertLike |
|---|---|---|---|---|---|---|---|
| 1 | 1 | 0 | 1 | 0 | 1 | 0 | 1 |
| 2 | 1 | 0 | 0 | 0 | 0 | 0 | 0 |
| 3 | 0 | 0 | 0 | 0 | 1 | 0 | 0 |
| 4 | 0 | 0 | 0 | 1 | 1 | 0 | 1 |

### d.

| CandidateID | HasSql | HasSqlNoWild | Yrs7Int | Yrs7Txt | RemoteBit | RemoteInt | CertLike |
|---|---|---|---|---|---|---|---|
| 1 | 1 | 1 | 1 | 0 | 1 | 0 | 1 |
| 2 | 1 | 1 | 0 | 0 | 0 | 0 | NULL |
| 3 | 0 | 0 | 0 | 0 | 1 | 0 | NULL |
| 4 | 0 | 0 | 0 | 1 | 1 | 0 | 1 |

## Correct Answer

**b**

Verified on SQL Server 2025 (RTM, 17.0.1000.7). The engine returns exactly:

```text
CandidateID HasSql HasSqlNoWild Yrs7Int Yrs7Txt RemoteBit RemoteInt CertLike
----------- ------ ------------ ------- ------- --------- --------- --------
1           1      0            1       0       1         0         1
2           1      0            0       0       0         0         NULL
3           0      0            0       0       1         0         NULL
4           0      0            0       1       1         0         1
```

## Explanation

`JSON_CONTAINS( target , search_value [ , path ] [ , search_mode ] )` returns an **int**: `1` if the search value is contained at the path, `0` if it is not, and `NULL` when the path is **not found** in the document (or the target is `NULL`).

The question turns on four independent rules.

### Rule 1 — the SQL type of the search value drives the comparison

Unlike a `JSON_VALUE(...) = '7'` predicate (where everything is compared as a character string), `JSON_CONTAINS` compares the **SQL type** of the search value against the **JSON type** at the path. A scalar matches only if the two are *comparable and equal*:

- `Yrs7Int` — search value is the SQL **int** `7`. It equals Ada's JSON *number* `7` (→ `1`), but it is **not comparable** to Dara's JSON *string* `"7"` (→ `0`).
- `Yrs7Txt` — search value is the character string `'7'`. It equals Dara's JSON *string* `"7"` (→ `1`), but it does **not** match Ada's JSON *number* `7` (→ `0`).
- `RemoteBit` vs `RemoteInt` — JSON `true`/`false` compares only against the SQL **bit** type. `CAST(1 AS bit)` matches `true` (rows 1, 3, 4 → `1`); the SQL **int** `1` never matches `true` (→ `0` on every row). This was confirmed on the live engine: `JSON_CONTAINS(@j, CAST(1 AS bit), '$.certs[*].active')` returns `1` while `JSON_CONTAINS(@j, 1, '$.certs[*].active')` returns `0` for the same document.

### Rule 2 — a path that lands on an array needs the `[*]` wildcard

`HasSql` uses `'$.skills[*]'` and finds `"sql"` inside Ada's and Bruno's arrays (→ `1`). `HasSqlNoWild` uses `'$.skills'` — the path points at the array *itself*, automatic unwrapping does not search its elements for a scalar comparison, and the column is `0` for **every** row, including rows where `"sql"` is plainly present. (The Microsoft Learn limitations section states that a wildcard is required when the path points to an array.)

### Rule 3 — path not found returns NULL, not 0

`CertLike` uses the path `'$.certs[*].id'`:

- Bruno has **no `certs` key** → path not found → `NULL`.
- Chen has `"certs":[]` → the wildcard step finds **no element**, so no `id` exists under the path → `NULL`.
- `0` is reserved for "the path exists, but the value is not contained" — which is exactly what makes option c wrong.

### Rule 4 — the fourth argument switches equality to LIKE semantics

`CertLike` passes `search_mode = 1`, so the character-string search value `'DP-%'` is evaluated with **LIKE** predicate semantics: `'DP-800'` LIKE `'DP-%'` → `1` for Ada and Dara. With the default `search_mode = 0` (equality), `'DP-%'` would match nothing and those cells would be `0`.

### Why option a is wrong

Option a assumes value-based coercion: it scores `Yrs7Int` and `Yrs7Txt` as `1` for *both* Ada and Dara (treating the number `7` and the string `"7"` as equal) and scores `RemoteInt` as `1` wherever `remote` is `true` (treating the int `1` as equal to `true`). `JSON_CONTAINS` does neither: comparison is by SQL type, `int` never equals a JSON string, and only **bit** compares with JSON `true`/`false`.

### Why option c is wrong (the subtle one)

Option c is identical to b except that Bruno's and Chen's `CertLike` cells are `0`. That confuses "not contained" with "path not found". `JSON_CONTAINS` returns `0` only when the path resolves and the value is absent; when the path itself does not resolve — a missing `certs` key, or an empty array that gives the `[*]` step nothing to stand on — the function returns `NULL`. The distinction matters in practice: a `WHERE JSON_CONTAINS(...) = 0` filter would silently drop Bruno and Chen.

### Why option d is wrong

Option d scores `HasSqlNoWild` as `1` for Ada and Bruno, assuming the engine automatically searches array elements when the path stops at the array. It does not: without `[*]`, the scalar search value is compared against the array as a whole and the result is `0`.

### Engine-verified remarks (SQL Server 2025 RTM)

- The **target** argument must be of the native `json` type on the RTM build: passing an `nvarchar` literal/column raises `Msg 8116` ("Argument data type nvarchar is invalid for argument 1 of json_contains function"), even though the Learn page says a character string target is accepted. Store the document in a `json` column (as here) or `CAST(... AS json)`.
- A `json`-typed **search value** is rejected with `Msg 8116` as well, and a string that merely *looks* like an array (`'["sql","azure"]'`) is compared as one literal string — array-containment via a string search value returns `0`.
- The function itself is version-gated, not compatibility-gated: it also runs at `COMPATIBILITY_LEVEL = 160` on this build. Level 170 is still the sensible setting for a 2025 JSON workload.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

## DP-800 Exam Rule to Remember

`JSON_CONTAINS` is a **typed** containment test with a three-valued result:

```text
1    → path resolves AND value contained
0    → path resolves AND value not contained
NULL → path not found (missing key, empty array under [*]) or NULL target
```

Memorize the type-pairing table:

```text
SQL int / decimal   ↔  JSON number
SQL (n)varchar      ↔  JSON string
SQL bit             ↔  JSON true / false
```

Cross-pairings (`int` vs `"7"`, `int 1` vs `true`) are simply *not comparable* → `0`. A path that ends on an array needs `[*]` to search its elements, and the optional fourth argument (`1`) switches the string comparison from `=` to `LIKE`.
