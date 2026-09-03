# SQL Server question — Copilot in Fabric 1

## Statement

Aurora Retail runs its operational store in a **SQL database in Microsoft Fabric** named `AuroraOps` and its sales history in a Fabric **lakehouse** named `AuroraLake` (queried through its SQL analytics endpoint). Both items live in the workspace `Aurora-Analytics`, assigned to a paid **F8** capacity in the **Australia East** region. The tenant has never used AI features.

The head of data wants:

1. **Copilot in Fabric** for the `AuroraOps` SQL database (chat pane, code completion, Fix/Explain), but only for the members of the security group `sg-aurora-copilot`, not the whole tenant.
2. Developers to use **GitHub Copilot in Visual Studio Code** with schema-aware help against `AuroraOps`, and to ask an agent questions about `AuroraLake` through an **MCP server**.
3. A written statement of the **security impact**: what leaves the tenant, and what an agent connected to the lakehouse can do.

Which plan is correct?

### a.

In the Fabric admin portal, under **Copilot and AI** tenant settings, keep **Users can use Copilot and other features powered by Azure OpenAI** enabled but scope it to `sg-aurora-copilot`; enable **Data sent to Azure OpenAI can be processed outside your capacity's geographic region, compliance boundary, or national cloud instance**, because Australia East is outside the US and the EU data boundary and Copilot stays disabled otherwise. Leave the workspace on the F8 capacity.

In Visual Studio Code, install the **MSSQL** extension plus **GitHub Copilot** and **GitHub Copilot Chat** (an active Copilot subscription is required), connect to `AuroraOps` with Microsoft Entra authentication, and use the **`@mssql`** chat participant or agent mode. For the lakehouse, add the Fabric Data Warehouse MCP server to `mcp.json` with `"type": "http", "url": "https://api.fabric.microsoft.com/v1/mcp/dataPlane/sqlEndpoint"`.

Security statement: Copilot sends the user's prompts, chat history, the executed query text and error messages, and the **database schema** — not table rows — to Azure OpenAI; prompts are not used to train foundation models. The MCP server runs `executeSQL` under the **signed-in user's identity**, respecting Fabric and SQL permissions; each tool call needs explicit approval in the client, and write statements must be reviewed before approval.

### b.

Move `Aurora-Analytics` to a **Fabric trial** capacity so that Copilot usage does not consume paid capacity units, and turn on **Users can use Copilot and other features powered by Azure OpenAI** for the whole organization. No cross-geo setting is needed because Copilot always processes data inside the capacity's region. Configure the same Visual Studio Code and MCP setup as option a.

### c.

Same tenant settings and Visual Studio Code setup as option a. For the lakehouse, create a SQL login and password on the `AuroraLake` SQL analytics endpoint, store them as inputs in the MCP server configuration, and use the MSSQL extension's `run_query` tool. Security statement: the MCP server runs as that SQL login, so the agent is limited by whatever permissions the login holds.

### d.

Turn Copilot in Fabric **off** for the tenant, because the SQL database Copilot streams table data to Azure OpenAI to answer questions such as "what are the top-selling products". Instead have developers rely on GitHub Copilot **inline completions** in `.sql` files, which are schema-aware once the MSSQL extension is connected, and connect the agent to `AuroraLake` through the same MCP server as option a.

## Correct Answer

**a**

## Explanation

The correct answer is **a**. Evaluate each option against the three requirements:

| Requirement | a | b (trial capacity) | c (SQL login for MCP) | d (Copilot off, inline completions) |
|---|---|---|---|---|
| 1. Copilot in Fabric for one security group, working in Australia East | satisfied | **violated** (trial SKU unsupported; no cross-geo setting; whole tenant) | satisfied | **violated** (Copilot turned off) |
| 2. Schema-aware GitHub Copilot + MCP agent on the lakehouse | satisfied | satisfied | **violated** (no SQL logins exist on the SQL analytics endpoint) | **violated** (inline completions are not schema-aware) |
| 3. Accurate security statement | satisfied | satisfied | **violated** (wrong identity model) | **violated** (claims table data is sent) |

### Where each control lives

```text
Fabric admin portal  -> Tenant settings > Copilot and AI          (who may use Copilot, cross-geo processing)
                     -> Capacity settings (if delegated)           (same switches per capacity)
Workspace            -> must sit on a paid F2+/P1+ capacity in a supported region
Fabric SQL database  -> Copilot chat pane, code completion, Fix / Explain (schema only, no data)
Visual Studio Code   -> MSSQL extension + GitHub Copilot + Copilot Chat; @mssql participant, agent-mode tools
mcp.json             -> "type": "http" server pointing at api.fabric.microsoft.com/v1/mcp/dataPlane/sqlEndpoint
Fabric permissions   -> workspace role / item permission / SQL GRANTs decide what executeSQL may run
```

### Why option a is correct

**Enabling Copilot in Fabric.** Copilot is controlled by the *Copilot and AI* tenant-setting group. *Users can use Copilot and other features powered by Azure OpenAI* is **enabled by default** for tenants with a paid capacity (F2 or higher, or P1 or higher), and every setting in the group can be applied to specific security groups rather than the whole organization — that is how requirement 1 is met. The setting can also be delegated to capacity administrators. The second setting is the one people forget: *Data sent to Azure OpenAI can be processed outside your capacity's geographic region, compliance boundary, or national cloud instance* is **disabled by default** and "If your tenant or capacity is outside the US or France, Copilot is disabled by default unless your Fabric tenant admin enables" it. For an Australia East capacity, Copilot for SQL database does not work until that setting is on. The workspace must stay on a supported paid capacity (F2+/P1+), which F8 is.

**GitHub Copilot in Visual Studio Code.** The documented prerequisites are the MSSQL extension, the *GitHub Copilot* and *GitHub Copilot Chat* extensions, a GitHub account with an active Copilot subscription, and a database connection. Schema awareness comes from the `@mssql` chat participant (ask mode), the MSSQL agent-mode tools (`connect`, `list_databases`, `show_schema`, `list_tables`, `run_query`, ...) and slash commands — each agent tool call "requires your approval before execution". Fabric SQL database, Fabric Data Warehouse and the Fabric lakehouse SQL analytics endpoint are all supported targets.

**MCP for the lakehouse.** The remote Fabric Data Warehouse MCP server "Applies to: SQL analytics endpoint and Warehouse in Microsoft Fabric". The global endpoint is `https://api.fabric.microsoft.com/v1/mcp/dataPlane/sqlEndpoint` (an item-scoped variant embeds `workspaces/<workspace-id>/items/<item-id>`); it exposes a single tool, `executeSQL`, and metadata discovery is done with `INFORMATION_SCHEMA` queries through that tool. It "uses the signed-in user's identity and respects Fabric permissions" — no credentials in the configuration.

**Security impact.** Copilot for SQL database "can only access the database schema that is accessible in the user's database"; by default it has access to previous messages in the session, the contents and error messages of queries the user executed, and the schemas — and "Copilot only has access to the database schema; it has no access to the data". Prompts and responses "aren't used to train foundation models". Data goes to the Azure OpenAI resources Fabric operates in the capacity's geography, unless the cross-geo processing setting allows otherwise. On the agent side, the documentation is explicit: `executeSQL` "should require explicit user approval in the MCP client", "Review generated T-SQL before allowing executeSQL to run, especially for write operations", and "Use least-privilege access for production warehouses". Copilot in the Fabric portal "doesn't autonomously execute queries"; in SSMS/VS Code, *Ask mode* is read-only by default while *Agent mode* can write only after user approval.

### Why option b is wrong

Two factual errors. "Copilot in Microsoft Fabric isn't supported on trial SKUs. Only paid SKUs (F2 or higher, or P1 or higher) are supported" — moving to a trial capacity switches Copilot off. And the cross-geo claim is backwards: outside the US and the EU data boundary, Copilot requests may need to be processed in another region, which is exactly why the tenant admin must consciously enable *Data sent to Azure OpenAI can be processed outside your capacity's geographic region...*; without it, Copilot stays disabled for an Australia East capacity. Enabling for the whole organization also ignores requirement 1 (and the documentation's warning to roll out by security group).

### Why option c is wrong

Fabric's SQL analytics endpoint (like the warehouse and the Fabric SQL database) supports **Microsoft Entra authentication only** — there are no SQL logins to create, so the plan cannot be implemented. Even where SQL logins exist, putting a password into an MCP server definition is the anti-pattern the security section of the exam targets: the identity should be the signed-in user's (or a managed identity's) token, so that Fabric workspace roles, item permissions and SQL permissions apply to the agent exactly as they apply to the person, and so that the audit trail names that person.

### Why option d is wrong

It gets the data flow wrong in both directions. Copilot for SQL database does **not** send table rows to Azure OpenAI; it sends the schema, the prompt, chat history and executed-query text, and it may "produce inaccurate results when the intent is to evaluate data" precisely because it "has no access to the data". Turning it off for that reason is unnecessary. Meanwhile GitHub Copilot **inline completions** (ghost text in `.sql` files) are generated by the Copilot model directly and "don't see your connected database schema" — the documentation's table marks them *not* schema-aware. Schema-aware help comes from `@mssql` chat, agent-mode tools, slash commands and the Schema Designer, all of which need the connection.

### The security statement, spelled out

What an auditor should be told about this setup:

- **Data flow of Copilot in Fabric (SQL database).** Sent to the Azure OpenAI resources that Fabric operates: the prompt, the session's previous messages and replies, the text and error messages of queries the user executed, and the schema the user can see. Not sent: rows from tables. Prompts and responses are protected under Microsoft's privacy commitments and are not used to train foundation models. With the cross-geo *processing* setting enabled, requests from an Australia East capacity may be processed in another geography; the separate *storage* setting only concerns conversation history for notebooks and data agents, not the SQL database Copilot.
- **Execution model.** In the Fabric portal, Copilot generates T-SQL but never runs it on its own; the user runs it. In SSMS/Visual Studio Code, *Ask mode* runs read-only queries, and *Agent mode* can perform writes only after explicit user approval of the tool call. The tool runs with the user's own connection and permissions — agent mode "doesn't introduce another authentication or permission changes".
- **MCP against the lakehouse.** The Fabric Data Warehouse MCP server exposes only `executeSQL`; whatever the agent runs is limited by the signed-in user's workspace role, item permission and SQL permissions, and every call must be approved. The recommended rollout is to start with read-only prompts and metadata exploration, review any `CREATE`/`ALTER`/`DROP`/`INSERT`/`UPDATE`/`DELETE` before approving it, and use least-privilege identities for production items. Prompt injection through data returned by a query cannot escalate beyond the user's own permissions, but it can still cause a destructive statement to be *proposed* — which is why per-call approval is not optional.
- **Fabric data agents as MCP servers.** When a Fabric data agent is consumed from a non-Fabric service (Foundry, Copilot Studio, Microsoft 365 Copilot, or as an MCP server), the admin portal warns that responses "may be sent outside of Fabric's compliance boundary or geographic region" and are handled under that service's terms — a separate decision from the Copilot tenant switches.

### Equivalent alternatives

Two variations of option a would be equally correct:

- Binding the MCP server to the lakehouse item instead of the global endpoint, so that the agent never has to be told which item to use:

  ```json
  { "servers": { "aurora-lake": { "type": "http",
      "url": "https://api.fabric.microsoft.com/v1/mcp/dataPlane/workspaces/<workspace-id>/items/<item-id>/sqlEndpoint" } } }
  ```

- Delegating the Copilot tenant settings to capacity administrators and enabling the same two switches on the F8 capacity's settings page, scoped to `sg-aurora-copilot`; the delegated capacity-level switches have the same names as the tenant-level ones.

Conceptual question (Azure / tooling); not executed against an engine.

## DP-800 Exam Rule to Remember

```text
Copilot in Fabric (admin portal > Tenant settings > Copilot and AI)
  "Users can use Copilot and other features powered by Azure OpenAI"      ON by default; scope to security groups
  "Data sent to Azure OpenAI can be processed outside your capacity's geographic region..."  OFF by default;
       REQUIRED outside US / EU data boundary (else Copilot is disabled there)
  "...can be stored outside..."  OFF by default; only for notebooks / data agents (conversation history)
  Paid capacity only: F2+ or P1+; trial SKUs and Pro/PPU workspaces do NOT support Copilot; settings can be
  delegated to capacity admins.
Copilot for SQL database in Fabric sees: schema, your prompt, chat history, executed query text + errors.
  It never reads table data; prompts are not used to train models; portal Copilot never runs queries by itself.
GitHub Copilot + MSSQL extension (VS Code): needs GitHub Copilot + Copilot Chat + subscription + a connection.
  Schema-aware: @mssql chat, agent-mode tools (connect, show_schema, list_tables, run_query ... each approved),
  slash commands, Schema Designer.  NOT schema-aware: inline completions (ghost text).
Fabric lakehouse / warehouse via MCP: https://api.fabric.microsoft.com/v1/mcp/dataPlane/sqlEndpoint
  ("type": "http"), one tool: executeSQL, runs as the signed-in Entra user (Entra-only auth, no SQL logins),
  respects Fabric + SQL permissions, approve every call, review writes, least privilege.
```
