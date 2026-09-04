# Instructor-Examiner guide — Audit Retention 1

Companion to [audit_retention_1.md](audit_retention_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**This question.** It is a multiple-choice question with four options, a to d, and only one is correct. Read all five requirements and all four options before taking an answer. The options are long portal and storage configurations; describe each one in words, naming the settings and values that matter, and offer to repeat any option. This is a conceptual Azure question: nothing is executed, and the learner is not expected to have run anything.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Implement data security and compliance.
- Task bullet: Configure auditing for Azure SQL Database, including destinations, retention and immutable storage.
- What is tested: server-level versus database-level audit policies, the storage account plus Log Analytics destination pattern, how a locked time-based immutability policy gives a WORM archive and how it must relate to the SQL retention setting, the separate switch for Microsoft support operations, and the file-level time filtering of sys dot fn underscore get underscore audit underscore file underscore v2.

## 2. Scenario to read aloud

**Piece 1, the story.** "A pharmaceutical manufacturer runs its batch-record system on Azure SQL Database. The logical server is called pharmatrace dash sql. It hosts the production database PharmaTrace, a copy called PharmaTraceDev, and a new database is added every quarter for each new plant. Manufacturing regulations impose five requirements on auditing. I will read them one at a time."

**Piece 2, requirement one.** "Requirement one. Every database on the server, existing and future, must be audited with the default action groups. The audit logs must be kept for seven years, that is two thousand five hundred fifty-five days, in a WORM store. WORM means write once, read many. Nobody in the subscription, DBAs and Owners included, may delete or shorten the logs before they expire. But the retention must be extendable if a regulator asks."

**Piece 3, requirement two.** "Requirement two. The security operations team must alert on batch events within minutes using KQL, the Kusto Query Language. They must keep the events interactively queryable for ninety days, and keep them cheaply for two years."

**Piece 4, requirements three and four.** "Requirement three. Operations performed by Microsoft support engineers on the server during support cases must be captured. Requirement four. PharmaTrace alone must additionally record permission changes and schema-object access. Those are the action groups DATABASE underscore OBJECT underscore PERMISSION underscore CHANGE underscore GROUP and SCHEMA underscore OBJECT underscore ACCESS underscore GROUP. They must go into a separate storage account owned by the compliance team, without changing what the other databases record."

**Piece 5, requirement five.** "Requirement five. Auditors must be able to pull, with T-SQL, only the events of a given two-hour window from the seven-year blob archive, without the function scanning every file in the archive. The question is: which configuration meets all five requirements? There are four options. I will describe each one."

**Piece 6, option a, part one.** "Option a. On the server, in the portal under Security and then Auditing, enable auditing with two destinations. The first is a general-purpose v2 storage account called ptaudit, authenticating with a managed identity, container sqldbauditlogs, and Retention in days set to two thousand six hundred. The second destination is the Log Analytics workspace pt dash law."

**Piece 7, option a, part two.** "Still option a. On the container sqldbauditlogs in ptaudit, configure a time-based retention policy of two thousand five hundred fifty-five days, set Allow protected append writes to Append blobs, and lock the policy. In the workspace pt dash law, set the AzureDiagnostics table to ninety days of analytics retention and seven hundred thirty days of total retention, and create a log alert on AzureDiagnostics where Category equals SQLSecurityAuditEvents and action underscore name underscore s equals BATCH COMPLETED."

**Piece 8, option a, part three.** "Still option a. On the server's Auditing page, switch Enable Auditing of Microsoft support operations to ON, with the same destinations. On PharmaTrace, enable database-level auditing to the compliance storage account with the PowerShell cmdlet Set dash AzSqlDatabaseAudit, passing AuditActionGroup with the two groups, permission change and schema object access. Auditors query with sys dot fn underscore get underscore audit underscore file underscore v2. Its first argument is the blob URL prefix: ptaudit dot blob dot core dot windows dot net, slash sqldbauditlogs, slash pharmatrace dash sql, slash master, slash SqlDbAuditing underscore ServerAudit, slash. Then DEFAULT, DEFAULT, and two more arguments: a start time of eight in the morning UTC and an end time of ten in the morning UTC on the fourth of September 2026."

**Piece 9, option b.** "Option b. Enable database-level auditing on PharmaTrace and PharmaTraceDev only, with no server policy, so that each database writes to its own folder. Destination ptaudit with Retention in days set to zero, plus the workspace pt dash law. On the container, place a legal hold tagged GxP dash 7y. Support engineers are said to be covered by BATCH underscore COMPLETED underscore GROUP, so the Microsoft support operations switch stays OFF. On PharmaTrace, replace the action groups with the permission change group and the schema object access group, and point the database audit at the compliance account. Auditors query the older function, sys dot fn underscore get underscore audit underscore file, with a URL that ends in PharmaTrace slash star dot xel, a wildcard, and then filter with a WHERE clause on event underscore time between two values."

**Piece 10, option c.** "Option c. Enable server-level auditing to the Log Analytics workspace pt dash law only. Set the AzureDiagnostics table to ninety days analytics retention and two thousand five hundred fifty-six days total retention, arguing that Log Analytics is append-only, so the workspace is the WORM archive. Microsoft support operations are said to be already included in server auditing. Add the permission change group and the schema object access group to the server policy, so that PharmaTrace records them. Auditors query the two-hour window with KQL, AzureDiagnostics where TimeGenerated between two values."

**Piece 11, option d.** "Option d. The same server-level auditing as option a, storage ptaudit plus the workspace, but with Retention in days set to zero on the SQL side, so that only the immutable policy governs deletion. On the container, a time-based retention policy of two thousand five hundred fifty-five days with Allow protected append writes set to Append blobs, left unlocked, so that compliance can adjust the period later. AzureDiagnostics at ninety and seven hundred thirty days, the KQL alert, the support operations switch ON, and the same PharmaTrace database-level audit as option a. Auditors query with the older function, sys dot fn underscore get underscore audit underscore file, with the same master prefix, DEFAULT, DEFAULT, and then a WHERE clause on event underscore time greater than or equal to eight in the morning and less than ten in the morning."

## 3. Setup script (reference only; do not read verbatim unless asked)

Option a, auditor query:

```sql
SELECT event_time, server_principal_name, database_name, statement
FROM sys.fn_get_audit_file_v2(
    'https://ptaudit.blob.core.windows.net/sqldbauditlogs/pharmatrace-sql/master/SqlDbAuditing_ServerAudit/',
    DEFAULT, DEFAULT, '2026-09-04T08:00:00Z', '2026-09-04T10:00:00Z');
```

Option a, PharmaTrace database-level audit (PowerShell):

```powershell
Set-AzSqlDatabaseAudit -AuditActionGroup "DATABASE_OBJECT_PERMISSION_CHANGE_GROUP","SCHEMA_OBJECT_ACCESS_GROUP" ...
```

Option a, KQL alert:

```kusto
AzureDiagnostics | where Category == "SQLSecurityAuditEvents" and action_name_s == "BATCH COMPLETED"
```

Option b, auditor query:

```sql
SELECT ... FROM sys.fn_get_audit_file('https://ptaudit.blob.core.windows.net/sqldbauditlogs/pharmatrace-sql/PharmaTrace/*.xel', DEFAULT, DEFAULT)
WHERE event_time BETWEEN ... AND ...;
```

Option d, auditor query:

```sql
SELECT event_time, server_principal_name, database_name, statement
FROM sys.fn_get_audit_file(
    'https://ptaudit.blob.core.windows.net/sqldbauditlogs/pharmatrace-sql/master/SqlDbAuditing_ServerAudit/',
    DEFAULT, DEFAULT)
WHERE event_time >= '2026-09-04T08:00:00' AND event_time < '2026-09-04T10:00:00';
```

Key settings per option:

| Setting | a | b | c | d |
|---|---|---|---|---|
| Audit scope | Server | Database only (two DBs) | Server | Server |
| Destinations | Storage + Log Analytics | Storage + Log Analytics | Log Analytics only | Storage + Log Analytics |
| SQL Retention (Days) | 2,600 | 0 | n/a | 0 |
| Storage immutability | Time-based 2,555 days, append blobs, locked | Legal hold | none | Time-based 2,555 days, append blobs, unlocked |
| Support operations switch | ON | OFF | not set | ON |
| PharmaTrace extra groups | Database-level policy, separate account | Replaces the database's groups | Added to the server policy | Database-level policy, separate account |
| Auditor read | fn_get_audit_file_v2 with start and end time | fn_get_audit_file with wildcard | KQL | fn_get_audit_file with WHERE |

## 4. The question (ask exactly this)

"Which configuration meets all five requirements? Option a, option b, option c, or option d?"

If the learner wants a reminder, re-read any option piece from section 2.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

- Server policy covers existing and future databases. Storage 2,555-day time-based policy, append blobs, locked, is the WORM state: it cannot be deleted or shortened, only extended. SQL Retention 2,600 days is longer than the storage policy, as required; SQL retention 0 with an immutability policy is unsupported. Log Analytics 90 days analytics and 730 days total gives the KQL alerting and two-year cheap tier. The Microsoft support operations switch is a separate server setting. A database-level policy on PharmaTrace runs side by side with the server policy, with its own groups and its own account. fn underscore get underscore audit underscore file underscore v2 with start and end times filters at file level, so only files in the window are opened.
- **b is wrong:** database-only auditing misses future databases; replacing PharmaTrace's groups drops the default groups; retention 0 plus an immutability policy is unsupported; a legal hold never expires and can be cleared by anyone with the permission, so it is not a seven-year clock; support operations need the dedicated switch; the blob URL form of fn underscore get underscore audit underscore file does not accept a wildcard.
- **c is wrong:** Log Analytics is not WORM, a workspace contributor can shorten retention, delete tables or purge records; server auditing does not include support operations; adding the two groups to the server policy applies them to every database; and there are no blob files for T-SQL to read.
- **d is wrong:** SQL retention 0 with a storage immutability policy is unsupported; an unlocked time-based policy can be shortened or deleted; and fn underscore get underscore audit underscore file with a WHERE clause opens every blob under the prefix before filtering rows.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with requirement one and the word future. Which audit scope, server or database, automatically covers a database that does not exist yet?"
2. "Now think about WORM. Which Azure service offers an immutability policy that a locked setting makes extend-only and undeletable? Is that a SQL setting, a storage setting, or a Log Analytics setting?"
3. "There is a documented rule linking the storage policy interval and the SQL Retention in days value. One must be shorter than the other, and one particular value on the SQL side is not supported at all. Which value?"
4. "Requirement three is about Microsoft support engineers. Is that part of the normal action groups, or is it its own switch on the server's Auditing page?"
5. "Requirement five says without scanning every file. Two of the options use the original fn underscore get underscore audit underscore file. One uses a newer function with two extra arguments. What do those arguments do?"
6. "You should now be between two options that look alike. Compare their retention values and whether the storage policy is locked or unlocked."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "b, each database gets its own folder, that is cleaner" | Thinks database-level policies are enough | "Which databases exist next year that do not exist today? Who audits them?" |
| "A legal hold is the strongest protection, so b" | Confuses legal hold with time-based retention | "Does a legal hold have an expiry? And who can clear it?" |
| "c, Log Analytics keeps data for up to twelve years" | Confuses long retention with immutability | "Can a workspace contributor shorten a table's retention or purge records? Is that WORM?" |
| "c, the server policy already captures support engineers" | Does not know the separate switch | "Look at the server's Auditing page in the portal. Is there a second toggle below the main one?" |
| "d, retention zero means unlimited, that is safest" | Does not know the unsupported combination | "Retention zero is unlimited on its own. But what does the documentation say when a storage immutability policy is also present?" |
| "d, unlocked lets compliance adjust it later, that matches extendable" | Confuses extendable with editable | "A locked policy can still be extended. What is the difference between extendable and shortenable?" |
| "The WHERE clause on event underscore time filters the window, so d is fine" | Row filter versus file filter | "The WHERE filters rows after the files are read. Which function filters the files themselves?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the four surfaces the five requirements map onto:

- **Scope.** A server auditing policy applies to all existing and newly created databases on the server. A database policy exists side by side with it, with its own action groups and its own destination, which is exactly how PharmaTrace gets its extra groups into a separate account without touching the other databases. The portal only exposes the default groups; non-default groups are set with Set dash AzSqlDatabaseAudit and the AuditActionGroup parameter.
- **Destinations.** Storage account, Log Analytics workspace and Event Hubs, in any combination. Storage holds append blobs in dot xel format under the container sqldbauditlogs. Log Analytics receives the SQLSecurityAuditEvents category into the AzureDiagnostics table. Storage is the archive; Log Analytics is the operational monitor. Never stretch one destination to do both jobs.
- **WORM.** Immutability is a storage feature, not a SQL setting. A time-based retention policy on the container, with Allow protected append writes set to Append blobs, and then locked. A locked policy cannot be deleted and cannot be shortened, only extended. An unlocked policy can be shortened or deleted. A legal hold has no expiry and is cleared by removing the tag. The storage interval must be shorter than the SQL retention, and SQL retention zero with a storage policy is unsupported. Hence 2,555 days on storage and 2,600 days on SQL.
- **Log Analytics retention.** Tables default to thirty days. Analytics retention can go to two years, total retention to twelve years, the tail being cheap long-term retention reached through search jobs. Ninety and seven hundred thirty days is a valid table setting. This is not WORM: retention is editable and data can be purged.
- **Support operations.** A separate server switch, Enable Auditing of Microsoft support operations. It appears in Log Analytics as Category DevOpsOperationsAudit.
- **Reading blobs with T-SQL.** sys dot fn underscore get underscore audit underscore file underscore v2 adds two arguments, a start time and an end time, and filters at file level first, then at record level. Its blob URL argument is a prefix, not a wildcard. After the July 2025 change, all server audit logs sit in one folder called master, so the prefix is server, slash master, slash SqlDbAuditing underscore ServerAudit.

Memory hook: "Server policy for every database. Storage for WORM, locked, shorter than SQL retention, never zero. Log Analytics for KQL. Support engineers have their own switch. Version 2 of the function filters files by time."

## 9. Follow-up oral questions (optional)

1. "If the storage container had a locked policy of 2,555 days and SQL Retention in days were set to 2,000, what would be wrong?" (The storage interval must be shorter than the SQL retention; here it is longer, which is unsupported.)
2. "Which Log Analytics category holds Microsoft support operations?" (DevOpsOperationsAudit.)
3. "Which feature gives cryptographic proof that the data in a table was not tampered with, as opposed to who did what?" (Ledger, not auditing.)

## 10. References

- Auditing for Azure SQL Database and Azure Synapse Analytics: https://learn.microsoft.com/en-us/azure/azure-sql/database/auditing-overview
- Set up auditing for Azure SQL Database: https://learn.microsoft.com/en-us/azure/azure-sql/database/auditing-setup
- Immutable storage for Azure Blob Storage: https://learn.microsoft.com/en-us/azure/storage/blobs/immutable-storage-overview
- Time-based retention policies: https://learn.microsoft.com/en-us/azure/storage/blobs/immutable-time-based-retention-policy-overview
- Legal holds: https://learn.microsoft.com/en-us/azure/storage/blobs/immutable-legal-hold-overview
- sys.fn_get_audit_file_v2: https://learn.microsoft.com/en-us/sql/relational-databases/system-functions/sys-fn-get-audit-file-v2-transact-sql
- Configure data retention in a Log Analytics workspace: https://learn.microsoft.com/en-us/azure/azure-monitor/logs/data-retention-configure
- Set-AzSqlDatabaseAudit: https://learn.microsoft.com/en-us/powershell/module/az.sql/set-azsqldatabaseaudit
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
