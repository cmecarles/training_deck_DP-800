# SQL Server question — JSON_VALUE 1

## Statement

A home-insurance carrier keeps flexible policy metadata as JSON documents beside the relational core of each policy.

The following script is run on a SQL Server 2025 instance, in this exact order, with no errors:

```sql
CREATE DATABASE PolicyVault;
GO
ALTER DATABASE PolicyVault SET COMPATIBILITY_LEVEL = 170;
GO
USE PolicyVault;
GO
CREATE SCHEMA Ins;
GO
CREATE TABLE Ins.Policies
(
    PolicyID INT           NOT NULL PRIMARY KEY,
    Meta     NVARCHAR(MAX) NOT NULL CHECK (ISJSON(Meta) = 1)
);
GO
INSERT INTO Ins.Policies (PolicyID, Meta) VALUES
 (1, N'{"policy no":"HP-2044-X","holder":{"name":"Marta Voss"},"riders":["flood","quake"],"limits":{"dwelling":450000},"active":true,"prior_claims":[{"yr":2021,"paid":12500.75},{"yr":2024,"paid":null}]}'),
 (2, CONVERT(NVARCHAR(MAX), N'{"policy no":"HP-2189-B","holder":{"name":"Ade Kaplan"},"riders":[],"limits":{"dwelling":300000,"deductible":2500},"active":false,"notes":"')
     + REPLICATE(CONVERT(NVARCHAR(MAX), N'x'), 4100) + N'"}');
GO
```

Note what the second `INSERT` builds: policy 2's `notes` property is a JSON string of **4,100** `x` characters, and the document is valid JSON (the `ISJSON` check passed). Policy 1 has no `notes` and no `limits.deductible`; policy 2 has an **empty** `riders` array and no `prior_claims`.

Two queries are then executed. **Predict the exact output of each — every value and every `NULL` — or the exact failure.**

### Query 1

```sql
SELECT
    p.PolicyID,
    JSON_VALUE(p.Meta, '$."policy no"')          AS PolicyNo,
    JSON_VALUE(p.Meta, '$.riders[1]')            AS SecondRider,
    JSON_VALUE(p.Meta, '$.riders')               AS RidersV,
    JSON_QUERY(p.Meta, '$.riders')               AS RidersQ,
    JSON_QUERY(p.Meta, '$.holder.name')          AS HolderQ,
    JSON_VALUE(p.Meta, '$.active')               AS ActiveV,
    JSON_VALUE(p.Meta, '$.limits.deductible')    AS Deductible,
    JSON_VALUE(p.Meta, '$.prior_claims[1].paid') AS Paid2,
    JSON_VALUE(p.Meta, '$.notes')                AS Notes
FROM Ins.Policies AS p
ORDER BY p.PolicyID;
```

### Query 2

```sql
SELECT
    p.PolicyID,
    JSON_VALUE(p.Meta, 'strict $.limits.deductible') AS Deductible
FROM Ins.Policies AS p
ORDER BY p.PolicyID;
```

## Correct Answer

Both outputs were produced by SQL Server 2025 (RTM, 17.0.1000.7).

### Query 1 — 2 rows

```text
PolicyID  PolicyNo   SecondRider  RidersV  RidersQ            HolderQ  ActiveV  Deductible  Paid2  Notes
--------  ---------  -----------  -------  -----------------  -------  -------  ----------  -----  -----
1         HP-2044-X  quake        NULL     ["flood","quake"]  NULL     true     NULL        NULL   NULL
2         HP-2189-B  NULL         NULL     []                 NULL     false    2500        NULL   NULL
```

### Query 2 — fails at run time, no rows

```text
Msg 13608, Level 16, State 2
Property cannot be found on the specified JSON path.
```

(The line number in the error depends on the batch; message number, severity, and text are fixed.)

## Explanation

`JSON_VALUE(expression, path)` extracts a **scalar** and returns it as **nvarchar(4000)**; `JSON_QUERY` extracts an **object or array** fragment. Each function returns `NULL` (lax mode, the default) or raises an error (`strict` prefix) when asked for the kind of thing it does not extract, when the path does not resolve — or, for `JSON_VALUE`, when the scalar is longer than 4,000 characters.

### Query 1, column by column

- **`PolicyNo` — `$."policy no"`** — a key containing a space (or any special character) must be double-quoted inside the path. It resolves normally: `HP-2044-X`, `HP-2189-B`.
- **`SecondRider` — `$.riders[1]`** — array indexes are **0-based**, so `[1]` is the *second* element: `quake` for policy 1 (not `flood`). Policy 2's `riders` is `[]`, index 1 does not exist, lax mode → `NULL`.
- **`RidersV` — `JSON_VALUE` on `$.riders`** — the path resolves to an **array**, and `JSON_VALUE` only returns scalars: lax mode → `NULL` for both rows, even for the non-empty array.
- **`RidersQ` — `JSON_QUERY` on `$.riders`** — the mirror image: `JSON_QUERY` returns the fragment, `["flood","quake"]` and `[]`. An empty array is the two-character text `[]`, not `NULL` — the property exists.
- **`HolderQ` — `JSON_QUERY` on `$.holder.name`** — the reverse trap: the path resolves to the **string** `"Marta Voss"` / `"Ade Kaplan"`, a scalar, and `JSON_QUERY` does not return scalars: lax mode → `NULL` for both rows. `JSON_VALUE` would have returned the names.
- **`ActiveV` — `$.active`** — JSON booleans are scalars, returned as the **text** `true` / `false` (nvarchar, not bit).
- **`Deductible` — `$.limits.deductible`** — exists only for policy 2 (`2500`); missing property, lax mode → `NULL` for policy 1.
- **`Paid2` — `$.prior_claims[1].paid`** — policy 1's second claim has `"paid": null`: the property **exists** and holds JSON `null`, which surfaces as SQL `NULL`. Policy 2 has no `prior_claims` at all — also `NULL`, for a different reason (path not found, lax). The output cannot distinguish the two cases; that is precisely why the column is a trap.
- **`Notes` — `$.notes`** — the deep trap. Policy 2's `notes` **exists and is a valid scalar string**, but it is 4,100 characters long. `JSON_VALUE` returns **nvarchar(4000)**; when the value exceeds 4,000 characters, lax mode returns `NULL` instead of truncating. Policy 1 simply lacks the key → `NULL`. So the column is `NULL` on both rows — one from the length limit, one from a missing path. (Engine-verified: `LEN(JSON_VALUE(Meta, '$.notes'))` is `NULL` for policy 2, and `JSON_VALUE(Meta, 'strict $.notes')` raises `Msg 13625, Level 16 — "String value in the specified JSON path would be truncated."`. To read the full value, use `OPENJSON ... WITH (Notes NVARCHAR(MAX) '$.notes')`, or `JSON_VALUE(... RETURNING nvarchar(max))` on a `json`-typed input.)

### Query 2 — strict mode turns NULL into an error

`strict $.limits.deductible` resolves for policy 2 (`2500`) but **not** for policy 1. In strict mode a missing property is not smoothed over with `NULL`; it aborts the statement at run time:

```text
Msg 13608, Level 16, State 2
Property cannot be found on the specified JSON path.
```

No result set is returned at all — you do **not** get policy 2's row with `2500`. One failing row fails the whole query, which is why `strict` belongs in data-quality checks and not in general-purpose reports over heterogeneous documents.

### Equivalent alternatives

- The results are identical if `Meta` is declared with the native **json** type instead of `NVARCHAR(MAX) + ISJSON` — the JSON functions behave the same over both storage choices. The `json` type additionally unlocks `JSON_VALUE(... RETURNING <data_type>)` (SQL Server 2025), e.g. `RETURNING int` to get `2500` as an **int** instead of text; on this build, `RETURNING` over a plain `nvarchar` input is rejected.
- `RidersQ`/`RidersV` and `HolderQ` behave exactly like `OPENJSON ... WITH` columns with and without `AS JSON` — the same scalar-vs-fragment split under a different syntax.

## DP-800 Exam Rule to Remember

Split the world in two and never cross the line:

```text
JSON_VALUE → scalars only   (string, number, true/false)
JSON_QUERY → fragments only (objects, arrays)
```

Asked for the wrong kind, each returns `NULL` in lax mode (the default) and an **error** in strict mode. Then stack the `JSON_VALUE` specifics:

- Return type is **nvarchar(4000)**: a scalar over 4,000 chars → lax `NULL` / strict `Msg 13625` (use `OPENJSON` or `RETURNING nvarchar(max)` on the `json` type for longer values).
- Booleans and numbers come back as **text** (`true`, `2500`) unless `RETURNING` (2025, `json`-type input) casts them.
- Array indexes are **0-based**; quote unusual keys: `$."policy no"`.
- Lax mode makes missing paths, wrong-kind paths, JSON `null`, and oversized scalars all collapse into the same `NULL` — and `strict` on a mixed document set fails the **entire statement**, not just the offending row.
