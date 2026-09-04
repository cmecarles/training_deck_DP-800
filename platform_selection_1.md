# SQL Server question — Platform Selection 1

## Statement

Tallgrass Grain Partners is moving four SQL Server-based workloads to Microsoft's cloud platforms. The company already owns a **Microsoft Fabric F64 capacity** used by its analytics team, and an Azure subscription. Each workload has its own hard requirements:

| Workload | Requirements |
|---|---|
| `GrainLedger` + `GrainRef` | Two on-premises SQL Server 2019 databases on one instance. The application runs **38 SQL Server Agent jobs**, joins the two databases with **three-part names** (`GrainRef.dbo.Commodity`), calls a **CLR assembly** for a legacy checksum, queries a partner's SQL Server through a **linked server**, and uses **Service Broker** queues. It must move with **no application code changes** to a **fully managed** service (no OS patching, automatic backups). |
| `SiloSense` | A vendor telemetry product certified only on **SQL Server 2022 at a vendor-approved cumulative update**. It stores sensor images with **FILESTREAM**, and a vendor Windows service installed **on the same machine** reads those files through the local file system. The vendor dictates **when patches are applied**. |
| `AgroAdvisor` | A new AI assistant the analytics team is building **inside its Fabric workspace**: T-SQL vector search and RAG, data must be queryable by Spark notebooks and Power BI **without any ETL**, the database must be an item in the workspace so it is **versioned with Fabric git integration and promoted with Fabric deployment pipelines**, **billed against the existing F64 capacity** with no Azure resource to provision, and accessed only with **Microsoft Entra** identities. |
| `MarketDesk` | An **existing Azure SQL Database** (General Purpose, 8 vCores) used by a packaged trading application that authenticates with **SQL logins**, is reached through a **private endpoint**, and is protected by **active geo-replication**. The database **must stay exactly where it is**. Analysts need a **near-real-time Delta copy in OneLake** for Power BI and Spark, with **no ETL pipeline and no additional compute charge** for the replication. |

Which platform should host each workload?

### a.

| Workload | Platform |
|---|---|
| `GrainLedger` + `GrainRef` | **Azure SQL Managed Instance** |
| `SiloSense` | **SQL Server 2022 on an Azure Virtual Machine**, registered with the SQL IaaS Agent extension |
| `AgroAdvisor` | **SQL database in Microsoft Fabric** |
| `MarketDesk` | Keep the Azure SQL Database; add **Fabric mirroring** of the database into OneLake |

### b.

| Workload | Platform |
|---|---|
| `GrainLedger` + `GrainRef` | **Azure SQL Database (Hyperscale)**, one database per source database, cross-database joins rewritten as **elastic query** external tables and the Agent jobs recreated as **elastic jobs** |
| `SiloSense` | SQL Server 2022 on an Azure Virtual Machine |
| `AgroAdvisor` | SQL database in Microsoft Fabric |
| `MarketDesk` | Keep the Azure SQL Database; add Fabric mirroring |

### c.

| Workload | Platform |
|---|---|
| `GrainLedger` + `GrainRef` | Azure SQL Managed Instance |
| `SiloSense` | **Azure SQL Managed Instance** with the *SQL Server 2022* update policy, so the engine version matches the vendor certification |
| `AgroAdvisor` | SQL database in Microsoft Fabric |
| `MarketDesk` | Keep the Azure SQL Database; add Fabric mirroring |

### d.

| Workload | Platform |
|---|---|
| `GrainLedger` + `GrainRef` | Azure SQL Managed Instance |
| `SiloSense` | SQL Server 2022 on an Azure Virtual Machine |
| `AgroAdvisor` | **Azure SQL Database** in the Azure subscription, with Fabric mirroring so the data lands in OneLake |
| `MarketDesk` | **Migrate the database into a SQL database in Fabric**, which mirrors itself to OneLake automatically |

## Correct Answer

**a**

## Explanation

The question is a four-way constraint match across the platforms named in the DP-800 audience profile: **Azure SQL Database**, **Azure SQL Managed Instance**, **SQL Server on Azure Virtual Machines**, **SQL database in Microsoft Fabric**, and the related **Fabric mirroring** of an Azure SQL Database. Every decisive fact below comes from the Azure SQL "What is Azure SQL?" overview, the Azure SQL Database vs Managed Instance **feature comparison**, the Fabric SQL database **limitations** table, and the Fabric **mirrored databases from Azure SQL Database** article.

### Why option a is correct

- **`GrainLedger` → Azure SQL Managed Instance.** Every one of the five features listed is an *instance-scoped* capability that Azure SQL Database lacks and Managed Instance has. The feature comparison reads, for Azure SQL Database vs Managed Instance: **SQL Server Agent** "No, see Elastic jobs" vs "Yes"; **Cross-database/three-part name queries** "No, see Elastic queries" vs "Yes"; **Linked servers** "No" vs "Yes. Only to SQL Server and SQL Database"; **Common language runtime - CLR** "No" vs "Yes, but without access to file system"; **Service Broker** "No" vs "Yes". The overview sums it up: Managed Instance offers "almost 100% feature parity with the SQL Server database engine", is "best for most migrations to the cloud", and "supports database migration from on-premises with minimal to no database changes" — while remaining PaaS with "built-in backups, patching, recovery".
- **`SiloSense` → SQL Server on an Azure VM.** Three requirements are impossible in any PaaS option: **FILESTREAM** is "No" for *both* Azure SQL Database and Managed Instance; a vendor service "on the same machine" needs a host you control ("you have the ability to deploy application or services on the host where SQL Server is placed"); and dictating the exact CU and patch timing needs OS-level control ("You can choose when to start maintenance activities including system updates... and you can fully customize the SQL Server database engine"). The overview names the case explicitly: "Best for migrations and applications that require OS-level access". Registering the VM with the **SQL IaaS Agent extension** adds portal management, automated backup and patching windows without giving up that control.
- **`AgroAdvisor` → SQL database in Fabric.** It "uses the same SQL Database Engine as Azure SQL Database" (so the vector type and AI functions are there), is "the home in Fabric for OLTP workloads", and creating one "creates these items in your Fabric workspace": the database plus a **SQL analytics endpoint**, with data "automatically replicated into the OneLake and converted to Parquet" — no ETL. It is a Fabric item, so it is "integrated with Fabric continuous integration/continuous development" (git integration and deployment pipelines), it is billed through the **Fabric capacity** ("A single Fabric capacity can provide resources for Fabric SQL databases in different workspaces"), and it relies on **Microsoft Entra authentication** only. Nothing is provisioned in the Azure subscription.
- **`MarketDesk` → keep Azure SQL Database, add Fabric mirroring.** Mirroring "provides an easy experience to avoid complex ETL" and lets you "continuously replicate your existing Azure SQL Databases directly into Fabric's OneLake", exposing a read-only SQL analytics endpoint over the Delta tables. The pricing is what the requirement asks for: "Fabric compute used to replicate your data into Fabric OneLake is free. Storage in OneLake is free of cost based the capacity size." The source database is untouched — same logical server, SQL logins, private endpoint and geo-replication — and all vCore service tiers are supported as a source.

### Why option b is wrong

This is the subtle distractor because each substitution *exists*: elastic query really is the Azure SQL Database answer to cross-database joins, and elastic jobs really are its answer to Agent jobs. But the requirement was **no application code changes**, and both are rewrites — external tables and a shard-map or RDBMS data source instead of `GrainRef.dbo.Commodity`, and job steps re-authored as elastic jobs. Two requirements have no workaround at all: the feature comparison marks **CLR** "No" for Azure SQL Database (the overview repeats "CLR isn't supported with SQL Database, but is supported in SQL Managed Instance"), and **Linked servers** and **Service Broker** are "No". Hyperscale changes the storage architecture, not the feature surface. The three other rows are correct, which is why the option looks plausible.

### Why option c is wrong

Managed Instance's *SQL Server 2022 update policy* does pin the engine's feature set to SQL Server 2022, but that does not satisfy `SiloSense`: FILESTREAM is "No" in Managed Instance ("see SQL managed instances features"), there is no host to install the vendor's Windows service on (**File system access**: "No"), and Microsoft, not the vendor, applies patches — Managed Instance offers only "configurable maintenance windows", not vendor-chosen CU levels. A certification that names an exact cumulative update and a co-located service is the textbook IaaS case.

### Why option d is wrong

Two rows are swapped, and both break a stated requirement:

- `AgroAdvisor` on Azure SQL Database + mirroring would put the data in OneLake, but the database would be an **Azure** resource billed to the subscription, not a workspace item billed to the F64 capacity, and it could not be versioned with Fabric git integration or promoted with Fabric deployment pipelines (only Fabric items can). The overview is explicit that "a SQL database in Microsoft Fabric is distinct from an Azure SQL Database or a mirrored database from Azure SQL Database".
- Migrating `MarketDesk` into a SQL database in Fabric violates "must stay exactly where it is" and three concrete limitations of the Fabric flavour: "Logins are not supported. Only users representing Microsoft Entra principals are supported" (the packaged app uses SQL logins), **Active geo-replication** is "Not currently" available, and workspace-level private links "are not available in SQL database". Mirroring gives the analysts the same OneLake copy without touching the source.

Conceptual question (Azure / Fabric platform selection); not executed against an engine.

## DP-800 Exam Rule to Remember

```text
Instance-scoped SQL Server features  -> Azure SQL Managed Instance
  (SQL Agent, cross-DB three-part names, linked servers, CLR, Service Broker, DbMail,
   Resource Governor, native RESTORE FROM URL)        ~100% engine parity, still PaaS
OS / host control                    -> SQL Server on Azure VM (IaaS)
  (FILESTREAM, exact CU, patch timing, software on the box, 100% compatibility)
   + SQL IaaS Agent extension for portal management
Cloud-born single database           -> Azure SQL Database (GP / BC / Hyperscale, serverless, elastic pools)
  cross-DB only via elastic query; jobs via elastic jobs; no CLR / linked servers / Agent
Fabric-native OLTP                   -> SQL database in Fabric
  same engine as Azure SQL DB; auto-mirrored to OneLake + SQL analytics endpoint;
  Entra-only (no logins); billed to Fabric capacity; git integration + deployment pipelines;
  not available: ledger, Always Encrypted, in-memory, CDC, geo-replication, SQL Agent
Existing Azure SQL DB + analytics    -> Fabric MIRRORING (source untouched, replication compute free,
                                        OneLake storage free within capacity, read-only analytics endpoint)
```

Read each workload for its **one non-negotiable**: an instance feature, the operating system, the Fabric workspace, or "leave the database where it is". That single word usually decides the platform; every other requirement in the row is there to make the wrong platform look reasonable.
