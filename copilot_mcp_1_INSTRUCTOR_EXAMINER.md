# Instructor-Examiner guide — GitHub Copilot and MCP 1

Companion to [copilot_mcp_1.md](copilot_mcp_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Read all five requirements and all four options before taking an answer. Three options contain a JSON file and one contains a Markdown file; describe each file in words, naming the file path, the top-level keys and the values that matter, exactly as in section 2. Options a, b and c share most of the JSON, so stress the differences.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Implement AI capabilities in database solutions (25–30%).
- Skill: Use AI-assisted development tools.
- Task bullet: Configure GitHub Copilot agent mode with the Microsoft SQL Server MCP server in Visual Studio Code.
- What is tested: where a team-shared MCP server definition lives, how the inputs mechanism keeps a password out of the repository, which layer actually stops a prompt-injected destructive statement, and what a custom instructions file can and cannot do.

## 2. Scenario to read aloud

**Piece 1, the story.** "The data engineering team at Meridian Analytics develops T-SQL in Visual Studio Code by using GitHub Copilot in agent mode. The team wants Copilot to work directly against the development database MeridianDev, on the server dev dash sql zero one, through the Microsoft SQL Server MCP server. That MCP server runs locally as a stdio process. It is stored in the repository at dot slash tools slash mssql dash mcp slash index dot js, and it reads its SQL connection string from the environment variable CONNECTION underscore STRING."

**Piece 2, requirements one to three.** "Five requirements. One: the MCP server definition must live in the Git repository, so every developer who clones it and opens the workspace gets the same server definition automatically. Two: the SQL authentication password must not be committed to the repository in plaintext. Three: through the MCP tools, Copilot must be able to list tables, read schema metadata, and run SELECT validation queries against MeridianDev."

**Piece 3, requirements four and five.** "Four: prompt injection is an accepted threat. Text stored in table data or in fetched documents can reach the model through tool output and can try to make the agent issue a destructive statement, for example DROP TABLE or DELETE. If that happens, the statement must fail at the database engine. The protection must not depend on a developer noticing the statement in an approval dialog. Five: developers must still review and approve each MCP tool invocation individually by default."

**Piece 4, option a.** "Option a commits a file at dot vscode slash mcp dot json. The file has two top-level keys. First, inputs: an array with one input, id mssql dash password, type promptString, description Password for the mssql underscore copilot underscore dev SQL login, and password set to true. Second, servers: one server named meridian dash mssql, type stdio, command node, args with one element, dot slash tools slash mssql dash mcp slash index dot js, and an env object with CONNECTION underscore STRING equal to Server dev dash sql zero one, Database MeridianDev, User Id mssql underscore copilot underscore dev, Password equal to the placeholder dollar open brace input colon mssql dash password close brace, TrustServerCertificate True. In MeridianDev, create the dedicated login and user mssql underscore copilot underscore dev, add it to the db underscore datareader fixed database role, and grant it VIEW DEFINITION. Grant it no DDL or data-modification permissions. Keep the default tool-approval behavior: do not enable chat dot tools dot global dot autoApprove, and do not select a broad auto-approval scope in the confirmation dialog."

**Piece 5, option b.** "Option b commits the same dot vscode slash mcp dot json as option a, with one difference in the env value: the User Id is dev underscore admin instead of mssql underscore copilot underscore dev. The dev underscore admin user is a member of the db underscore owner fixed database role, so Copilot can help with any future task without permission changes. Keep the default tool-approval behavior. The claim is that, because Visual Studio Code shows each tool invocation with its input parameters before it runs, a developer will reject any injected destructive statement in the confirmation dialog, so the elevated permissions are safe."

**Piece 6, option c.** "Option c has each developer run the command MCP colon Open User Configuration and add the server to the mcp dot json file in their own user profile. The JSON is identical to option a: same inputs entry with promptString and password true, same server meridian dash mssql with stdio, node, the index dot js path, and the connection string with User Id mssql underscore copilot underscore dev and the password placeholder. Use the same least-privilege login as in option a, and document the setup steps in the repository README so every developer can reproduce the configuration."

**Piece 7, option d.** "Option d commits a Markdown file at dot github slash copilot dash instructions dot md. The text says: Copilot instructions for Meridian Analytics. Register the following MCP server for this repository and use it for all database work. Command: node dot slash tools slash mssql dash mcp slash index dot js. Environment: CONNECTION underscore STRING equal to Server dev dash sql zero one, Database MeridianDev, User Id mssql underscore copilot underscore dev, Password, ask the developer, TrustServerCertificate True. You may list tables, read schema metadata, and run SELECT queries. Never run DROP, DELETE, UPDATE, or INSERT statements. The claim is that, because this file is automatically included in every chat request, Copilot registers the MCP server for the whole team, and the closing instruction guarantees that destructive statements are never executed."

## 3. Setup script (reference only; do not read verbatim unless asked)

Option a and option c JSON (option a at `.vscode/mcp.json`, option c in the user-profile `mcp.json`):

```json
{
    "inputs": [
        {
            "id": "mssql-password",
            "type": "promptString",
            "description": "Password for the mssql_copilot_dev SQL login",
            "password": true
        }
    ],
    "servers": {
        "meridian-mssql": {
            "type": "stdio",
            "command": "node",
            "args": ["./tools/mssql-mcp/index.js"],
            "env": {
                "CONNECTION_STRING": "Server=dev-sql01;Database=MeridianDev;User Id=mssql_copilot_dev;Password=${input:mssql-password};TrustServerCertificate=True"
            }
        }
    }
}
```

Option b, the only changed line:

```json
"CONNECTION_STRING": "Server=dev-sql01;Database=MeridianDev;User Id=dev_admin;Password=${input:mssql-password};TrustServerCertificate=True"
```

Option d, `.github/copilot-instructions.md`:

```markdown
# Copilot instructions for Meridian Analytics

Register the following MCP server for this repository and use it for all
database work:

- command: node ./tools/mssql-mcp/index.js
- environment: CONNECTION_STRING=Server=dev-sql01;Database=MeridianDev;User Id=mssql_copilot_dev;Password=<ask the developer>;TrustServerCertificate=True

You may list tables, read schema metadata, and run SELECT queries.
Never run DROP, DELETE, UPDATE, or INSERT statements.
```

Option a database side, in words: create login and user mssql_copilot_dev, ALTER ROLE db_datareader ADD MEMBER mssql_copilot_dev, GRANT VIEW DEFINITION TO mssql_copilot_dev. No INSERT, UPDATE, DELETE, ALTER or CONTROL.

## 4. The question (ask exactly this)

"Which approach should the team use?

a. Commit dot vscode slash mcp dot json with an inputs entry, id mssql dash password, type promptString, password true, and a servers entry meridian dash mssql, type stdio, command node, args dot slash tools slash mssql dash mcp slash index dot js, env CONNECTION underscore STRING with User Id mssql underscore copilot underscore dev and Password equal to the input placeholder. In MeridianDev, create login and user mssql underscore copilot underscore dev, add it to db underscore datareader, grant VIEW DEFINITION, and no DDL or data-modification permissions. Keep default tool approval: do not enable chat dot tools dot global dot autoApprove and do not pick a broad auto-approval scope.

b. Commit the same file, but with User Id dev underscore admin, a db underscore owner member, so Copilot can help with any future task. Keep default tool approval; because Visual Studio Code shows each invocation with its parameters, a developer will reject any injected destructive statement, so the elevated permissions are safe.

c. Have each developer run MCP colon Open User Configuration and add the same JSON as option a to their user-profile mcp dot json, with the same least-privilege login, and document the steps in the README.

d. Commit dot github slash copilot dash instructions dot md describing the server command and the connection string with the password to be asked from the developer, allowing list, schema and SELECT, and saying never run DROP, DELETE, UPDATE or INSERT. Because the file is included in every chat request, Copilot registers the server for the team and the closing instruction guarantees destructive statements never execute.

Which letter, and why do the other three fail?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

| Option | Verdict | Why |
|---|---|---|
| a | Correct | Workspace file dot vscode slash mcp dot json under the servers key is shared through the repo (req 1). inputs with promptString and password true keeps the secret out; the placeholder is substituted at server start (req 2). db_datareader plus VIEW DEFINITION covers listing, metadata and SELECT (req 3). No INSERT, UPDATE, DELETE, ALTER or CONTROL, so an injected DROP or DELETE fails with a permission error at the engine, regardless of approval (req 4). Default per-invocation approval kept, autoApprove not enabled (req 5). |
| b | Wrong | Configuration mechanics are right; the security model is wrong. db_owner can drop, delete and alter anything, so an injected DROP TABLE succeeds at the engine. The approval dialog is not a security boundary: injected instructions can be hidden in comments or formatting in tool output, and a developer can auto-approve a tool for the session, workspace or all future calls. Requirement 4 fails. |
| c | Wrong | Correct JSON and correct login, but placed in the user-profile mcp.json via MCP: Open User Configuration. That applies to one developer's profile and is not in the repository, so a fresh clone gets no server definition. Copies drift. Requirement 1 fails. |
| d | Wrong | dot github slash copilot dash instructions dot md is a custom instructions file: natural-language context added to every chat request. It cannot register an MCP server, launch a process or set environment variables. No server, no tools, requirement 3 fails. The never-run line lives in the prompt layer, exactly what injection attacks, so requirement 4 fails too. |

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with requirement one. Where does Visual Studio Code look for a workspace MCP server definition that travels with the repository: a file under dot vscode, or the user profile?"
2. "Now option d. What kind of file is dot github slash copilot dash instructions dot md? Does Visual Studio Code parse it to start processes, or does it just add text to the prompt?"
3. "Requirement four says the statement must fail at the database engine, and must not depend on a developer noticing. Which layer decides what the engine allows: the approval dialog, the instructions file, or the permissions of the login in the connection string?"
4. "Compare options a and b. They differ in one thing: the User Id. One login is db underscore owner, the other is db underscore datareader with VIEW DEFINITION. If an injected DROP TABLE is approved by mistake, which login makes it fail?"
5. "Only one option keeps the file in the workspace, keeps the password out through inputs, uses a read-only login and keeps default approval. Which letter?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "Option b, the developer sees every call before it runs" | Treats the approval dialog as the security boundary | "Can a tool be auto-approved for the session or the workspace? And can an injected instruction be hidden in a comment inside tool output? What then stops the statement?" |
| "Option b, db owner is needed for future tasks" | Ignores least privilege | "Requirement three lists exactly what Copilot must do. Does any of it need db underscore owner?" |
| "Option c, user config is more secure" | Confuses secret handling with definition location | "The password is handled the same way in a and c. What differs is where the definition lives. Does a new clone get it?" |
| "Option c, the README makes it reproducible" | Thinks documentation equals shared configuration | "Requirement one says automatically on clone. Is copying JSON from a README automatic?" |
| "Option d, instructions are included in every request" | Believes an instructions file can register tools | "Included as what: executable configuration, or prose the model reads? Can prose start a node process?" |
| "Option d, the never-run line prevents DROP" | Thinks prompt-level rules are enforced | "Where does prompt injection attack: the engine, or the prompt context? Which of the two is the never-run line in?" |
| "Option a, but the password is still in the file" | Does not know the inputs placeholder | "Read the Password value in the committed file. Is it a value, or a dollar input reference resolved at start time?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the three independent concepts:

- **Where shared MCP configuration lives.** Visual Studio Code reads workspace MCP servers from dot vscode slash mcp dot json, under the top-level servers key. Because it is inside the repository, every clone gets it. The user-profile mcp dot json, opened with MCP colon Open User Configuration, applies to one developer's profile across their workspaces and is not versioned. That is why option c fails requirement 1.
- **How secrets stay out of the file.** The inputs array defines an input with type promptString and password true. The committed file holds only the placeholder dollar open brace input colon mssql dash password close brace. Visual Studio Code prompts each developer with hidden typing and substitutes the value into the CONNECTION underscore STRING environment variable when the server starts.
- **Which layer stops an injected destructive statement.** The MCP server runs every statement as the principal in its connection string. What can happen equals the database permissions of that login. What does happen equals whichever invocations get approved or auto-approved. db underscore datareader gives SELECT on all tables and views, VIEW DEFINITION exposes metadata, and with no INSERT, UPDATE, DELETE, ALTER or CONTROL, an injected DROP TABLE or DELETE is rejected by SQL Server with a permission error, even if approved blindly, even if the tool was auto-approved earlier. That is requirement 4, and why option b fails: db underscore owner lets the injected statement succeed.
- **Why the dialog is not the boundary.** Injected instructions can be hidden in comments or obscured through formatting inside tool output, and a destructive operation can be buried in an innocuous-looking call. The confirmation dialog also lets a developer auto-approve for the session, the workspace, or all future invocations. chat dot tools dot global dot autoApprove disables the prompts entirely. Option a keeps the default and avoids both. That is requirement 5, defense in depth.
- **Custom instructions versus MCP configuration.** dot github slash copilot dash instructions dot md is natural-language context added to every chat request. It shapes style and conventions. It cannot register a server, launch a process or set environment variables, and a never-run line is just prompt text that a stronger injected instruction can override. That is why option d fails requirements 3 and 4.

Memory hook: "mcp dot json registers tools, copilot dash instructions dot md only talks. Permissions decide what can happen; approvals decide what does happen. Least privilege on the connection makes the engine say no."

## 9. Follow-up oral questions (optional)

1. "What are the two top-level keys in an mcp dot json file, and what does each hold?" (servers, the server definitions; inputs, prompted variables such as a promptString with password true.)
2. "Which Visual Studio Code setting would silence all tool confirmations, and why is it a bad idea here?" (chat dot tools dot global dot autoApprove; it removes the human review layer that requirement 5 demands.)
3. "Which role and which permission give the read-only login what requirement three needs?" (db underscore datareader for SELECT on tables and views, plus VIEW DEFINITION for schema metadata.)

## 10. References

- MCP servers in Visual Studio Code, including .vscode/mcp.json, inputs and approval: https://code.visualstudio.com/docs/copilot/chat/mcp-servers
- Agent mode in Visual Studio Code, tool approval and auto-approve settings: https://code.visualstudio.com/docs/copilot/chat/chat-agent-mode
- Custom instructions for GitHub Copilot in Visual Studio Code: https://code.visualstudio.com/docs/copilot/copilot-customization
- Microsoft SQL Server MCP server sample: https://github.com/Azure-Samples/SQL-AI-samples/tree/main/MssqlMcp
- Database-level roles, including db_datareader: https://learn.microsoft.com/en-us/sql/relational-databases/security/authentication-access/database-level-roles
- GRANT database permissions, including VIEW DEFINITION: https://learn.microsoft.com/en-us/sql/t-sql/statements/grant-database-permissions-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
