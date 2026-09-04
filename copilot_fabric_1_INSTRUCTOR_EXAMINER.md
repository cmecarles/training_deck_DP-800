# Instructor-Examiner guide — Copilot in Fabric 1

Companion to [copilot_fabric_1.md](copilot_fabric_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Each option is a plan with three parts: tenant settings, Visual Studio Code and MCP setup, and a security statement. Read all three requirements and all four options before taking an answer. This is a conceptual Fabric and tooling question: name the tenant settings, the extensions, the mcp dot json keys and the MCP URL precisely in words.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Implement AI capabilities in database solutions (25–30%).
- Skill: Use AI-assisted development tools.
- Task bullet: Enable and govern Copilot in Fabric, use GitHub Copilot with the MSSQL extension in Visual Studio Code, and connect agents to Fabric through MCP.
- What is tested: which Fabric tenant settings control Copilot and where they live, the capacity and region prerequisites, what data Copilot for SQL database sends to Azure OpenAI, which GitHub Copilot features are schema-aware, and the identity model of the Fabric Data Warehouse MCP server.

## 2. Scenario to read aloud

**Piece 1, the story.** "Aurora Retail runs its operational store in a SQL database in Microsoft Fabric named AuroraOps, and its sales history in a Fabric lakehouse named AuroraLake, queried through its SQL analytics endpoint. Both items live in the workspace Aurora dash Analytics, assigned to a paid F8 capacity in the Australia East region. The tenant has never used AI features."

**Piece 2, the requirements.** "The head of data wants three things. One: Copilot in Fabric for the AuroraOps SQL database, meaning the chat pane, code completion, and Fix and Explain, but only for the members of the security group sg dash aurora dash copilot, not the whole tenant. Two: developers use GitHub Copilot in Visual Studio Code with schema-aware help against AuroraOps, and can ask an agent questions about AuroraLake through an MCP server. Three: a written statement of the security impact: what leaves the tenant, and what an agent connected to the lakehouse can do."

**Piece 3, option a, tenant settings.** "Option a, first part. In the Fabric admin portal, under the Copilot and AI tenant settings, keep the setting Users can use Copilot and other features powered by Azure OpenAI enabled, but scope it to sg dash aurora dash copilot. Enable the setting Data sent to Azure OpenAI can be processed outside your capacity's geographic region, compliance boundary, or national cloud instance, because Australia East is outside the US and the EU data boundary and Copilot stays disabled otherwise. Leave the workspace on the F8 capacity."

**Piece 4, option a, Visual Studio Code and MCP.** "Option a, second part. In Visual Studio Code, install the MSSQL extension plus GitHub Copilot and GitHub Copilot Chat; an active Copilot subscription is required. Connect to AuroraOps with Microsoft Entra authentication, and use the at mssql chat participant or agent mode. For the lakehouse, add the Fabric Data Warehouse MCP server to mcp dot json with type http and the URL https colon slash slash api dot fabric dot microsoft dot com slash v1 slash mcp slash dataPlane slash sqlEndpoint."

**Piece 5, option a, security statement.** "Option a, third part. Copilot sends the user's prompts, chat history, the executed query text and error messages, and the database schema, not table rows, to Azure OpenAI. Prompts are not used to train foundation models. The MCP server runs executeSQL under the signed-in user's identity, respecting Fabric and SQL permissions. Each tool call needs explicit approval in the client, and write statements must be reviewed before approval."

**Piece 6, option b.** "Option b. Move Aurora dash Analytics to a Fabric trial capacity so that Copilot usage does not consume paid capacity units, and turn on Users can use Copilot and other features powered by Azure OpenAI for the whole organization. No cross-geo setting is needed because Copilot always processes data inside the capacity's region. Configure the same Visual Studio Code and MCP setup as option a."

**Piece 7, option c.** "Option c. Same tenant settings and Visual Studio Code setup as option a. For the lakehouse, create a SQL login and password on the AuroraLake SQL analytics endpoint, store them as inputs in the MCP server configuration, and use the MSSQL extension's run underscore query tool. Security statement: the MCP server runs as that SQL login, so the agent is limited by whatever permissions the login holds."

**Piece 8, option d.** "Option d. Turn Copilot in Fabric off for the tenant, because the SQL database Copilot streams table data to Azure OpenAI to answer questions such as what are the top-selling products. Instead have developers rely on GitHub Copilot inline completions in dot sql files, which are schema-aware once the MSSQL extension is connected, and connect the agent to AuroraLake through the same MCP server as option a."

## 3. Setup script (reference only; do not read verbatim unless asked)

Option a mcp.json entry, global endpoint:

```json
{ "servers": { "aurora-lake": { "type": "http",
    "url": "https://api.fabric.microsoft.com/v1/mcp/dataPlane/sqlEndpoint" } } }
```

Equivalent item-scoped variant:

```json
{ "servers": { "aurora-lake": { "type": "http",
    "url": "https://api.fabric.microsoft.com/v1/mcp/dataPlane/workspaces/<workspace-id>/items/<item-id>/sqlEndpoint" } } }
```

Where each control lives:

```text
Fabric admin portal  -> Tenant settings > Copilot and AI          (who may use Copilot, cross-geo processing)
                     -> Capacity settings (if delegated)           (same switches per capacity)
Workspace            -> must sit on a paid F2+/P1+ capacity in a supported region
Fabric SQL database  -> Copilot chat pane, code completion, Fix / Explain (schema only, no data)
Visual Studio Code   -> MSSQL extension + GitHub Copilot + Copilot Chat; @mssql participant, agent-mode tools
mcp.json             -> "type": "http" server pointing at api.fabric.microsoft.com/v1/mcp/dataPlane/sqlEndpoint
Fabric permissions   -> workspace role / item permission / SQL GRANTs decide what executeSQL may run
```

MSSQL extension agent-mode tools: connect, list_databases, show_schema, list_tables, run_query, and others. Each call requires approval before execution.

## 4. The question (ask exactly this)

"Which plan is correct?

a. In the Fabric admin portal, under Copilot and AI tenant settings, keep Users can use Copilot and other features powered by Azure OpenAI enabled but scope it to sg dash aurora dash copilot; enable Data sent to Azure OpenAI can be processed outside your capacity's geographic region, compliance boundary, or national cloud instance; leave the workspace on the F8 capacity. In Visual Studio Code, install the MSSQL extension plus GitHub Copilot and GitHub Copilot Chat with an active subscription, connect to AuroraOps with Entra authentication, and use the at mssql chat participant or agent mode. For the lakehouse, add the Fabric Data Warehouse MCP server to mcp dot json with type http and URL api dot fabric dot microsoft dot com slash v1 slash mcp slash dataPlane slash sqlEndpoint. Security statement: Copilot sends prompts, chat history, executed query text and errors, and the schema, not table rows; prompts are not used to train models; the MCP server runs executeSQL as the signed-in user, respecting Fabric and SQL permissions; each tool call needs explicit approval and writes must be reviewed.

b. Move Aurora dash Analytics to a Fabric trial capacity, turn on Users can use Copilot and other features powered by Azure OpenAI for the whole organization, no cross-geo setting because Copilot always processes data inside the capacity's region, same Visual Studio Code and MCP setup as a.

c. Same tenant settings and Visual Studio Code setup as a. For the lakehouse, create a SQL login and password on the AuroraLake SQL analytics endpoint, store them as inputs in the MCP server configuration, and use the MSSQL extension's run underscore query tool. Security statement: the MCP server runs as that SQL login, so the agent is limited by the login's permissions.

d. Turn Copilot in Fabric off for the tenant, because the SQL database Copilot streams table data to Azure OpenAI. Have developers rely on GitHub Copilot inline completions in dot sql files, which are schema-aware once the MSSQL extension is connected, and connect the agent to AuroraLake through the same MCP server as a.

Which letter, and why do the other three fail?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

| Option | Verdict | Why |
|---|---|---|
| a | Correct | Copilot and AI tenant settings, main switch scoped to the security group, cross-geo processing switch enabled because Australia East is outside the US or France, paid F8 capacity kept. MSSQL plus GitHub Copilot plus Copilot Chat plus subscription plus connection gives schema-aware at mssql chat and agent tools. Fabric Data Warehouse MCP server, type http, global sqlEndpoint URL, single tool executeSQL, runs as the signed-in Entra user. Security statement matches the docs. |
| b | Wrong | Copilot in Fabric is not supported on trial SKUs, only paid F2 or higher or P1 or higher. The cross-geo claim is backwards: outside the US or France the processing setting must be enabled or Copilot stays disabled. Whole-organization enablement ignores requirement 1. |
| c | Wrong | The SQL analytics endpoint, warehouse and Fabric SQL database support Microsoft Entra authentication only; there are no SQL logins to create. Storing a password in an MCP config is the anti-pattern; the identity should be the signed-in user's token so Fabric roles, item permissions, SQL grants and the audit trail apply to the person. |
| d | Wrong | Copilot for SQL database never sends table rows; it sends schema, prompt, chat history and executed query text, and has no access to the data. Turning it off is unnecessary. GitHub Copilot inline completions do not see the connected schema; schema-aware help comes from at mssql chat, agent-mode tools, slash commands and the Schema Designer. |

Requirement scorecard: a satisfies 1, 2, 3. b violates 1. c violates 2 and 3. d violates 1, 2 and 3.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with requirement one and the capacity. Which capacity SKUs support Copilot in Fabric: paid, trial, or both? One option moves the workspace to a trial."
2. "Now the region. The F8 capacity is in Australia East. Is that in the US or in France? What must an admin enable so Copilot works outside those two places?"
3. "Think about how you authenticate to a Fabric SQL analytics endpoint. Is there such a thing as a SQL login and password on it, or is it Entra only?"
4. "Two claims in option d. First: does Copilot for SQL database in Fabric read table rows, or only the schema? Second: which GitHub Copilot feature is not schema-aware, ghost-text inline completions or at mssql chat?"
5. "Only one option keeps a paid capacity, enables the cross-geo processing setting, uses Entra identity for MCP and says schema not rows. Which letter?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "Option b, a trial capacity saves capacity units" | Does not know Copilot needs paid F2 or higher or P1 or higher | "Check the documented SKU requirement for Copilot in Fabric. Are trial SKUs listed as supported?" |
| "Option b, Copilot always runs in the capacity's region" | Has the cross-geo setting backwards | "Where are the Azure OpenAI resources that Fabric uses? What happens in a region outside the US or France when the processing setting is off?" |
| "Option c, a SQL login gives least privilege" | Thinks the analytics endpoint has SQL authentication | "Which authentication methods does the SQL analytics endpoint support? Can you run CREATE LOGIN there?" |
| "Option c, storing credentials in mcp dot json is fine" | Misses the identity and audit model | "If the agent runs as a shared login, whose name appears in the audit trail, and do the user's Fabric workspace roles still apply?" |
| "Option d, Copilot sends data so turn it off" | Believes Copilot for SQL database reads rows | "What does the documentation say Copilot for SQL database can access: the schema, the data, or both?" |
| "Option d, inline completions know the schema" | Confuses ghost text with the at mssql participant | "Which features are listed as schema-aware in the MSSQL extension docs? Is ghost text among them?" |
| "Option a, but the cross-geo setting is a data leak" | Confuses processing with storage settings | "There are two cross-geo switches. Which one is about processing requests and which one is about storing conversation history? Which applies to the SQL database Copilot?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the four control points:

- **Tenant settings, Copilot and AI group, in the Fabric admin portal.** Users can use Copilot and other features powered by Azure OpenAI is on by default for tenants with a paid capacity, and every setting in the group can be scoped to security groups. That is how requirement 1 is met. Data sent to Azure OpenAI can be processed outside your capacity's geographic region, compliance boundary, or national cloud instance is off by default, and outside the US or France Copilot is disabled until an admin enables it. Australia East needs it. A third switch, can be stored outside, only concerns conversation history for notebooks and data agents, not the SQL database Copilot. The settings can be delegated to capacity administrators and set on the capacity's settings page with the same names.
- **Capacity.** Copilot in Fabric needs a paid SKU, F2 or higher or P1 or higher. Trial SKUs are not supported. F8 qualifies. That is why option b fails.
- **What Copilot for SQL database sends.** The prompt, the session's previous messages, the text and error messages of executed queries, and the schema the user can see. Never table rows. Prompts are not used to train foundation models. It may give inaccurate results when asked to evaluate data, precisely because it has no access to data. Portal Copilot never runs queries by itself. That is why option d's reason for turning it off is wrong.
- **GitHub Copilot in Visual Studio Code.** Prerequisites: MSSQL extension, GitHub Copilot and GitHub Copilot Chat extensions, an active Copilot subscription, and a database connection. Schema-aware: the at mssql chat participant in ask mode, agent-mode tools such as connect, list underscore databases, show underscore schema, list underscore tables and run underscore query, each requiring approval, slash commands, and the Schema Designer. Not schema-aware: inline completions, the ghost text. Ask mode is read-only by default; agent mode can write only after approval, with the user's own connection and permissions. Fabric SQL database, Fabric warehouse and the lakehouse SQL analytics endpoint are all supported targets.
- **MCP for the lakehouse.** The Fabric Data Warehouse MCP server applies to SQL analytics endpoint and Warehouse. mcp dot json entry: type http, URL api dot fabric dot microsoft dot com slash v1 slash mcp slash dataPlane slash sqlEndpoint, or an item-scoped URL with workspaces slash id slash items slash id. One tool, executeSQL; metadata discovery through INFORMATION underscore SCHEMA queries. It uses the signed-in user's identity and respects Fabric permissions; no credentials in the config. Entra-only authentication, no SQL logins. That is why option c fails.
- **Security statement for the agent.** executeSQL should require explicit approval in the client. Review generated T-SQL before approving, especially CREATE, ALTER, DROP, INSERT, UPDATE, DELETE. Use least privilege for production. Prompt injection through returned data cannot escalate beyond the user's permissions, but can cause a destructive statement to be proposed, so per-call approval is not optional. When a Fabric data agent is consumed from Foundry, Copilot Studio, Microsoft 365 Copilot or as an MCP server, responses may leave Fabric's compliance boundary; that is a separate decision.

Memory hook: "Paid capacity, scoped switch, cross-geo processing on outside the US or France. Copilot sees schema, not rows. Ghost text is blind. MCP runs as you, and you approve every call."

## 9. Follow-up oral questions (optional)

1. "Name the two tenant settings that must be checked for Copilot on an Australia East capacity." (Users can use Copilot and other features powered by Azure OpenAI, scoped to the group; and Data sent to Azure OpenAI can be processed outside your capacity's geographic region, compliance boundary, or national cloud instance.)
2. "What single tool does the Fabric Data Warehouse MCP server expose, and how do you discover tables through it?" (executeSQL; run INFORMATION underscore SCHEMA queries through it.)
3. "In Visual Studio Code, which mode of the MSSQL Copilot experience can perform writes, and under what condition?" (Agent mode, only after explicit user approval of the tool call, using the user's own connection.)

## 10. References

- Copilot in Fabric overview and requirements: https://learn.microsoft.com/en-us/fabric/fundamentals/copilot-fabric-overview
- Enable Copilot in Fabric, tenant settings: https://learn.microsoft.com/en-us/fabric/fundamentals/copilot-enable-fabric
- Copilot tenant settings reference: https://learn.microsoft.com/en-us/fabric/admin/service-admin-portal-copilot
- Copilot for SQL database in Fabric: https://learn.microsoft.com/en-us/fabric/database/sql/copilot
- Copilot for SQL database privacy and security: https://learn.microsoft.com/en-us/fabric/database/sql/copilot-privacy-security
- GitHub Copilot with the MSSQL extension for Visual Studio Code: https://learn.microsoft.com/en-us/sql/tools/visual-studio-code-extensions/mssql/mssql-copilot-extension
- Fabric Data Warehouse MCP server: https://learn.microsoft.com/en-us/fabric/data-warehouse/mcp-server
- MCP servers in Visual Studio Code: https://code.visualstudio.com/docs/copilot/chat/mcp-servers
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
