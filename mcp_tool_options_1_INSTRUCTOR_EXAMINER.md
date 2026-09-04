# Instructor-Examiner guide — Model and MCP Tool Options 1

Companion to [mcp_tool_options_1.md](mcp_tool_options_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with one correct answer. Read all four options, a to d, in full before taking an answer. This is a Visual Studio Code and GitHub Copilot tooling question, not a T-SQL question; there is no SQL to run. The learner must weigh each option against five numbered requirements, so keep the five requirements handy and re-read any of them on request. When the learner picks an option, ask them to say which requirement, if any, that option breaks; that is where the hints go.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked. For JSON, name the keys and values in words; say "hash" for the `#` prefix in tool references such as `#mssql_run_query`.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Use AI-assisted tooling for database development.
- Task bullet: Configure and use GitHub Copilot with the MSSQL extension for Visual Studio Code, including agent mode, MCP servers, tool approval and model selection.
- What is tested: which chat mode runs tools autonomously, how the tools picker scopes a session, what the tool-approval scopes mean, how `.vscode/mcp.json` shares MCP servers and secrets safely, and how the model picker affects premium-request cost.

## 2. Scenario to read aloud

**Piece 1, the story.** "A haulage company called Halden Freight keeps its dispatch database, FreightOps, on a development SQL Server. A developer opens the team's repository in Visual Studio Code. Three extensions are installed: the MSSQL extension, GitHub Copilot, and GitHub Copilot Chat. The developer is already connected to FreightOps through a saved MSSQL connection profile named HaldenDev."

**Piece 2, the shared MCP file.** "The repository contains a shared workspace file at dot vscode slash mcp dot json. Every developer who opens the repository gets it. It has two top-level keys: inputs and servers. The inputs array holds one input. Its id is docs dash token, its type is promptString, its description says it is the bearer token for the Halden internal docs MCP server, and its password flag is true, which means Visual Studio Code prompts for it with hidden typing."

**Piece 3, the two servers.** "Under servers there are two entries. The first is named halden dash docs. Its type is http, its url is https colon slash slash docs dot halden dot internal slash mcp, and it has a headers object with one header, Authorization, whose value is the word Bearer followed by the placeholder dollar, open brace, input colon docs dash token, close brace. The second server is named shell dash tools. Its type is stdio, its command is node, and its args array holds one item, the relative path dot slash tools slash shell dash mcp slash index dot js."

**Piece 4, what the servers do.** "halden dash docs exposes read-only search tools over the company's T-SQL coding standards. shell dash tools exposes a tool called run underscore shell that executes arbitrary operating-system commands."

**Piece 5, the task.** "The developer wants to run one chat session that refactors the six stored procedures in the folder sql slash dispatch so that they follow the coding standards. The developer sets five requirements for that session. I will read them one at a time."

**Piece 6, requirements one and two.** "Requirement one: Copilot must work autonomously. It must connect to HaldenDev, inspect the schema, run validation SELECT queries, look up the standards, and edit the dot sql files, chaining those steps itself, without the developer typing each one. Requirement two: only the MSSQL extension's database tools and the halden dash docs tools may be callable in this session. shell dash tools must not be callable, but its definition must stay in the shared mcp dot json for the other teams that need it."

**Piece 7, requirements three to five.** "Requirement three: every mssql underscore run underscore query invocation must be confirmed individually by the developer. The read-only schema tools, mssql underscore list underscore tables, mssql underscore show underscore schema and mssql underscore list underscore views, may be approved once and then run without prompts for this repository. Requirement four: the halden dash docs bearer token must never be committed. Requirement five: the developer is on a legacy premium-request plan and wants the exploratory prompts to use a low-multiplier model, and only the final refactoring prompt to use a reasoning model."

**Piece 8, option a.** "Option a. In the Chat view, select Agent in the mode picker. Open the tools picker, the Configure Tools button, and deselect the shell dash tools server for this session, leaving the MSSQL tools and halden dash docs selected. Keep the default approval behaviour. When the confirmation dialog appears for list tables, show schema or list views, choose Allow in this Workspace. For run query, approve each invocation with the single-use option. Leave the committed mcp dot json as it is; Visual Studio Code prompts for docs dash token when the server starts. In the model picker, choose a zero point three three x model, for example GPT-5 mini, for the exploratory prompts, and switch to a reasoning model such as Claude Sonnet for the refactoring prompt."

**Piece 9, option b.** "Option b. In the Chat view, select Edit in the mode picker, because the goal is to change files. Attach the six dot sql files as context, and write the prompt with explicit tool references: hash mssql underscore connect, hash mssql underscore show underscore schema, hash mssql underscore run underscore query, and hash halden dash docs, so that Copilot invokes those tools while it edits the files. Because Edit mode never shows shell dash tools, requirement two is satisfied without touching the tools picker. Approve the tools as in option a and pick the models as in option a."

**Piece 10, option c.** "Option c. In the Chat view, select Agent. To avoid the repeated confirmation dialogs, set the setting chat dot tools dot global dot autoApprove to true in the workspace settings, or run slash yolo in chat, since the HaldenDev connection uses a login that only has db underscore datareader anyway. Deselect shell dash tools in the tools picker. To stop the token prompt at every server start, replace the placeholder dollar input colon docs dash token in mcp dot json with the actual bearer token. Pick the models as in option a."

**Piece 11, option d.** "Option d. In the Chat view, select Agent. Instead of using the tools picker, add the setting chat dot mcp dot access with the value none to dot vscode slash settings dot json, so that shell dash tools cannot run, and reference hash halden dash docs in the prompt so that the docs tools are still available. When the confirmation dialog appears for run query, choose Always allow; it only applies to the current workspace, so it satisfies requirement three while removing the noise. Leave mcp dot json unchanged and pick the models as in option a."

## 3. Setup script (reference only; do not read verbatim unless asked)

The shared file `.vscode/mcp.json`:

```json
{
    "inputs": [
        {
            "id": "docs-token",
            "type": "promptString",
            "description": "Bearer token for the Halden internal docs MCP server",
            "password": true
        }
    ],
    "servers": {
        "halden-docs": {
            "type": "http",
            "url": "https://docs.halden.internal/mcp",
            "headers": { "Authorization": "Bearer ${input:docs-token}" }
        },
        "shell-tools": {
            "type": "stdio",
            "command": "node",
            "args": ["./tools/shell-mcp/index.js"]
        }
    }
}
```

Settings and commands named in the options:

```text
Option a: mode picker = Agent; Configure Tools -> deselect server "shell-tools";
          approvals: "Allow in this Workspace" for mssql_list_tables / mssql_show_schema / mssql_list_views,
                     single-use approval for mssql_run_query;
          .vscode/mcp.json unchanged; model picker: 0.33x model, then a reasoning model.
Option b: mode picker = Edit; six .sql files attached; prompt references
          #mssql_connect #mssql_show_schema #mssql_run_query #halden-docs.
Option c: mode picker = Agent; workspace setting "chat.tools.global.autoApprove": true  (or /yolo);
          deselect "shell-tools"; ${input:docs-token} replaced by the literal token in .vscode/mcp.json.
Option d: mode picker = Agent; .vscode/settings.json  "chat.mcp.access": "none";
          prompt references #halden-docs; approval "Always allow" for mssql_run_query.
```

## 4. The question (ask exactly this)

"Which way of configuring the chat session meets all five requirements?

- a. Agent mode. Deselect the shell-tools server in the Configure Tools picker. Keep default approvals: Allow in this Workspace for the three read-only schema tools, single-use approval for each run query. Leave mcp dot json unchanged so Visual Studio Code prompts for the token. Pick a 0.33x model for exploration and a reasoning model for the refactor.
- b. Edit mode, because the goal is to change files. Attach the six sql files and reference hash mssql connect, hash mssql show schema, hash mssql run query and hash halden-docs in the prompt. Edit mode never shows shell-tools, so requirement two is met without the tools picker. Approvals and models as in a.
- c. Agent mode. Set chat dot tools dot global dot autoApprove to true in workspace settings, or run slash yolo, because the login is db underscore datareader only. Deselect shell-tools. Replace the docs-token placeholder in mcp dot json with the real bearer token. Models as in a.
- d. Agent mode. Add chat dot mcp dot access equals none to dot vscode slash settings dot json so shell-tools cannot run, and reference hash halden-docs in the prompt. Choose Always allow for run query, since it only applies to the current workspace. Leave mcp dot json unchanged. Models as in a."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

| Requirement | a | b | c | d |
|---|---|---|---|---|
| 1. Autonomous multi-step tool use | met | broken | met | met |
| 2. Exclude shell-tools for this session only | met | broken | met | broken, kills halden-docs too |
| 3. Per-invocation approval of run query | met | met | broken | broken |
| 4. Token never committed | met | met | broken | met |
| 5. Model per prompt | met | met | met | met |

Why each wrong option is wrong, one line each:

- **b** is wrong because Edit mode applies edits to files in context but never runs the tool loop; a hash tool reference in Edit mode attaches nothing that can execute, so requirement 1 fails, and shell-tools is "hidden" only because no tool at all can run.
- **c** is wrong because `chat.tools.global.autoApprove` (same as `/yolo`) disables every confirmation, including run query, breaking requirement 3; and pasting the bearer token into the committed `.vscode/mcp.json` breaks requirement 4.
- **d** is wrong because `chat.mcp.access: none` disables every MCP server, halden-docs included, breaking requirement 2; and "Always allow" approves the tool for all future invocations in every workspace, not just this one, so a standing approval for run query breaks requirement 3.

## 6. Hint ladder (one hint per attempt, in order)

1. "Go back to requirement one. Copilot must chain connect, inspect, query, look up and edit by itself. Which of the chat modes is the one that runs tools on its own? Check each option's mode."
2. "Now requirement four. Read each option again and ask: does anything end up written into the committed mcp dot json that should not be there?"
3. "Requirement three has two halves. Run query must be confirmed every single time. Which options remove that confirmation, whether by a setting, a slash command, or a standing approval?"
4. "One option uses a setting called chat dot mcp dot access. Think about what that setting governs. Does it act on one server, or on all MCP servers at once? What happens to halden dash docs?"
5. "That eliminates option b, which cannot run the tool loop. Three options remain, all in Agent mode. Which of them keeps the default approval behaviour and leaves the token as a prompted input?"
6. "That also eliminates option c, which switches off all approvals and hard-codes the token. Two remain: a and d. Compare how each one keeps shell dash tools out, and what scope Always allow really has."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "b, because we are editing files, so Edit mode is the right mode" | Thinks the mode is chosen by the kind of output rather than by whether tools run | "Editing is the last step. Who runs the connect, schema and query steps before that, and in which mode can that happen?" |
| "b, because hash references make the tools run in any mode" | Believes a hash reference forces a tool invocation | "A hash reference points at a tool. Which mode actually invokes tools?" |
| "c, the login is read-only so auto-approve is safe" | Confuses database permissions with the review requirement | "Requirement three is about the developer confirming each call, not about what the database allows. Does a read-only login give you that confirmation?" |
| "c, prompting for the token every time is annoying, so put it in the file" | Forgets the file is committed | "Where does mcp dot json live, and who else sees it? Re-read requirement four." |
| "d, chat dot mcp dot access none is a clean way to block the shell server" | Thinks the setting is per server | "Does that setting take a server name? What are its possible values, and what happens to the other server?" |
| "d, Always allow is scoped to the workspace" | Mixes up the approval scopes | "There is a separate choice whose name contains the word Workspace. What does that tell you about the scope of Always allow?" |
| "a is wrong because deselecting in the picker is not persistent" | Thinks requirement two needs a permanent change | "Requirement two says for this session. Does the tools picker act on the session?" |
| "a is wrong because the MSSQL tools are MCP tools and would need their own server entry" | Does not know MSSQL tools are extension-contributed | "Where do the MSSQL tools come from: an MCP server in mcp dot json, or the extension itself?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain that the question is really four controls that live in the chat session, plus one file:

- **The mode picker decides whether tools run at all.** Ask answers and proposes code but does not edit files or run commands. Edit applies multi-file edits to the files you put in context, without a tool loop. Agent decides which files to change, calls built-in tools, extension-contributed tools and MCP tools, runs terminal commands, and iterates until the task is done. Newer builds add Plan, which researches with read-only tools and produces a plan without changing anything. The MSSQL documentation says agent mode picks up the MSSQL tools automatically, no at-mssql mention needed, and that every tool call requires approval before execution. That is why b fails requirement 1.
- **The tools picker decides which tools this session can see.** The Configure Tools button in the chat input lists every tool grouped by source: built-in, extension tools such as the MSSQL set, and each MCP server. Deselecting the shell-tools server removes run underscore shell from this session without touching the shared file, which is exactly what requirement 2 asks. A request may have at most 128 tools enabled. Tools, servers and tool sets can be referenced with a hash prefix, and related tools can be grouped in a dot toolsets dot jsonc file and referenced as hash name.
- **The approval scope decides how often a human looks at a call.** The confirmation dialog offers a single use, Allow in this Session, Allow in this Workspace, and Always allow. Always allow means all future invocations everywhere, which is why d fails requirement 3. Saved approvals can be reviewed with Chat: Manage Tool Approval and cleared with Chat: Reset Tool Confirmations. The setting chat dot tools dot global dot autoApprove, toggled from chat with slash yolo or slash autoApprove, is documented as disabling critical security protections; it approves every tool, including any that reach the session later, which is why c fails requirement 3. A read-only SQL login limits the damage a query can do, but it does not restore the per-call review, and it does nothing for non-SQL tools.
- **The MSSQL tools are extension tools, not an MCP server.** The extension contributes connect, disconnect, change database, get connection details, list servers, list databases, show schema, list schemas, list tables, list views, list functions and run query, exposed in chat as hash mssql underscore connect and so on. They use the same connection and credentials as the extension, so they appear in the tools picker regardless of any MCP setting. That is why option d would look half-working: the MSSQL tools survive while halden-docs disappears.
- **chat dot mcp dot access is all-or-nothing.** Its values are all, registry and none, and it sits behind the enterprise ChatMCP policy. none switches off every MCP server; a hash halden-docs reference cannot start a server that is not allowed to run.
- **Secrets stay out of the repository through inputs.** An entry in the inputs array with type promptString and password true makes Visual Studio Code prompt, with hidden typing, when the server starts, and substitute the value into the dollar input placeholder. The committed file holds only the placeholder. The documentation says to avoid hard-coding API keys and to use input variables or environment files instead. That is why c fails requirement 4.
- **The model picker decides what each prompt costs.** The model can be changed at any prompt; Chat: Manage Language Models shows every model with its capabilities and billing. On the legacy premium-request plans each model carries a multiplier: small models such as GPT-5 mini, Claude Haiku 4.5 or GPT-4o mini at 0.33x, reasoning models higher. Only the prompts you send count as premium requests; the tool calls Copilot makes autonomously inside an agent run do not. So a long run with dozens of run query calls is one request at the chosen model's multiplier, and the cost lever is the model per prompt, not the number of tools.

Memory hook: "Mode says whether tools run. Picker says which tools. Approval scope says how often a human looks. Model picker says what it costs. Never trade the approval scope for convenience, and never trade inputs for a hard-coded secret."

## 9. Follow-up oral questions (optional)

1. "If the developer wanted the shell-tools exclusion to be reusable across sessions without editing mcp dot json, what could they create?" (A tool set in a dot toolsets dot jsonc file containing only the MSSQL and halden-docs tools, referenced as hash name.)
2. "A long agent run makes forty mssql run query calls after one prompt. How many premium requests is that on a legacy plan?" (One. Only the prompt counts; autonomous tool calls do not.)
3. "What is the difference between Allow in this Workspace and Always allow?" (Workspace scope applies to the current repository only; Always allow applies to all future invocations in every workspace.)

## 10. References

- Use agent mode in Visual Studio Code: https://code.visualstudio.com/docs/copilot/chat/chat-agent-mode
- Chat modes in Visual Studio Code, Ask, Edit and Agent: https://code.visualstudio.com/docs/copilot/chat/copilot-chat
- Use MCP servers in Visual Studio Code, including mcp dot json, inputs and chat dot mcp dot access: https://code.visualstudio.com/docs/copilot/chat/mcp-servers
- Tool approval and tool sets in agent mode: https://code.visualstudio.com/docs/copilot/chat/chat-agent-mode#_manage-tool-approvals
- Language models and the model picker: https://code.visualstudio.com/docs/copilot/language-models
- MSSQL extension, GitHub Copilot agent mode and the mssql tools: https://learn.microsoft.com/en-us/sql/tools/visual-studio-code-extensions/mssql/mssql-copilot-agent-mode
- MSSQL extension, GitHub Copilot overview: https://learn.microsoft.com/en-us/sql/tools/visual-studio-code-extensions/mssql/github-copilot-overview
- Premium requests and model multipliers: https://docs.github.com/en/copilot/concepts/billing/copilot-requests
- Enterprise policy ChatMCP and other Visual Studio Code policies: https://code.visualstudio.com/docs/enterprise/enterprise-support
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
