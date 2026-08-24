# SQL Server question — Embeddings 1

## Statement

A cooking-recipe platform stores its catalog in an Azure SQL Database named `RecipeBox`.

The database uses the **DTU purchasing model at the Standard S1 service objective**, and the team is not allowed to change the service objective.

Recipes are stored in the following table:

```sql
CREATE TABLE Kitchen.Recipes
(
    RecipeId    int NOT NULL PRIMARY KEY,
    Title       nvarchar(200) NOT NULL,
    Ingredients nvarchar(max) NOT NULL,
    Steps       nvarchar(max) NOT NULL,
    Embedding   vector(1536) NULL
);
```

An external model has already been registered in the database and works correctly:

```sql
CREATE EXTERNAL MODEL RecipeEmbeddingModel
WITH (
    LOCATION = 'https://recipebox-openai.cognitiveservices.azure.com/openai/deployments/text-embedding-ada-002/embeddings?api-version=2023-05-15',
    API_FORMAT = 'Azure OpenAI',
    MODEL_TYPE = EMBEDDINGS,
    MODEL = 'text-embedding-ada-002',
    CREDENTIAL = [https://recipebox-openai.cognitiveservices.azure.com/]
);
```

Users constantly add and edit recipes through the website. You must design the **embedding maintenance** architecture that keeps `Kitchen.Recipes.Embedding` up to date with the text in `Title`, `Ingredients`, and `Steps`.

The solution must satisfy **all** of the following requirements:

1. A user's `INSERT` or `UPDATE` transaction must **not wait for the embedding model call**, and the user's write must **succeed even when the Azure OpenAI endpoint is unreachable**.
2. A new or edited recipe must get a refreshed embedding in **near real time** (within about a minute under normal operation), not on a daily cadence.
3. If the embedding processor is **offline for up to one day** (deployment, outage), every recipe changed during that time must still be embedded automatically after the processor comes back. No change may be missed.
4. Only the rows that actually changed may be reprocessed. Re-embedding the entire table is **not acceptable** because every model call is billed.
5. Every feature used must be **supported on the current service objective** (Azure SQL Database, DTU model, Standard S1).

Which architecture should you implement?

### a.

Generate the embedding inside the database transaction with a DML trigger:

```sql
CREATE TRIGGER Kitchen.trg_Recipes_Embed
ON Kitchen.Recipes
AFTER INSERT, UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    UPDATE r
    SET Embedding = AI_GENERATE_EMBEDDINGS(
            CONCAT_WS(N' ', i.Title, i.Ingredients, i.Steps)
            USE MODEL RecipeEmbeddingModel)
    FROM Kitchen.Recipes AS r
    INNER JOIN inserted AS i
        ON i.RecipeId = r.RecipeId;
END;
```

### b.

Enable change tracking and process changes with an Azure Function that uses the SQL trigger binding:

```sql
ALTER DATABASE RecipeBox
SET CHANGE_TRACKING = ON
(CHANGE_RETENTION = 2 DAYS, AUTO_CLEANUP = ON);

ALTER TABLE Kitchen.Recipes
ENABLE CHANGE_TRACKING;
```

Deploy an Azure Function whose trigger is:

```text
[SqlTrigger("[Kitchen].[Recipes]", "SqlConnectionString")]
```

For each change batch that the binding delivers, the function calls the embedding model for the changed rows only and updates `Kitchen.Recipes.Embedding` for those rows.

### c.

Enable change data capture and poll the change table from a timer-based Azure Function:

```sql
EXEC sys.sp_cdc_enable_db;
GO

EXEC sys.sp_cdc_enable_table
    @source_schema = N'Kitchen',
    @source_name   = N'Recipes',
    @role_name     = NULL,
    @supports_net_changes = 1;
GO
```

A timer-triggered Azure Function runs every 30 seconds, reads
`cdc.fn_cdc_get_net_changes_Kitchen_Recipes` for the new LSN range, calls the embedding model for the changed rows only, and updates `Kitchen.Recipes.Embedding`.

### d.

Use an Azure Logic Apps workflow with a Recurrence trigger that runs every night at 02:00 and executes:

```sql
UPDATE r
SET Embedding = AI_GENERATE_EMBEDDINGS(
        CONCAT_WS(N' ', r.Title, r.Ingredients, r.Steps)
        USE MODEL RecipeEmbeddingModel)
FROM Kitchen.Recipes AS r;
```

## Correct Answer

**b**

## Explanation

The correct answer is **b**.

The question is a constraint-matching exercise. Evaluate every option against the five numbered requirements:

| Requirement | a (sync trigger) | b (change tracking + SQL trigger binding) | c (CDC polling) | d (nightly full re-embed) |
| --- | --- | --- | --- | --- |
| 1. No model call inside the user transaction | **violated** | satisfied | satisfied | satisfied |
| 2. Near-real-time freshness | satisfied | satisfied | satisfied | **violated** |
| 3. No missed changes across a one-day outage | satisfied | satisfied | satisfied | satisfied |
| 4. Changed rows only | satisfied | satisfied | satisfied | **violated** |
| 5. Supported at Standard S1 (DTU model) | satisfied | satisfied | **violated** | satisfied |

Only option **b** satisfies all five.

### Why option b is correct

The Azure SQL trigger binding for Azure Functions is built on **SQL change tracking**. The documented behavior maps to each requirement:

- **Requirement 1 — decoupled writes.** Change tracking records, synchronously and cheaply, only the primary-key values of changed rows in internal side tables. The user's `INSERT`/`UPDATE` never calls the model endpoint. The Function's polling loop (default `PollingIntervalMs` = 1000 ms) runs completely outside the user transaction, so an Azure OpenAI outage delays embedding refresh but never blocks or fails a user write.
- **Requirement 2 — near real time.** The binding polls for changes about once per second and delivers them in batches, so embeddings refresh within seconds under normal operation.
- **Requirement 3 — no missed changes.** The binding persists its progress in leases/state tables in the `az_func` schema and, per the documentation, "picks up processing changes where it left off" after a restart or broken connection. With `CHANGE_RETENTION = 2 DAYS`, the database retains change history longer than the stated one-day maximum outage, so changes made while the Function is down are still delivered when it resumes.
- **Requirement 4 — changed rows only.** Change tracking answers exactly the question "which rows changed since version X?", so only modified recipes are re-embedded.
- **Requirement 5 — tier support.** Change tracking is supported in Azure SQL Database with no service-tier restriction, so it works at Standard S1.

### Why option a is wrong

Option a **violates requirement 1**.

An `AFTER` DML trigger executes **inside the user's transaction**. `AI_GENERATE_EMBEDDINGS` performs an HTTPS REST call to the external model endpoint at execution time, so:

- Every recipe `INSERT`/`UPDATE` now waits for a network round trip to Azure OpenAI, adding the model latency directly to the user's write path.
- If the endpoint is unreachable or throttled, the trigger raises an error and the **user's transaction fails**, exactly what requirement 1 forbids.

The option is otherwise fresh and incremental, but a synchronous trigger couples transactional writes to an external service's availability, which is the classic anti-pattern this epigraph tests.

### Why option c is wrong

Option c **violates requirement 5**.

In Azure SQL Database, change data capture is supported for any vCore service tier, but in the **DTU purchasing model CDC requires S3 or higher; the subcore tiers (Basic, S0, S1, S2) are not supported**. The scenario pins the database to **Standard S1** and forbids scaling, so `sys.sp_cdc_enable_db` is not a usable choice here.

Even where the tier allows it, CDC is heavier than the task needs: it asynchronously scans the transaction log (the Azure SQL Database capture scheduler runs every 20 seconds, with no latency SLA), and it copies the **full column values** — including the `nvarchar(max)` `Ingredients` and `Steps` — into change tables, when the embedding processor only needs to know *which* rows changed and can read the current values from `Kitchen.Recipes` itself. When you only need "which rows changed", change tracking is the documented lightweight fit.

### Why option d is wrong

Option d **violates requirements 2 and 4**.

- A Recurrence trigger at 02:00 means a recipe edited at 08:00 keeps a stale embedding for up to ~18 hours. That fails the near-real-time requirement.
- The `UPDATE` has no `WHERE` clause and no change detection: it re-embeds **every row every night**, paying one model call per recipe regardless of whether it changed. That fails the changed-rows-only requirement.

Logic Apps is a legitimate embedding-maintenance option in general, but only when the workflow is driven by *detected changes*, not by a blind full-table recurrence.

## DP-800 Exam Rule to Remember

Choosing an embedding **maintenance** method is a constraint-matching exercise. Anchor on three questions:

```text
1. May the user's write transaction pay for the model call?
   -> If no: rule out synchronous DML triggers that call the model.

2. How fresh must embeddings be, and may unchanged rows be reprocessed?
   -> Scheduled full re-embeds fail near-real-time and cost constraints.

3. What does the platform/tier actually support?
   -> Change tracking: all Azure SQL Database tiers; records PKs only ("which rows changed").
   -> CDC: full before/after data from the log scan; in Azure SQL Database it is
      NOT supported on DTU subcore tiers (Basic, S0, S1, S2) - S3+ or vCore only.
```

The **Azure Functions SQL trigger binding** is the packaged answer for "decoupled, near-real-time, no missed changes": it is built on change tracking, polls outside the user transaction, and persists its position in `az_func` leases tables so it resumes where it left off — as long as `CHANGE_RETENTION` covers the longest expected processor downtime.
