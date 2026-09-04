# Instructor-Examiner guide — Fabric SQL Database 1

Companion to [fabric_sql_database_1.md](fabric_sql_database_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Lab rule.** This is a hands-on Microsoft Fabric lab question. Before reading the scenario, ask: "Have you already run this lab from your own Fabric account?" If yes, ask what they observed at each step (which items appeared, which statements failed, what the analytics endpoint listed, what landed in Git) before you quiz them; use their observations as the starting point. If no, walk through the steps in words using section 2, so that the question can still be answered from the documented facts alone. Do not require the learner to run anything during the call.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%); also Configure and secure the database engine (15–20%).
- Skill: Design and implement a database solution on Microsoft Fabric.
- Task bullet: Create and configure a SQL database in Fabric; identify differences from Azure SQL Database; configure source control.
- What is tested: which Azure SQL Database features are missing in the Fabric flavour (logins, ledger, in-memory, CDC, EXECUTE AS), how the always-on mirror restricts columnstore, what the SQL analytics endpoint exposes, and what Git integration commits.

## 2. Scenario to read aloud

**Piece 1, the story.** "TidePool Marinas runs a berth-booking system. They want it in a SQL database inside Microsoft Fabric, so the same data is immediately available for analytics in OneLake. You build it yourself, then predict how the Fabric flavour of the SQL engine reacts to a script written for Azure SQL Database."

**Piece 2, prerequisites.** "You need a Fabric capacity. A sixty-day trial is enough, and a trial allows at most three SQL databases. You need the Admin or Member role in a workspace you create. SQL database in Fabric is on by default; if the tile is missing, a Fabric admin checks tenant settings. Everything uses your own Microsoft Entra account. The database has no SQL logins."

**Piece 3, creating the workspace and database in the portal.** "In the Fabric portal, choose Workspaces, then New workspace. Name it ws dash dp800 dash tidepool. Expand Advanced, pick the workspace type that points at your capacity, Trial or Fabric capacity, and select Apply. Inside the workspace choose New item, pick the SQL database tile, type TidePoolOps, and select Create. An equivalent path is the Databases workload home page, New, SQL database."

**Piece 4, the CLI and REST alternatives.** "With the Fabric CLI, installed by pip install ms dash fabric dash cli, you run fab auth login, then fab create ws dash dp800 dash tidepool dot Workspace slash TidePoolOps dot SQLDatabase, then fab ls on the workspace to list items. With the REST API, you POST to api dot fabric dot microsoft dot com, v1, workspaces, workspace id, items, with a JSON body whose displayName is TidePoolOps and whose type is SQLDatabase. A GET on workspaces, workspace id, SqlDatabases lists them, and a GET on one database returns properties ServerFqdn and DatabaseName."

**Piece 5, what appears and how to connect.** "Back in the workspace item list, note which items now exist with the name TidePoolOps. On the database item choose the three dots, Settings, Connection strings. The Data Source looks like tcp colon, an id, dot database dot fabric dot microsoft dot com, comma 1433. Initial Catalog is the database name. On the same Settings page, SQL endpoint shows that the SQL analytics endpoint has a different server name, an id, dot tenant, dot fabric dot microsoft dot com. The Open in button on the query editor fills in SSMS 21 or the MSSQL extension for VS Code. From a terminal you can use sqlcmd with dash S the server, dash G for Entra authentication, dash d TidePoolOps, and dash i setup dot sql."

**Piece 6, the setup script, schema and tables.** "In the database query editor, New query, you create a schema called Marina and three tables. Marina dot Berth has BerthID, an integer primary key; Pontoon, a single character; and LengthM, a decimal five two. Marina dot Booking has BookingID, an integer primary key; BerthID, an integer foreign key to Berth; Nights, an integer; and BookedAt, a datetime2 with six digits, defaulting to SYSUTCDATETIME. Marina dot TideReading has ReadingID, a bigint primary key that is NONCLUSTERED; ReadAt, a datetime2 six; and HeightM, a decimal four two."

**Piece 7, the data.** "Berth gets three rows: berth 1 on pontoon A, twelve and a half metres; berth 2 on pontoon A, fifteen metres; berth 3 on pontoon B, nine point seven five metres. Booking gets two rows: booking 100 on berth 1 for three nights, and booking 101 on berth 3 for one night. TideReading gets two rows: reading 1 on the first of September 2026 at six in the morning, height four point one zero, and reading 2 at noon the same day, height zero point eight five."

**Piece 8, statements S1 to S3.** "Then ten statements run one at a time, each as its own batch, on the SQL database side of the editor, not the analytics endpoint.

- S1 creates a login called dockmaster with a password.
- S2 creates a user for a colleague's Entra account, using the syntax CREATE USER, the user principal name in square brackets, FROM EXTERNAL PROVIDER.
- S3 creates a temporal table, Marina dot BerthRate, with BerthID as primary key, NightlyRate a decimal, ValidFrom and ValidTo as datetime2 six period columns generated always as row start and row end, PERIOD FOR SYSTEM underscore TIME, and WITH SYSTEM underscore VERSIONING ON, history table Marina dot BerthRateHistory."

**Piece 9, statements S4 to S7.** "

- S4 creates Marina dot FeeLedger, FeeID primary key and Amount decimal, WITH LEDGER equals ON, APPEND underscore ONLY equals ON.
- S5 creates Marina dot GateEvent, a memory-optimized table: EventID with a NONCLUSTERED HASH primary key and bucket count 1024, Gate a two-character column, WITH MEMORY underscore OPTIMIZED ON and DURABILITY SCHEMA underscore AND underscore DATA.
- S6 creates a clustered columnstore index called CCI underscore TideReading on the existing table Marina dot TideReading.
- S7 creates a brand new table Marina dot SensorLog with LogID bigint, SensorID int, Value decimal nine three, and an inline index CCI underscore SensorLog CLUSTERED COLUMNSTORE, declared inside the CREATE TABLE."

**Piece 10, statements S8 to S10.** "

- S8 runs EXEC sys dot sp underscore cdc underscore enable underscore db, to enable change data capture.
- S9 runs EXECUTE AS USER equals the colleague's name, then SELECT USER underscore NAME, then REVERT.
- S10 creates Marina dot WaveReport with ReportID as primary key and a second column whose name is Wave, space, Height, in square brackets, decimal four two."

**Piece 11, observing the analytics side and Git.** "Afterwards you switch the query editor drop-down to SQL analytics endpoint, expand Explorer and list the tables. In the database item, Replication, Monitor replication, you note any table with status NotSupported. Then you connect the workspace to a Git repository through Workspace settings, Git integration, open Source control, tick TidePoolOps and Commit. You note what folder and file types appear in the repository."

## 3. Setup script (reference only; do not read verbatim unless asked)

```text
fab auth login
fab create ws-dp800-tidepool.Workspace/TidePoolOps.SQLDatabase
fab ls ws-dp800-tidepool.Workspace

POST https://api.fabric.microsoft.com/v1/workspaces/{workspaceId}/items
{ "displayName": "TidePoolOps", "type": "SQLDatabase", "description": "DP-800 lab" }
GET  https://api.fabric.microsoft.com/v1/workspaces/{workspaceId}/SqlDatabases
GET  https://api.fabric.microsoft.com/v1/workspaces/{workspaceId}/SqlDatabases/{databaseId}

sqlcmd -S <id>.database.fabric.microsoft.com,1433 -G -d TidePoolOps -i setup.sql
```

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

## 4. The question (ask exactly this)

Part 1: "For S1 to S10, state whether each statement succeeds or fails, and why. Let's go one at a time, starting with S1."

Part 2: "Which of the tables Berth, Booking, TideReading, BerthRate, BerthRateHistory and SensorLog are visible through the SQL analytics endpoint after replication catches up?"

Part 3: "What is committed to Git for the database, and which two database-level settings are not included?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

| Stmt | Outcome | Why |
|---|---|---|
| S1 | Fails | Logins are not supported; Entra is the only identity provider, SQL authentication is not supported |
| S2 | Succeeds | Users representing Entra principals are the only supported users; CREATE USER ... FROM EXTERNAL PROVIDER is the documented syntax |
| S3 | Succeeds | Temporal tables are supported; BerthRateHistory is created but excluded from mirroring |
| S4 | Fails | Ledger tables cannot be created |
| S5 | Fails | In-memory (memory-optimized) tables cannot be created |
| S6 | Fails | While mirroring is active, a clustered columnstore index cannot be created on an existing table; mirroring is always on |
| S7 | Succeeds | An inline CCI at table creation is allowed, but the table will never be mirrored |
| S8 | Fails | Change data capture is not available |
| S9 | Fails | EXECUTE AS is not available in Fabric SQL database |
| S10 | Fails | Column names cannot contain spaces (nor comma, semicolon, braces, parens, newline, tab, equals) |

Part 2. Visible through the SQL analytics endpoint: Marina dot Berth, Marina dot Booking, Marina dot TideReading, Marina dot BerthRate. Not visible: Marina dot BerthRateHistory (temporal history tables are excluded from mirroring) and Marina dot SensorLog (CCI table, shown as NotSupported in Monitor replication). FeeLedger, GateEvent and WaveReport were never created.

Part 3. Git receives a folder TidePoolOps dot SQLDatabase containing a SQL database project: a dot sqlproj file plus one dot sql file per object (for example Marina slash Tables slash Berth dot sql) and the Fabric item metadata. The two settings not included are the database collation and the compatibility level.

Also useful if the learner asks: creating the database provisioned two items, the SQL database and its SQL analytics endpoint; no default semantic model is created any more.

## 6. Hint ladder (one hint per attempt, in order)

**S1, CREATE LOGIN**
1. "What kind of principal is a login: server level or database level? Does a Fabric SQL database expose a server you can create things on?"
2. "Which identity provider does the scenario say is the only one? Think about who you signed in as."
3. "The limitations page has one line: logins are not supported, only users representing Microsoft Entra principals. Apply it."

**S2, CREATE USER FROM EXTERNAL PROVIDER**
1. "This is the counterpart of S1. If logins are out, how do you still give a colleague a database identity?"
2. "The clause FROM EXTERNAL PROVIDER is exactly the Azure SQL way to map an Entra account to a database user. Is that provider Entra?"

**S3, temporal table**
1. "Is system versioning on the list of unavailable features, or on the list of available ones?"
2. "Temporal tables are marked Yes. There is a twist about the history table and the mirror, but does the statement itself succeed?"

**S4, ledger table**
1. "Ledger is one of the specialized table types. The Fabric feature table gives it a one-word answer. Which word?"
2. "The table-level limitations say that in-memory, ledger, ledger history and Always Encrypted tables cannot be created."

**S5, memory-optimized table**
1. "Look at the WITH clause: MEMORY underscore OPTIMIZED equals ON. Same list as S4."
2. "In-memory OLTP is in the same sentence as ledger in the limitations."

**S6, CCI on an existing table**
1. "In Azure SQL Database this would work, because the primary key is nonclustered. What is different about a Fabric SQL database from the moment it is created?"
2. "Think of the mirror into OneLake. It is on from creation and cannot be turned off. What does it forbid on an existing table?"
3. "The rule is: when mirroring is active, clustered columnstore indexes cannot be created on an existing table."

**S7, inline CCI at CREATE TABLE**
1. "This is not an existing table. The index is declared inside the CREATE TABLE. Is that the same case as S6?"
2. "The documentation allows a CCI created at the same time as the table using the inline index syntax. There is a consequence for the mirror. What is it?"

**S8, sp underscore cdc underscore enable underscore db**
1. "Change data capture is a row in the Azure SQL versus Fabric comparison table. What does the Fabric column say?"
2. "Fabric already has an always-on change mechanism into OneLake. CDC is not offered."

**S9, EXECUTE AS USER**
1. "Impersonation is in the comparison table too. Azure SQL Database allows EXECUTE AS USER. Does Fabric?"
2. "The Fabric column says No. How would you test a principal's permissions instead?"

**S10, column named Wave Height**
1. "Look carefully at the second column name. What character does it contain?"
2. "Why would a column name need to be representable in Parquet or Delta? Which characters would that forbid?"
3. "Fabric forbids spaces and the characters comma, semicolon, braces, parens, newline, tab and equals in column names."

**Part 2, the analytics endpoint**
1. "Start by listing which tables actually exist after the ten statements. Three were never created."
2. "Two of the existing tables are not mirrored. One is a history table. One has a particular index."
3. "Temporal history tables are excluded from mirroring, and a CCI table shows NotSupported in Monitor replication."

**Part 3, Git**
1. "What does a database look like when it is turned into files? Think of the SQL projects format used by SqlPackage and the SQL Database Projects extension."
2. "The folder is named after the item and its type. Inside there is a project file and one file per object."
3. "Two properties of the whole database, not of any object, are left out. One is about sorting text; the other is about engine version behaviour."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "S1 succeeds, it is just a strong password" | Assumes SQL authentication exists in Fabric | "Who is the only identity provider in a Fabric SQL database? Does a login belong to that provider?" |
| "S2 fails, there are no users either" | Confuses logins with users | "Logins and users are different levels. Which of them represents an Entra principal inside the database?" |
| "S3 fails because of the history table and mirroring" | Mixes the mirror rule with statement success | "The mirror decides what gets copied. Does it decide whether the CREATE TABLE runs?" |
| "S6 succeeds, the primary key is nonclustered" | Applies the Azure SQL rule without the mirror | "That reasoning is right for Azure SQL Database. What is always running in a Fabric SQL database from the moment it exists?" |
| "S7 fails, same as S6" | Does not distinguish inline CCI at creation from CCI on an existing table | "Read the CREATE TABLE again. Is the index added later, or declared at the same time as the table?" |
| "S8 succeeds, CDC is in Azure SQL Database" | Assumes feature parity | "Feature parity is the trap of this question. Look at the CDC row of the comparison." |
| "S9 succeeds, it is just impersonation" | Same parity assumption | "Impersonation has its own row in the comparison table. What does Fabric say?" |
| "S10 succeeds, brackets make any name legal" | Knows T-SQL delimited identifiers, not the Fabric naming rule | "Brackets handle T-SQL. Where else does this column have to exist, and in what file format?" |
| "BerthRateHistory is visible on the endpoint" | Forgets temporal history exclusion | "The current table replicates. What does the mirroring limitation say about the history table?" |
| "SensorLog is visible on the endpoint" | Forgets that inline CCI means never mirrored | "S7 succeeded, but at a price. What did the documentation say about mirroring that table?" |
| "Git contains a dot bacpac or dot dacpac" | Confuses build artifacts with source | "Source control stores source, not a compiled package. Which file type describes a database as source?" |
| "The two excluded settings are recovery model and file growth" | Thinks of on-premises SQL Server settings | "Those do not even exist in a Fabric SQL database. Think of settings that describe text sorting and engine version." |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the model in one sentence first: a SQL database in Fabric is the Azure SQL Database engine plus always-on mirroring plus Fabric identity. Then go through the groups.

- **Identity is Entra only (S1, S2).** Logins are not supported. Only users representing Microsoft Entra principals exist, created with CREATE USER name FROM EXTERNAL PROVIDER, and often you do not need CREATE USER at all because workspace roles and item permissions create the users automatically. SQL authentication never works; sqlcmd needs dash G and SSMS needs Microsoft Entra ID authentication.
- **Specialized tables (S3, S4, S5).** Temporal tables: Yes. Ledger: No. Always Encrypted: No. In-memory: No. For temporal tables the current table is mirrored but the history table is excluded. The lab uses datetime2 with six digits on purpose, because Delta supports six fractional digits; a datetime2 seven column loses its seventh digit and a datetime2 seven primary key makes a table unmirrorable.
- **Columnstore and the always-on mirror (S6, S7).** Mirroring starts at creation, has no settings, cannot be turned off, and mirrors every supported table. While it is active, a clustered columnstore index cannot be created on an existing table. The only route is stopMirroring through the REST API, add the index, start again, and then the table is not mirrored. An inline CCI in CREATE TABLE is allowed, but the new table is never mirrored. Nonclustered columnstore has no such restriction.
- **Engine features missing (S8, S9, S10).** CDC: No. EXECUTE AS: No. Column names may not contain spaces or the characters comma, semicolon, braces, parens, newline, tab and equals, because the same column must exist in the Delta or Parquet copy. Other gaps worth remembering: no BACKUP command, no server-level roles, no application roles, no elastic query or elastic jobs, no failover groups, geo-restore or long-term retention, OPENROWSET BULK only from OneLake. Limits: 32 vCores, 4 TB, 150 databases per workspace.
- **The SQL analytics endpoint.** Creating the database provisions two items: the database and its SQL analytics endpoint. Mirrored data lands as Delta Parquet in OneLake and the endpoint is the same read-only experience as a lakehouse endpoint. Computed columns and image, text, xml, rowversion, sql underscore variant, UDT, geometry, geography and hierarchyid columns are skipped silently; tables with json or vector columns are not mirrored. Views and procedures are not mirrored. Source row-level security, permissions and masking are not propagated.
- **Source control.** Git integration turns the live database into a SQL database project, a dot sqlproj file with one dot sql per object, under the folder TidePoolOps dot SQLDatabase. Update from the repository is a project build plus a SqlPackage publish. Collation and compatibility level are not part of source control or deployment pipelines. Static data goes in a query saved to Shared Queries and marked Set as Post-deployment Script.

Memory hook: "Entra only, mirror always on, no ledger, no in-memory, no CDC, no EXECUTE AS, no spaces in names. Git gets a sqlproj, but not collation or compat level."

## 9. Follow-up oral questions (optional)

1. "You need Marina dot TideReading to be a clustered columnstore table and still analytics-friendly. What are your two realistic options?" (Either recreate it with an inline CCI, accepting it will not be mirrored, or use a nonclustered columnstore index, which has no restriction and keeps the table mirrored.)
2. "Which two server names do you meet with one Fabric SQL database, and which one is read-only?" (The database on id dot database dot fabric dot microsoft dot com port 1433, and the SQL analytics endpoint on id dot tenant dot fabric dot microsoft dot com; the endpoint is read-only.)
3. "How do you take a backup of a Fabric SQL database?" (You do not. There is no BACKUP command; automatic zone-redundant backups with seven days of retention and point-in-time restore are provided.)

## 10. References

- Limitations for SQL database in Microsoft Fabric: https://learn.microsoft.com/en-us/fabric/database/sql/limitations
- SQL database in Microsoft Fabric overview: https://learn.microsoft.com/en-us/fabric/database/sql/overview
- Feature comparison, Azure SQL Database and SQL database in Fabric: https://learn.microsoft.com/en-us/fabric/database/sql/feature-comparison-sql-database-fabric
- Create a SQL database in Fabric: https://learn.microsoft.com/en-us/fabric/database/sql/create
- Connect to your SQL database in Fabric: https://learn.microsoft.com/en-us/fabric/database/sql/connect
- Authentication in SQL database in Fabric: https://learn.microsoft.com/en-us/fabric/database/sql/authentication
- Mirroring overview for SQL database in Fabric: https://learn.microsoft.com/en-us/fabric/database/sql/mirroring-overview
- Limitations of mirroring for SQL database in Fabric: https://learn.microsoft.com/en-us/fabric/database/sql/mirroring-limitations
- Source control integration for SQL database in Fabric: https://learn.microsoft.com/en-us/fabric/database/sql/source-control
- Create a SQL database with the Fabric CLI: https://learn.microsoft.com/en-us/fabric/database/sql/create-fabric-cli
- Create a SQL database with the REST API: https://learn.microsoft.com/en-us/fabric/database/sql/create-rest-api
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
