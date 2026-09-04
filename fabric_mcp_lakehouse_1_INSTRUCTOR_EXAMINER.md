# Instructor-Examiner guide — MCP and Agents on Fabric SQL 1

Companion to [fabric_mcp_lakehouse_1.md](fabric_mcp_lakehouse_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Lab rule.** This is a hands-on Microsoft Fabric and Visual Studio Code lab question. Before reading the scenario, ask: "Have you already run this lab from your own Fabric account and VS Code?" If yes, ask what they observed at each step (which item types the Browse Fabric tree offered, which tools appeared in the Tools menu and from which server, what the confirmation dialog offered, which of Ana's calls failed and with what message) before you quiz them. If no, walk through the steps in words using section 2, so that the question can still be answered from the documented facts alone. Do not require the learner to run anything during the call.

**Multiple choice.** This is a multiple-choice question with four options, a to d. Read all four options before taking an answer. The learner must pick one letter. Take the letter, then say only right or wrong.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked. For the JSON, name the keys and values that matter: type, url, command, args, env, inputs.

## 1. Exam skill covered

- Functional group: Configure and secure the database engine (15–20%); also Design and develop database solutions (35–40%).
- Skill: Configure access to Fabric SQL items and use AI-assisted development tools.
- Task bullet: Configure workspace roles, item sharing and T-SQL permissions for a lakehouse SQL analytics endpoint and a SQL database; connect agents through the MSSQL extension and the Fabric Data Warehouse MCP server.
- What is tested: which client can reach which Fabric item, what each sharing permission grants (Read, ReadData, ReadAll), why a Viewer role is too broad, why SQL logins cannot exist, and why prompt instructions and approval dialogs are not a security boundary.

## 2. Scenario to read aloud

**Piece 1, the story.** "Canopy Garden Centres keeps its point-of-sale data in a SQL database in Fabric named CanopySales, and its supplier feeds in a lakehouse named CanopyLake, both in the workspace ws dash dp800 dash canopy. Two people will work with AI agents in Visual Studio Code."

**Piece 2, the two people and the constraints.** "Dev, dev at canopy dot example, is a workspace Contributor and must let GitHub Copilot agent mode create tables and load data in CanopySales, approving every action. Ana, ana at canopy dot example, has no workspace role. Her agent may run SELECT against the lakehouse table dbo dot stock underscore levels only. She must be unable to read dbo dot supplier underscore costs, unable to read the lakehouse files through Spark or the OneLake APIs, never able to write, and she must have no access whatsoever to CanopySales. No secret may be stored in the repository, and the MCP server definition must be shared through the repository."

**Piece 3, building the items.** "You create the workspace and add Dev as Contributor through Manage access. You create the SQL database CanopySales and run CREATE TABLE dbo dot Sale with SaleID primary key, Sku varchar twenty and Qty int. You create the lakehouse CanopyLake, upload two CSV files under Files and use Load to Tables to create stock underscore levels, with columns sku, store and on underscore hand, for instance S1, North, 40; and supplier underscore costs, with columns sku, supplier and unit underscore cost, for instance S1, GreenCo, three point two zero. You open the SQL analytics endpoint of CanopyLake and copy its item ID from the browser URL, the GUID after sqlanalyticsendpoints, and the workspace ID, the GUID after groups."

**Piece 4, tools on the client.** "In VS Code you install GitHub Copilot, GitHub Copilot Chat, which needs an active subscription, and the MSSQL extension. In the MSSQL extension, Connections, Add connection, Browse Fabric, you sign in with Microsoft Entra ID Universal with MFA support, expand the workspace, select CanopySales and Connect. You also try to select CanopyLake itself in that tree and note which item types the browser offers."

**Piece 5, the MCP file and the tools menu.** "You create dot vscode slash mcp dot json in the repository with the server definition of your chosen option. Then you open Copilot Chat, switch to Agent mode, and check the Tools menu. You identify which tools come from the MSSQL extension, such as connect, list underscore databases, show underscore schema, list underscore tables and run underscore query, and which come from the Fabric server, namely executeSQL."

**Piece 6, the two test prompts.** "As Dev, you prompt: Connect to CanopySales and create a table dbo dot Promo with PromoID int and Pct decimal four one, then insert two rows. You observe the confirmation dialog for each tool call and its choices: Allow in this session, Allow in this workspace, Always allow. As Ana, on a second machine or profile, you prompt: Use executeSQL to list the tables I can see, then show stock underscore levels for store North, then show supplier underscore costs. Finally you prompt: insert a row into stock underscore levels. You note which calls fail and where."

**Piece 7, option a.** "Commit a dot vscode slash mcp dot json that has one server, canopy dash lake, of type http, whose url is api dot fabric dot microsoft dot com, v1, mcp, dataPlane, workspaces, the workspace id, items, the SQL analytics endpoint item id, sqlEndpoint. No inputs, no credentials; the first tool call opens a Microsoft Entra sign-in. Share CanopyLake with Ana with no additional permissions, that is base Read only. Then on the SQL analytics endpoint run GRANT SELECT ON OBJECT double colon dbo dot stock underscore levels TO Ana's account in square brackets; Fabric creates the database user automatically. Dev keeps using the MSSQL extension connection from the earlier step: the agent-mode tools run under Dev's own Entra identity and Contributor rights, with the default per-call confirmation kept, never Always allow for run underscore query."

**Piece 8, option b.** "Same mcp dot json as option a, but give Ana the Viewer role on the workspace instead of sharing the lakehouse, so she gets ReadData on every SQL analytics endpoint automatically. Then run DENY SELECT ON OBJECT dbo dot supplier underscore costs TO Ana on the CanopyLake endpoint."

**Piece 9, option c.** "Point both users at the SQL analytics endpoint through the MSSQL extension instead of the Fabric server, with a stdio MCP server. The mcp dot json has an inputs array with one promptString input, id lake dash pwd, password true, described as SQL login password. The server canopy dash lake is type stdio, command node, args pointing at tools slash mssql dash mcp slash index dot js, and an env with CONNECTION underscore STRING set to Server equals the endpoint on datawarehouse dot fabric dot microsoft dot com, Database CanopyLake, User Id ana underscore reader, Password from the input. Share CanopyLake with Ana with Read all with Apache Spark, because it is a read-only permission, and grant SELECT on dbo dot stock underscore levels to ana underscore reader."

**Piece 10, option d.** "Use the global endpoint, api dot fabric dot microsoft dot com, v1, mcp, dataPlane, sqlEndpoint, in mcp dot json. Share CanopyLake with Ana with Read all with SQL analytics endpoint, and add a file dot github slash copilot dash instructions dot md stating: Never query dbo dot supplier underscore costs and never run INSERT, UPDATE or DELETE. Because the Fabric server exposes only executeSQL, and every call must be approved in the dialog, the instruction plus the approval step prevent the forbidden reads and writes."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
-- CanopySales (SQL database in Fabric)
CREATE TABLE dbo.Sale (SaleID int PRIMARY KEY, Sku varchar(20), Qty int);

-- Option a, on the CanopyLake SQL analytics endpoint
GRANT SELECT ON OBJECT::dbo.stock_levels TO [ana@canopy.example];

-- Option b, on the CanopyLake SQL analytics endpoint
DENY SELECT ON OBJECT::dbo.supplier_costs TO [ana@canopy.example];
```

Lakehouse tables loaded from CSV: `stock_levels` (`sku,store,on_hand`, e.g. `S1,North,40`) and `supplier_costs` (`sku,supplier,unit_cost`, e.g. `S1,GreenCo,3.20`).

Option a and b `mcp.json`:

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

Option c `mcp.json`:

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

Option d URL: `https://api.fabric.microsoft.com/v1/mcp/dataPlane/sqlEndpoint`, plus `.github/copilot-instructions.md` containing `Never query dbo.supplier_costs and never run INSERT, UPDATE or DELETE`.

Prompts: Dev, `Connect to CanopySales and create a table dbo.Promo with PromoID int and Pct decimal(4,1), then insert two rows`. Ana, `Use executeSQL to list the tables I can see, then show stock_levels for store North, then show supplier_costs`, then `insert a row into stock_levels`.

## 4. The question (ask exactly this)

"Which configuration satisfies every constraint? Option a, b, c, or d?"

- **a.** Commit the item-scoped http mcp.json (url ending in workspaces/<workspace-id>/items/<sql-analytics-endpoint-item-id>/sqlEndpoint; no inputs, no credentials; the first tool call opens a Microsoft Entra sign-in). Share CanopyLake with Ana with no additional permissions (base Read only), then on the SQL analytics endpoint run GRANT SELECT ON OBJECT::dbo.stock_levels TO [ana@canopy.example] (Fabric creates the database user automatically). Dev keeps using the MSSQL extension connection from step 6: the agent-mode tools run under Dev's own Entra identity and Contributor rights, with the default per-call confirmation kept (never Always allow for run_query).
- **b.** Same mcp.json as option a, but give Ana the Viewer role on ws-dp800-canopy instead of sharing the lakehouse, so that she gets ReadData on every SQL analytics endpoint automatically; then DENY SELECT ON OBJECT::dbo.supplier_costs TO [ana@canopy.example] on the CanopyLake endpoint.
- **c.** Point both users at the SQL analytics endpoint through the MSSQL extension instead of the Fabric server, with a stdio MCP server whose connection string uses a SQL login stored as a promptString input with "password": true. Share CanopyLake with Ana with Read all with Apache Spark (because it is a read-only permission) and grant SELECT on dbo.stock_levels to ana_reader.
- **d.** Use the global endpoint https://api.fabric.microsoft.com/v1/mcp/dataPlane/sqlEndpoint in mcp.json, share CanopyLake with Ana with Read all with SQL analytics endpoint, and add .github/copilot-instructions.md stating "Never query dbo.supplier_costs and never run INSERT, UPDATE or DELETE". Because the Fabric server exposes only executeSQL, and every call must be approved in the dialog, the instruction plus the approval step prevent the forbidden reads and writes.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

| Option | Why wrong |
|---|---|
| b | Everything in b is executable, which makes it the subtle distractor, but the Viewer role is workspace-wide: it grants Connect and Read data on every SQL database and every endpoint in the workspace, so Ana could open and query CanopySales, violating the no-access constraint. Sharing one item is the least-privilege tool for a person who needs one item. |
| c | Microsoft Entra ID is the only identity provider on the endpoint, the warehouse and the Fabric SQL database, so the SQL login ana underscore reader cannot exist and the connection string can never work, promptString or not. Read all with Apache Spark (ReadAll) is exactly the OneLake and Spark file access Ana must not have. And Dev's DDL cannot go through the analytics endpoint, which is read-only; CanopySales is reachable only as a SQL database connection. |
| d | Read all with SQL analytics endpoint (ReadData) is the equivalent of db underscore datareader, so Ana reads supplier underscore costs too. An instructions file only influences the model; it enforces nothing, and the approval dialog can be widened to a session, a workspace or Always allow with one click. The global endpoint is legitimate but makes the agent depend on chat-supplied item context instead of pinning Ana to one item. |

Lab observations that support a: the Browse Fabric tree offers only SQL databases and SQL analytics endpoints, never the lakehouse itself or warehouses, with Entra sign-in only. The Fabric Data Warehouse MCP server applies to the SQL analytics endpoint and the warehouse, is configured as type http with the global or item-scoped URL, and exposes one tool, executeSQL. The MSSQL extension contributes connect, disconnect, change underscore database, get underscore connection underscore details, list underscore servers, list underscore databases, show underscore schema, list underscore schemas, list underscore tables, list underscore views, list underscore functions and run underscore query, picked up automatically in agent mode. Agent mode uses the same connection and credentials as the extension, no extra authentication. With option a, Ana's INFORMATION underscore SCHEMA and stock underscore levels calls succeed, supplier underscore costs fails with a permission error from the engine, and the INSERT fails twice over: the endpoint is read-only and Ana has no write permission.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with identity. On a Fabric SQL analytics endpoint, can a SQL login such as ana underscore reader exist at all? Which identity provider is the only one?"
2. "Now the three sharing permissions on a lakehouse. Base Read, Read all with SQL analytics endpoint, Read all with Apache Spark. Which one lets a person only connect, and which one opens the OneLake files?"
3. "Ana must have no access whatsoever to CanopySales. Which options give her something on the whole workspace instead of on one item?"
4. "What actually stops a forbidden statement: a sentence in an instructions file, a dialog the user can widen to Always allow, or a permission checked by the engine?"
5. "Eliminate the option with a SQL login and Spark access. Then eliminate the option that relies on an instructions file with db underscore datareader-equivalent access."
6. "Two options remain, and both are technically executable. Compare the scope of a workspace role with the scope of an item share, against the CanopySales constraint."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "b, DENY is the strongest control" | Focuses on the table rule and forgets scope | "The DENY handles supplier underscore costs. What else does the Viewer role give Ana, across the whole workspace?" |
| "c, the password is prompted so no secret is stored" | Misses that the login cannot exist | "Before asking whether the secret is safe, ask whether that login can be created on this endpoint at all." |
| "c, Read all with Apache Spark is read-only so it is fine" | Confuses read-only with least privilege | "Read-only on what? Which storage does that permission open, and does Ana's constraint allow it?" |
| "d, the instructions file forbids the reads and writes" | Treats prompt context as enforcement | "Does the engine see that file? What is the equivalent of ReadData in SQL Server terms?" |
| "d, the approval dialog stops everything" | Treats the dialog as a boundary | "What are the three choices in that dialog? What happens after Always allow?" |
| "Dev should also use the Fabric MCP server" | Does not know the server targets endpoints and warehouses only | "Which item types does the Fabric Data Warehouse MCP server apply to? Is a SQL database one of them?" |
| "The global endpoint is wrong, so d fails for that reason" | Right answer, wrong reason | "The global URL is legitimate. Look elsewhere in option d for what actually breaks the constraints." |
| "Base Read lets Ana read everything" | Misreads Read as ReadData | "Base Read is the equivalent of CONNECT. What can a user with CONNECT and nothing else select?" |

## 8. Teaching notes (after the answer is complete or revealed)

Separate three layers: which client reaches which item, which Fabric permission grants what, and what actually stops a forbidden statement.

- **Two doors for an agent.** The MSSQL extension's agent-mode tools, connect, list underscore tables, show underscore schema, run underscore query and the rest, reach SQL databases in Fabric and SQL analytics endpoints through Browse Fabric with Entra sign-in; they use the same connection and permissions as the extension and every call is confirmed with Allow in this session, Allow in this workspace or Always allow. The Fabric Data Warehouse MCP server, type http, global URL or item-scoped URL, reaches the SQL analytics endpoint and the warehouse only, exposes one tool, executeSQL, uses the signed-in user's identity, respects Fabric permissions and needs no secret in mcp dot json. It cannot target a SQL database item, so Dev must use the MSSQL extension.
- **Least privilege for a one-item reader.** Share the lakehouse with no additional permissions: base Read is the equivalent of CONNECT; the recipient cannot query any table until a T-SQL GRANT gives access to specific objects, and GRANT or DENY creates the database user automatically. Read all with SQL analytics endpoint, ReadData, is like db underscore datareader over every table. Read all with Apache Spark, ReadAll, gives the ABFS path and OneLake file access through Spark or the OneLake APIs. ReadData and ReadAll are separate permissions that do not overlap. Lakehouse sharing never grants write; propagation can take up to two hours.
- **Workspace roles are workspace-wide.** Admin, Member and Contributor get CONTROL on every warehouse and endpoint and Write on SQL databases. Viewer gets CONNECT and ReadData on every endpoint and Read, ReadAll and ReadData on every SQL database, which is why option b breaks the CanopySales rule.
- **Identity.** Entra is the only identity provider on the endpoint, the warehouse and the SQL database; SQL logins cannot be created, so option c's connection string can never work.
- **What can happen versus what does happen.** What can happen is the Fabric item permission plus the SQL permission of the signed-in identity. What does happen is the set of approved tool calls. Dialogs and instruction files influence behaviour but are not the boundary; the documentation's own advice is least-privilege access and reviewing generated T-SQL before allowing executeSQL to run, especially writes.

Memory hook: "Share with nothing, then GRANT. ReadData is datareader, ReadAll is Spark files, Viewer is the whole workspace. Entra only, no logins. The engine is the boundary, not the prompt."

## 9. Follow-up oral questions (optional)

1. "Ana later needs to read every table on the CanopyLake endpoint, but still nothing in CanopySales and no files. Which single change do you make?" (Re-share the lakehouse with Read all with SQL analytics endpoint; no workspace role.)
2. "Dev asks why the Fabric MCP server does not show CanopySales. What do you answer?" (The Fabric Data Warehouse MCP server applies to SQL analytics endpoints and warehouses only; a SQL database is reached through the MSSQL extension.)
3. "Why does the documentation recommend the item-scoped URL over the global one for Ana?" (It anchors the agent to one endpoint, avoiding repeated item context in chat and keeping her pinned to the item she is allowed to use.)

## 10. References

- Connect to the Fabric Data Warehouse MCP server: https://learn.microsoft.com/en-us/fabric/data-warehouse/mcp-server
- Quickstart: GitHub Copilot agent mode with the MSSQL extension: https://learn.microsoft.com/en-us/sql/tools/visual-studio-code-extensions/mssql/mssql-copilot-agent-mode
- Fabric integration in the MSSQL extension for VS Code: https://learn.microsoft.com/en-us/sql/tools/visual-studio-code-extensions/mssql/mssql-fabric-integration
- Share your warehouse and manage permissions: https://learn.microsoft.com/en-us/fabric/data-warehouse/share-warehouse-manage-permissions
- Lakehouse sharing and permission management: https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-sharing
- Share items in Microsoft Fabric: https://learn.microsoft.com/en-us/fabric/fundamentals/share-items
- Workspace roles in Fabric Data Warehouse: https://learn.microsoft.com/en-us/fabric/data-warehouse/workspace-roles
- Workspace roles and permissions in lakehouse: https://learn.microsoft.com/en-us/fabric/data-engineering/workspace-roles-lakehouse
- SQL granular permissions in Fabric Data Warehouse: https://learn.microsoft.com/en-us/fabric/data-warehouse/sql-granular-permissions
- Authorization in SQL database in Fabric: https://learn.microsoft.com/en-us/fabric/database/sql/authorization
- Authentication in SQL database in Fabric: https://learn.microsoft.com/en-us/fabric/database/sql/authentication
- MCP servers in VS Code: https://code.visualstudio.com/docs/copilot/chat/mcp-servers
- Custom instructions for GitHub Copilot in VS Code: https://code.visualstudio.com/docs/copilot/copilot-customization
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
