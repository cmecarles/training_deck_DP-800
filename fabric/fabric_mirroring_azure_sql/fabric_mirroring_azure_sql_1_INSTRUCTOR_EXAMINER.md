# Instructor-Examiner guide — Mirroring Azure SQL Database into Fabric 1

Companion to [fabric_mirroring_azure_sql_1.md](fabric_mirroring_azure_sql_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Lab rule.** This is a hands-on Microsoft Fabric lab question. Before reading the scenario, ask: "Have you already run this lab from your own Azure and Fabric accounts?" If yes, ask what they observed at each step (whether the mirror started, what Monitor replication showed per table, which tables and columns appeared on the SQL analytics endpoint, what happened after the ALTER TABLE) before you quiz them. If no, walk through the steps in words using section 2, so that the question can still be answered from the documented facts alone. Do not require the learner to run anything during the call.

**Multiple choice.** This is a multiple-choice question with four options, a to d. Read all four options before taking an answer. The learner must pick one letter. Take the letter, then say only right or wrong.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Integrate SQL solutions with Azure services.
- Task bullet: Configure mirroring of an Azure SQL Database into Fabric; identify prerequisites, eligible tables and columns, and the effect of DDL.
- What is tested: the managed identity, networking and permission prerequisites, the incompatibility of mirroring with CDC, which table types and column types are excluded, what a heap does, and what DDL on a mirrored table triggers.

## 2. Scenario to read aloud

**Piece 1, the story.** "The Beacon Point Lighthouse Museum sells tickets from an Azure SQL Database called BeaconTickets, General Purpose, two vCores, on the logical server beacon dash sql dash suffix. Analysts want the data in Fabric without building pipelines, so you mirror the database into the workspace ws dash dp800 dash beacon and validate a runbook written by a junior engineer. The runbook contains exactly one wrong step."

**Piece 2, prerequisites.** "You need an Azure subscription with rights on the logical server, and a Fabric capacity, a trial is fine, but it must be running, because a paused capacity stops replication. You need the Member or Admin role in the workspace; Contributors lack the Reshare permission the wizard needs. Two Fabric tenant settings must be on: Service principals can call Fabric public APIs, and Users can access data stored in OneLake with apps external to Fabric."

**Piece 3, the source tables.** "In the Azure portal you create the database as a vCore General Purpose database, not the DTU Free or Basic tiers. Then, in BeaconTickets, you create a schema Sales and six tables. Sales dot Tickets: TicketID integer primary key, VisitorID, Price decimal, SoldAt datetime2 six, Notes of type xml, Photo of type image, and RowVer of type rowversion. Sales dot Visitors: VisitorID primary key, FullName nvarchar eighty, Country char two. Sales dot VisitLog: VisitedAt and Gate, a heap with no key at all. Sales dot Turnstile: TurnstileID and Clicks, with an inline clustered columnstore index. Sales dot TicketAudit: AuditID primary key, TicketID, Action, created WITH LEDGER ON, APPEND ONLY. And Sales dot PriceHistory: a system-versioned temporal table with PriceID primary key, Price, ValidFrom and ValidTo period columns, and a history table Sales dot PriceHistoryArchive."

**Piece 4, the data.** "Two visitors: Mara Quinn from Ireland and Tobias Lund from Norway. Two tickets at twelve euros each, sold on the first of September 2026 at ten and ten oh five. Two visit log rows at ten ten and ten twelve, gates A and B. One price history row, price twelve."

**Piece 5, the runbook, steps 1 to 4.** "Step 1: in the Azure portal, on the logical server, under Security, Identity, turn the System assigned managed identity on and save. Verify with SELECT star FROM sys dot dm underscore server underscore managed underscore identities, expecting one row with is underscore primary equal to one. Step 2: under Security, Networking, Public access, set Allow Azure services and resources to access this server to Yes, or configure a virtual network or an on-premises data gateway if public access is off. Step 3: in master, create a login fabric underscore login with a password; in BeaconTickets, create the user fabric underscore user for that login and grant it SELECT, ALTER ANY EXTERNAL MIRROR, VIEW DATABASE PERFORMANCE STATE and VIEW DATABASE SECURITY STATE. Step 4: in BeaconTickets, run EXEC sys dot sp underscore cdc underscore enable underscore db, quote, so that the log reader job exists before Fabric attaches, end quote."

**Piece 6, the runbook, steps 5 to 9.** "Step 5: in the Fabric portal, in the workspace, New item, Mirrored Azure SQL Database, name it BeaconTickets underscore Mirror, Create. Step 6: under New sources choose Azure SQL Database; server beacon dash sql dash suffix dot database dot windows dot net, database BeaconTickets, data gateway none, authentication kind Basic with fabric underscore login, then Connect. Step 7: on Configure mirroring keep Mirror all data on, then Mirror database. Wait two to five minutes, open Monitor replication, and wait for status Running and a Last completed time per table. Step 8: open the mirror's SQL analytics endpoint and run SELECT COUNT star FROM Sales dot Tickets, and SELECT star FROM INFORMATION underscore SCHEMA dot TABLES. Step 9: back in Azure SQL, run ALTER TABLE Sales dot Visitors ADD Email nvarchar 120 NULL, and watch Monitor replication."

**Piece 7, option a.** "Step 4 is wrong and must be removed: an Azure SQL Database with change data capture enabled cannot be mirrored, because mirroring has its own change feed. Everything else is correct: a managed identity is required even with Basic authentication, and ALTER ANY EXTERNAL MIRROR plus SELECT and the two VIEW DATABASE STATE permissions are the documented minimum. After mirroring, Sales dot Tickets, Visitors, VisitLog and PriceHistory appear; PriceHistoryArchive, TicketAudit and Turnstile do not; Tickets lacks the Notes, Photo and RowVer columns; step 9 triggers a full re-snapshot of Visitors. Replication compute is free and mirrored storage is free up to one terabyte per capacity unit."

**Piece 8, option b.** "Step 3 is wrong: ALTER ANY EXTERNAL MIRROR is not sufficient; the mirroring principal must be db underscore owner, or hold CONTROL. Step 4 is harmless because mirroring is built on top of CDC's log reader. After mirroring, every table in Sales appears, including the temporal history and ledger tables, because mirroring copies whole Delta snapshots of the transaction log; xml and image columns are stored as varchar max."

**Piece 9, option c.** "Step 1 is wrong: a system-assigned managed identity is only needed when the authentication kind is Service principal or Workspace identity; with Basic SQL authentication the login in step 3 is the only identity involved. Step 4 is required, otherwise Monitor replication shows Failed. After mirroring, Sales dot VisitLog is skipped because it has no primary key, and Sales dot Turnstile is mirrored since columnstore is just a storage layout."

**Piece 10, option d.** "Step 7 is wrong: Mirror all data must be switched off and Sales dot VisitLog must be given a primary key first, because heaps are never eligible and selecting them blocks the whole database. Step 4 is correct. After mirroring, Sales dot TicketAudit appears because ledger tables are ordinary tables with hidden columns, and Sales dot PriceHistoryArchive appears because history tables are just tables with a clustered index; step 9 only replicates the new column values incrementally without reseeding."

## 3. Setup script (reference only; do not read verbatim unless asked)

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

Runbook T-SQL fragments:

```sql
-- Step 1 verification
SELECT * FROM sys.dm_server_managed_identities;
-- Step 3, in master
CREATE LOGIN fabric_login WITH PASSWORD = '<strong password>';
-- Step 3, in BeaconTickets
CREATE USER fabric_user FOR LOGIN fabric_login;
GRANT SELECT, ALTER ANY EXTERNAL MIRROR, VIEW DATABASE PERFORMANCE STATE, VIEW DATABASE SECURITY STATE TO fabric_user;
-- Step 4
EXEC sys.sp_cdc_enable_db;
-- Step 8, on the SQL analytics endpoint of the mirror
SELECT COUNT(*) FROM Sales.Tickets;
SELECT * FROM INFORMATION_SCHEMA.TABLES;
-- Step 9, on the source
ALTER TABLE Sales.Visitors ADD Email nvarchar(120) NULL;
-- Cleanup, on the source, if the change feed is still active
EXEC sp_change_feed_disable_db;
```

## 4. The question (ask exactly this)

"Which option correctly identifies the wrong step and predicts what the SQL analytics endpoint shows once the corrected runbook has run? Option a, b, c, or d?"

- **a.** Step 4 is wrong and must be removed: an Azure SQL Database with change data capture enabled cannot be mirrored (mirroring has its own change feed). Everything else is correct: a managed identity is required even with Basic authentication, and ALTER ANY EXTERNAL MIRROR plus SELECT and the two VIEW DATABASE ... STATE permissions are the documented minimum. After mirroring: Sales.Tickets, Sales.Visitors, Sales.VisitLog and Sales.PriceHistory appear; Sales.PriceHistoryArchive, Sales.TicketAudit and Sales.Turnstile do not; Tickets lacks the Notes, Photo and RowVer columns; step 9 triggers a full re-snapshot of Visitors. Replication compute is free and mirrored storage is free up to one terabyte per capacity unit.
- **b.** Step 3 is wrong: ALTER ANY EXTERNAL MIRROR is not sufficient; the mirroring principal must be db_owner (or hold CONTROL). Step 4 is harmless because mirroring is built on top of CDC's log reader. After mirroring, every table in Sales appears, including the temporal history and ledger tables, because mirroring copies whole Delta snapshots of the transaction log; xml and image columns are stored as varchar(max).
- **c.** Step 1 is wrong: a system-assigned managed identity is only needed when Authentication kind is Service principal or Workspace identity; with Basic (SQL) authentication the login in step 3 is the only identity involved. Step 4 is required, otherwise the Monitor replication page shows Failed. After mirroring, Sales.VisitLog is skipped because it has no primary key, and Sales.Turnstile is mirrored since columnstore is just a storage layout.
- **d.** Step 7 is wrong: Mirror all data must be switched off and Sales.VisitLog must be given a primary key first, because heaps are never eligible and selecting them blocks the whole database. Step 4 is correct. After mirroring, Sales.TicketAudit appears because ledger tables are ordinary tables with hidden columns, and Sales.PriceHistoryArchive appears because history tables are just tables with a clustered index; step 9 only replicates the new column values incrementally without reseeding.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.** The single wrong step is step 4. An Azure SQL Database cannot be mirrored if it has change data capture enabled, Azure Synapse Link for SQL, or is already mirrored in another workspace. Mirroring uses the change feed, not CDC's capture job, and the two are mutually exclusive on the same database.

| Option | Why wrong |
|---|---|
| b | db underscore owner or CONTROL are sufficient but not necessary; the granular grant in step 3 is the documented least-privilege set. CDC is a blocker, not the transport. Ledger and temporal history tables are explicitly excluded, and unsupported column types are dropped, not converted to varchar max. |
| c | The managed identity requirement is independent of the authentication kind; the identity is the server's own principal that writes into OneLake and must be the primary identity regardless. Heaps have been eligible since April 2025. Clustered columnstore tables are not supported. |
| d | A table without a primary key no longer blocks anything, and selecting it does not affect other tables. Ledger and temporal history tables are excluded by name. DDL on a mirrored table restarts a full snapshot of that table; there is no incremental back-fill. |

Endpoint contents after the corrected runbook: Sales dot Tickets without Notes, Photo and RowVer; Sales dot Visitors, with nvarchar shown as varchar max; Sales dot VisitLog, a heap, mirrored; Sales dot PriceHistory, the current temporal table. Not mirrored: Sales dot Turnstile (CCI), Sales dot TicketAudit (ledger), Sales dot PriceHistoryArchive (temporal history). Step 9 causes a complete re-snapshot and reseed of Visitors.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with step 4. Mirroring needs to read changes from the source. Does it use CDC for that, or something of its own? And can both be on at once?"
2. "Now the managed identity. It is not how Fabric logs in to SQL. It is how the SQL server pushes data into OneLake. Does that depend on the authentication kind chosen in step 6?"
3. "Permissions. The tutorial lists four grants. Is a bigger role required, or only sufficient?"
4. "Table eligibility. Think of three categories that are excluded: a storage layout, a tamper-evident table type, and the history side of system versioning. Which of the six tables fall into those?"
5. "The heap. Since April 2025, does a missing primary key block a table? Eliminate any option that says heaps are never eligible or are skipped."
6. "Two options remain. One says every table appears and xml becomes varchar max. Check that against the excluded column types."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "b, mirroring runs on the CDC log reader" | Confuses the change feed with CDC | "If mirroring were built on CDC, would the docs forbid CDC on a mirrored database? Check the blockers list." |
| "b, db underscore owner is required" | Confuses sufficient with necessary | "ALTER ANY EXTERNAL MIRROR is included in CONTROL and db underscore owner. Does that make the smaller grant insufficient?" |
| "c, Basic authentication makes the identity unnecessary" | Mixes the two directions of authentication | "There are two directions: Fabric reading SQL, and SQL writing OneLake. Which one is the managed identity for?" |
| "c, a heap cannot be mirrored" | Outdated rule from before April 2025 | "That was true once. What changed in April 2025?" |
| "c, columnstore is just storage" | Does not know CCI is excluded | "Look at the limitations list for indexes. Is clustered columnstore on it?" |
| "d, heaps block the whole database" | Invents a cascade rule | "Does the documentation say anything about one table blocking others? Or is eligibility per table?" |
| "d, ledger tables are ordinary tables" | Does not know ledger exclusion | "Ledger structures have their own line in the limitations. What does it say?" |
| "Step 9 updates only the new column" | Assumes incremental schema evolution | "What does the documentation say happens to a table when a DDL change is detected?" |
| "Tickets appears with all columns" | Forgets excluded column types | "Three of its columns are xml, image and rowversion. Are those representable in OneLake?" |

## 8. Teaching notes (after the answer is complete or revealed)

- **The wrong step.** Step 4 enables CDC. An Azure SQL Database cannot be mirrored if it has CDC enabled, Azure Synapse Link for SQL, or is already mirrored in another workspace. Mirroring uses the change feed, visible through sp underscore help underscore change underscore feed and the dm underscore change underscore feed DMVs, not CDC's capture job. Delayed transaction durability and a read-only secondary are also blockers; mirroring is only supported on a writable primary.
- **Why every other step is right.** Step 1: either the system-assigned or a user-assigned managed identity of the logical server must be enabled and be the primary identity; it publishes data into OneLake, is granted Read and write on the mirrored item automatically, and this is independent of how Fabric authenticates to SQL. Step 2: public access with Allow Azure services, or a virtual network or gateway. Step 3: the minimum grant is SELECT, ALTER ANY EXTERNAL MIRROR, VIEW DATABASE PERFORMANCE STATE and VIEW DATABASE SECURITY STATE; db underscore owner works but is not required; Basic, Organization account, Service principal and Workspace identity are all accepted authentication kinds. Step 5: Member or Admin is required; a Contributor gets an error about not having permission to reshare. Step 7: Mirror all data replicates up to one thousand tables, alphabetically by schema then table, and new tables created later are picked up; turning it off lets you choose. Monitor replication shows database status and per-table status, rows replicated and Last completed. Step 9: when there is a DDL change, a complete data snapshot is restarted for that table and data is reseeded; altering the primary key or switching partitions is not allowed.
- **What the endpoint shows.** The mirror creates a mirrored database item and a SQL analytics endpoint; no default semantic model since September 2025. The source schema hierarchy is kept. Tickets is mirrored without Notes, Photo and RowVer, because xml, image and rowversion columns cannot be mirrored; computed, text and ntext, sql underscore variant, UDT and geo columns are skipped too. Visitors is mirrored and nvarchar appears as varchar max. VisitLog, a heap, is mirrored since April 2025. Turnstile is excluded because clustered columnstore is not supported. TicketAudit is excluded because ledger structures are not supported, together with Always Encrypted, in-memory, graph and external tables. PriceHistory is mirrored but PriceHistoryArchive, its history table, is not. A datetime2 seven column loses its seventh digit; a datetime2 seven, datetimeoffset seven or time seven primary key makes the table ineligible; LOBs over one megabyte are truncated; json and vector columns make a table ineligible.
- **Security and cost.** Source row-level security, object permissions and dynamic data masking are not propagated; granular security must be reconfigured in the mirror. Replication compute is free, storage is free up to one terabyte per capacity unit, queries are billed normally, and the capacity must be running.
- **Positioning.** Fabric mirroring gives an analytics copy in OneLake read through the endpoint, Spark or Direct Lake. Change event streaming streams changes as CloudEvents to Event Hubs or Eventstream for event-driven apps. CDC keeps change tables inside the database for your own ETL and blocks mirroring.

Memory hook: "Managed identity on and primary, Allow Azure services, four grants, no CDC. Heaps yes; CCI, ledger, temporal history no; xml, image, rowversion dropped; any DDL reseeds the table."

## 9. Follow-up oral questions (optional)

1. "The museum later moves BeaconTickets to a Basic DTU tier to save money. What happens to mirroring?" (It is unsupported; DTU Free, Basic and Standard below 100 DTU cannot be mirrored; any vCore tier is fine.)
2. "You stop replication for maintenance and start it again. What happens to the data in OneLake?" (Every table is reseeded from a fresh snapshot.)
3. "Analysts had row-level security on Sales dot Tickets in Azure SQL. Does the mirror enforce it?" (No. Source RLS, permissions and masking are not propagated; they must be recreated on the mirrored database.)

## 10. References

- Mirrored databases from Azure SQL Database: https://learn.microsoft.com/en-us/fabric/mirroring/azure-sql-database
- Limitations and behaviors for mirrored databases from Azure SQL Database: https://learn.microsoft.com/en-us/fabric/mirroring/azure-sql-database-limitations
- Tutorial: configure mirrored databases from Azure SQL Database: https://learn.microsoft.com/en-us/fabric/mirroring/azure-sql-database-tutorial
- Troubleshoot mirrored databases from Azure SQL Database: https://learn.microsoft.com/en-us/fabric/mirroring/azure-sql-database-troubleshoot
- Monitor mirrored database replication: https://learn.microsoft.com/en-us/fabric/mirroring/monitor
- Mirroring overview in Microsoft Fabric: https://learn.microsoft.com/en-us/fabric/mirroring/overview
- Change event streaming overview: https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/change-event-streaming/overview
- Power BI semantic models in Fabric Data Warehouse: https://learn.microsoft.com/en-us/fabric/data-warehouse/semantic-models
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
