# Instructor-Examiner guide — Lakehouse SQL Analytics Endpoint 1

Companion to [fabric_lakehouse_sql_endpoint_1.md](fabric_lakehouse_sql_endpoint_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Lab rule.** This is a hands-on Microsoft Fabric lab question. Before reading the scenario, ask: "Have you already run this lab from your own Fabric account?" If yes, ask what they observed at each step (the validation message for the table name, the inferred column types, which statements failed on the endpoint, the S10 result rows, how the count in step 12 caught up) before you quiz them. If no, walk through the steps in words using section 2, so that the question can still be answered from the documented facts alone. Do not require the learner to run anything during the call.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Integrate SQL solutions with Azure services.
- Task bullet: Use the SQL analytics endpoint of a lakehouse; write cross-item queries; understand metadata sync and Direct Lake.
- What is tested: how Delta types map to T-SQL types on the endpoint, what "read-only" allows and forbids, three-part-name queries between a lakehouse and a warehouse, OPENROWSET on lakehouse files, the warehouse engine's unsupported statements, metadata refresh options, and Direct Lake fallback behaviour.

## 2. Scenario to read aloud

**Piece 1, the story.** "Glacier Peak Resort logs every chair-lift ride as CSV. You land the file in a Fabric lakehouse, turn it into a Delta table, query it through the lakehouse's SQL analytics endpoint, join it to a Fabric Data Warehouse in the same workspace, and predict which T-SQL statements the read-only endpoint accepts. You need a Fabric capacity, a trial is fine, and the Admin role in the workspace. Your identity is a Microsoft Entra account; SQL logins do not exist on any of these items."

**Piece 2, the lakehouse and the CSV.** "You create the workspace ws dash dp800 dash glacier, then a Lakehouse named GlacierLake, leaving lakehouse schemas off so tables land in dbo. You save a CSV file called lift underscore rides dot csv with a header row and six columns: ride underscore id, lift underscore id, rider underscore pass, ride underscore ts, duration underscore s, and peak underscore wind. Five data rows. Ride 1 on lift L1, pass P100, at nine oh two on the first of February 2026, 412 seconds, wind twelve point five. Ride 2, L1, P101, nine oh three thirty, 408 seconds, twelve point five. Ride 3, L2, P100, nine forty, 655 seconds, eighteen. Ride 4, L3, P102, ten fifteen, 301 seconds, nine point two. Ride 5, L2, P103, ten twenty, 660 seconds, eighteen point four."

**Piece 3, loading to a table.** "In the Lakehouse explorer you create a subfolder raw under Files and upload the CSV into it. Then, on the file, you choose Load to Tables, New table. In the dialog Load file to new table you enter the table name lift underscore rides. Only letters, digits and underscore are accepted; the lab asks you to try lift dash rides first and read the validation message. You keep Column header ticked and the separator as a comma, then Load. When the table appears under Tables you open it and note the Delta column types the loader inferred. Then you run Load to Tables on the same file again, into the Existing table lift underscore rides, with Load mode Append. That doubles the rows to ten."

**Piece 4, the warehouse.** "You create a Warehouse named GlacierDW. In its query editor you create dbo dot lift underscore capacity with lift underscore id varchar ten, seats smallint, and lift underscore name varchar forty. Three rows: L1 with six seats, Summit Six; L2 with four seats, Ridge Quad; L3 with two seats, Bunny Double."

**Piece 5, opening the endpoint.** "You open GlacierLake and, at the top right of the ribbon, switch the view from Lakehouse to SQL analytics endpoint. The endpoint is also listed as its own item in the workspace, with the lakehouse's name. If lift underscore rides is not in Explorer yet, you select the Refresh icon in the Explorer toolbar and wait; metadata sync normally lags less than a minute. You select plus Warehouses in Explorer, tick GlacierDW, and confirm it appears in the tree. Then you run nine statements, each as its own batch."

**Piece 6, statements S1 to S5.** "

- S1 queries sys dot columns for dbo dot lift underscore rides, returning the column name, the SQL type name, max length and collation, ordered by column id.
- S2 inserts a row into dbo dot lift underscore rides: ride 99, lift L1, pass P999, a timestamp, 400 seconds, wind ten.
- S3 creates a table dbo dot lift underscore notes with lift underscore id varchar ten and note varchar two hundred.
- S4 creates a view dbo dot v underscore rides underscore per underscore lift that groups lift underscore rides by lift underscore id and returns lift underscore id, a count called rides, and the average duration cast to float, called avg underscore seconds.
- S5 creates a procedure dbo dot usp underscore busiest underscore lift that selects the top one lift underscore id and rides from that view, ordered by rides descending."

**Piece 7, statements S6 to S9.** "

- S6 joins the view to GlacierDW dot dbo dot lift underscore capacity, using the three-part name, on lift underscore id, returning lift underscore id, lift underscore name, rides and seats.
- S7 reads the raw file, not the table: SELECT TOP 3 star FROM OPENROWSET BULK, path slash Files slash raw slash lift underscore rides dot csv, FORMAT CSV, HEADER underscore ROW TRUE.
- S8 selects lift underscore id and rides from the view, FOR XML PATH, lift.
- S9 runs SET TRANSACTION ISOLATION LEVEL SNAPSHOT."

**Piece 8, S10 in the warehouse.** "You switch to the GlacierDW query editor, add GlacierLake with plus Warehouses, and run S10: create a table dbo dot rides underscore by underscore lift with lift underscore id varchar ten and rides int; insert into it a SELECT of lift underscore id and COUNT star from GlacierLake dot dbo dot lift underscore rides grouped by lift underscore id; then select everything from the new table ordered by lift underscore id."

**Piece 9, the stale count.** "Back in the lakehouse you append the CSV once more, and immediately run SELECT COUNT star FROM dbo dot lift underscore rides on the endpoint several times. You note how the count catches up and where the manual Refresh lives."

## 3. Setup script (reference only; do not read verbatim unless asked)

```text
ride_id,lift_id,rider_pass,ride_ts,duration_s,peak_wind
1,L1,P100,2026-02-01T09:02:00,412,12.5
2,L1,P101,2026-02-01T09:03:30,408,12.5
3,L2,P100,2026-02-01T09:40:00,655,18.0
4,L3,P102,2026-02-01T10:15:00,301,9.2
5,L2,P103,2026-02-01T10:20:00,660,18.4
```

```sql
-- GlacierDW (Warehouse)
CREATE TABLE dbo.lift_capacity (lift_id varchar(10) NOT NULL, seats smallint NOT NULL, lift_name varchar(40) NOT NULL);
INSERT INTO dbo.lift_capacity VALUES ('L1', 6, 'Summit Six'), ('L2', 4, 'Ridge Quad'), ('L3', 2, 'Bunny Double');

-- GlacierLake SQL analytics endpoint
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

-- S10  (run in the warehouse)
CREATE TABLE dbo.rides_by_lift (lift_id varchar(10) NOT NULL, rides int NOT NULL);
INSERT INTO dbo.rides_by_lift
SELECT lift_id, COUNT(*) FROM GlacierLake.dbo.lift_rides GROUP BY lift_id;
SELECT * FROM dbo.rides_by_lift ORDER BY lift_id;
```

## 4. The question (ask exactly this)

Part 1: "For S1, state the SQL data type and collation that the endpoint assigns to a Delta string column such as lift underscore id, whatever the loader inferred for the numeric columns."

Part 2: "For S2 to S10, state whether each statement succeeds or fails and why. For S10 give the result rows, after the load and the first append, before the last append. One statement at a time, starting with S2."

Part 3: "Name the three documented ways to force a metadata refresh when the count in step 12 is stale, and state which of them is available only when the workspace has the New metadata sync preview turned on."

Part 4: "If you build a Direct Lake semantic model on this endpoint and include dbo dot v underscore rides underscore per underscore lift, what happens under Direct Lake on SQL and under Direct Lake on OneLake?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Part 1.** A Delta string column becomes varchar 8000 with collation Latin1 underscore General underscore 100 underscore BIN2 underscore UTF8 in the endpoint of a lakehouse. varchar max is only used by the endpoints of mirrored items and Fabric databases. Other mappings: integer to int, long to bigint, double to float, timestamp to datetime2 with six digits, boolean to bit, decimal p s to decimal p s.

**Part 2.**

| Stmt | Outcome | Why |
|---|---|---|
| S2 | Fails | The endpoint is read-only: you cannot insert, update or delete through it |
| S3 | Fails | Creating, altering and dropping tables is only supported in the warehouse, not in the endpoint |
| S4 | Succeeds | Views, functions and stored procedures can be created on the endpoint |
| S5 | Succeeds | Same rule; the procedure persists in the endpoint, not in the Delta lake |
| S6 | Succeeds | Cross-database three-part-name query to a warehouse in the same workspace, added with plus Warehouses |
| S7 | Succeeds | OPENROWSET BULK with a relative slash Files path works only when querying a lakehouse through its own endpoint; OneLake reads via OPENROWSET are in preview |
| S8 | Fails | SELECT FOR XML is in the unsupported list of the warehouse engine |
| S9 | Fails | SET TRANSACTION ISOLATION LEVEL is in the unsupported list |
| S10 | Succeeds | Warehouse tables accept CREATE TABLE and INSERT; the INSERT SELECT reads the lakehouse by three-part name. Result rows: L1, 4; L2, 4; L3, 2 |

**Part 3.** One: the Refresh icon in the Explorer toolbar of the endpoint editor. Two: the Refresh SQL endpoint metadata REST API. Three: EXEC sys dot sp underscore dw underscore refresh underscore ext underscore table with the table name. Option three works only for endpoints created after Workspace settings, Warehouse settings, New metadata sync preview was enabled.

**Part 4.** Under Direct Lake on SQL, a table based on a non-materialized SQL view falls back to DirectQuery, or fails if fallback is disabled. Under Direct Lake on OneLake, a table based on a SQL view is not supported; only Delta tables, or a lakehouse materialized view, can be used. No default semantic model is created any more; you create one yourself and Direct Lake is its storage mode.

## 6. Hint ladder (one hint per attempt, in order)

**Part 1, the string type**
1. "The endpoint does not let you pick a type. It derives one from the Delta type. A Delta string has no length. What does the warehouse engine use as the wide varchar for a lakehouse?"
2. "The number is eight thousand, and the collation is the UTF-8 binary one that the warehouse engine always uses."
3. "Say the collation in full: Latin1 General 100, then a binary flag, then UTF8."

**S2, INSERT**
1. "What single word describes the endpoint's relationship to the Delta tables?"
2. "Read-only means the data cannot be changed from T-SQL. Where would you insert instead?"

**S3, CREATE TABLE**
1. "A new table on the endpoint would have to become a Delta folder under Tables. Can the endpoint write Delta?"
2. "Creating, altering and dropping tables is a warehouse thing. The endpoint's tables are autogenerated."

**S4 and S5, view and procedure**
1. "Read-only refers to the data and the autogenerated tables. Does it also forbid objects that hold no data?"
2. "You can extend the endpoint with your own schemas, views, procedures and functions. Where do they live?"

**S6, the three-part join**
1. "Both items are in the same workspace, and you added the warehouse with plus Warehouses. What does that button enable?"
2. "Item name dot schema dot table. Is that a valid reference on the endpoint?"

**S7, OPENROWSET**
1. "OPENROWSET reads files, not tables. Which files can a lakehouse endpoint reach with a relative path?"
2. "The relative path starting with slash Files works only from a lakehouse's own endpoint."

**S8, FOR XML**
1. "The endpoint runs the warehouse engine. The warehouse has an unsupported list. Is FOR XML on it?"

**S9, isolation level**
1. "Same unsupported list. What does the warehouse say about SET TRANSACTION ISOLATION LEVEL?"

**S10, the warehouse insert**
1. "This runs in the warehouse, which is writable. Can the warehouse read the lakehouse by three-part name?"
2. "For the rows, count rides per lift after the load and one append. Five rides doubled to ten. How many on L1, L2, L3?"
3. "Original rides: two on L1, two on L2, one on L3. Double each."

**Part 3, refresh options**
1. "One option is a button you already used in step 8. Where is it?"
2. "One option is a REST API for the whole endpoint. One is a stored procedure for a single table."
3. "The stored procedure is named sp underscore dw underscore refresh underscore ext underscore table. Which workspace setting must have been on when the endpoint was created?"

**Part 4, Direct Lake**
1. "A SQL view is not a Delta table. Direct Lake reads Delta files. What must happen when the source is a view?"
2. "There are two flavours of Direct Lake. One can fall back to DirectQuery. The other never falls back. Which is which?"
3. "Direct Lake on SQL goes through the endpoint and can fall back. Direct Lake on OneLake reads OneLake directly, so a non-materialized view is simply not supported."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "String becomes varchar max" | Confuses lakehouse endpoint with mirrored-item endpoint | "varchar max is used by one kind of endpoint. Is a lakehouse that kind?" |
| "String becomes nvarchar" | Expects Unicode via nvarchar | "The warehouse engine handles Unicode differently. Which collation family ends in UTF8?" |
| "S4 fails, the endpoint is read-only" | Overextends read-only to all objects | "Read-only applies to the data and the autogenerated tables. Does a view contain data?" |
| "S6 fails, cross-database queries are not allowed" | Does not know same-workspace three-part names | "You added the warehouse with plus Warehouses. Why would that button exist?" |
| "S7 fails, OPENROWSET is only for Azure storage" | Does not know the relative slash Files path | "There are three location forms for OPENROWSET. One is a relative path. Where does it work?" |
| "S8 succeeds, FOR XML is standard T-SQL" | Assumes full SQL Server surface | "The endpoint shares the warehouse engine, and the warehouse has a documented unsupported list." |
| "S9 succeeds, snapshot isolation is what the warehouse uses" | Confuses the engine's internal isolation with the SET statement | "The engine may use snapshot internally. Can you set it yourself with SET TRANSACTION ISOLATION LEVEL?" |
| "S10 returns L1 two, L2 two, L3 one" | Forgets the first append | "How many rows did the table have after step 6?" |
| "The stored procedure works everywhere" | Misses the preview prerequisite | "That procedure needs a particular workspace setting on before the endpoint was created. Which one?" |
| "Direct Lake on OneLake falls back to DirectQuery" | Swaps the two flavours | "Which flavour talks to the endpoint and therefore has something to fall back to?" |

## 8. Teaching notes (after the answer is complete or revealed)

- **What the endpoint is.** Every lakehouse automatically provisions a SQL analytics endpoint, a read-only T-SQL surface over the Delta tables under Tables. It runs on the same engine as the Fabric Data Warehouse, so the warehouse's T-SQL surface, data types and limitations apply, minus writes to the autogenerated tables. Files under Files are not tables; S7 reads them as files. Load to Tables writes Delta with V-order; table names accept letters, digits and underscore only, up to 256 characters, no dashes or spaces; an unticked Column header gives columns underscore c0, underscore c1 and so on.
- **Types are derived, not chosen (S1).** Delta string becomes varchar 8000 with Latin1 General 100 BIN2 UTF8 on a lakehouse endpoint, varchar max on mirrored items. Timestamp becomes datetime2 with six digits, double becomes float, integer int, long bigint, boolean bit. Arrays, maps and structs are not surfaced as columns. Data in varchar columns of a lakehouse endpoint is truncated at 8 KB.
- **What read-only means (S2 to S5).** No INSERT, UPDATE, DELETE, MERGE, no CREATE, ALTER or DROP TABLE; those belong to the warehouse or to Spark. But you can add schemas, views, procedures and inlineable scalar functions, GRANT and DENY, row-level and column-level security, and dynamic data masking. Because it shares the warehouse engine, the warehouse's unsupported list also applies: FOR XML, SET TRANSACTION ISOLATION LEVEL, SET ROWCOUNT, recursive queries, triggers, synonyms, materialized views, CREATE USER, the vector type, PREDICT, BULK LOAD.
- **Cross-item queries (S6, S10).** In the current workspace you can query warehouses and endpoints by item dot schema dot table after adding them with plus Warehouses. Data flows only into a warehouse, never back into the lakehouse from T-SQL. Cross-region connections are not supported; at most 150 warehouse plus endpoint items per workspace.
- **OPENROWSET (S7).** Reads Parquet, CSV and JSONL from Azure Blob, ADLS or OneLake (preview). Three location forms: an absolute onelake URL, a relative path plus DATA underscore SOURCE, and a relative path starting with slash Files, which works only from a lakehouse's own endpoint. HEADER underscore ROW TRUE turns the first line into column names.
- **Metadata sync.** Lag is normally under one minute; the background process runs while the endpoint is active and halts after fifteen minutes of inactivity. Manual refresh: the Refresh icon, the Refresh SQL endpoint metadata REST API, and sp underscore dw underscore refresh underscore ext underscore table for one table, which needs the New metadata sync preview enabled before the endpoint was created.
- **Direct Lake.** No default semantic model since September 2025. Direct Lake on SQL uses the endpoint for discovery and permissions and falls back to DirectQuery for SQL views or SQL-based granular security. Direct Lake on OneLake never falls back, and a table on a non-materialized SQL view is not supported.

Memory hook: "Endpoint equals warehouse engine, read-only on Delta tables: views yes, inserts no, FOR XML no. String is varchar 8000 BIN2 UTF8. Three refreshes, the proc needs the new sync. Direct Lake on SQL falls back, on OneLake it does not."

## 9. Follow-up oral questions (optional)

1. "You need dbo dot v underscore rides underscore per underscore lift in a Direct Lake on OneLake model. What do you build instead of the view?" (A Delta table, for example a lakehouse materialized view or a table written by Spark or a pipeline.)
2. "A colleague creates a foreign-key constraint on lift underscore rides in the endpoint. What is the side effect?" (Further schema changes on that table from the lakehouse side are blocked until the constraint is removed.)
3. "How would you get the ride data into the warehouse permanently rather than querying it live?" (INSERT SELECT in the warehouse from the three-part name, as S10 did, or a pipeline or dataflow.)

## 10. References

- What is the SQL analytics endpoint for a lakehouse: https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-sql-analytics-endpoint
- Load to Delta Lake tables: https://learn.microsoft.com/en-us/fabric/data-engineering/load-to-tables
- T-SQL surface area in Fabric Data Warehouse: https://learn.microsoft.com/en-us/fabric/data-warehouse/tsql-surface-area
- Data types in Fabric Data Warehouse: https://learn.microsoft.com/en-us/fabric/data-warehouse/data-types
- Query the Warehouse or SQL analytics endpoint: https://learn.microsoft.com/en-us/fabric/data-warehouse/query-warehouse
- Browse file content with OPENROWSET: https://learn.microsoft.com/en-us/fabric/data-warehouse/browse-file-content-with-openrowset
- SQL analytics endpoint performance and metadata sync considerations: https://learn.microsoft.com/en-us/fabric/data-warehouse/sql-analytics-endpoint-performance
- Direct Lake overview: https://learn.microsoft.com/en-us/fabric/fundamentals/direct-lake-overview
- Power BI semantic models in Fabric Data Warehouse: https://learn.microsoft.com/en-us/fabric/data-warehouse/semantic-models
- Limitations of Fabric Data Warehouse: https://learn.microsoft.com/en-us/fabric/data-warehouse/limitations
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
