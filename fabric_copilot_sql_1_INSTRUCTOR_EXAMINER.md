# Instructor-Examiner guide — Copilot for SQL in Fabric 1

Companion to [fabric_copilot_sql_1.md](fabric_copilot_sql_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Lab rule.** This is a hands-on Microsoft Fabric lab question. Before reading the scenario, ask: "Have you already run this lab from your own Fabric account?" If yes, ask what they observed at each step (what the Copilot pane showed on the trial capacity, which tenant settings were on or off by default, what the execution mode selector did, when Fix became enabled, whether the warehouse Copilot remembered the previous prompt) before you quiz them. If no, walk through the steps in words using section 2, so that the question can still be answered from the documented facts alone. Do not require the learner to run anything during the call.

**Multiple choice.** This is a multiple-choice question with four options, a to d. Read all four options before taking an answer. The learner must pick one letter. Take the letter, then say only right or wrong.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Configure and secure the database engine (15–20%); also Design and develop database solutions (35–40%).
- Skill: Configure Microsoft Fabric tenant and workspace settings for AI features; use Copilot for SQL.
- Task bullet: Enable Copilot for a SQL database and a warehouse in Fabric; control what Copilot may execute; describe what data Copilot receives.
- What is tested: capacity requirements for Copilot, the tenant settings in the Copilot and AI group and which one is needed outside the US and EU, the execution modes of the SQL database chat pane, where completions are switched off, and the privacy statement.

## 2. Scenario to read aloud

**Piece 1, the story.** "Orchard Coop is a fruit-growers' cooperative. Its member CRM is a SQL database in Fabric called OrchardCrm. Its harvest facts are in a Fabric Data Warehouse called OrchardDW. Both live in the workspace ws dash dp800 dash orchard. The Fabric tenant's home region is Canada Central. You enable and exercise Copilot on both items, then pick the plan that satisfies the cooperative's rules."

**Piece 2, prerequisites.** "You are a Fabric tenant admin, or can ask one, and a workspace Admin. Copilot needs a paid capacity, F2 or larger, or P1 or larger. To see it work you assign the workspace to a paid capacity at least for the exercise. The observation steps on a trial capacity are part of the lab."

**Piece 3, building the lab.** "You create the workspace ws dash dp800 dash orchard on your Trial capacity for now. You create the SQL database OrchardCrm and run a script: schema Crm, table Crm dot Grower with GrowerID primary key, GrowerName and Region; and table Crm dot Delivery with DeliveryID primary key, GrowerID as a foreign key, Fruit, Kg as a decimal, and DeliveredOn a date. Three growers: Hillcrest Farm in Okanagan, Bright Acres in Niagara, Cedar Row in Okanagan. Four deliveries: apples from grower 1, peaches from grower 2, apples from grower 3, pears from grower 1. Then you create the warehouse OrchardDW with one table dbo dot HarvestFact, two rows, Okanagan apples and Niagara peaches."

**Piece 4, tenant settings.** "In the Admin portal, Tenant settings, group Copilot and AI, you record the state of five settings. One: Users can use Copilot and other features powered by Azure OpenAI. Two: Data sent to Azure OpenAI can be processed outside your capacity's geographic region, compliance boundary, or national cloud instance. Three: the same sentence but with stored instead of processed. Four: Users can use Copilot, AI Agents and other AI experiences powered by OpenAI as a Microsoft Subprocessor. Five: Capacities can be designated as Copilot in Fabric capacities. For each you note whether it can be applied to specific security groups and whether it can be delegated to capacity admins."

**Piece 5, exercising Copilot in the database.** "You open OrchardCrm, New query, and select the Copilot ribbon button while the workspace is still on the trial capacity, and observe the pane. Then you move the workspace to a paid capacity through Workspace settings, Workspace type, Edit, Fabric capacity, and reopen the pane. At the bottom of the chat pane there is an execution mode selector with two options: Read-only, and Read and write with approval. In Read-only mode you type show the total kilograms delivered per region, and observe whether a query runs. Then you type create a table Crm dot Payout with PayoutID int and Amount decimal, and observe. You switch the mode and repeat that second prompt."

**Piece 6, quick actions and completions.** "In the editor you paste a query with deliberate mistakes: it selects GrowerName and SUM of Kg from Delivery joined to Grower on GrowerID, with a GROUP BY Region, so the grouping does not match the select list. You highlight it and look at the Explain and Fix buttons next to Run. You select Fix before running, then Run, then Fix again. Next, in the database Settings, Copilot pane, you find Show Copilot completions, and you look at the status bar at the bottom of the editor. You type a comment, dash dash list growers with more than 1000 kg delivered, wait for ghost text, and press Tab."

**Piece 7, the warehouse and the endpoint.** "You open OrchardDW, New SQL query, select Copilot on the ribbon, ask count deliveries by fruit, then ask now only for Okanagan, and observe whether the second prompt is understood in context. You repeat the Explain and Fix exercise in the warehouse. Finally you open the SQL analytics endpoint of OrchardCrm and check whether a Copilot button exists there too."

**Piece 8, the cooperative's four rules.** "Rule 1: Copilot must be available in OrchardCrm and OrchardDW only to the security group sg dash orchard dash analysts, in the Canada Central capacity. Rule 2: in the OrchardCrm chat pane, analysts may let Copilot run SELECT queries, but Copilot must never execute DDL or DML. Rule 3: inline ghost-text completions must be switched off in OrchardCrm, while the chat pane, Explain and Fix stay available. Rule 4: the compliance officer needs a one-paragraph statement of what leaves the tenant when an analyst uses Copilot in OrchardCrm."

**Piece 9, option a.** "Keep the workspace on the paid F2 or larger capacity, because the trial SKU does not support Copilot. In Tenant settings, Copilot and AI, leave Users can use Copilot and other features powered by Azure OpenAI enabled but apply it only to sg dash orchard dash analysts, and enable Data sent to Azure OpenAI can be processed outside your capacity's region, also scoped to the group, because Canada Central is outside the US and the EU data boundary. In the OrchardCrm chat pane set the execution mode to Read-only: SELECT prompts still run automatically, while CREATE TABLE is only drafted. Turn off Show Copilot completions in the database Settings, Copilot pane. The statement says: Copilot receives the prompt, the session's previous messages, the text and error messages of queries the analyst executed, and the database schema; it has no access to table data; prompts are not used to train foundation models; and because of the second setting, requests may be processed by Azure OpenAI outside the capacity's region."

**Piece 10, option b.** "Leave the workspace on the trial capacity, which is an F64 equivalent and therefore satisfies the F2 or higher requirement, and enable Users can use Copilot and other features powered by Azure OpenAI for the group. No cross-region setting is needed, because Copilot always processes requests in the capacity's own region. Meet rule 2 by choosing Read and write with approval, since in Read-only mode Copilot cannot run any query, including SELECT. Meet rule 3 by disabling completions from the same tenant setting."

**Piece 11, option c.** "Same capacity and group scoping as option a, but enable the stored outside setting instead of the processed outside setting, claiming it is the one that unblocks Copilot for the SQL database outside the US and EU. Meet rule 2 with Read and write with approval so the analyst can approve each SELECT. Meet rule 3 by removing the analyst's Write permission on OrchardCrm, which hides the completion feature."

**Piece 12, option d.** "Disable Users can use Copilot and other features powered by Azure OpenAI for the whole tenant, and instead enable Users can use Copilot, AI Agents and other AI experiences powered by OpenAI as a Microsoft Subprocessor for the group; this routes the SQL database Copilot to OpenAI-operated models without any cross-region setting. Meet rule 2 with Read-only mode. Meet rule 3 by turning off Show Copilot completions, which also disables the Fix and Explain quick actions since they share the completion engine."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
-- OrchardCrm (SQL database in Fabric)
CREATE SCHEMA Crm;
GO
CREATE TABLE Crm.Grower   (GrowerID int PRIMARY KEY, GrowerName varchar(80) NOT NULL, Region varchar(40) NOT NULL);
CREATE TABLE Crm.Delivery (DeliveryID int PRIMARY KEY, GrowerID int NOT NULL REFERENCES Crm.Grower(GrowerID),
                           Fruit varchar(30) NOT NULL, Kg decimal(9,2) NOT NULL, DeliveredOn date NOT NULL);
INSERT INTO Crm.Grower VALUES (1,'Hillcrest Farm','Okanagan'), (2,'Bright Acres','Niagara'), (3,'Cedar Row','Okanagan');
INSERT INTO Crm.Delivery VALUES (10,1,'Apple',1200.50,'2026-08-20'), (11,2,'Peach',430.00,'2026-08-21'),
                                (12,3,'Apple',980.25,'2026-08-22'), (13,1,'Pear',300.00,'2026-08-23');

-- OrchardDW (Warehouse)
CREATE TABLE dbo.HarvestFact (HarvestID int NOT NULL, Region varchar(40) NOT NULL, Fruit varchar(30) NOT NULL,
                              Kg decimal(9,2) NOT NULL, HarvestDate date NOT NULL);
INSERT INTO dbo.HarvestFact VALUES (1,'Okanagan','Apple',5000.00,'2026-08-15'), (2,'Niagara','Peach',2100.00,'2026-08-16');

-- Step 7: the deliberately wrong query pasted into the editor
SELECT GrowerName, SUM(Kg) FROM Crm.Delivery d JOIN Crm.Grower g ON g.GrowerID = d.GrowerId GROUP BY Region

-- Step 8: the comment typed to trigger ghost text
-- list growers with more than 1000 kg delivered
```

Prompts used in the chat pane: `show the total kilograms delivered per region`; `create a table Crm.Payout with PayoutID int and Amount decimal(9,2)`; in the warehouse `count deliveries by fruit` then `now only for Okanagan`.

## 4. The question (ask exactly this)

"Which plan is correct? Option a, option b, option c, or option d?"

- **a.** Keep the workspace on the paid F2 (or larger) capacity — the trial SKU does not support Copilot. In Tenant settings > Copilot and AI, leave "Users can use Copilot and other features powered by Azure OpenAI" enabled but apply it to sg-orchard-analysts only, and enable "Data sent to Azure OpenAI can be processed outside your capacity's geographic region, compliance boundary, or national cloud instance" (also scoped to the group), because Canada Central is outside the US and the EU data boundary. In the OrchardCrm chat pane set the execution mode to Read-only: SELECT prompts still run automatically, while CREATE TABLE is only drafted. Turn off "Show Copilot completions" in the database Settings > Copilot pane. Statement: Copilot receives the prompt, the session's previous messages, the text and error messages of queries the analyst executed, and the database schema; it has no access to table data, prompts are not used to train foundation models, and, because of the second setting, requests may be processed by Azure OpenAI outside the capacity's region.
- **b.** Leave the workspace on the trial capacity, which is an F64-equivalent and therefore satisfies the "F2 or higher" requirement, and enable "Users can use Copilot and other features powered by Azure OpenAI" for sg-orchard-analysts. No cross-region setting is needed because Copilot always processes requests in the capacity's own region. Meet rule 2 by choosing Read and write with approval, since in Read-only mode Copilot cannot run any query, including SELECT. Meet rule 3 by disabling completions from the same tenant setting.
- **c.** Same capacity and group scoping as option a, but enable "Data sent to Azure OpenAI can be stored outside your capacity's geographic region, compliance boundary, or national cloud instance" instead of the processed setting; it is the one that unblocks Copilot for the SQL database outside the US and EU. Meet rule 2 with Read and write with approval so that the analyst can approve each SELECT. Meet rule 3 by removing the analyst's Write permission on OrchardCrm, which hides the completion feature.
- **d.** Disable "Users can use Copilot and other features powered by Azure OpenAI" for the whole tenant, and instead enable "Users can use Copilot, AI Agents and other AI experiences powered by OpenAI as a Microsoft Subprocessor" for sg-orchard-analysts; this routes the SQL database Copilot to OpenAI-operated models without any cross-region setting. Meet rule 2 with Read-only mode. Meet rule 3 by turning off the "Show Copilot completions" switch, which also disables the Fix and Explain quick actions since they share the completion engine.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

| Option | Why wrong |
|---|---|
| b | Trial SKUs do not support Copilot at all, whatever their size. Outside the US and EU boundary the processed outside setting is required. Read-only mode does run SELECT automatically, and Read and write with approval would let Copilot execute DDL after a click. There is no tenant switch for completions; it is a per-database setting. |
| c | The stored outside setting only applies to Copilot in notebooks and data agents; the SQL database needs the processed outside setting. Read and write with approval is the mode where DDL and DML can be executed, the opposite of rule 2, and SELECT needs no approval in either mode. Removing Write permission changes what the analyst can do in the database, not whether ghost text appears. |
| d | The subprocessor setting is a separate, off-by-default model provider with its own cross-region companion, not a replacement for the Azure OpenAI switch that Copilot for SQL database depends on. Disabling the Azure OpenAI switch tenant-wide turns Copilot off for everyone. Fix and Explain are independent quick actions; the completions toggle does not disable them. |

Documented facts the lab shows: Copilot is not supported on trial SKUs, only paid F2 or higher, P1 or higher. Default states: Azure OpenAI switch ON, Capacities can be designated ON, both processed outside settings OFF, stored outside OFF, both subprocessor settings OFF. Every setting can be scoped to security groups; the Azure OpenAI switch and the subprocessor switch can be managed at tenant and capacity level. In Read-only mode Copilot does not run DDL or DML, it suggests the code for you; a SELECT prompt is generated and run automatically regardless of mode. Fix is enabled only after a run returned an error and takes the error message into context. Show Copilot completions is a per-database switch mirrored in the status bar; Tab accepts, Ctrl plus Right accepts word by word. Warehouse Copilot covers warehouse and SQL analytics endpoint, does not remember previous prompts, and does not use table data.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with rule 1 and the capacity. Which SKUs does Copilot support? Is a trial one of them, whatever its size?"
2. "Still rule 1. Canada Central is outside the US and the EU boundary. Which of the five tenant settings is documented as required in that case: the one about processing, or the one about storing?"
3. "Now rule 2. In the lab, in Read-only mode, did the SELECT prompt run? Did the CREATE TABLE prompt run? Which mode is the one where DDL can actually be executed?"
4. "Rule 3. Where did you find the Show Copilot completions switch: in the tenant settings, in item permissions, or in the database's own Settings? And did switching it off touch Explain and Fix?"
5. "One option is built around the wrong regional setting, and another around the wrong model provider. Eliminate the option that switches off the Azure OpenAI setting for the whole tenant."
6. "Eliminate the option that keeps the trial capacity. Two options remain; compare their choice of regional setting and their execution mode against rules 1 and 2."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "b, the trial is an F64, that is more than F2" | Confuses trial size with trial SKU support | "Size is not the criterion. What did the Copilot pane show while the workspace was still on the trial?" |
| "c, the stored outside setting is the one" | Mixes the two cross-region settings | "One of those settings governs conversation history for notebooks and data agents. Which one is about processing a request?" |
| "d, the OpenAI subprocessor setting is the modern replacement" | Thinks the subprocessor switch supersedes the Azure OpenAI switch | "Are those two settings alternatives for the same feature, or two different model providers, each with its own switch?" |
| "Read and write with approval is safer because the analyst approves" | Misreads what approval permits | "Approval gates what? If the analyst approves a CREATE TABLE, who executes it? Does rule 2 allow that?" |
| "Read-only means Copilot cannot run anything" | Ignores the lab observation | "In step 6, in Read-only mode, what happened when you asked for total kilograms per region?" |
| "Completions are turned off in tenant settings" | Wrong scope for the switch | "Which page had the Show Copilot completions switch? Was it tenant-wide, or per database?" |
| "Removing Write hides ghost text" | Confuses item permission with feature toggle | "Permissions control what the analyst may do to the data. What controls whether Copilot shows ghost text?" |
| "Turning off completions also disables Fix and Explain" | Assumes shared engine | "In step 8, after you turned completions off, were the Explain and Fix buttons still next to Run?" |

## 8. Teaching notes (after the answer is complete or revealed)

Walk the four rules in order.

- **Rule 1, capacity and tenant settings.** Copilot in Fabric is not supported on trial SKUs; only paid F2 or higher and P1 or higher. The trial page itself lists Copilot as unsupported. In the Admin portal, Tenant settings, Copilot and AI: Users can use Copilot and other features powered by Azure OpenAI is ON by default, can be scoped to security groups, and can be delegated to capacity admins. Data sent to Azure OpenAI can be processed outside your capacity's region is OFF by default and is required when the tenant or capacity is outside the US or the EU boundary, otherwise Copilot is disabled. The stored outside setting is only for Copilot in notebooks and data agents. The OpenAI as a Microsoft Subprocessor setting is a different, off-by-default model provider with its own cross-region companion.
- **Rule 2, execution modes.** The SQL database chat pane has an execution mode selector. In Read-only mode Copilot does not run DDL or DML; it suggests the code for you to review and run. A SELECT prompt is generated and run automatically regardless of mode. In Read and write with approval, Copilot drafts the DDL or DML and prompts you to approve execution, so Copilot can execute it. Rule 2 therefore maps to Read-only.
- **Rule 3, completions versus quick actions.** Show Copilot completions is a per-database switch under Settings, Copilot, mirrored in the query editor status bar. Completions use the schema and query tab context, are accepted with Tab, or word by word with Ctrl plus Right, and cover DDL, DQL and DML. Explain and Fix are quick actions next to Run: Explain works on any highlighted text and writes a summary plus inline comments; Fix is enabled only after a run returned an error and takes the error message into context. The completions toggle does not affect them.
- **Rule 4, the data statement.** In the SQL database, Copilot can only access the database schema accessible to the user. By default it also sees previous messages and replies in that session, the contents and error messages of queries the user executed. It has no access to table data. Prompts and responses are not used to train foundation models. With the processed outside setting on, requests may be processed outside the capacity's region.
- **Warehouse Copilot.** Applies to the warehouse and the SQL analytics endpoint, so the button exists on the endpoint too. It does not understand previous inputs, so now only for Okanagan is not resolved against the previous answer. Natural language to SQL supports English only. It does not use table data.

Memory hook: "Paid capacity, Azure OpenAI switch scoped to the group, processed outside for non-US non-EU, Read-only lets SELECT run, completions are a database setting, Fix needs an error first."

## 9. Follow-up oral questions (optional)

1. "The cooperative opens a second tenant in the United States. Which tenant setting from option a becomes unnecessary?" (The processed outside setting; inside the US it is not required.)
2. "An analyst says the Fix button is greyed out. What is the most likely reason?" (Fix only becomes enabled after the query has been run and returned an error.)
3. "In the warehouse, an analyst asks a follow-up prompt that depends on the previous one and gets a wrong answer. Why?" (The warehouse Copilot does not remember previous inputs; each prompt must be self-contained.)

## 10. References

- Copilot and AI tenant settings in the Fabric admin portal: https://learn.microsoft.com/en-us/fabric/admin/service-admin-portal-copilot
- Enable and configure Copilot in Fabric: https://learn.microsoft.com/en-us/fabric/fundamentals/copilot-enable-fabric
- Copilot in the SQL database workload, overview: https://learn.microsoft.com/en-us/fabric/database/sql/copilot
- Copilot chat pane in SQL database: https://learn.microsoft.com/en-us/fabric/database/sql/copilot-chat-pane
- Copilot quick actions in SQL database: https://learn.microsoft.com/en-us/fabric/database/sql/copilot-quick-actions
- Copilot code completion in SQL database: https://learn.microsoft.com/en-us/fabric/database/sql/copilot-code-completion
- Privacy, security and responsible use of Copilot in SQL database: https://learn.microsoft.com/en-us/fabric/database/sql/copilot-privacy-security
- Copilot in the Fabric Data Warehouse workload: https://learn.microsoft.com/en-us/fabric/data-warehouse/copilot
- Fabric trial capacity: https://learn.microsoft.com/en-us/fabric/fundamentals/fabric-trial
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
