# SQL Server question — MCP and Agents on Fabric SQL 1

## Statement

Canopy Garden Centres keeps its point-of-sale data in a **SQL database in Fabric** named `CanopySales` and its supplier feeds in a **lakehouse** named `CanopyLake`, both in the workspace `ws-dp800-canopy`. Two people will work with AI agents in Visual Studio Code:

- **Dev** (`dev@canopy.example`) is a workspace **Contributor** and must let GitHub Copilot **agent mode** create tables and load data in `CanopySales`, approving every action.
- **Ana** (`ana@canopy.example`) has **no workspace role**. Her agent may run **SELECT** statements against the lakehouse table `dbo.stock_levels` only, must be **unable** to read `dbo.supplier_costs`, must be unable to read the lakehouse files through Spark or the OneLake APIs, must never be able to write, and must have **no access whatsoever** to `CanopySales`.
- No secret may be stored in the repository, and the MCP server definition must be shared through the repository.

### Part 1 — Build the items

1. **Workspaces** > **New workspace** > `ws-dp800-canopy` > **Advanced** > your capacity > **Apply**. Add Dev as **Contributor** (**Manage access** > **Add people or groups**).
2. **New item** > **SQL database** > `CanopySales` > **Create**; run `CREATE TABLE dbo.Sale (SaleID int PRIMARY KEY, Sku varchar(20), Qty int);`.
3. **New item** > **Lakehouse** > `CanopyLake` > **Create**. Upload two CSV files under **Files** and use **Load to Tables** > **New table** to create `stock_levels` (`sku,store,on_hand`) and `supplier_costs` (`sku,supplier,unit_cost`) — for instance `S1,North,40` and `S1,GreenCo,3.20`.
4. Open the **SQL analytics endpoint** of `CanopyLake`; copy its item ID from the browser URL (the GUID after `/sqlanalyticsendpoints/` or in **Settings**), and the workspace ID (the GUID after `/groups/`).

### Part 2 — Tools on the client

5. Install in VS Code: **GitHub Copilot**, **GitHub Copilot Chat** (an active Copilot subscription is required) and the **MSSQL** extension.
6. MSSQL extension > **Connections** > **Add connection** > **Browse Fabric** > sign in with **Microsoft Entra ID - Universal with MFA support** > expand `ws-dp800-canopy` > select `CanopySales` > **Connect**. Try to select `CanopyLake` itself in the same tree and note which item types the browser offers.
7. Create `.vscode/mcp.json` in the repository with the server definition of your chosen option, then open Copilot Chat, switch to **Agent** mode, and check the **Tools** menu: identify which tools come from the MSSQL extension (`connect`, `list_databases`, `show_schema`, `list_tables`, `run_query`, ...) and which from the Fabric server (`executeSQL`).
8. As Dev, prompt: `Connect to CanopySales and create a table dbo.Promo with PromoID int and Pct decimal(4,1), then insert two rows`. Observe the confirmation dialog for each tool call and its choices (**Allow in this session**, **Allow in this workspace**, **Always allow**).
9. As Ana (a second machine or profile), prompt: `Use executeSQL to list the tables I can see, then show stock_levels for store North, then show supplier_costs`, and finally `insert a row into stock_levels`. Note which calls fail and where.

### Which configuration satisfies every constraint?

### a.

Commit this `.vscode/mcp.json` (item-scoped to the lakehouse's SQL analytics endpoint; no `inputs`, no credentials — the first tool call opens a Microsoft Entra sign-in):

```json
{
  "servers": {
    "canopy-lake": {
      "type": "http",
      "url": "https://api.fabric.microsoft.com/v1/mcp/dataPlane/workspaces/<workspace-id>/items/<sql-analytics-endpoint-item-id>/sqlEndpoint"
    }
  }
}
```

Share `CanopyLake` with Ana with **no additional permissions** (base *Read* only), then on the SQL analytics endpoint run `GRANT SELECT ON OBJECT::dbo.stock_levels TO [ana@canopy.example];` (Fabric creates the database user automatically). Dev keeps using the MSSQL extension connection from step 6: the agent-mode tools run under Dev's own Entra identity and Contributor rights, with the default per-call confirmation kept (never **Always allow** for `run_query`).

### b.

Same `mcp.json` as option a, but give Ana the **Viewer** role on `ws-dp800-canopy` instead of sharing the lakehouse, so that she gets *ReadData* on every SQL analytics endpoint automatically; then `DENY SELECT ON OBJECT::dbo.supplier_costs TO [ana@canopy.example];` on the `CanopyLake` endpoint.

### c.

Point both users at the SQL analytics endpoint through the MSSQL extension instead of the Fabric server, with a stdio MCP server whose connection string uses a SQL login stored as a `promptString` input with `"password": true`:

```json
{
  "inputs": [ { "id": "lake-pwd", "type": "promptString", "password": true, "description": "SQL login password" } ],
  "servers": {
    "canopy-lake": {
      "type": "stdio", "command": "node", "args": ["./tools/mssql-mcp/index.js"],
      "env": { "CONNECTION_STRING": "Server=<endpoint>.datawarehouse.fabric.microsoft.com;Database=CanopyLake;User Id=ana_reader;Password=${input:lake-pwd}" }
    }
  }
}
```

Share `CanopyLake` with Ana with **Read all with Apache Spark** (because it is a read-only permission) and grant `SELECT` on `dbo.stock_levels` to `ana_reader`.

### d.

Use the **global** endpoint `https://api.fabric.microsoft.com/v1/mcp/dataPlane/sqlEndpoint` in `mcp.json`, share `CanopyLake` with Ana with **Read all with SQL analytics endpoint**, and add `.github/copilot-instructions.md` stating `Never query dbo.supplier_costs and never run INSERT, UPDATE or DELETE`. Because the Fabric server exposes only `executeSQL`, and every call must be approved in the dialog, the instruction plus the approval step prevent the forbidden reads and writes.

**Capacity and cleanup.** Every `executeSQL`/`run_query` call consumes capacity like any query. When done: **Workspace settings** > **General** > **Remove this workspace** > **Delete**; remove the item share for Ana beforehand if you keep the workspace; pause a paid capacity.

## Correct Answer

**a**

## Explanation

The correct answer is **a**. The question separates three layers: **which client can reach which item**, **which Fabric permission grants what**, and **what actually stops a forbidden statement**.

| Constraint | a | b (Viewer role) | c (SQL login, ReadAll) | d (global endpoint + instructions) |
|---|---|---|---|---|
| Ana: `stock_levels` only, engine-enforced | satisfied | satisfied (DENY) | **violated** (login cannot exist) | **violated** (ReadData = all tables; prompt-level only) |
| Ana: no Spark/OneLake file access | satisfied | satisfied | **violated** (ReadAll) | satisfied |
| Ana: no access to `CanopySales` | satisfied | **violated** (Viewer reads every SQL database in the workspace) | satisfied | satisfied |
| No secrets, definition in repo | satisfied | satisfied | **violated** (SQL auth impossible; prompt only hides a secret that cannot work) | satisfied |
| Dev: agent-mode DDL/DML with approval | satisfied | satisfied | **violated** (MCP over the analytics endpoint is read-only; Dev needs the SQL database) | satisfied |

### What the lab shows

- **Step 6** — the MSSQL extension's Fabric browser supports "only SQL databases and SQL analytics endpoints"; the lakehouse itself, warehouses and other item types are not offered, and sign-in is Entra only ("you don't need connection strings or personal access tokens").
- **Step 7** — the **Fabric Data Warehouse MCP server** "applies to: SQL analytics endpoint and Warehouse", is configured as `"type": "http"` with either the global URL or the item-scoped URL `.../workspaces/<workspace-id>/items/<item-id>/sqlEndpoint`, and "exposes one tool", `executeSQL`; schema discovery is done by asking the agent to run `INFORMATION_SCHEMA` queries through it. The MSSQL extension contributes `connect`, `disconnect`, `change_database`, `get_connection_details`, `list_servers`, `list_databases`, `show_schema`, `list_schemas`, `list_tables`, `list_views`, `list_functions` and `run_query`; agent mode "picks up MSSQL extension tools automatically. No `@mssql` mention required."
- **Step 8** — "All actions use the same connection context and credentials as the MSSQL extension. Agent mode doesn't introduce another authentication or permission changes." Every tool call shows a confirmation dialog with **Allow in this session**, **Allow in this workspace**, **Always allow**; `run_query` can execute any statement Dev's identity may run, so DDL and inserts succeed after approval.
- **Step 9** — with option a, the `INFORMATION_SCHEMA` and `stock_levels` calls succeed, `supplier_costs` fails with a permission error from the engine, and the `INSERT` fails twice over: the analytics endpoint is read-only and Ana has no write permission. "The server doesn't bypass Fabric security" — "Users can only run SQL statements that their identity is allowed to execute".

### Why option a is correct

The item-scoped URL anchors the agent to the `CanopyLake` endpoint ("avoid repeatedly providing warehouse context in chat") and needs no secret because the server "uses the signed-in user's identity and respects Fabric permissions". Sharing with **no additional permissions** gives Ana only *Read* — "which only allows the recipient to *connect* to the SQL analytics endpoint, the equivalent of CONNECT permissions in SQL Server. The shared recipient won't be able to query any table or view ... unless they're provided access to objects ... using T-SQL GRANT"; that is precisely the documented recipe: "share the Warehouse with no additional permissions, then provide granular access to specific objects using T-SQL GRANT". `CREATE USER` is not run explicitly — "When you run `GRANT` or `DENY`, Fabric creates the database user automatically." Because Ana has no workspace role and no share on `CanopySales`, she cannot even connect to it (connecting requires at least *Read* on the item). Dev's side uses the only client that can reach a SQL database in Fabric from agent mode — the MSSQL extension — where least privilege is the Contributor role itself plus the per-call dialog; the Fabric MCP server cannot target a SQL database item at all.

### Why option b is wrong

Everything in b is technically executable, which makes it the subtle distractor. The **Viewer** role, however, is workspace-wide: for SQL databases it grants **Connect**, **Read data and metadata** (Read + ReadAll + ReadData) on every database, and for warehouses/endpoints "CONNECT and ReadData". Ana would be able to open and query `CanopySales`, violating the "no access whatsoever" constraint. The `DENY` is fine as far as `supplier_costs` goes — but the constraint that matters is scope, and sharing one item is the least-privilege tool for a person who needs one item.

### Why option c is wrong

"Microsoft Entra ID is the only identity provider" on the SQL analytics endpoint, the warehouse and the Fabric SQL database; a SQL login `ana_reader` cannot be created, so the connection string can never work, `promptString` or not. **Read all with Apache Spark** (ReadAll) is the permission that lets a recipient "find the ABFS path ... and use this path within a Spark Notebook to read this data" — exactly the OneLake/Spark access Ana must not have; ReadData and ReadAll "are separate permissions that do not overlap". And routing Dev's DDL through the analytics endpoint cannot work: it is read-only, and `CanopySales` is reachable only as a SQL database connection.

### Why option d is wrong

**Read all with SQL analytics endpoint** (ReadData) is "the equivalent of *db_datareader*": Ana reads `supplier_costs` too. An instructions file is natural-language context injected into requests; it "influence[s] the model" but registers nothing and enforces nothing, and the approval dialog can be broadened to a session, a workspace or **Always allow** with one click. The documentation's own guidance is the engine-side one: "Use least-privilege access for production warehouses" and "review generated T-SQL before allowing `executeSQL` to run, especially for write operations". The global endpoint is legitimate, but it makes the agent depend on chat-supplied item context, which is the opposite of pinning Ana to one item.

### Permission map used in this question

```text
Workspace role     Admin/Member/Contributor: CONTROL on every warehouse & SQL analytics endpoint; Write on SQL databases
                   Viewer: CONNECT + ReadData on every endpoint; Read/ReadAll/ReadData on every SQL database
Item share         Read                       -> connect only (CONNECT); nothing readable until GRANT
                   Read all with SQL analytics endpoint (ReadData) -> all tables via T-SQL (~ db_datareader)
                   Read all with Apache Spark  (ReadAll)  -> OneLake files via Spark/OneLake APIs
                   (lakehouse sharing never grants write; propagation can take up to 2 hours)
SQL granular       GRANT/DENY/REVOKE, roles, RLS, CLS, DDM; user auto-created by GRANT; DENY wins
Clients            MSSQL extension (Browse Fabric): SQL database + SQL analytics endpoint; tools connect ... run_query
                   Fabric DW MCP server (http): SQL analytics endpoint + Warehouse only; one tool executeSQL
```

Documentation relied upon (ms.date): Connect to Fabric Data Warehouse MCP server 2026-06-16; Quickstart: GitHub Copilot agent mode (MSSQL extension) 2026-06-01; How GitHub Copilot works with the MSSQL extension 2026-06-01; Fabric integration (MSSQL extension) 2026-03-13; Share your warehouse and manage permissions 2025-07-14; Lakehouse sharing and permission management 2026-03-01; Share items in Fabric 2025-04-06; Workspace roles in Fabric Data Warehouse 2025-06-26; Workspace roles and permissions in lakehouse 2025-09-22; SQL granular permissions 2026-06-25; Authorization in SQL database 2024-10-11; Authentication in SQL database 2024-11-20.

Hands-on question (Microsoft Fabric capacity required); behaviour is taken from the official documentation as of the ms.date cited above.

## DP-800 Exam Rule to Remember

```text
Agent -> Fabric SQL: two doors
  MSSQL extension agent-mode tools (connect, list_tables, show_schema, run_query ...): SQL database in Fabric
      and SQL analytics endpoints; same connection + permissions as the extension; every call approved
  Fabric Data Warehouse MCP server ("type": "http", .../mcp/dataPlane/sqlEndpoint or item-scoped URL):
      SQL analytics endpoint + Warehouse only; single tool executeSQL; signed-in Entra user; no secrets in mcp.json
Least privilege for a one-item reader: share with NO extra permission (Read = CONNECT) + GRANT SELECT on objects
  ReadData = all tables via SQL (db_datareader)      ReadAll = OneLake files via Spark
  Viewer role = every endpoint/database in the workspace (too broad for one item)
What CAN happen = Fabric item permission + SQL permission of the signed-in identity
What DOES happen = approved tool calls; dialogs and instruction files are not the boundary
```
