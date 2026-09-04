# SQL Server question — Fabric SQL Database 1

## Statement

TidePool Marinas wants its berth-booking system to live in a **SQL database in Microsoft Fabric** so that the same data is immediately available for analytics in OneLake. You will build it yourself and then predict how the Fabric flavour of the SQL Database Engine reacts to a script written for Azure SQL Database.

**Prerequisites.** A Fabric capacity (a 60-day Fabric trial is enough; a trial is limited to three SQL databases) and the **Admin** (or Member) role in the workspace you create. SQL database in Fabric is enabled by default; if the *SQL database* tile is missing, ask the Fabric admin to check the tenant settings. Everything below uses your own **Microsoft Entra** account — the database has no SQL logins.

### Part 1 — Create the workspace and the database (portal, CLI, or REST)

1. In the Fabric portal, select **Workspaces** > **New workspace**. Name it `ws-dp800-tidepool`, expand **Advanced**, choose the workspace type that points at your capacity (**Trial** or **Fabric capacity**) and select **Apply**.
2. In the workspace, select **New item**, choose the **SQL database** tile, enter `TidePoolOps` and select **Create**. (Equivalent: the **Databases** workload home page > **New** > **SQL database**.)
3. Alternative — Fabric CLI (`pip install ms-fabric-cli`):

   ```text
   fab auth login
   fab create ws-dp800-tidepool.Workspace/TidePoolOps.SQLDatabase
   fab ls ws-dp800-tidepool.Workspace
   ```

4. Alternative — REST API (bearer token for `https://api.fabric.microsoft.com`):

   ```text
   POST https://api.fabric.microsoft.com/v1/workspaces/{workspaceId}/items
   { "displayName": "TidePoolOps", "type": "SQLDatabase", "description": "DP-800 lab" }

   GET  https://api.fabric.microsoft.com/v1/workspaces/{workspaceId}/SqlDatabases
   GET  https://api.fabric.microsoft.com/v1/workspaces/{workspaceId}/SqlDatabases/{databaseId}
        -> properties.ServerFqdn, properties.DatabaseName
   ```

5. Go back to the workspace item list and note **which items now exist** with the name `TidePoolOps`.
6. On the database item, select **...** > **Settings** > **Connection strings**. Note the `Data Source=` value (shape `tcp:<id>.database.fabric.microsoft.com,1433`) and `Initial Catalog=`. On the same **Settings** page, open **SQL endpoint** and note that the SQL analytics endpoint has a *different* server name (`<id>.<tenant>.fabric.microsoft.com`). The **Open in** button on the query editor ribbon fills SSMS 21+ or the MSSQL extension for VS Code for you.
7. Connect with sqlcmd (Entra authentication, `-G`) if you prefer a terminal:

   ```text
   sqlcmd -S <id>.database.fabric.microsoft.com,1433 -G -d TidePoolOps -i setup.sql
   ```

### Part 2 — Setup script (run in the database's query editor, **New query**)

```sql
CREATE SCHEMA Marina;
GO
CREATE TABLE Marina.Berth
(
    BerthID   int          NOT NULL PRIMARY KEY,
    Pontoon   char(1)      NOT NULL,
    LengthM   decimal(5,2) NOT NULL
);
CREATE TABLE Marina.Booking
(
    BookingID  int  NOT NULL PRIMARY KEY,
    BerthID    int  NOT NULL REFERENCES Marina.Berth (BerthID),
    Nights     int  NOT NULL,
    BookedAt   datetime2(6) NOT NULL DEFAULT SYSUTCDATETIME()
);
CREATE TABLE Marina.TideReading
(
    ReadingID  bigint       NOT NULL PRIMARY KEY NONCLUSTERED,
    ReadAt     datetime2(6) NOT NULL,
    HeightM    decimal(4,2) NOT NULL
);
INSERT INTO Marina.Berth VALUES (1, 'A', 12.50), (2, 'A', 15.00), (3, 'B', 9.75);
INSERT INTO Marina.Booking (BookingID, BerthID, Nights) VALUES (100, 1, 3), (101, 3, 1);
INSERT INTO Marina.TideReading VALUES (1, '2026-09-01T06:00:00', 4.10), (2, '2026-09-01T12:00:00', 0.85);
GO
```

### Part 3 — Statements to predict

Run the following statements **one at a time, each as its own batch**, in the `TidePoolOps` query editor (the **SQL database** side of the drop-down, not the SQL analytics endpoint). Replace `<colleague>@<tenant>` with a real user of your tenant (you need the Entra *Directory Readers* role for `FROM EXTERNAL PROVIDER`).

```sql
-- S1
CREATE LOGIN dockmaster WITH PASSWORD = 'Tide!Pool#2026Strong';

-- S2
CREATE USER [<colleague>@<tenant>] FROM EXTERNAL PROVIDER;

-- S3
CREATE TABLE Marina.BerthRate
(
    BerthID     int          NOT NULL PRIMARY KEY,
    NightlyRate decimal(8,2) NOT NULL,
    ValidFrom   datetime2(6) GENERATED ALWAYS AS ROW START NOT NULL,
    ValidTo     datetime2(6) GENERATED ALWAYS AS ROW END   NOT NULL,
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
)
WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = Marina.BerthRateHistory));

-- S4
CREATE TABLE Marina.FeeLedger
(
    FeeID  int          NOT NULL PRIMARY KEY,
    Amount decimal(8,2) NOT NULL
)
WITH (LEDGER = ON (APPEND_ONLY = ON));

-- S5
CREATE TABLE Marina.GateEvent
(
    EventID int NOT NULL PRIMARY KEY NONCLUSTERED HASH WITH (BUCKET_COUNT = 1024),
    Gate    char(2) NOT NULL
)
WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_AND_DATA);

-- S6
CREATE CLUSTERED COLUMNSTORE INDEX CCI_TideReading ON Marina.TideReading;

-- S7
CREATE TABLE Marina.SensorLog
(
    LogID    bigint       NOT NULL,
    SensorID int          NOT NULL,
    Value    decimal(9,3) NOT NULL,
    INDEX CCI_SensorLog CLUSTERED COLUMNSTORE
);

-- S8
EXEC sys.sp_cdc_enable_db;

-- S9
EXECUTE AS USER = '<colleague>@<tenant>';
SELECT USER_NAME();
REVERT;

-- S10
CREATE TABLE Marina.WaveReport
(
    ReportID      int NOT NULL PRIMARY KEY,
    [Wave Height] decimal(4,2) NOT NULL
);
```

### Part 4 — Observe the analytics side and source control

1. Switch the query editor drop-down to **SQL analytics endpoint** (or open the `TidePoolOps` SQL analytics endpoint item), expand **Explorer**, and list the tables you see. In the database item, open **Replication** > **Monitor replication** and note any table with status **NotSupported**.
2. Connect the workspace to a Git repository (**Workspace settings** > **Git integration**; the Git-integration tenant setting must be on), then select **Source control** > tick `TidePoolOps` > **Commit**. Note what folder and file types appear in the repository.

### Question

1. For S1–S10, state whether each statement **succeeds or fails** and why.
2. Which of the tables `Berth`, `Booking`, `TideReading`, `BerthRate`, `BerthRateHistory`, `SensorLog` are visible through the **SQL analytics endpoint** after replication catches up?
3. What is committed to Git for the database, and which two database-level settings are **not** included?

**Capacity and cleanup.** The SQL database autopauses when idle, but the workspace still counts against capacity: when finished, open **Workspace settings** > **General** > **Manage** > **Remove this workspace** > **Delete** (content is purged after a 7-day retention), and pause a paid F capacity in the Azure portal.

## Correct Answer

| Stmt | Outcome | Why (documented behaviour) |
|------|---------|----------------------------|
| S1 | **Fails** | Logins (server principals) aren't supported; Microsoft Entra ID is the only identity provider, "SQL authentication isn't supported" |
| S2 | **Succeeds** | Users representing Microsoft Entra principals are the only supported users; `CREATE USER ... FROM EXTERNAL PROVIDER` is the documented syntax |
| S3 | **Succeeds** | Temporal tables: Yes. (`BerthRateHistory` is created, but it is excluded from mirroring) |
| S4 | **Fails** | Ledger: No — "in-memory, ledger, ledger history, and Always Encrypted tables cannot be created" |
| S5 | **Fails** | In-memory tables cannot be created |
| S6 | **Fails** | "When mirroring is active, Clustered columnstore indexes cannot be created on an existing table" (mirroring is always on and cannot be disabled; you could only stop it through the REST API first) |
| S7 | **Succeeds** | A CCI "created at the same time the table is created using the inline index syntax" is allowed — "however, the new table cannot be mirrored" |
| S8 | **Fails** | Change data capture: No |
| S9 | **Fails** | `EXECUTE AS`: No in Fabric SQL database (Azure SQL Database allows `EXECUTE AS USER`) |
| S10 | **Fails** | Column names cannot contain spaces nor `, ; { } ( ) \n \t =` |

Tables visible through the SQL analytics endpoint: `Marina.Berth`, `Marina.Booking`, `Marina.TideReading`, `Marina.BerthRate`. **Not** visible: `Marina.BerthRateHistory` (temporal history tables are excluded) and `Marina.SensorLog` (CCI table; **Monitor replication** shows it as **NotSupported**). `FeeLedger`, `GateEvent` and `WaveReport` were never created.

Git: the commit produces a folder `TidePoolOps.SQLDatabase` containing a **SQL database project** (`.sqlproj` plus one `.sql` file per object, for example `Marina/Tables/Berth.sql`) and the Fabric item metadata. Database **collation** and **compatibility level** are not part of source control or deployment pipelines.

## Explanation

### What was created alongside the database

Creating a SQL database in Fabric provisions **two items**: the SQL database and its **SQL analytics endpoint**; data "is automatically replicated into OneLake and converted to Parquet". Mirroring "starts upon creation of your SQL database in Fabric with no user action required", there are no settings to configure, it "is always on and can't be turned off", and "all supported tables are mirrored with no option to skip tables". No default Power BI semantic model is created any more (default semantic models stopped being auto-created on September 5, 2025); build one with a new semantic model on the SQL analytics endpoint if you need Direct Lake.

Two servers, two connection strings: the database itself answers on `<id>.database.fabric.microsoft.com,1433` (an Azure-SQL-style name) and the read-only analytics copy on `<id>.<tenant>.fabric.microsoft.com` (a warehouse-style name). Both are found under **Settings** > **Connection strings** / **SQL endpoint** and both accept **Microsoft Entra** authentication only (`sqlcmd -G`, SSMS "Microsoft Entra ID - Universal with MFA support"). The connection policy is fixed to **Default** (redirect on ports 11000–11999 plus the gateway on 1433).

### S1 and S2 — identity is Entra-only

The limitations table says it in one line: "Logins are not supported. Only users representing Microsoft Entra principals are supported." `CREATE LOGIN` therefore fails (S1), while `CREATE USER [name] FROM EXTERNAL PROVIDER` (S2) is exactly how the authentication article tells you to create users for an Entra user, service principal or group (when connected as a service principal you use the `WITH SID = ..., TYPE = E|X` form instead). You often do not need `CREATE USER` at all: workspace roles and item permissions, or the portal's database-role management, create the users automatically.

### S3, S4, S5 — which "specialized tables" exist in Fabric

The feature table marks **Temporal tables: Yes**, **Ledger: No**, **Always Encrypted: No**, and the table-level limitations add that "in-memory, ledger, ledger history, and Always Encrypted tables cannot be created". So the system-versioned `BerthRate` (S3) is fine, the ledger table (S4) and the memory-optimized table (S5) are rejected. The subtle part of S3 is the mirroring rule: "for temporal tables, the data table is mirrored, but the history table is excluded from mirroring". Choosing `datetime2(6)` for the period columns is deliberate: Delta supports six fractional digits, so `datetime2(7)` columns lose their seventh digit and a `datetime2(7)` primary key makes a table unmirrorable.

### S6 and S7 — clustered columnstore and the always-on mirror

Because replication runs from creation, "when mirroring is active, clustered columnstore indexes cannot be created on an existing table" — S6 fails even though `TideReading` has a nonclustered PK that would allow it in Azure SQL Database. The only path is to stop mirroring with the *sqldatabase* REST API (`stopMirroring`), add the CCI, and start it again — and "the table will not be mirrored". S7 uses the inline `INDEX ... CLUSTERED COLUMNSTORE` syntax, which is explicitly allowed at creation time, again at the cost of the table never being replicated. Nonclustered columnstore indexes have no such restriction.

### S8, S9, S10 — engine features that Azure SQL Database has and Fabric does not

- **CDC (S8)**: Azure SQL Database supports it from S3 upwards; Fabric SQL database: **No**. Change tracking, temporal tables and the built-in mirroring cover the "what changed" scenarios instead.
- **`EXECUTE AS` (S9)**: Azure SQL Database allows `EXECUTE AS USER`; the Fabric column says **No**, so impersonation tests must be done by signing in as the principal.
- **Column names (S10)**: Fabric adds a naming rule that Azure SQL Database does not have — no spaces and none of `, ; { } ( ) \n \t =` — because the same column must be representable in the Delta/Parquet copy. Database names have their own forbidden list (`! [ ] < > * % & : / ? # = @ ^ " ' ; ( )`), a database name cannot be re-used after deletion, and a PK cannot be **hierarchyid**, **sql_variant** or **timestamp**.

Other differences worth memorising from the same table: no `BACKUP` command (automatic ZRS backups, 7-day retention, point-in-time restore only), no server-level roles, no application roles, no elastic query/elastic jobs/failover groups/geo-restore/LTR, `OPENROWSET BULK` only from OneLake, full-text search in preview, `ALTER DATABASE SET` options in preview, and resource limits of 32 vCores, 4 TB, 150 databases per workspace.

### The SQL analytics endpoint after replication

Mirrored data lands as Delta Parquet in OneLake and "the SQL analytics endpoint points to those files"; it "works just like the Lakehouse SQL analytics endpoint. It is the same read-only experience". Column-level exclusions apply silently: computed columns and **image, text/ntext, xml, rowversion/timestamp, sql_variant, UDT, geometry, geography, hierarchyid** columns are skipped, LOBs over 1 MB are truncated, and tables with **json** or **vector** columns are not mirrored at all. Views and stored procedures are not mirrored — create them again on the endpoint. Row-level security, object permissions and dynamic data masking of the source are **not** propagated to the replicated data. Cross-database three-part-name queries (joining a lakehouse or warehouse of the same workspace) are possible "via the SQL analytics endpoint", not from the transactional database connection.

### Source control

Fabric git integration converts the live database into a **SQL database project** (`.sqlproj`, Microsoft.Build.Sql) under `<database>.SQLDatabase`; **Update** from the repository "combines a SQL project build and SqlPackage publish operation" with `/p:IncludeTransactionalScripts=true` and `/p:GenerateSmartDefaults=true` among others. Edits to the `.sqlproj` are overwritten on the next commit from Fabric, and "database-level settings such as collation and compatibility level aren't included". Static data is handled with a query moved to **Shared Queries** and marked **Set as Post-deployment Script**.

Documentation relied upon (ms.date): Limitations for SQL database 2026-08-24; Overview 2026-05-19; Create a SQL database 2025-09-05; Connect 2026-03-03; Authentication 2024-11-20; Mirroring overview 2025-07-02; Limitations of mirroring for SQL database 2026-02-24; Source control integration 2026-02-27; Create with Fabric CLI 2025-02-19; Create with REST API 2026-02-19; Power BI semantic models 2025-12-05; Clean up tutorial resources 2025-04-06.

Hands-on question (Microsoft Fabric capacity required); behaviour is taken from the official documentation as of the ms.date cited above.

## DP-800 Exam Rule to Remember

```text
SQL database in Fabric = Azure SQL Database engine + always-on mirroring + Fabric identity
  Created with it: SQL analytics endpoint (read-only, Delta in OneLake). No default semantic model.
  Two servers:  <id>.database.fabric.microsoft.com,1433 (OLTP)   <id>.<tenant>.fabric.microsoft.com (analytics)
  Auth: Microsoft Entra only  -> CREATE USER ... FROM EXTERNAL PROVIDER; CREATE LOGIN / SQL auth: never
  NOT available: ledger, Always Encrypted, in-memory, CDC, EXECUTE AS, server roles, app roles,
                 BACKUP, elastic query/jobs, geo-replication/LTR, trace flags, SQL Agent (use pipelines)
  Available:     temporal (history not mirrored), RLS/DDM/column encryption, NCCI, Query Store, AI functions
  Mirroring rules: no CCI on an existing table while mirroring; inline CCI at CREATE = table not mirrored;
                   json/vector tables, computed cols, xml/image/text/rowversion/sql_variant/geo/hierarchyid
                   skipped; datetime2(7) trimmed; no spaces in column names; 1000 tables max
  Source control: Fabric git = SQL database project (.sqlproj); collation & compat level not included;
                  Update = build + SqlPackage publish; post-deployment script via Shared Queries
```
