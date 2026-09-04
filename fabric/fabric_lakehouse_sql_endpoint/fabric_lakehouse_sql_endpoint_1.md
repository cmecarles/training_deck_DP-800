# SQL Server question — Lakehouse SQL Analytics Endpoint 1

## Statement

Glacier Peak Resort logs every chair-lift ride as CSV. You will land the file in a **Fabric lakehouse**, turn it into a Delta table, query it through the lakehouse's **SQL analytics endpoint**, join it to a **Fabric Data Warehouse** in the same workspace, and predict which T-SQL statements the read-only endpoint accepts.

**Prerequisites.** A Fabric capacity (trial is fine) and the Admin role in the workspace you create. Your identity is a Microsoft Entra account; SQL logins do not exist on any of these items.

### Part 1 — Lakehouse and Delta table

1. **Workspaces** > **New workspace** > `ws-dp800-glacier` > **Advanced** > your capacity > **Apply**.
2. **New item** > **Lakehouse** > name `GlacierLake` > **Create**. If the dialog offers lakehouse schemas, leave it off so that tables land in `dbo`.
3. Save this text locally as `lift_rides.csv`:

   ```text
   ride_id,lift_id,rider_pass,ride_ts,duration_s,peak_wind
   1,L1,P100,2026-02-01T09:02:00,412,12.5
   2,L1,P101,2026-02-01T09:03:30,408,12.5
   3,L2,P100,2026-02-01T09:40:00,655,18.0
   4,L3,P102,2026-02-01T10:15:00,301,9.2
   5,L2,P103,2026-02-01T10:20:00,660,18.4
   ```

4. In the **Lakehouse explorer**, select **Files** > **...** > **New subfolder** `raw`, open it, **...** > **Upload** > **Upload files** > choose `lift_rides.csv`.
5. Right-click (or **...**) on `lift_rides.csv` > **Load to Tables** > **New table**. In the dialog **Load file to new table** enter **New table name** `lift_rides` (only letters, digits and `_` are accepted — try `lift-rides` first and read the validation message), keep **Column header** checked and **Separator** `,`, then **Load**. Wait for the table to appear under **Tables**, open it and note the Delta column types the loader inferred.
6. Also try **Load to Tables** on the same file into **Existing table** `lift_rides` with **Load mode** **Append** — it doubles the rows to 10.

### Part 2 — Warehouse

7. **New item** > **Warehouse** > `GlacierDW` > **Create**. In its query editor run:

   ```sql
   CREATE TABLE dbo.lift_capacity (lift_id varchar(10) NOT NULL, seats smallint NOT NULL, lift_name varchar(40) NOT NULL);
   INSERT INTO dbo.lift_capacity VALUES ('L1', 6, 'Summit Six'), ('L2', 4, 'Ridge Quad'), ('L3', 2, 'Bunny Double');
   ```

### Part 3 — Query the SQL analytics endpoint

8. Open `GlacierLake`, and in the top-right of the ribbon switch the view from **Lakehouse** to **SQL analytics endpoint** (the endpoint is also listed as its own item, with the lakehouse's name, in the workspace). If `lift_rides` is not in **Explorer** yet, select the **Refresh** icon in the Explorer toolbar and wait; metadata sync normally lags less than a minute.
9. Select **+ Warehouses** in the Explorer, tick `GlacierDW`, and confirm it appears in the tree.
10. Select **New SQL query** and run each of the following as its own batch:

```sql
-- S1  (schema the endpoint generated)
SELECT c.name, TYPE_NAME(c.user_type_id) AS sql_type, c.max_length, c.collation_name
FROM sys.columns AS c WHERE c.object_id = OBJECT_ID('dbo.lift_rides') ORDER BY c.column_id;

-- S2
INSERT INTO dbo.lift_rides (ride_id, lift_id, rider_pass, ride_ts, duration_s, peak_wind)
VALUES (99, 'L1', 'P999', '2026-02-01T11:00:00', 400, 10.0);

-- S3
CREATE TABLE dbo.lift_notes (lift_id varchar(10), note varchar(200));

-- S4
CREATE VIEW dbo.v_rides_per_lift AS
SELECT lift_id, COUNT(*) AS rides, AVG(CAST(duration_s AS float)) AS avg_seconds
FROM dbo.lift_rides GROUP BY lift_id;

-- S5
CREATE PROCEDURE dbo.usp_busiest_lift AS
SELECT TOP (1) lift_id, rides FROM dbo.v_rides_per_lift ORDER BY rides DESC;

-- S6
SELECT v.lift_id, c.lift_name, v.rides, c.seats
FROM dbo.v_rides_per_lift AS v
JOIN GlacierDW.dbo.lift_capacity AS c ON c.lift_id = v.lift_id;

-- S7  (read the raw file, not the table)
SELECT TOP (3) * FROM OPENROWSET(BULK '/Files/raw/lift_rides.csv', FORMAT = 'CSV', HEADER_ROW = TRUE) AS f;

-- S8
SELECT lift_id, rides FROM dbo.v_rides_per_lift FOR XML PATH('lift');

-- S9
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
```

11. Switch to the `GlacierDW` query editor, add `GlacierLake` with **+ Warehouses**, and run:

```sql
-- S10  (run in the warehouse)
CREATE TABLE dbo.rides_by_lift (lift_id varchar(10) NOT NULL, rides int NOT NULL);
INSERT INTO dbo.rides_by_lift
SELECT lift_id, COUNT(*) FROM GlacierLake.dbo.lift_rides GROUP BY lift_id;
SELECT * FROM dbo.rides_by_lift ORDER BY lift_id;
```

12. Back in the lakehouse, append the CSV once more (step 6) and immediately run `SELECT COUNT(*) FROM dbo.lift_rides` on the endpoint several times. Note how the count catches up and where the manual **Refresh** lives.

### Question

1. For S1, state the **SQL data type and collation** that the endpoint assigns to a Delta `string` column such as `lift_id`, whatever the loader inferred for the numeric columns.
2. For S2–S10, state whether each statement **succeeds or fails** and why. For S10 give the result rows (after steps 5–6, before step 12).
3. Name the three documented ways to force a metadata refresh when the count in step 12 is stale, and state which of them is available only when the workspace has the **New metadata sync (preview)** turned on.
4. If you build a Direct Lake semantic model on this endpoint and include `dbo.v_rides_per_lift`, what happens under **Direct Lake on SQL** and under **Direct Lake on OneLake**?

**Capacity and cleanup.** The lakehouse, its endpoint and the warehouse all consume capacity when queried. When done: **Workspace settings** > **General** > **Manage** > **Remove this workspace** > **Delete**, and pause a paid capacity in the Azure portal.

## Correct Answer

**S1.** A Delta `string` column becomes **`varchar(8000)`** with collation **`Latin1_General_100_BIN2_UTF8`** in the SQL analytics endpoint of a *lakehouse* (`varchar(max)` is only used by the endpoints of mirrored items and Fabric databases). Delta `integer` maps to `int`, `long` to `bigint`, `double` to `float`, `timestamp` to `datetime2` (6 digits), `boolean` to `bit`, `decimal(p,s)` to `decimal(p,s)`.

| Stmt | Outcome | Why |
|------|---------|-----|
| S2 | **Fails** | The endpoint is read-only: "you can't insert, update, or delete data through it" |
| S3 | **Fails** | "Creating, altering, and dropping tables ... are only supported in Warehouse ..., not in the SQL analytics endpoint" |
| S4 | **Succeeds** | Views, functions and stored procedures can be created on the endpoint |
| S5 | **Succeeds** | Same rule; it persists in the endpoint, not in the Delta lake |
| S6 | **Succeeds** | Cross-database three-part-name query to a warehouse in the same workspace (added with **+ Warehouses**) |
| S7 | **Succeeds** | `OPENROWSET(BULK '/Files/...')` — the relative `/Files` path "works only when querying a Lakehouse through its SQL analytics endpoint"; OneLake reads via OPENROWSET are in preview |
| S8 | **Fails** | `SELECT ... FOR XML` is in the unsupported list |
| S9 | **Fails** | `SET TRANSACTION ISOLATION LEVEL` is in the unsupported list |
| S10 | **Succeeds** | Warehouse tables accept `CREATE TABLE`/`INSERT`; the `INSERT ... SELECT` reads the lakehouse through its three-part name. Result: `L1, 4` / `L2, 4` / `L3, 2` |

**Refresh options.** (1) The **Refresh** icon in the Explorer toolbar of the endpoint editor; (2) the **Refresh SQL endpoint metadata** REST API; (3) `EXEC sys.sp_dw_refresh_ext_table 'dbo.lift_rides';` — option (3) works only for endpoints created after **Workspace settings** > **Warehouse settings** > **New metadata sync (preview)** was enabled.

**Direct Lake.** Under *Direct Lake on SQL* a table based on a (non-materialized) SQL view **falls back to DirectQuery** (or fails if fallback is disabled); under *Direct Lake on OneLake* a table based on a SQL view is **not supported** — only Delta tables (or a lakehouse materialized view) can be used. No default semantic model is created any more; you create one yourself and Direct Lake is its storage mode.

## Explanation

### What the endpoint is

"The SQL analytics endpoint gives you a read-only T-SQL query surface over the Delta tables in your lakehouse. Every lakehouse automatically provisions a SQL analytics endpoint when created." It "runs on the same engine as the Fabric Data Warehouse", so the T-SQL surface, data types and limitations of the warehouse apply — minus writes to the autogenerated tables. Only Delta tables under `/Tables` are discovered; files under `/Files` are not tables (S7 reads them as files instead). Loading with **Load to Tables** keeps "output in Delta format with V-order optimization"; names accept "alphanumeric characters and underscores (`_`) only, up to 256 characters. Dashes (`-`) and spaces aren't allowed", and unchecked **Column header** yields `_c0`, `_c1`, ...

### S1 — types are derived, not chosen

"The column types in the SQL analytics endpoint tables are derived from the source Delta types", following the mapping table: `STRING` → `varchar(8000)` for a lakehouse endpoint (and `varchar(max)` for mirrored items), with the `Latin1_General_100_BIN2_UTF8` collation; `VARCHAR(n<2000)` → `varchar(4n)`; `TIMESTAMP` → `datetime2` limited to six fractional digits; `BINARY` → `varbinary(n)`. Types with no supported equivalent (arrays, maps, structs, `uniqueidentifier` written by a warehouse) are simply not surfaced as columns. Data in `varchar` columns of a lakehouse endpoint is still truncated at 8 KB.

### S2, S3 versus S4, S5 — what "read-only" means

Read-only refers to the **data and the autogenerated tables**: "You can extend the model with your own SQL schemas, views, stored procedures, and functions", apply `GRANT`/`DENY`, row-level and column-level security, and dynamic data masking. Inserting, updating, deleting, creating, altering or dropping tables belongs to the warehouse (or to Spark on the lakehouse side). Scalar UDFs are supported when inlineable. Because the endpoint shares the warehouse engine, the warehouse's unsupported list also applies: `FOR XML` (S8), `SET TRANSACTION ISOLATION LEVEL` (S9), `SET ROWCOUNT`, recursive queries, triggers, synonyms, materialized views, `CREATE USER` (users are auto-created by `GRANT`), the `vector` type and vector functions, `PREDICT`, `BULK LOAD`.

### S6, S10 — cross-item queries in one workspace

"You can write cross database queries to warehouses and databases in the current active workspace": add the item with **+ Warehouses**, then use `Item.schema.table`. Data flows only into a warehouse (`INSERT INTO GlacierDW... SELECT FROM GlacierLake...`), never back into the lakehouse from T-SQL. S10's `INSERT ... SELECT` is the documented pattern. Cross-region connections are not supported, and a workspace holds at most 150 warehouse + SQL analytics endpoint items.

### S7 — OPENROWSET

`OPENROWSET(BULK ...)` reads Parquet, CSV and JSONL from Azure Blob/ADLS or **Fabric OneLake** (OneLake reads are in preview). Three location forms exist: an absolute URL (`https://onelake.dfs.fabric.microsoft.com/<workspaceId>/<lakehouseId>/Files/...`), a relative path plus `DATA_SOURCE`, and a relative path starting with `/Files`, which "works only when querying a Lakehouse through its SQL analytics endpoint". `HEADER_ROW = TRUE` turns the first line into column names; `sp_describe_first_result_set` and the `WITH (...)` clause let you pin the inferred types.

### Metadata sync and Delta features

"Under normal operating conditions, the lag between a lakehouse and SQL analytics endpoint is less than one minute"; the background process runs only while the endpoint is active and halts after 15 minutes of inactivity. It reads the Delta logs from `/Tables`, and handles table discovery, data freshness and schema-change detection. Manual refresh: the **Refresh** icon in the Explorer toolbar, the *Refresh SQL endpoint metadata* REST API (for schema changes on the whole item), and `sys.sp_dw_refresh_ext_table` for one table — the last "only available if the SQL analytics endpoint was created after enabling the New metadata sync (preview)", a workspace-level warehouse setting introduced in May 2026 that also exposes `sys.dm_db_external_tables_log_status`. Delta features: column mapping by **name** yes, by **ID** no; deletion vectors yes; liquid clustering yes; `TIMESTAMP_NTZ` no; V2 checkpoints not listed correctly; a foreign-key constraint added on the endpoint blocks further schema changes on that table.

### Direct Lake

Direct Lake "is the storage mode for new Power BI semantic models created on a Warehouse or SQL analytics endpoint"; since September 5, 2025 default semantic models are no longer created automatically. **Direct Lake on SQL** uses the endpoint for discovery and permission checks and "fall[s] back to DirectQuery table storage mode when they can't load the data directly from a Delta table, such as when the data source is a SQL view or when the warehouse uses SQL-based granular access control". **Direct Lake on OneLake** never falls back; "it isn't supported to create a Direct Lake on OneLake table based on a non-materialized SQL view". Complex Delta column types, binary and GUID columns are unsupported in both.

Documentation relied upon (ms.date): What is the SQL analytics endpoint for a lakehouse 2026-05-19; Load to Delta Lake tables 2026-03-01; T-SQL surface area 2026-08-26; Data types in Fabric Data Warehouse 2026-06-11; Query the Warehouse or SQL analytics endpoint 2026-01-06; Query external files 2026-02-11; Browse file content with OPENROWSET 2026-06-23; SQL analytics endpoint metadata sync 2026-05-29; SQL analytics endpoint performance considerations 2026-08-07; Delta Lake table format interoperability 2026-05-08; Direct Lake overview 2026-06-15; Power BI semantic models 2025-12-05; Limitations of Fabric Data Warehouse 2026-07-29.

Hands-on question (Microsoft Fabric capacity required); behaviour is taken from the official documentation as of the ms.date cited above.

## DP-800 Exam Rule to Remember

```text
Lakehouse SQL analytics endpoint = warehouse engine, READ-ONLY over Delta tables in /Tables
  OK:   SELECT, CREATE VIEW / FUNCTION (inlineable scalar) / PROCEDURE, GRANT/DENY, RLS, CLS, DDM,
        three-part names to other warehouses/endpoints in the SAME workspace (+ Warehouses),
        OPENROWSET(BULK '/Files/...') on the lakehouse's own files (OneLake read = preview)
  NO:   INSERT/UPDATE/DELETE/MERGE, CREATE/ALTER/DROP TABLE (warehouse or Spark only), FOR XML,
        SET TRANSACTION ISOLATION LEVEL, SET ROWCOUNT, recursive CTE, triggers, synonyms, vector, CREATE USER
  Types: string -> varchar(8000) Latin1_General_100_BIN2_UTF8 (varchar(max) only for mirrored items),
         timestamp -> datetime2(6), double -> float, integer -> int, boolean -> bit; no arrays/maps/structs
  Sync: < 1 min typical; Refresh icon | Refresh SQL endpoint metadata API | sp_dw_refresh_ext_table (new sync only)
  Direct Lake: no default semantic model since 2025-09-05; on SQL -> falls back to DirectQuery for views/RLS;
               on OneLake -> no fallback, views unsupported
```
