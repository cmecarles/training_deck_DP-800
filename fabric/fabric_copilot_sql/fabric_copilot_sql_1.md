# SQL Server question — Copilot for SQL in Fabric 1

## Statement

Orchard Coop, a fruit-growers' cooperative, keeps its member CRM in a **SQL database in Fabric** named `OrchardCrm` and its harvest facts in a **Fabric Data Warehouse** named `OrchardDW`. Both live in the workspace `ws-dp800-orchard`. The Fabric tenant's home region is **Canada Central**. You will enable and exercise Copilot on both items, then pick the plan that satisfies the cooperative's rules.

**Prerequisites.** You are a Fabric tenant admin (or can ask one) and a workspace Admin. Copilot needs a **paid** F2+ or P1+ capacity: to actually see it working you need to assign the workspace to a paid capacity at least for the exercise; the observation steps on a trial capacity are part of the lab.

### Part 1 — Build the lab

1. **Workspaces** > **New workspace** > `ws-dp800-orchard` > **Advanced** > choose your **Trial** capacity for now > **Apply**.
2. **New item** > **SQL database** > `OrchardCrm` > **Create**. In the query editor (**New query**) run:

   ```sql
   CREATE SCHEMA Crm;
   GO
   CREATE TABLE Crm.Grower   (GrowerID int PRIMARY KEY, GrowerName varchar(80) NOT NULL, Region varchar(40) NOT NULL);
   CREATE TABLE Crm.Delivery (DeliveryID int PRIMARY KEY, GrowerID int NOT NULL REFERENCES Crm.Grower(GrowerID),
                              Fruit varchar(30) NOT NULL, Kg decimal(9,2) NOT NULL, DeliveredOn date NOT NULL);
   INSERT INTO Crm.Grower VALUES (1,'Hillcrest Farm','Okanagan'), (2,'Bright Acres','Niagara'), (3,'Cedar Row','Okanagan');
   INSERT INTO Crm.Delivery VALUES (10,1,'Apple',1200.50,'2026-08-20'), (11,2,'Peach',430.00,'2026-08-21'),
                                   (12,3,'Apple',980.25,'2026-08-22'), (13,1,'Pear',300.00,'2026-08-23');
   ```

3. **New item** > **Warehouse** > `OrchardDW` > **Create**. In its query editor run:

   ```sql
   CREATE TABLE dbo.HarvestFact (HarvestID int NOT NULL, Region varchar(40) NOT NULL, Fruit varchar(30) NOT NULL,
                                 Kg decimal(9,2) NOT NULL, HarvestDate date NOT NULL);
   INSERT INTO dbo.HarvestFact VALUES (1,'Okanagan','Apple',5000.00,'2026-08-15'), (2,'Niagara','Peach',2100.00,'2026-08-16');
   ```

### Part 2 — Tenant settings (Admin portal)

4. Open **Settings (gear)** > **Admin portal** > **Tenant settings** and expand the group **Copilot and AI**. Record the state of:
   - **Users can use Copilot and other features powered by Azure OpenAI**
   - **Data sent to Azure OpenAI can be processed outside your capacity's geographic region, compliance boundary, or national cloud instance**
   - **Data sent to Azure OpenAI can be stored outside your capacity's geographic region, compliance boundary, or national cloud instance**
   - **Users can use Copilot, AI Agents and other AI experiences powered by OpenAI as a Microsoft Subprocessor**
   - **Capacities can be designated as Copilot in Fabric capacities**

   For each setting note whether it can be applied to **specific security groups** and whether it can be **delegated** to capacity admins (**Capacity settings** > your capacity > **Delegated tenant settings**).

### Part 3 — Exercise Copilot

5. Open `OrchardCrm`, **New query**, and select the **Copilot** ribbon button. Observe the pane while the workspace is on the trial capacity. Then move the workspace to a paid capacity (**Workspace settings** > **Workspace type** > **Edit** > **Fabric capacity** > pick the F SKU) and reopen the pane.
6. At the bottom of the chat pane locate the **execution mode selector** with the options **Read-only** and **Read and write with approval**. In **Read-only** mode type `show the total kilograms delivered per region` and observe whether a query runs. Then type `create a table Crm.Payout with PayoutID int and Amount decimal(9,2)` and observe. Switch the mode and repeat the second prompt.
7. In the editor, paste `SELECT GrowerName, SUM(Kg) FROM Crm.Delivery d JOIN Crm.Grower g ON g.GrowerID = d.GrowerId GROUP BY Region` (note the deliberate mistakes), highlight it, and look at the **Explain** and **Fix** buttons next to **Run**. Select **Fix** before running; then **Run**, then **Fix** again.
8. Open the database **Settings** > **Copilot** pane and find **Show Copilot completions**; also look at the status bar at the bottom of the query editor. Type `-- list growers with more than 1000 kg delivered` on a new line and wait for ghost text; press **Tab**.
9. Open `OrchardDW`, **New SQL query**, select **Copilot** on the ribbon, ask `count deliveries by fruit`, then ask `now only for Okanagan` and observe whether the second prompt is understood in context. Repeat step 7 in the warehouse.
10. Open the **SQL analytics endpoint** of `OrchardCrm` and check whether a **Copilot** button exists there too.

### The cooperative's rules

1. Copilot must be available in `OrchardCrm` and `OrchardDW` **only** to the security group `sg-orchard-analysts`, in the Canada Central capacity.
2. In the `OrchardCrm` chat pane, analysts may let Copilot **run SELECT queries**, but Copilot must **never execute** DDL or DML.
3. Inline ghost-text completions must be **switched off in `OrchardCrm`** (auditors do not want unreviewed code inserted), while the chat pane, **Explain** and **Fix** stay available.
4. The compliance officer needs a one-paragraph statement of what leaves the tenant when an analyst uses Copilot in `OrchardCrm`.

Which plan is correct?

### a.

Keep the workspace on the paid F2 (or larger) capacity — the trial SKU does not support Copilot. In **Tenant settings** > **Copilot and AI**, leave **Users can use Copilot and other features powered by Azure OpenAI** enabled but apply it to `sg-orchard-analysts` only, and enable **Data sent to Azure OpenAI can be processed outside your capacity's geographic region, compliance boundary, or national cloud instance** (also scoped to the group), because Canada Central is outside the US and the EU data boundary. In the `OrchardCrm` chat pane set the execution mode to **Read-only**: `SELECT` prompts still run automatically, while `CREATE TABLE` is only drafted. Turn off **Show Copilot completions** in the database **Settings** > **Copilot** pane. Statement: Copilot receives the prompt, the session's previous messages, the text and error messages of queries the analyst executed, and the database schema; it has no access to table data, prompts are not used to train foundation models, and — because of the second setting — requests may be processed by Azure OpenAI outside the capacity's region.

### b.

Leave the workspace on the trial capacity, which is an F64-equivalent and therefore satisfies the "F2 or higher" requirement, and enable **Users can use Copilot and other features powered by Azure OpenAI** for `sg-orchard-analysts`. No cross-region setting is needed because Copilot always processes requests in the capacity's own region. Meet rule 2 by choosing **Read and write with approval**, since in **Read-only** mode Copilot cannot run any query, including `SELECT`. Meet rule 3 by disabling completions from the same tenant setting.

### c.

Same capacity and group scoping as option a, but enable **Data sent to Azure OpenAI can be stored outside your capacity's geographic region, compliance boundary, or national cloud instance** instead of the *processed* setting — it is the one that unblocks Copilot for the SQL database outside the US and EU. Meet rule 2 with **Read and write with approval** so that the analyst can approve each `SELECT`. Meet rule 3 by removing the analyst's **Write** permission on `OrchardCrm`, which hides the completion feature.

### d.

Disable **Users can use Copilot and other features powered by Azure OpenAI** for the whole tenant, and instead enable **Users can use Copilot, AI Agents and other AI experiences powered by OpenAI as a Microsoft Subprocessor** for `sg-orchard-analysts`; this routes the SQL database Copilot to OpenAI-operated models without any cross-region setting. Meet rule 2 with **Read-only** mode. Meet rule 3 by turning off the **Show Copilot completions** switch, which also disables the **Fix** and **Explain** quick actions since they share the completion engine.

**Capacity and cleanup.** Copilot calls consume capacity units of the workspace's capacity. When done, **Workspace settings** > **General** > **Remove this workspace** > **Delete**, pause the paid F capacity in the Azure portal, and revert any tenant settings you changed.

## Correct Answer

**a**

## Explanation

The correct answer is **a**.

| Rule | a | b (trial + R/W mode) | c (storage setting) | d (OpenAI subprocessor) |
|---|---|---|---|---|
| 1. Copilot for the group in Canada Central | satisfied | **violated** (trial SKU; no cross-region setting) | **violated** (wrong setting) | **violated** (wrong setting group; Azure OpenAI switch off) |
| 2. SELECT runs, DDL/DML never executes | satisfied (Read-only) | **violated** (R/W with approval can execute DDL) | **violated** | satisfied |
| 3. Completions off, Explain/Fix on | satisfied | **violated** (no such tenant switch) | **violated** (permission does not control it) | **violated** (Fix/Explain are independent) |
| 4. Accurate data statement | satisfied | — | — | — |

### What you observed in the lab

- **Trial capacity**: "Copilot in Microsoft Fabric isn't supported on trial SKUs. Only paid SKUs (F2 or higher, or P1 or higher) are supported at this time." The trial page itself lists "Copilot and Trusted Workspace Access aren't supported". The pane only becomes usable after the move to the paid capacity (step 5).
- **Tenant settings** live in the **Copilot and AI** group. Enabled by default: *Users can use Copilot and other features powered by Azure OpenAI* and *Capacities can be designated as Copilot in Fabric capacities*. Disabled by default: both *processed outside* settings, the *stored outside* setting, and the two *OpenAI as a Microsoft Subprocessor* settings. The Azure OpenAI switch and the subprocessor switch "can be managed at both the tenant and the capacity levels", and every setting can be applied to specific security groups.
- **Execution mode selector** (step 6): in **Read-only** mode "Copilot doesn't run Data Definition Language (DDL) or Data Manipulation Language (DML) statements that change data or schema. Instead, Copilot suggests SQL code for you to review and run manually", whereas a `SELECT` prompt is generated **and run automatically "regardless of the selected mode"**. In **Read and write with approval** the `CREATE TABLE` is drafted and Copilot "prompts you to approve execution".
- **Fix** (step 7) "only gets enabled after you run your T-SQL query and it has returned an error"; it "automatically takes the SQL error message into context" and leaves a comment where it edited. **Explain** works on any highlighted text and writes a summary plus in-line comments.
- **Show Copilot completions** (step 8) is a per-database switch in **Settings** > **Copilot**, mirrored in the query editor status bar; completions "use your database schema and query tab context", are accepted with **Tab** (or word-by-word with **Ctrl+Right**), and cover DDL, DQL and DML.
- **Warehouse Copilot** (steps 9–10) applies to "SQL analytics endpoint and Warehouse", so the button exists on the SQL analytics endpoint too; unlike the SQL database chat pane, the warehouse Copilot "doesn't understand previous inputs", so `now only for Okanagan` is not resolved against the previous answer, and "natural language to SQL supports English language to T-SQL". Copilot for the warehouse "doesn't use data in tables to generate T-SQL suggestions".

### Why option a is correct

Rule 1 needs three things the option supplies: a paid capacity, the group-scoped Azure OpenAI switch, and — for a capacity outside the US and the EU data boundary — the **processed outside** setting: "If your tenant or capacity is outside the US or France, Copilot is disabled by default unless your Fabric tenant admin enables" it. Rule 2 maps to the **Read-only** execution mode, whose documented behaviour is exactly "SELECT runs automatically, DDL/DML is only suggested". Rule 3 maps to the **Show Copilot completions** switch, which is independent of chat and of the quick actions. The data statement reproduces the privacy article: "In database, Copilot can only access the database schema that is accessible in the user's database" and by default has access to "previous messages sent to and replies from Copilot for that user in that session", "contents of SQL query that the user has executed", "error messages of a SQL query that the user has executed", and "schemas of the database"; "prompts and responses ... aren't used to train foundation models".

### Why option b is wrong

Three factual errors. The trial capacity is sized like an F64 but Copilot is explicitly unsupported on trial SKUs. Outside the US and the EU boundary the cross-region *processing* setting is required — Copilot is otherwise disabled. And **Read-only** mode does run `SELECT` prompts automatically; choosing **Read and write with approval** would let Copilot execute DDL after a click, which rule 2 forbids. There is no tenant-level switch for completions; it is a database setting.

### Why option c is wrong

The *stored outside* setting "is only applicable for customers who want to use Copilot in notebooks and data agents" — it governs conversation-history storage across sessions and does nothing for the SQL database Copilot; the required switch is the *processed outside* one. **Read and write with approval** is the mode in which DDL/DML can be executed, the opposite of rule 2 (and `SELECT` needs no approval in either mode). Removing **Write** on the item changes what the analyst may do in the database, not whether Copilot shows ghost text.

### Why option d is wrong

The subprocessor setting enables experiences "powered by OpenAI-operated models"; it is a separate, off-by-default model-provider option with its own cross-region companion setting (*Data sent to OpenAI as a Microsoft Subprocessor can be processed outside...*), not a replacement for the Azure OpenAI switch that Copilot for SQL database depends on. Disabling the Azure OpenAI switch tenant-wide turns Copilot off for everyone. Finally, **Fix** and **Explain** are quick actions with their own prerequisites; the completions toggle does not disable them.

### Where each capability lives

```text
Admin portal > Tenant settings > Copilot and AI
   Users can use Copilot and other features powered by Azure OpenAI ............ ON by default, group-scoped, delegable
   Data sent to Azure OpenAI can be PROCESSED outside your capacity's region ... OFF; required outside US / EU boundary
   Data sent to Azure OpenAI can be STORED outside ............................. OFF; notebooks + data agents only
   Users can use Copilot, AI Agents ... OpenAI as a Microsoft Subprocessor ..... OFF; different model provider
SQL database query editor      Copilot ribbon button -> chat pane (NL2SQL, doc Q&A, remembers the session)
                               execution mode: Read-only (SELECT runs, DDL/DML drafted) | Read and write with approval
                               Explain / Fix next to Run (Fix enabled only after an error)
                               Settings > Copilot > Show Copilot completions (Tab / Ctrl+Right)
Warehouse + SQL analytics endpoint   same three surfaces; no memory of previous prompts; English only
SSMS / VS Code (MSSQL)         Ask mode read-only; Agent mode writes only after approval; execution-plan analysis
```

Documentation relied upon (ms.date): Copilot and Agent admin settings 2026-08-11; Enable and configure Copilot in Fabric 2026-08-10; Copilot in the SQL database workload overview 2026-02-26; Copilot chat pane 2025-11-18; Copilot quick actions 2025-11-18; Copilot code completion 2025-11-18; Privacy, security and responsible use of Copilot in the SQL database workload 2025-11-18; Copilot in the Data Warehouse workload 2026-09-02; Fabric trial capacity 2026-08-12.

Hands-on question (Microsoft Fabric capacity required); behaviour is taken from the official documentation as of the ms.date cited above.

## DP-800 Exam Rule to Remember

```text
Copilot for SQL database / Warehouse in Fabric
  Needs: paid F2+/P1+ (never trial) + "Users can use Copilot ... Azure OpenAI" (ON by default, scope to a group)
         + outside US/EU boundary: "Data sent to Azure OpenAI can be PROCESSED outside..." (OFF by default)
  "...STORED outside..." = notebooks/data agents only.  "OpenAI as a Microsoft Subprocessor" = other provider.
  Chat pane sees: schema + your prompt + session history + executed query text/errors.  Never table data.
  SQL database chat pane modes: Read-only (SELECT auto-runs, DDL/DML only drafted)
                                Read and write with approval (DDL/DML run after you approve)
  Quick actions: Explain (comments) | Fix (enabled only after a run returned an error, uses the error text)
  Ghost-text completions: per-database Settings > Copilot > Show Copilot completions; Tab accepts.
  Warehouse Copilot = SQL analytics endpoint + Warehouse; no memory of earlier prompts; English only.
```
