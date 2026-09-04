# SQL Server question — OPENJSON 1

## Statement

A meteorological network ingests raw weather-station observation feeds. Each station's latest observation arrives as one JSON document and is stored as text.

The following script is run on a SQL Server 2025 instance, in this exact order, with no errors:

```sql
CREATE DATABASE SkyWatch;
GO
ALTER DATABASE SkyWatch SET COMPATIBILITY_LEVEL = 170;
GO
USE SkyWatch;
GO
CREATE SCHEMA Wx;
GO
CREATE TABLE Wx.Stations
(
    StationID   INT IDENTITY(1,1) PRIMARY KEY,
    StationCode CHAR(4)       NOT NULL UNIQUE,
    Obs         NVARCHAR(MAX) NOT NULL CHECK (ISJSON(Obs) = 1)
);
GO
INSERT INTO Wx.Stations (StationCode, Obs) VALUES
 ('KSEA', N'{"temp_c":12.5,"wind":{"dir":"NW","kts":18},"gusts":[22,27,31],"qc_pass":true,"obs_time":"06:00Z","ceiling_ft":null}'),
 ('EGLL', N'{"temp_c":-3,"wind":{"dir":"N","kts":5},"gusts":[],"qc_pass":false,"obs_time":"06:10Z","ceiling_ft":1200}');
GO
```

Three queries are then executed. **Predict the exact result of each one** — every column value of every row, including `NULL`s, or the exact failure if a query cannot complete.

### Query A — default schema

```sql
SELECT j.[key], j.[value], j.[type]
FROM Wx.Stations AS s
CROSS APPLY OPENJSON(s.Obs) AS j
WHERE s.StationCode = 'KSEA'
ORDER BY j.[key];
```

### Query B — explicit schema

```sql
SELECT
    s.StationCode,
    j.TempC,
    j.WindKts,
    j.Gusts,
    j.GustsScalar,
    j.CeilingFt,
    j.QcPass
FROM Wx.Stations AS s
CROSS APPLY OPENJSON(s.Obs)
WITH (
    TempC       DECIMAL(4,1)  '$.temp_c',
    WindKts     INT           'strict $.wind.kts',
    Gusts       NVARCHAR(MAX) '$.gusts' AS JSON,
    GustsScalar NVARCHAR(40)  '$.gusts',
    CeilingFt   INT           '$.ceiling_ft',
    QcPass      BIT
) AS j
ORDER BY s.StationCode;
```

### Query C — strict mapping

```sql
SELECT s.StationCode, j.RunwayVis
FROM Wx.Stations AS s
CROSS APPLY OPENJSON(s.Obs)
WITH (
    RunwayVis INT 'strict $.runway_vis'
) AS j
WHERE s.StationCode = 'KSEA';
```

## Correct Answer

All outputs below were produced by SQL Server 2025 (RTM, 17.0.1000.7).

### Query A — 6 rows

```text
key         value                  type
----------  ---------------------  ----
ceiling_ft  NULL                   0
gusts       [22,27,31]             4
obs_time    06:00Z                 1
qc_pass     true                   3
temp_c      12.5                   2
wind        {"dir":"NW","kts":18}  5
```

### Query B — 2 rows

```text
StationCode  TempC  WindKts  Gusts       GustsScalar  CeilingFt  QcPass
-----------  -----  -------  ----------  -----------  ---------  ------
EGLL         -3.0   5        []          NULL         1200       NULL
KSEA         12.5   18       [22,27,31]  NULL         NULL       NULL
```

### Query C — fails at run time, no rows

```text
Msg 13608, Level 16, State 6
Property cannot be found on the specified JSON path.
```

(The line number in the error depends on the batch; message number, severity, and text are fixed.)

## Explanation

### Query A — the default three-column schema

Without a `WITH` clause, `OPENJSON` returns one row per **first-level** key with exactly three columns:

- `key` — **nvarchar(4000)**, BIN2 collation;
- `value` — **nvarchar(max)**, the value rendered as text;
- `type` — **tinyint** type code (Microsoft Learn documents the column as **int**, but the engine's actual column metadata, verified on SQL Server 2025 RTM, is **tinyint**).

The type codes are the first trap; they must be produced from the JSON type of each value:

| type | JSON type | row here |
|---|---|---|
| 0 | null | `ceiling_ft` |
| 1 | string | `obs_time` |
| 2 | number | `temp_c` |
| 3 | true/false | `qc_pass` |
| 4 | array | `gusts` |
| 5 | object | `wind` |

Three details decide the exact grid:

1. **`ceiling_ft` yields SQL `NULL` with type `0`** — a JSON `null` becomes a relational `NULL` in `value`, not the four-character string `null`, while the type column still records that the *key exists* and holds JSON `null`. (Contrast: `qc_pass` yields the literal text `true`, because booleans have no relational representation in an `nvarchar` column.)
2. **Nested values are not expanded.** Only first-level properties become rows; `wind` comes back as the raw fragment `{"dir":"NW","kts":18}` (type 5) and `gusts` as `[22,27,31]` (type 4). There is no row for `dir`, `kts`, or the individual gusts.
3. **Row order follows `ORDER BY j.[key]`**, not document order — `ceiling_ft` sorts first even though it is the last key in the document. Without the `ORDER BY`, rows come back in document order, and relying on that is unsafe.

### Query B — `WITH` clause, column by column

`CROSS APPLY` runs `OPENJSON` once per station row and, because the document is a single object, each application returns exactly one row — so the join yields two rows, sorted `EGLL` before `KSEA` by `StationCode`.

- **`TempC DECIMAL(4,1) '$.temp_c'`** — plain lax mapping with a cast: `-3.0` and `12.5`.
- **`WindKts INT 'strict $.wind.kts'`** — a `strict` mapping is *not* an automatic error: both documents contain `wind.kts`, so it resolves to `5` and `18`. Strict only changes what happens when the path fails (see Query C).
- **`Gusts ... '$.gusts' AS JSON`** — `AS JSON` (type must be `NVARCHAR(MAX)`) returns the inner fragment like `JSON_QUERY` would: `[]` for EGLL and `[22,27,31]` for KSEA. Note that an empty array is returned as the two characters `[]`, not as `NULL` — the property exists.
- **`GustsScalar NVARCHAR(40) '$.gusts'`** — the *same path* **without** `AS JSON` behaves like `JSON_VALUE`: the path points at an array, an array is not a scalar, and in lax mode the column is silently `NULL` for **both** rows. This is the classic "my column is always NULL" bug when `AS JSON` is forgotten.
- **`CeilingFt INT '$.ceiling_ft'`** — EGLL has the number `1200`; KSEA has JSON `null`, which surfaces as SQL `NULL`. Same path, two different reasons to read carefully.
- **`QcPass BIT`** — the second trap. With no explicit path, `OPENJSON` matches the **column name** against the JSON keys with a case-sensitive, collation-unaware (BIN2) comparison. `QcPass` ≠ `qc_pass`, the match fails, and in lax mode the column is `NULL` for both rows — even though both documents carry a perfectly good `qc_pass` boolean. Writing the column as `qc_pass BIT` or adding the path `'$.qc_pass'` would have returned `1` and `0`. (Engine-verified: this exact query returns `NULL`, `NULL`.)

### Query C — strict mapping on a missing property

Neither document contains `runway_vis`. In lax mode the column would simply be `NULL`; the `strict` prefix instead raises a **run-time error** and the statement returns no result set:

```text
Msg 13608, Level 16, State 6
Property cannot be found on the specified JSON path.
```

The `WHERE s.StationCode = 'KSEA'` filter does not save the query — the error occurs while evaluating the applied rowset, and one failing row is enough to abort the whole statement.

### Equivalent alternatives

- `OPENJSON` requires `COMPATIBILITY_LEVEL >= 130` (or the `ALLOW_BUILTIN_TVF_IN_ALL_COMPAT_LEVELS` scoped configuration); 170 comfortably satisfies it. The queries behave identically if `Obs` is declared with the native `json` type instead of `NVARCHAR(MAX)` with an `ISJSON` check.
- `GustsScalar`'s always-NULL behavior can equally be predicted from `JSON_VALUE(Obs, '$.gusts')`, and `Gusts` from `JSON_QUERY(Obs, '$.gusts')` — the `WITH` clause without/with `AS JSON` mirrors that function pair exactly.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

## DP-800 Exam Rule to Remember

Default-schema `OPENJSON` returns `key`, `value`, `type` for **first-level** properties only, and the type codes are:

```text
0 null   1 string   2 number   3 true/false   4 array   5 object
```

JSON `null` → SQL `NULL` + type 0; `true`/`false` → the literal text, type 3; nested objects/arrays → the raw fragment, types 5/4.

In a `WITH` clause:

```text
no AS JSON  → JSON_VALUE semantics (scalar; object/array → NULL lax / error strict)
AS JSON     → JSON_QUERY semantics (fragment; requires NVARCHAR(MAX))
no path     → column name matched to key, CASE-SENSITIVE (BIN2)
lax (default) path miss → NULL        strict path miss → Msg 13608 aborts the query
```

Use `CROSS APPLY OPENJSON(t.col)` to shred a JSON column per table row; one bad row under a `strict` mapping fails the entire statement.
