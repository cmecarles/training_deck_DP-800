# Instructor-Examiner guide — Auditing 1

Companion to [auditing_1.md](auditing_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Each option is a list of audit rows. Read all four options before taking an answer, describing each option as a list of which actions, A1 to A9, appear and with which action code. A good approach is to first let the learner say, for each of the nine actions, whether it is recorded and as what; then match to a letter. Accept the letter as the final answer.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure and protect data (15–20%).
- Skill: Implement auditing and compliance features.
- Task bullet: Configure SQL Server Audit with server and database audit specifications.
- What is tested: a server audit is created disabled and the state change itself is audited, object-level audit actions record the permission used on the base object, `BY public` includes `dbo`, and which action groups cover principal changes versus permission changes.

## 2. Scenario to read aloud

**Piece 1, the story.** "A port authority runs a SQL Server 2025 instance hosting a database called DockYard. Customs regulations require an audit trail of every read of the cargo manifests and of every change to database principals. The trail goes to a binary file that auditors read back with T-SQL. The security administrator, a sysadmin, sets it all up. The folder C colon backslash dp800audit exists and the service account can write to it."

**Piece 2, the server audit.** "In master, the administrator creates a server audit called DockYard underscore Audit. Its target is a file in that folder, with a max size of ten megabytes and two rollover files. Its options are QUEUE DELAY zero and ON FAILURE CONTINUE. Note that nothing in this statement turns the audit on."

**Piece 3, the server audit specification.** "Then a server audit specification, DockYard ServerSpec, for that audit. It adds two action groups: FAILED LOGIN GROUP and SERVER PRINCIPAL CHANGE GROUP. It is created WITH STATE ON."

**Piece 4, the database and its objects.** "The database DockYard is created, with a schema Cargo. Table Cargo dot Manifests has ManifestID, an integer primary key; Vessel; Shipper; and DeclaredValue, a decimal. Two rows: manifest 1, MV Tramontana, Nordic Steel, two hundred fifty thousand; manifest 2, MV Garbi, Iberia Citrus, forty-eight thousand. A second table, Cargo dot Berths, has BerthID and Name, with two rows, North-1 and South-3. A view, Cargo dot vManifestSummary, selects ManifestID and Vessel from Manifests. A user called Customs is created without login and granted SELECT and UPDATE on the whole Cargo schema."

**Piece 5, the database audit specification.** "Then a database audit specification, DockYard DbSpec, for the same server audit. It adds two things. First, the action SELECT ON OBJECT Cargo dot Manifests BY public. Second, the group DATABASE PRINCIPAL CHANGE GROUP. It is created WITH STATE ON."

**Piece 6, the nine actions.** "Then nine actions run in order. All succeed.

- A1, as Customs: SELECT ManifestID from Cargo dot Manifests where ManifestID is 1. At this point the server audit has not been enabled yet.
- A2, as the administrator, in master: ALTER SERVER AUDIT DockYard Audit WITH STATE ON.
- A3, as Customs: SELECT Vessel and DeclaredValue from Manifests where ManifestID is 1.
- A4, as Customs: SELECT Name from Cargo dot Berths.
- A5, as Customs: UPDATE Manifests SET DeclaredValue to forty-nine thousand where ManifestID is 2.
- A6, as Customs: SELECT Vessel from the view vManifestSummary.
- A7, as the administrator: CREATE USER Inspector WITHOUT LOGIN.
- A8, as the administrator: GRANT SELECT ON Cargo dot Berths TO Inspector.
- A9, as the administrator, that is dbo: SELECT COUNT star from Cargo dot Manifests."

**Piece 7, the auditor's query.** "Finally the auditor selects action id, database principal name, object name and statement from sys dot fn underscore get underscore audit underscore file, over the DockYard Audit files, ordered by event time and sequence number. You will be asked which rows come back, one per audited event, in order."

**Piece 8, option a.** "Option a has seven rows. A1 as SL on Manifests by Customs. Then AUSC, the audit state change. Then A3 as SL on Manifests. A4 as SL on Berths. A6 as SL on Manifests. A7 as CR, create, on Inspector by dbo. And A9 as SL on Manifests by dbo. A5 and A8 are absent."

**Piece 9, option b.** "Option b has five rows. AUSC first. A3 as SL on Manifests by Customs. A6 as SL, but with object name vManifestSummary, the view. A7 as CR on Inspector. And A8 as G, grant, on Berths by dbo. A1, A4, A5 and A9 are absent."

**Piece 10, option c.** "Option c has six rows. AUSC first. A3 as SL on Manifests by Customs. A5, the UPDATE statement, recorded as SL on Manifests by Customs. A6 as SL on Manifests. A7 as CR on Inspector by dbo. And A9 as SL on Manifests by dbo. A1, A4 and A8 are absent."

**Piece 11, option d.** "Option d has five rows. AUSC first. A3 as SL on Manifests. A5 recorded as UP, update, on Manifests by Customs. A6 as SL on Manifests. And A7 as CR on Inspector. A1, A4, A8 and A9 are absent. So d differs from c in two places: A5 is UP instead of SL, and A9 is missing."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
USE master;
GO
CREATE SERVER AUDIT DockYard_Audit
TO FILE (FILEPATH = 'C:\dp800audit\', MAXSIZE = 10 MB, MAX_ROLLOVER_FILES = 2)
WITH (QUEUE_DELAY = 0, ON_FAILURE = CONTINUE);
GO
CREATE SERVER AUDIT SPECIFICATION DockYard_ServerSpec
FOR SERVER AUDIT DockYard_Audit
ADD (FAILED_LOGIN_GROUP),
ADD (SERVER_PRINCIPAL_CHANGE_GROUP)
WITH (STATE = ON);
GO
CREATE DATABASE DockYard;
GO
USE DockYard;
GO
CREATE SCHEMA Cargo;
GO
CREATE TABLE Cargo.Manifests
(
    ManifestID    INT           NOT NULL PRIMARY KEY,
    Vessel        VARCHAR(40)   NOT NULL,
    Shipper       VARCHAR(60)   NOT NULL,
    DeclaredValue DECIMAL(12,2) NOT NULL
);
INSERT INTO Cargo.Manifests VALUES
  (1, 'MV Tramontana', 'Nordic Steel AS',  250000.00),
  (2, 'MV Garbi',      'Iberia Citrus SL',  48000.00);
CREATE TABLE Cargo.Berths (BerthID INT NOT NULL PRIMARY KEY, Name VARCHAR(20) NOT NULL);
INSERT INTO Cargo.Berths VALUES (1, 'North-1'), (2, 'South-3');
GO
CREATE VIEW Cargo.vManifestSummary AS SELECT ManifestID, Vessel FROM Cargo.Manifests;
GO
CREATE USER Customs WITHOUT LOGIN;
GRANT SELECT, UPDATE ON SCHEMA::Cargo TO Customs;
GO
CREATE DATABASE AUDIT SPECIFICATION DockYard_DbSpec
FOR SERVER AUDIT DockYard_Audit
ADD (SELECT ON OBJECT::Cargo.Manifests BY public),
ADD (DATABASE_PRINCIPAL_CHANGE_GROUP)
WITH (STATE = ON);
GO
-- A1  (as Customs)                          -- the server audit has NOT been enabled yet
SELECT ManifestID FROM Cargo.Manifests WHERE ManifestID = 1;
-- A2  (as the administrator, in master)
ALTER SERVER AUDIT DockYard_Audit WITH (STATE = ON);
-- A3  (as Customs)
SELECT Vessel, DeclaredValue FROM Cargo.Manifests WHERE ManifestID = 1;
-- A4  (as Customs)
SELECT Name FROM Cargo.Berths;
-- A5  (as Customs)
UPDATE Cargo.Manifests SET DeclaredValue = 49000.00 WHERE ManifestID = 2;
-- A6  (as Customs)
SELECT Vessel FROM Cargo.vManifestSummary;
-- A7  (as the administrator)
CREATE USER Inspector WITHOUT LOGIN;
-- A8  (as the administrator)
GRANT SELECT ON Cargo.Berths TO Inspector;
-- A9  (as the administrator, i.e. dbo)
SELECT COUNT(*) AS N FROM Cargo.Manifests;
-- Auditor
SELECT action_id, database_principal_name, object_name, statement
FROM sys.fn_get_audit_file('C:\dp800audit\DockYard_Audit_*.sqlaudit', DEFAULT, DEFAULT)
ORDER BY event_time, sequence_number;
```

## 4. The question (ask exactly this)

"Which rows does the auditor's query return, one row per audited event, in order? Option a, b, c or d?"

- a. A1 SL Manifests; AUSC; A3 SL Manifests; A4 SL Berths; A6 SL Manifests; A7 CR Inspector; A9 SL Manifests.
- b. AUSC; A3 SL Manifests; A6 SL vManifestSummary; A7 CR Inspector; A8 G Berths.
- c. AUSC; A3 SL Manifests; A5 SL Manifests; A6 SL Manifests; A7 CR Inspector; A9 SL Manifests.
- d. AUSC; A3 SL Manifests; A5 UP Manifests; A6 SL Manifests; A7 CR Inspector.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: c.** Verified on SQL Server 2025 RTM. The engine returned:

| action_id | class | database_principal_name | object_name | statement |
|---|---|---|---|---|
| AUSC | A | | | ALTER SERVER AUDIT ... STATE = ON |
| SL | U | Customs | Manifests | A3 |
| SL | U | Customs | Manifests | A5, the UPDATE |
| SL | U | Customs | Manifests | A6, through the view |
| CR | SU | dbo | Inspector | A7 |
| SL | U | dbo | Manifests | A9 |

Per action: A1 not recorded, audit still disabled. A2 recorded as AUSC. A3 recorded, SL. A4 not recorded, Berths is not in the specification. A5 recorded as SL, because the UPDATE's WHERE reads the table and the SELECT permission on Manifests is used. A6 recorded as SL on the base table Manifests, not the view. A7 recorded as CR by DATABASE_PRINCIPAL_CHANGE_GROUP. A8 not recorded, GRANT is a permission change, not a principal change. A9 recorded, because BY public includes dbo.

Why each wrong option is wrong:

- a: records A1, made before the audit was enabled, and A4 on Berths, which is not audited. It also omits A5. It mistakes the specification's STATE ON for the audit being on, and an object action for a schema action.
- b: names the view for A6, records the GRANT, and omits A5 and A9. The audit names the base object whose permission was used; GRANT is not a principal change; dbo is in public.
- d: labels A5 as UP and drops A9. The UPDATE itself was not audited, only its read of the table, so the row is SL. And dbo is not exempt from BY public. This is the closest distractor.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with A1. The database audit specification says STATE ON, but what state is a server audit in when it is created? Has anyone turned the audit itself on before A1?"
2. "A2 turns the audit on. Is the act of changing the audit's state itself recorded, even without any action group for it? That gives you the first row."
3. "A4 reads Berths. The specification names one object. Does an action ON OBJECT Manifests cover Berths? That rules out one option."
4. "A8 is a GRANT. Is a GRANT a principal change, or a permission change? Which group would you need for it? That rules out another option."
5. "Two options remain. Look at A9. The action says BY public. Is dbo a member of public, or is the administrator exempt?"
6. "Finally A5, the UPDATE with a WHERE clause. The specification only lists SELECT. To evaluate the WHERE, which permission on Manifests does the engine use? What action code does that produce?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| a | Believes STATE ON on the specification means auditing is on | "A specification feeds an audit. What is the audit's own state after CREATE SERVER AUDIT?" |
| b | Thinks the audit records the object named in the statement | "Which object's permission is actually checked when you read through a view with an unbroken ownership chain?" |
| d | Thinks the statement type decides the action code, and that dbo is exempt | "Does the specification contain an UPDATE action at all? And who is in public?" |
| "A5 is not recorded; only SELECT is audited" | Misses that the UPDATE reads the table | "Does an UPDATE with a WHERE clause need to read rows first? Which permission does that use?" |
| "A2 is not recorded; there is no AUDIT_CHANGE_GROUP" | Does not know state changes are intrinsically audited | "Is turning an audit on something the engine records on its own?" |
| "A8 is recorded by DATABASE_PRINCIPAL_CHANGE_GROUP" | Confuses permission and principal changes | "Did A8 create, alter or drop a user or role?" |

## 8. Teaching notes (after the answer is complete or revealed)

Walk through the audit model:

- **A server audit is created disabled.** A specification created WITH STATE ON only feeds the audit; until ALTER SERVER AUDIT WITH STATE ON, the target receives nothing. So A1 is lost. ALTER SERVER AUDIT must be issued from master; from a user database it fails with error 33074.
- **Enabling the audit is itself audited**, action AUSC, class A, with no action group required. That is the first row.
- **Object-level actions are object-specific.** SELECT ON OBJECT Manifests covers only that table. To cover the schema use ON SCHEMA Cargo; to cover every object access in the database use SCHEMA_OBJECT_ACCESS_GROUP. A4 on Berths is not recorded.
- **What gets audited is the permission used, on the base object.** An UPDATE with a WHERE must read the table, the SELECT permission on Manifests is used, and the SELECT audit action fires as SL. The update itself is not audited; an UP row would need ADD UPDATE ON OBJECT Manifests. A read through the view records Manifests, not vManifestSummary, because with an unbroken ownership chain the access still counts as access to the table.
- **DATABASE_PRINCIPAL_CHANGE_GROUP** captures CREATE USER as CR, class SU, object name Inspector. **GRANT is a permission change**, covered by DATABASE_OBJECT_PERMISSION_CHANGE_GROUP or SCHEMA_OBJECT_PERMISSION_CHANGE_GROUP, which were not added.
- **BY public includes dbo.** Administrators are not exempt; to audit administrators, audit dbo. A9 is recorded.

Options and reading: QUEUE_DELAY 0 means synchronous delivery, so rows are in the file immediately; the default of 1000 milliseconds could leave the last events in memory for up to a second. ON_FAILURE CONTINUE keeps the instance running and loses events if the file cannot be written; SHUTDOWN stops the instance and requires the SHUTDOWN permission; FAIL_OPERATION fails only the audited actions. Reading the file with sys.fn_get_audit_file requires VIEW SERVER SECURITY AUDIT on SQL Server 2022 and later, CONTROL SERVER before.

Structural rules: a database-level action cannot go in a server audit specification, error 156; a server-only group such as FAILED_LOGIN_GROUP cannot go in a database specification, error 102. One server audit specification per audit, one database audit specification per database per audit, multiple audits per instance.

Azure SQL Database has no CREATE SERVER AUDIT; auditing is configured on the logical server or per database, to a storage account, Log Analytics or Event Hubs, with default groups BATCH_COMPLETED_GROUP and the two database authentication groups. Azure SQL Managed Instance keeps the CREATE SERVER AUDIT syntax with URL or EXTERNAL_MONITOR targets. Auditing answers who did what, reads included; Ledger gives tamper-evident proof of data history; temporal tables give queryable history with no security guarantee.

Memory hook: "Created off, turning on is logged. Audit the permission on the base object. Public includes dbo."

## 9. Follow-up oral questions (optional)

1. "What single addition to the database audit specification would make A5 appear as UP?" (ADD UPDATE ON OBJECT Cargo dot Manifests BY public.)
2. "Which action group would capture A8, the GRANT?" (DATABASE_OBJECT_PERMISSION_CHANGE_GROUP or SCHEMA_OBJECT_PERMISSION_CHANGE_GROUP.)
3. "What error do you get if you run ALTER SERVER AUDIT from inside the DockYard database?" (Error 33074, cannot alter a server audit from a user database; it must be done in master.)

## 10. References

- SQL Server Audit, overview of audits and specifications: https://learn.microsoft.com/en-us/sql/relational-databases/security/auditing/sql-server-audit-database-engine
- SQL Server Audit action groups and actions: https://learn.microsoft.com/en-us/sql/relational-databases/security/auditing/sql-server-audit-action-groups-and-actions
- CREATE SERVER AUDIT (Transact-SQL): https://learn.microsoft.com/en-us/sql/t-sql/statements/create-server-audit-transact-sql
- CREATE DATABASE AUDIT SPECIFICATION (Transact-SQL): https://learn.microsoft.com/en-us/sql/t-sql/statements/create-database-audit-specification-transact-sql
- sys.fn_get_audit_file (Transact-SQL): https://learn.microsoft.com/en-us/sql/relational-databases/system-functions/sys-fn-get-audit-file-transact-sql
- Auditing for Azure SQL Database: https://learn.microsoft.com/en-us/azure/azure-sql/database/auditing-overview
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
