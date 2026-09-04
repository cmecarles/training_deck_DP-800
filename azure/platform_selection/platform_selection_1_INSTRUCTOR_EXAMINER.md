# Instructor-Examiner guide — Platform Selection 1

Companion to [platform_selection_1.md](platform_selection_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.
7. **This is a multiple-choice question with no code.** It is a four-workload matching exercise. Read all four options, pieces 6 to 9, before taking an answer. Take one letter as the answer. If the learner prefers, let them first say a platform for each workload, then map that to a letter, but the final answer is one letter.
8. There are four workloads and each has one non-negotiable requirement. Make sure every requirement in pieces 2 to 5 has been heard before asking.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Integrate SQL solutions with Azure services.
- Task bullet: Evaluate and select the appropriate SQL platform: Azure SQL Database, Azure SQL Managed Instance, SQL Server on Azure Virtual Machines, SQL database in Microsoft Fabric.
- What is tested: matching hard requirements to the platform that satisfies them, including instance-scoped features, OS-level control, Fabric-native items, and Fabric mirroring of an existing Azure SQL Database.

## 2. Scenario to read aloud

**Piece 1, the story.** "Tallgrass Grain Partners is moving four SQL Server-based workloads to Microsoft's cloud platforms. The company already owns a Microsoft Fabric F64 capacity used by its analytics team, and it has an Azure subscription. Each workload has its own hard requirements. I will describe the four workloads one at a time."

**Piece 2, GrainLedger and GrainRef.** "The first workload is a pair: GrainLedger and GrainRef. They are two on-premises SQL Server 2019 databases on one instance. The application runs thirty-eight SQL Server Agent jobs. It joins the two databases with three-part names, for example GrainRef dot dbo dot Commodity. It calls a CLR assembly for a legacy checksum. It queries a partner's SQL Server through a linked server. And it uses Service Broker queues. It must move with no application code changes, to a fully managed service, meaning no OS patching and automatic backups."

**Piece 3, SiloSense.** "The second workload is SiloSense, a vendor telemetry product. It is certified only on SQL Server 2022 at a vendor-approved cumulative update. It stores sensor images with FILESTREAM, and a vendor Windows service installed on the same machine reads those files through the local file system. The vendor dictates when patches are applied."

**Piece 4, AgroAdvisor.** "The third workload is AgroAdvisor, a new AI assistant the analytics team is building inside its Fabric workspace. It needs T-SQL vector search and RAG. The data must be queryable by Spark notebooks and Power BI without any ETL. The database must be an item in the workspace, so it is versioned with Fabric git integration and promoted with Fabric deployment pipelines. It must be billed against the existing F64 capacity, with no Azure resource to provision. And it is accessed only with Microsoft Entra identities."

**Piece 5, MarketDesk.** "The fourth workload is MarketDesk, an existing Azure SQL Database, General Purpose, eight vCores. A packaged trading application uses it, authenticates with SQL logins, reaches it through a private endpoint, and it is protected by active geo-replication. The database must stay exactly where it is. But analysts need a near-real-time Delta copy in OneLake for Power BI and Spark, with no ETL pipeline and no additional compute charge for the replication."

**Piece 6, option a.** "Option a. GrainLedger and GrainRef go to Azure SQL Managed Instance. SiloSense goes to SQL Server 2022 on an Azure Virtual Machine, registered with the SQL IaaS Agent extension. AgroAdvisor goes to a SQL database in Microsoft Fabric. MarketDesk keeps its Azure SQL Database and adds Fabric mirroring of the database into OneLake."

**Piece 7, option b.** "Option b. GrainLedger and GrainRef go to Azure SQL Database Hyperscale, one database per source database, with the cross-database joins rewritten as elastic query external tables and the Agent jobs recreated as elastic jobs. SiloSense goes to SQL Server 2022 on an Azure Virtual Machine. AgroAdvisor goes to a SQL database in Fabric. MarketDesk keeps its Azure SQL Database and adds Fabric mirroring."

**Piece 8, option c.** "Option c. GrainLedger and GrainRef go to Azure SQL Managed Instance. SiloSense goes to Azure SQL Managed Instance with the SQL Server 2022 update policy, so the engine version matches the vendor certification. AgroAdvisor goes to a SQL database in Fabric. MarketDesk keeps its Azure SQL Database and adds Fabric mirroring."

**Piece 9, option d.** "Option d. GrainLedger and GrainRef go to Azure SQL Managed Instance. SiloSense goes to SQL Server 2022 on an Azure Virtual Machine. AgroAdvisor goes to an Azure SQL Database in the Azure subscription, with Fabric mirroring so the data lands in OneLake. MarketDesk is migrated into a SQL database in Fabric, which mirrors itself to OneLake automatically."

## 3. Setup script (reference only; do not read verbatim unless asked)

There is no code in this question. The requirements table, verbatim:

| Workload | Requirements |
|---|---|
| `GrainLedger` + `GrainRef` | Two on-premises SQL Server 2019 databases on one instance. The application runs 38 SQL Server Agent jobs, joins the two databases with three-part names (`GrainRef.dbo.Commodity`), calls a CLR assembly for a legacy checksum, queries a partner's SQL Server through a linked server, and uses Service Broker queues. It must move with no application code changes to a fully managed service (no OS patching, automatic backups). |
| `SiloSense` | A vendor telemetry product certified only on SQL Server 2022 at a vendor-approved cumulative update. It stores sensor images with FILESTREAM, and a vendor Windows service installed on the same machine reads those files through the local file system. The vendor dictates when patches are applied. |
| `AgroAdvisor` | A new AI assistant the analytics team is building inside its Fabric workspace: T-SQL vector search and RAG, data must be queryable by Spark notebooks and Power BI without any ETL, the database must be an item in the workspace so it is versioned with Fabric git integration and promoted with Fabric deployment pipelines, billed against the existing F64 capacity with no Azure resource to provision, and accessed only with Microsoft Entra identities. |
| `MarketDesk` | An existing Azure SQL Database (General Purpose, 8 vCores) used by a packaged trading application that authenticates with SQL logins, is reached through a private endpoint, and is protected by active geo-replication. The database must stay exactly where it is. Analysts need a near-real-time Delta copy in OneLake for Power BI and Spark, with no ETL pipeline and no additional compute charge for the replication. |

## 4. The question (ask exactly this)

"Which platform should host each workload? Option a, b, c or d?"

- **a.** GrainLedger + GrainRef: **Azure SQL Managed Instance**. SiloSense: **SQL Server 2022 on an Azure Virtual Machine**, registered with the SQL IaaS Agent extension. AgroAdvisor: **SQL database in Microsoft Fabric**. MarketDesk: keep the Azure SQL Database; add **Fabric mirroring** of the database into OneLake.
- **b.** GrainLedger + GrainRef: **Azure SQL Database (Hyperscale)**, one database per source database, cross-database joins rewritten as **elastic query** external tables and the Agent jobs recreated as **elastic jobs**. SiloSense: SQL Server 2022 on an Azure Virtual Machine. AgroAdvisor: SQL database in Microsoft Fabric. MarketDesk: keep the Azure SQL Database; add Fabric mirroring.
- **c.** GrainLedger + GrainRef: Azure SQL Managed Instance. SiloSense: **Azure SQL Managed Instance** with the *SQL Server 2022* update policy, so the engine version matches the vendor certification. AgroAdvisor: SQL database in Microsoft Fabric. MarketDesk: keep the Azure SQL Database; add Fabric mirroring.
- **d.** GrainLedger + GrainRef: Azure SQL Managed Instance. SiloSense: SQL Server 2022 on an Azure Virtual Machine. AgroAdvisor: **Azure SQL Database** in the Azure subscription, with Fabric mirroring so the data lands in OneLake. MarketDesk: **migrate the database into a SQL database in Fabric**, which mirrors itself to OneLake automatically.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

| Workload | Platform | Decisive requirement |
|---|---|---|
| GrainLedger + GrainRef | Azure SQL Managed Instance | SQL Agent, three-part names, linked servers, CLR and Service Broker are instance-scoped features; Managed Instance has them, Azure SQL Database does not. No code changes, still PaaS. |
| SiloSense | SQL Server 2022 on an Azure VM, with the SQL IaaS Agent extension | FILESTREAM is not in any PaaS option. A co-located vendor service and vendor-controlled patching need OS-level control. |
| AgroAdvisor | SQL database in Fabric | Must be a workspace item: git integration, deployment pipelines, billed to the F64 capacity, Entra-only, auto-mirrored to OneLake with a SQL analytics endpoint. |
| MarketDesk | Keep Azure SQL Database, add Fabric mirroring | Source stays untouched, SQL logins, private endpoint and geo-replication remain. Mirroring replication compute is free; OneLake storage is free within the capacity. |

Why the wrong options are wrong:

- **b.** Elastic query and elastic jobs exist, but both are rewrites, and the requirement was no application code changes. And there is no workaround at all for CLR, linked servers and Service Broker in Azure SQL Database. Hyperscale changes storage, not the feature surface.
- **c.** The SQL Server 2022 update policy pins the feature set but does not give FILESTREAM, which is No in Managed Instance; there is no host for the vendor's Windows service; and Microsoft applies patches, with only configurable maintenance windows, not vendor-chosen CU levels.
- **d.** AgroAdvisor on Azure SQL Database would be an Azure resource billed to the subscription, not a workspace item, and could not be versioned with Fabric git or promoted with deployment pipelines. Migrating MarketDesk breaks "stay exactly where it is", and SQL database in Fabric does not support SQL logins, active geo-replication, or private links.

## 6. Hint ladder (one hint per attempt, in order)

1. "Each workload has one word that decides it. For the first, it is a list of instance features. For the second, it is the operating system. For the third, it is the Fabric workspace. For the fourth, it is 'stay where it is'. Find that word in each row before choosing."
2. "For GrainLedger: SQL Agent, three-part names, linked servers, CLR and Service Broker. Which of the PaaS platforms says Yes to all five in the feature comparison, and which says No, see elastic jobs, No, see elastic query?"
3. "For SiloSense: FILESTREAM, a Windows service on the same machine, and the vendor choosing the patch date. Is there any platform where Microsoft is not the one patching the box and you can install your own software next to SQL Server?"
4. "For AgroAdvisor: git integration, deployment pipelines and billing to the F64 capacity only apply to things that are items in a Fabric workspace. Is an Azure SQL Database a Fabric item?"
5. "For MarketDesk: the database must not move, the app uses SQL logins, a private endpoint and geo-replication. Which Fabric feature copies an Azure SQL Database into OneLake without touching the source, and what does it charge for replication compute?"
6. "Two options remain after removing the one that rewrites GrainLedger and the one that swaps the last two rows. They differ only on SiloSense. Does Managed Instance support FILESTREAM, and can you install a vendor service on its host?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "b, Hyperscale is the most modern and elastic query handles the joins" | Treats a rewrite as no code change | "The requirement says no application code changes. Is an external table the same object as GrainRef dot dbo dot Commodity? And what about CLR and Service Broker?" |
| "c, the 2022 update policy matches the vendor's certified version" | Thinks version parity equals full feature parity | "Look at the FILESTREAM row for Managed Instance in the feature comparison. And where would the vendor's Windows service be installed?" |
| "d, an Azure SQL Database with mirroring gives the same OneLake data" | Ignores the workspace-item requirements | "The data lands in OneLake, yes. But can an Azure resource be versioned with Fabric git integration or billed to the F64 capacity?" |
| "d, move MarketDesk into Fabric so it mirrors itself" | Ignores 'stay where it is' and the Fabric limitations | "Three things the trading app relies on: SQL logins, a private endpoint, geo-replication. Does SQL database in Fabric offer any of them?" |
| "GrainLedger should be a VM, it is the safest for compatibility" | Forgets fully managed | "The requirement asks for no OS patching and automatic backups. Is a VM fully managed?" |
| "AgroAdvisor could be Managed Instance, it has vector search" | Misses the Fabric workspace and capacity requirements | "Where must the database live, and what capacity must it be billed to?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the platform ladder:

- **Instance-scoped SQL Server features point to Azure SQL Managed Instance.** The feature comparison for Azure SQL Database versus Managed Instance reads: SQL Server Agent, No, see elastic jobs, versus Yes. Cross-database three-part name queries, No, see elastic queries, versus Yes. Linked servers, No, versus Yes, to SQL Server and SQL Database only. CLR, No, versus Yes, without file system access. Service Broker, No, versus Yes. Managed Instance has almost one hundred percent engine parity, is best for most migrations with minimal to no database changes, and is still PaaS with built-in backups, patching and recovery. That is GrainLedger.
- **OS or host control points to SQL Server on an Azure VM.** FILESTREAM is No for both Azure SQL Database and Managed Instance. A vendor service on the same machine needs a host you control. Choosing the exact cumulative update and the patch date needs OS-level access. The SQL IaaS Agent extension adds portal management, automated backup and patching windows without giving up that control. That is SiloSense.
- **Fabric-native OLTP points to SQL database in Fabric.** It uses the same engine as Azure SQL Database, so the vector type and AI functions are there. Creating one creates workspace items: the database plus a SQL analytics endpoint, with data automatically replicated into OneLake as Parquet, no ETL. It is a Fabric item, so it takes part in git integration and deployment pipelines, is billed through the Fabric capacity, and uses Microsoft Entra authentication only. Nothing is provisioned in Azure. That is AgroAdvisor.
- **An existing Azure SQL Database plus analytics points to Fabric mirroring.** Mirroring continuously replicates the database into OneLake, exposes a read-only SQL analytics endpoint over Delta tables, and the compute used to replicate is free; OneLake storage is free within the capacity size. The source is untouched: same logical server, SQL logins, private endpoint, geo-replication. All vCore tiers are supported as a source. That is MarketDesk.
- **Why the distractors fail.** Option b turns a lift-and-shift into a rewrite and still cannot provide CLR, linked servers or Service Broker. Option c confuses engine version with host control; Managed Instance has no FILESTREAM, no file system access, and Microsoft patches it. Option d confuses "data in OneLake" with "item in the workspace", and forgets that SQL database in Fabric has no logins, no active geo-replication, and no private links.

Memory hook: "Instance features, Managed Instance. Touch the OS, a VM. Live in the workspace, SQL database in Fabric. Leave it where it is, mirror it."

## 9. Follow-up oral questions (optional)

1. "If GrainLedger had no Agent jobs, no CLR, no linked server and no Service Broker, and the two databases were merged into one, which platform would become reasonable?" (Azure SQL Database, since no instance-scoped feature remains.)
2. "What does registering a SQL Server VM with the SQL IaaS Agent extension give you?" (Portal management, automated backup, automated patching windows, license management, without losing OS control.)
3. "Name three things a SQL database in Fabric does not offer that an Azure SQL Database does." (Any three of: SQL logins, active geo-replication, private links, ledger, Always Encrypted, in-memory OLTP, CDC, SQL Agent.)

## 10. References

- What is Azure SQL? (platform overview): https://learn.microsoft.com/en-us/azure/azure-sql/azure-sql-iaas-vs-paas-what-is-overview
- Features comparison: Azure SQL Database and Azure SQL Managed Instance: https://learn.microsoft.com/en-us/azure/azure-sql/database/features-comparison
- SQL Server on Azure Virtual Machines overview: https://learn.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/sql-server-on-azure-vm-iaas-what-is-overview
- SQL Server IaaS Agent extension: https://learn.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/sql-server-iaas-agent-extension-automate-management
- SQL database in Microsoft Fabric overview: https://learn.microsoft.com/en-us/fabric/database/sql/overview
- Limitations in SQL database in Microsoft Fabric: https://learn.microsoft.com/en-us/fabric/database/sql/limitations
- Mirroring Azure SQL Database in Microsoft Fabric: https://learn.microsoft.com/en-us/fabric/database/mirrored-database/azure-sql-database
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
