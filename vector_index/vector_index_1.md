# SQL Server question — Vector Index 1

## Statement

An online gallery stores "style" embeddings of artworks in a SQL Server 2025 database named `ArtLens`. For this exercise the embeddings are 3-dimensional; the three axes encode *(warm colours, seascape, monochrome)*. The database is built by the following complete script (the vector index and `VECTOR_SEARCH` are preview features on SQL Server 2025, hence the scoped configuration):

```sql
CREATE DATABASE ArtLens;
GO
USE ArtLens;
GO
ALTER DATABASE SCOPED CONFIGURATION SET PREVIEW_FEATURES = ON;
GO
CREATE SCHEMA Gallery;
GO
CREATE TABLE Gallery.Artworks
(
    ArtworkId   INT          NOT NULL CONSTRAINT PK_Artworks PRIMARY KEY CLUSTERED,
    Title       NVARCHAR(80) NOT NULL,
    StyleVector VECTOR(3)    NOT NULL
);
GO
INSERT INTO Gallery.Artworks (ArtworkId, Title, StyleVector) VALUES
    (1, N'Harbour at Dawn',          '[3, 4, 0]'),
    (2, N'Red Square Study',         '[1, 0, 0]'),
    (3, N'Blue Monochrome',          '[0, 0, 2]'),
    (4, N'Harbour at Dawn (poster)', '[6, 8, 0]'),
    (5, N'Inverse Red',              '[-1, 0, 0]');
GO
```

The following eleven statements are then executed **in order, each in its own batch**:

```sql
-- S1
SELECT DATALENGTH(StyleVector)                   AS Bytes,
       VECTORPROPERTY(StyleVector, 'Dimensions') AS Dims,
       VECTORPROPERTY(StyleVector, 'BaseType')   AS BaseType
FROM Gallery.Artworks WHERE ArtworkId = 1;

-- S2
CREATE TABLE Gallery.HiRes (ArtworkId INT NOT NULL PRIMARY KEY, Vec VECTOR(1999) NULL);

-- S3
SELECT CAST(VECTOR_NORMALIZE(StyleVector, 'norm2') AS VARCHAR(80)) AS UnitVector
FROM Gallery.Artworks WHERE ArtworkId = 1;

-- S4
DECLARE @q VECTOR(3) = '[1, 0, 0]';
SELECT ArtworkId,
       CAST(VECTOR_DISTANCE('cosine', StyleVector, @q) AS DECIMAL(10,4)) AS CosDist,
       CAST(VECTOR_DISTANCE('dot', VECTOR_NORMALIZE(StyleVector, 'norm2'),
                                   VECTOR_NORMALIZE(@q, 'norm2')) AS DECIMAL(10,4)) AS DotOfUnit
FROM Gallery.Artworks ORDER BY ArtworkId;

-- S5
CREATE VECTOR INDEX VX_Artworks_Style ON Gallery.Artworks (StyleVector)
    WITH (METRIC = 'cosine', TYPE = 'diskann');

-- S6
INSERT INTO Gallery.Artworks (ArtworkId, Title, StyleVector) VALUES (6, N'Green Field', '[0, 1, 0]');

-- S7
UPDATE Gallery.Artworks SET Title = N'Harbour at Dusk' WHERE ArtworkId = 1;

-- S8
DECLARE @q VECTOR(3) = '[1, 0, 0]';
SELECT a.ArtworkId, CAST(r.distance AS DECIMAL(10,4)) AS Distance
FROM VECTOR_SEARCH(TABLE = Gallery.Artworks AS a, COLUMN = StyleVector,
                   SIMILAR_TO = @q, METRIC = 'euclidean', TOP_N = 3) AS r
ORDER BY r.distance, a.ArtworkId;

-- S9
DECLARE @q VECTOR(3) = '[1, 0, 0]';
SELECT a.ArtworkId, a.Title, CAST(r.distance AS DECIMAL(10,4)) AS Distance
FROM VECTOR_SEARCH(TABLE = Gallery.Artworks AS a, COLUMN = StyleVector,
                   SIMILAR_TO = @q, METRIC = 'cosine', TOP_N = 3) AS r
ORDER BY r.distance, a.ArtworkId;

-- S10
DROP INDEX VX_Artworks_Style ON Gallery.Artworks;

-- S11
INSERT INTO Gallery.Artworks (ArtworkId, Title, StyleVector) VALUES (6, N'Green Field', '[0, 1, 0]');
```

For each statement state whether it **succeeds or raises an error**, and give the exact result of every statement that returns rows.

## Correct Answer

| Stmt | Outcome | Detail |
|---|---|---|
| S1 | **Succeeds** | `Bytes = 20`, `Dims = 3`, `BaseType = float32` |
| S2 | **Fails** | `Msg 2717` — `The size (1999) given to the column 'Vec' exceeds the maximum allowed (1998).` |
| S3 | **Succeeds** | `[6.0000002e-001,8.0000001e-001,0.0000000e+000]` |
| S4 | **Succeeds** | see table below |
| S5 | **Succeeds** | index created (the engine prints `Warning: The join order has been enforced because a local join hint is used.`); `sys.indexes` shows `VX_Artworks_Style` with `type_desc = VECTOR` |
| S6 | **Fails** | `Msg 42231` — `Data modification statement failed because table 'Artworks' has a vector index on it.` |
| S7 | **Fails** | `Msg 42231` — same message, although only `Title` is updated |
| S8 | **Fails** | `Msg 42227` — `Cannot find a vector index with metric 'euclidean' on column 'StyleVector'.` |
| S9 | **Succeeds** | see table below |
| S10 | **Succeeds** | index dropped |
| S11 | **Succeeds** | `(1 rows affected)` — the table is writable again |

S4:

| ArtworkId | CosDist | DotOfUnit |
|---|---|---|
| 1 | 0.4000 | −0.6000 |
| 2 | 0.0000 | −1.0000 |
| 3 | 1.0000 | 0.0000 |
| 4 | 0.4000 | −0.6000 |
| 5 | 2.0000 | 1.0000 |

S9:

| ArtworkId | Title | Distance |
|---|---|---|
| 2 | Red Square Study | 0.0000 |
| 1 | Harbour at Dawn | 0.4000 |
| 4 | Harbour at Dawn (poster) | 0.4000 |

## Explanation

### S1 — storage size and `VECTORPROPERTY`

A `vector(n)` value is stored as **4 bytes per dimension plus an 8-byte header**: 3 × 4 + 8 = **20** bytes. The same rule gives 6,152 bytes for `vector(1536)` (verified with `DATALENGTH`) and 8,000 bytes for the maximum `vector(1998)` — which is exactly why 1,998 is the cap: 1,998 × 4 + 8 = 8,000 bytes, the row-size limit. `VECTORPROPERTY(v, 'Dimensions')` returns the declared dimension count and `VECTORPROPERTY(v, 'BaseType')` returns `float32` (the only base type today); any other property name returns `NULL`.

### S2 — the 1,998-dimension limit

Declaring `VECTOR(1999)` is rejected at parse time with error 2717. A model that emits 3,072 values (`text-embedding-3-large` by default) must be shortened with its `dimensions` parameter before it can be stored.

### S3 — `VECTOR_NORMALIZE`

`VECTOR_NORMALIZE(v, 'norm2')` divides by the Euclidean length: (3, 4, 0) / 5 = (0.6, 0.8, 0). The printed `6.0000002e-001` is the float32 representation of 0.6 — expected, not an error. Other norms are `'norm1'` (÷ Σ|xᵢ|, giving (0.4286, 0.5714, 0)) and `'norminf'` (÷ max|xᵢ|, giving (0.75, 1, 0)); an unknown name such as `'euclidean'` fails with `Msg 42210 The requested norm function 'euclidean' is not supported by vector_norm/vector_normalize.` A zero vector normalizes to itself.

### S4 — cosine distance vs dot product on unit vectors

`VECTOR_DISTANCE('cosine', v, q)` = 1 − cos θ. With q = (1, 0, 0): artwork 1 has cos θ = 3/5 = 0.6 → 0.4; artwork 2 is identical in direction → 0; artwork 3 is orthogonal → 1; artwork 4 = 2 × artwork 1 has the *same* angle → 0.4 (magnitude is divided out); artwork 5 points the opposite way → 2.

`VECTOR_DISTANCE('dot', a, b)` returns the **negative** dot product. On unit vectors the dot product *is* cos θ, so `DotOfUnit = −cos θ = CosDist − 1`: −0.6, −1, 0, −0.6, 1. The two columns differ by a constant, hence they produce the **same ranking**. That is the reason embeddings are often stored pre-normalized: the cheaper dot-product metric (no per-row division) becomes equivalent to cosine. (Raw, un-normalized dot would rank artwork 4 (−6) above artwork 2 (−1): magnitude leaks in. Euclidean on unit vectors is also rank-equivalent to cosine: ‖a − b‖² = 2 − 2 cos θ.)

### S5 — `CREATE VECTOR INDEX` requirements

`CREATE VECTOR INDEX name ON table (vector_column) WITH (METRIC = 'cosine' | 'dot' | 'euclidean' [, TYPE = 'diskann'] [, MAXDOP = n])` builds a DiskANN graph. Requirements verified on this build:

- `PREVIEW_FEATURES = ON`, otherwise `Msg 343 Unknown object type 'VECTOR' used in a CREATE, DROP, or ALTER statement.`
- The table needs a **clustered primary key on a single 4-byte `INT`** column: a table with a `BIGINT` clustered PK fails with `Msg 42217 Table 'Gallery.Sketches' must have a clustered primary key on a single 4 byte INT column to create a vector index.`
- One vector index per column (`Msg 42230 Cannot create vector index on column ... because it already has an existing vector index.`), `TYPE` defaults to DiskANN, and `METRIC` accepts only the three values (`'manhattan'` is a syntax error).
- The session must have `QUOTED_IDENTIFIER ON` (otherwise `Msg 1934`, as for indexed views), and `sys.vector_indexes` exposes `vector_index_type = DiskANN`, `distance_metric = COSINE` and the DiskANN `build_parameters`.

On this RTM build the index built on **5 rows**; the current Azure SQL Database "latest version" index requires at least 100 rows with non-`NULL` vectors (`Msg 42266`).

### S6 and S7 — the table becomes read-only

With the (first-version) vector index in place **every** DML statement is rejected with error 42231 — `INSERT`, `DELETE`, and `UPDATE`, *even an `UPDATE` that does not touch the vector column* (S7 is the subtle one). `TRUNCATE TABLE` fails too (`Msg 42232 TRUNCATE TABLE statement failed because table 'Artworks' has a vector index on it.`). To change data you drop the index, modify, and rebuild — which is why vector indexes suit read-mostly tables loaded in bulk. Azure SQL Database's latest index version lifts this ("Full DML support"), and older Azure indexes could opt into stale results with the `ALLOW_STALE_VECTOR_INDEX` scoped configuration; neither exists on this build (the configuration name is a syntax error here).

### S8 — the metric must match the index

`VECTOR_SEARCH` looks for an index on the named column **with the same metric**. There is only a cosine index, so the euclidean search fails with error 42227 — on this build there is no silent fallback to an exact scan. The metric you index with is the metric you can search with.

### S9 — approximate search over the index

`VECTOR_SEARCH(TABLE = t AS a, COLUMN = c, SIMILAR_TO = @q, METRIC = 'cosine', TOP_N = 3)` returns the table's columns plus `distance`. The three nearest neighbours of (1, 0, 0) by cosine are artworks 2 (0), 1 (0.4), and 4 (0.4); the `ORDER BY r.distance, a.ArtworkId` tiebreaker puts 1 before 4. On five vectors the approximate result equals the exact one (`SELECT TOP (3) ... ORDER BY VECTOR_DISTANCE('cosine', StyleVector, @q)` returns the same rows); at scale ANN may miss a true neighbour — that is the recall trade-off. The `TOP_N` argument is the syntax this build accepts; the newer `SELECT TOP (n) WITH APPROXIMATE ... ORDER BY r.distance` form documented for Azure SQL Database fails here with `Msg 102 Incorrect syntax near 'APPROXIMATE'`.

### S10 and S11 — drop the index, DML returns

`DROP INDEX` removes the vector index like any other index; the very next batch inserts artwork 6 successfully (`COUNT(*)` = 6). Two details verified: after the drop, `VECTOR_SEARCH` with `METRIC = 'cosine'` fails with the same `Msg 42227` (no index, no ANN), while `ORDER BY VECTOR_DISTANCE` keeps working because exact search never uses an index; and if `DROP INDEX` and the `INSERT` are placed in the **same batch**, the `INSERT` still fails with 42231 because the batch was compiled while the index existed — keep them in separate batches.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

## DP-800 Exam Rule to Remember

```text
vector(n)   : float32 elements, n ≤ 1998, storage = 4·n + 8 bytes (1536 → 6152, 1998 → 8000)
VECTORPROPERTY(v,'Dimensions' | 'BaseType')      VECTOR_NORMALIZE(v,'norm2' | 'norm1' | 'norminf')
unit vectors: dot distance = cosine distance − 1  →  same ranking; euclidean also rank-equivalent

CREATE VECTOR INDEX ix ON t (v) WITH (METRIC = 'cosine'|'dot'|'euclidean', TYPE = 'diskann', MAXDOP = n)
   needs PREVIEW_FEATURES = ON, clustered PK on one 4-byte INT, QUOTED_IDENTIFIER ON, one index per column
   first-version index (SQL Server 2025 RTM): table becomes READ-ONLY (42231 / 42232) until DROP INDEX
VECTOR_SEARCH(TABLE = t AS a, COLUMN = v, SIMILAR_TO = @q, METRIC = <same as index>, TOP_N = k)
   metric ≠ index metric → 42227;   VECTOR_DISTANCE = exact, never uses the index
```

Choose the metric once: cosine for un-normalized text embeddings, dot for pre-normalized vectors (cheapest), euclidean when magnitudes carry meaning — and index with the metric you will query with.
