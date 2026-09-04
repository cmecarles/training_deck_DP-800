# SQL Server question — Mirroring Azure SQL Database into Fabric 1

## Statement

The Beacon Point Lighthouse Museum sells tickets from an **Azure SQL Database** named `BeaconTickets` (General Purpose, 2 vCores, on the logical server `beacon-sql-<suffix>`). Analysts want the data in Fabric without building pipelines, so you will **mirror** the database into the workspace `ws-dp800-beacon` and validate a runbook written by a junior engineer.

**Prerequisites.** An Azure subscription with rights on the logical server; a Fabric capacity (trial is fine; it must be *running* — a paused capacity stops replication); the **Member** or **Admin** role in the workspace (Contributors lack the *Reshare* permission the wizard needs). Two Fabric tenant settings must be on: **Service principals can call Fabric public APIs** and **Users can access data stored in OneLake with apps external to Fabric**.

### Part 1 — Source database

Create the database (Azure portal > **Create a resource** > **SQL Database**, vCore General Purpose; do **not** pick the DTU *Free*/*Basic* tiers), then run in `BeaconTickets`:

```sql
CREATE SCHEMA Sales;
GO
CREATE TABLE Sales.Tickets   (TicketID int NOT NULL PRIMARY KEY, VisitorID int NOT NULL, Price decimal(7,2) NOT NULL,
                              SoldAt datetime2(6) NOT NULL, Notes xml NULL, Photo image NULL, RowVer rowversion NOT NULL);
CREATE TABLE Sales.Visitors  (VisitorID int NOT NULL PRIMARY KEY, FullName nvarchar(80) NOT NULL, Country char(2) NOT NULL);
CREATE TABLE Sales.VisitLog  (VisitedAt datetime2(6) NOT NULL, Gate char(1) NOT NULL);         -- heap, no key
CREATE TABLE Sales.Turnstile (TurnstileID int NOT NULL, Clicks int NOT NULL, INDEX cci CLUSTERED COLUMNSTORE);
CREATE TABLE Sales.TicketAudit (AuditID int NOT NULL PRIMARY KEY, TicketID int NOT NULL, Action varchar(10) NOT NULL)
    WITH (LEDGER = ON (APPEND_ONLY = ON));
CREATE TABLE Sales.PriceHistory
(
    PriceID int NOT NULL PRIMARY KEY, Price decimal(7,2) NOT NULL,
    ValidFrom datetime2(6) GENERATED ALWAYS AS ROW START NOT NULL,
    ValidTo   datetime2(6) GENERATED ALWAYS AS ROW END NOT NULL,
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
) WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = Sales.PriceHistoryArchive));
INSERT INTO Sales.Visitors VALUES (1, N'Mara Quinn', 'IE'), (2, N'Tobias Lund', 'NO');
INSERT INTO Sales.Tickets (TicketID, VisitorID, Price, SoldAt) VALUES (1, 1, 12.00, '2026-09-01T10:00:00'), (2, 2, 12.00, '2026-09-01T10:05:00');
INSERT INTO Sales.VisitLog VALUES ('2026-09-01T10:10:00', 'A'), ('2026-09-01T10:12:00', 'B');
INSERT INTO Sales.PriceHistory (PriceID, Price) VALUES (1, 12.00);
```

### Part 2 — The junior engineer's runbook (contains exactly one wrong step)

1. Azure portal > the logical server > **Security** > **Identity** > **System assigned managed identity** > **Status: On** > **Save**. Verify with `SELECT * FROM sys.dm_server_managed_identities;` (one row, `is_primary = 1`).
2. Logical server > **Security** > **Networking** > **Public access** > **Allow Azure services and resources to access this server**: **Yes** (or configure a virtual network / on-premises data gateway if public access is off).
3. In `master`: `CREATE LOGIN fabric_login WITH PASSWORD = '<strong password>';` In `BeaconTickets`: `CREATE USER fabric_user FOR LOGIN fabric_login; GRANT SELECT, ALTER ANY EXTERNAL MIRROR, VIEW DATABASE PERFORMANCE STATE, VIEW DATABASE SECURITY STATE TO fabric_user;`
4. In `BeaconTickets`: `EXEC sys.sp_cdc_enable_db;` — "so that the log reader job exists before Fabric attaches".
5. Fabric portal > `ws-dp800-beacon` > **New item** > **Mirrored Azure SQL Database** > name `BeaconTickets_Mirror` > **Create**.
6. Under **New sources** select **Azure SQL Database**; **Server** `beacon-sql-<suffix>.database.windows.net`, **Database** `BeaconTickets`, **Data gateway** (None), **Authentication kind** **Basic** with `fabric_login` > **Connect**.
7. On **Configure mirroring** keep **Mirror all data** on > **Mirror database**. Wait 2–5 minutes, open **Monitor replication**, and wait for status **Running** and a **Last completed** time per table.
8. Open the mirror's **SQL analytics endpoint** and run `SELECT COUNT(*) FROM Sales.Tickets;` and `SELECT * FROM INFORMATION_SCHEMA.TABLES;`
9. Back in Azure SQL, run `ALTER TABLE Sales.Visitors ADD Email nvarchar(120) NULL;` and watch **Monitor replication**.

### Question

Which option correctly identifies the wrong step **and** predicts what the SQL analytics endpoint shows once the (corrected) runbook has run?

### a.

Step 4 is wrong and must be removed: an Azure SQL Database with change data capture enabled cannot be mirrored (mirroring has its own change feed). Everything else is correct — a managed identity is required even with Basic authentication, and `ALTER ANY EXTERNAL MIRROR` plus `SELECT` and the two `VIEW DATABASE ... STATE` permissions are the documented minimum. After mirroring: `Sales.Tickets`, `Sales.Visitors`, `Sales.VisitLog` and `Sales.PriceHistory` appear; `Sales.PriceHistoryArchive`, `Sales.TicketAudit` and `Sales.Turnstile` do not; `Tickets` lacks the `Notes`, `Photo` and `RowVer` columns; step 9 triggers a full re-snapshot of `Visitors`. Replication compute is free and mirrored storage is free up to one terabyte per capacity unit.

### b.

Step 3 is wrong: `ALTER ANY EXTERNAL MIRROR` is not sufficient — the mirroring principal must be `db_owner` (or hold `CONTROL`). Step 4 is harmless because mirroring is built on top of CDC's log reader. After mirroring, every table in `Sales` appears, including the temporal history and ledger tables, because mirroring copies whole Delta snapshots of the transaction log; `xml` and `image` columns are stored as `varchar(max)`.

### c.

Step 1 is wrong: a system-assigned managed identity is only needed when **Authentication kind** is *Service principal* or *Workspace identity*; with **Basic** (SQL) authentication the login in step 3 is the only identity involved. Step 4 is required, otherwise the **Monitor replication** page shows *Failed*. After mirroring, `Sales.VisitLog` is skipped because it has no primary key, and `Sales.Turnstile` is mirrored since columnstore is just a storage layout.

### d.

Step 7 is wrong: **Mirror all data** must be switched off and `Sales.VisitLog` must be given a primary key first, because heaps are never eligible and selecting them blocks the whole database. Step 4 is correct. After mirroring, `Sales.TicketAudit` appears because ledger tables are ordinary tables with hidden columns, and `Sales.PriceHistoryArchive` appears because history tables are just tables with a clustered index; step 9 only replicates the new column values incrementally without reseeding.

**Capacity and cleanup.** Replication compute is free but the workspace must sit on a running capacity. To stop: open the mirrored database > **Stop replication** (starting again reseeds every table), then **Workspace settings** > **General** > **Remove this workspace** > **Delete**; if you keep the source database, run `EXEC sp_change_feed_disable_db;` on it if the change feed is still active, and delete or pause the Azure SQL resources to stop Azure charges.

## Correct Answer

**a**

## Explanation

The correct answer is **a**. The single wrong step is **4**: "Azure SQL Database can't be mirrored if the database has: enabled Change Data Capture (CDC), Azure Synapse Link for SQL, or the database is already mirrored in another Fabric workspace." Mirroring uses the **change feed** (`sp_help_change_feed`, `sys.dm_change_feed_log_scan_sessions`, `sys.dm_change_feed_errors`), not CDC's capture job, and the two are mutually exclusive on the same database — as are delayed transaction durability and a read-only secondary (mirroring "is only supported on a writable primary database").

### Why every other step is right

- **Step 1 — managed identity.** "Either the System Assigned Managed Identity (SAMI) or the User Assigned Managed Identity (UAMI) of the Azure SQL logical server needs to be enabled and must be the primary identity." The identity is what *publishes* data into OneLake; during creation it is automatically granted **Read and write** on the mirrored database item, and removing that permission breaks replication. This is independent of how *Fabric* authenticates to *SQL* (step 6).
- **Step 2 — network.** Public access with *Allow Azure services*, or a virtual network / on-premises data gateway, are the two documented ways for Fabric to reach the server.
- **Step 3 — permissions.** The tutorial's minimum grant is exactly `SELECT, ALTER ANY EXTERNAL MIRROR, VIEW DATABASE PERFORMANCE STATE, VIEW DATABASE SECURITY STATE`; `ALTER ANY EXTERNAL MIRROR` "is included in higher level permission like CONTROL permission or the db_owner role", so `db_owner` works but is not required. Basic (SQL), Organization account (Entra), Service principal and Workspace identity are all accepted **Authentication kinds**.
- **Step 5 — role.** "User needs to be a member of the Admin/Member role for the workspace to create SQL Database mirroring"; a Contributor hits `Unable to grant required permission to the source server. User does not have permission to reshare`.
- **Step 7 — table selection.** *Mirror all data* replicates up to 1,000 tables (alphabetically by schema then table) and "any new tables created after Mirroring is started will be mirrored"; turning it off lets you pick tables. **Monitor replication** shows database status (*Running*, *Running with warning*, *Stopping/Stopped*, *Failed*, *Paused*) and per-table status, *Rows replicated* and *Last completed*.
- **Step 9 — DDL.** "When there's DDL change, a complete data snapshot is restarted for the changed table, and data is reseeded." Altering the primary key or switching partitions is not allowed at all.

### What the SQL analytics endpoint shows

Creating the mirror "creates these items in your Fabric workspace: the mirrored SQL database item ... [and] a SQL analytics endpoint" (no default semantic model since September 2025). The source schema hierarchy is kept (`Sales.Tickets`, not `Sales_Tickets`). Eligibility:

| Source table | Mirrored? | Rule |
|---|---|---|
| `Sales.Tickets` | Yes, without `Notes` (xml), `Photo` (image), `RowVer` (rowversion) | those data types "can't be mirrored to Fabric OneLake"; the other columns replicate |
| `Sales.Visitors` | Yes | `nvarchar` becomes `varchar(max)` on the endpoint of a mirrored item |
| `Sales.VisitLog` (heap) | Yes | "Starting in April 2025, a table can be mirrored even if it doesn't have a primary key" |
| `Sales.Turnstile` (CCI) | No | "Clustered columnstore indexes aren't currently supported" |
| `Sales.TicketAudit` (ledger) | No | ledger history tables and other ledger structures are excluded (Always Encrypted, in-memory, graph and external tables too) |
| `Sales.PriceHistory` | Yes (current table) | temporal current tables replicate |
| `Sales.PriceHistoryArchive` | No | "Temporal history tables and ledger history tables" can't be mirrored |

The endpoint is the usual read-only surface: T-SQL can "define and query data objects but not manipulate the data"; you can create views, inline TVFs and stored procedures, manage permissions, and join to other warehouses and lakehouses in the workspace. Source RLS, object permissions and dynamic data masking are **not** propagated — "any granular security established in the source database must be re-configured in the mirrored database". A `datetime2(7)` column would lose its seventh digit; a `datetime2(7)`/`datetimeoffset(7)`/`time(7)` primary key makes the table ineligible; LOBs over 1 MB are truncated; `json` and `vector` columns make a table ineligible.

### Why option b is wrong

`db_owner`/`CONTROL` are sufficient but not necessary — the granular grant in step 3 is the documented least-privilege set. CDC is not the transport; it is a blocker. Ledger and temporal history tables are explicitly excluded, and unsupported column types are dropped, not converted to `varchar(max)`.

### Why option c is wrong

The managed identity requirement has nothing to do with the **Authentication kind** used by the Fabric connection: the identity is the server's own principal that writes into OneLake and must be the *primary* identity regardless. Heaps have been eligible since April 2025 (though tables that lacked a PK *before* that date are not picked up automatically — stop/start replication or add them explicitly). Clustered columnstore tables are not supported.

### Why option d is wrong

A table without a primary key no longer blocks anything, and selecting it does not affect other tables. Ledger and temporal-history tables are excluded by name in the limitations. DDL on a mirrored table restarts a full snapshot of that table; there is no incremental column back-fill.

### Positioning: mirroring vs change event streaming vs CDC

```text
Fabric mirroring     analytics copy in OneLake (Delta), read via SQL analytics endpoint / Spark / Direct Lake;
                     change feed on the source; free replication compute; 1 TB free storage per CU (F64 = 64 TB);
                     incompatible with CDC and Synapse Link on the same database; near-real-time (~15 s publishes)
Change event         SQL Server 2025 / Azure SQL DB / MI / Fabric SQL database (preview): streams INSERT/UPDATE/DELETE
streaming (CES)      as CloudEvents to Azure Event Hubs or Fabric Eventstream for event-driven apps and microservices
CDC                  in-database change tables (cdc.<schema>_<table>_CT, __$operation) read by your own ETL;
                     Azure SQL DB needs S3+ on DTU; blocks Fabric mirroring of that database
```

Documentation relied upon (ms.date): Mirrored databases from Azure SQL Database 2025-07-03; Limitations and behaviors for mirrored databases from Azure SQL Database 2026-02-26; Tutorial: configure mirrored databases from Azure SQL Database 2025-11-25; Troubleshoot mirrored databases from Azure SQL Database 2025-11-25; Monitor mirrored database replication 2025-09-05; Mirroring overview 2026-08-28; Change event streaming overview 2026-07-29; Power BI semantic models 2025-12-05.

Hands-on question (Microsoft Fabric capacity required); behaviour is taken from the official documentation as of the ms.date cited above.

## DP-800 Exam Rule to Remember

```text
Mirror Azure SQL Database into Fabric — checklist
  Server:  system- (or user-) assigned managed identity ON and primary; Allow Azure services (or a gateway)
  DB:      writable primary; NOT CDC-enabled, NOT Synapse Link, NOT delayed durability, NOT mirrored elsewhere;
           DTU Free/Basic/Standard < 100 DTU unsupported; any vCore tier OK
  Login:   SELECT + ALTER ANY EXTERNAL MIRROR + VIEW DATABASE PERFORMANCE STATE + VIEW DATABASE SECURITY STATE
  Tenant:  Service principals can call Fabric public APIs; Users can access data stored in OneLake with apps external to Fabric
  Fabric:  Member/Admin; New item > Mirrored Azure SQL Database > connect > Configure mirroring (Mirror all data
           or pick <= 1000 tables) > Mirror database > Monitor replication (Running / Last completed)
  Tables:  no PK is fine (since Apr 2025); NOT: CCI, temporal HISTORY, ledger, in-memory, graph, external,
           json/vector columns; skipped columns: computed, xml, image, text/ntext, rowversion, sql_variant, UDT, geo
  DDL:     any DDL = full re-snapshot of that table; ALTER PK / SWITCH PARTITION forbidden; stop/start = reseed all
  Cost:    replication compute free; storage free up to 1 TB per CU; queries billed normally; capacity must be running
```
