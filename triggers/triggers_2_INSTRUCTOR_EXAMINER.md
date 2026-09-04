# Instructor-Examiner guide — Triggers 2

Companion to [triggers_2.md](triggers_2.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a long tracing question: five triggers, eleven statements, and two final result sets. Take the eleven statements one at a time, S1 to S11, and confirm each before moving on, because later statements depend on earlier ones. Offer to re-read any trigger body from section 2 at any time. Keep a running tally of the EventLog rows yourself, from section 5, so you can check partial answers. The learner may say "one row affected" or "rows affected one"; both are fine.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Implement programmability objects.
- Task bullet: Create triggers, DML triggers (INSTEAD OF and AFTER) and DDL triggers.
- What is tested: INSTEAD OF versus AFTER firing, `sp_settriggerorder`, the `nested triggers` server option versus the `RECURSIVE_TRIGGERS` database option, what `ROLLBACK` inside a DDL trigger undoes, identity gaps, and `DISABLE TRIGGER`.

## 2. Scenario to read aloud

**Piece 1, the story.** "A fish farm tracks its ponds, and the batches of fish stocked in them, in a SQL Server database called AquaFarm. Operators enter stock through a grouped view, and the DBA protects the schema with a DDL trigger. Everything I describe was run top to bottom in Management Studio, with default session options and no errors. The server option nested triggers is at its default value, one."

**Piece 2, the three tables.** "There is a schema called Farm with three tables. Farm dot Tanks has two columns: TankId, an integer, the primary key, and TankName, text up to thirty characters. Farm dot Batches has four columns: BatchId, an identity integer starting at one, the primary key. TankId, an integer that references Farm dot Tanks. Species, a text code up to twenty characters. And FishCount, an integer. Farm dot EventLog has three columns: LogId, an identity integer starting at one, the primary key. Source, text up to forty characters. And Detail, text up to two hundred characters, nullable."

**Piece 3, the data.** "Two tanks are inserted: tank one, East Pond, and tank two, West Pond. Two batches are inserted, so they get BatchId one and two: batch one is in tank one, species Salmon, four hundred fish. Batch two is in tank two, species Trout, two hundred fifty fish. The EventLog is empty."

**Piece 4, the view.** "There is a view, Farm dot TankStock. It joins Tanks to Batches on TankId, groups by TankId, TankName and Species, and returns four columns: TankId, TankName, Species, and the sum of FishCount as TotalFish. So it is a grouped view over two tables, with an aggregate."

**Piece 5, the first trigger, on the view.** "The first trigger is Farm dot trg underscore TankStock underscore Insert. It is an INSTEAD OF INSERT trigger on the view TankStock. Its body does three things. First, it inserts one row into EventLog with Source INSTEAD OF TankStock and Detail equal to the count of rows in the inserted table, followed by the words view row open paren s close paren. For example, two view row s. Second, it inserts into Farm dot Tanks the distinct TankId and TankName from inserted, but only where no tank with that TankId already exists. Third, it inserts into Farm dot Batches one row per inserted row: TankId, Species, and TotalFish used as the FishCount. It sets NOCOUNT on at the top."

**Piece 6, the two audit triggers.** "The second and third triggers are twins on Farm dot Batches, both AFTER INSERT. Farm dot trg underscore Batches underscore AuditA inserts one EventLog row with Source AFTER Batches A and Detail equal to the count of rows in inserted followed by batch row open paren s close paren. Farm dot trg underscore Batches underscore AuditB does exactly the same with Source AFTER Batches B. Both set NOCOUNT on."

**Piece 7, the cull trigger.** "The fourth trigger is Farm dot trg underscore Batches underscore Cull, an AFTER UPDATE trigger on Farm dot Batches. First line: if there is no row in inserted with FishCount greater than one thousand, it returns immediately. Otherwise it inserts into EventLog one row per inserted row with FishCount over one thousand, Source AFTER Batches Cull, Detail the word batch, then the BatchId, then the word at, then the FishCount. For example, batch one at five thousand. Then it updates Farm dot Batches, joined to inserted on BatchId, setting FishCount to FishCount divided by two, integer division, for the rows whose inserted FishCount is over one thousand. So it halves any batch that went over a thousand, and it does that by updating its own table."

**Piece 8, the DDL trigger.** "The fifth trigger is trg underscore ProtectSchema, a DDL trigger ON DATABASE, FOR DROP underscore TABLE and ALTER underscore TABLE. It reads EVENTDATA into an XML variable and builds a string called at what from three XML values: the EventType, a space, the SchemaName, a dot, and the ObjectName. For example, DROP underscore TABLE Farm dot Batches. Then it inserts an EventLog row with Source DDL attempt and Detail at what. Then it runs ROLLBACK. Then, after the rollback, it inserts another EventLog row with Source DDL blocked and Detail at what."

**Piece 9, statements S1 and S2.** "Now eleven statements run in order, each in its own batch, in the same session. S1 executes sp underscore settriggerorder with trigger name Farm dot trg underscore Batches underscore AuditB, order First, statement type INSERT. S2 executes sp underscore settriggerorder with trigger name Farm dot trg underscore TankStock underscore Insert, order First, statement type INSERT."

**Piece 10, statement S3.** "S3 inserts two rows into the view Farm dot TankStock, giving TankId, TankName, Species and TotalFish. Row one: tank three, North Pond, Trout, three hundred. Row two: tank one, East Pond, Trout, one hundred twenty."

**Piece 11, statements S4 to S6.** "S4 updates Farm dot Batches, setting FishCount to five thousand where BatchId is one. S5 runs ALTER DATABASE AquaFarm SET RECURSIVE underscore TRIGGERS ON. S6 updates Farm dot Batches, setting FishCount to five thousand where BatchId is two."

**Piece 12, statements S7 to S11.** "S7 is DROP TABLE Farm dot Batches. S8 is DISABLE TRIGGER trg underscore ProtectSchema ON DATABASE. S9 is ALTER TABLE Farm dot Batches ADD Notes, a nullable text column of fifty characters. S10 is DISABLE TRIGGER Farm dot trg underscore Batches underscore AuditA ON Farm dot Batches. S11 inserts one row directly into Farm dot Batches: tank two, Carp, ninety fish."

**Piece 13, the two final queries.** "At the end, two queries. The first selects LogId, Source and Detail from Farm dot EventLog ordered by LogId. The second selects BatchId, TankId, Species and FishCount from Farm dot Batches ordered by BatchId. You will be asked for the exact rows of both. Tell me when you are ready for the question."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE AquaFarm;
GO
USE AquaFarm;
GO
CREATE SCHEMA Farm;
GO
CREATE TABLE Farm.Tanks
(
    TankId   INT          NOT NULL PRIMARY KEY,
    TankName NVARCHAR(30) NOT NULL
);
CREATE TABLE Farm.Batches
(
    BatchId   INT IDENTITY(1,1) PRIMARY KEY,
    TankId    INT         NOT NULL REFERENCES Farm.Tanks(TankId),
    Species   VARCHAR(20) NOT NULL,
    FishCount INT         NOT NULL
);
CREATE TABLE Farm.EventLog
(
    LogId  INT IDENTITY(1,1) PRIMARY KEY,
    Source VARCHAR(40)   NOT NULL,
    Detail NVARCHAR(200) NULL
);
GO
INSERT INTO Farm.Tanks (TankId, TankName) VALUES (1, N'East Pond'), (2, N'West Pond');
INSERT INTO Farm.Batches (TankId, Species, FishCount) VALUES (1, 'Salmon', 400), (2, 'Trout', 250);
GO
CREATE VIEW Farm.TankStock
AS
SELECT t.TankId, t.TankName, b.Species, SUM(b.FishCount) AS TotalFish
FROM Farm.Tanks AS t
JOIN Farm.Batches AS b ON b.TankId = t.TankId
GROUP BY t.TankId, t.TankName, b.Species;
GO
CREATE TRIGGER Farm.trg_TankStock_Insert
ON Farm.TankStock
INSTEAD OF INSERT
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO Farm.EventLog (Source, Detail)
    SELECT 'INSTEAD OF TankStock', CONCAT(COUNT(*), N' view row(s)') FROM inserted;

    INSERT INTO Farm.Tanks (TankId, TankName)
    SELECT DISTINCT i.TankId, i.TankName
    FROM inserted AS i
    WHERE NOT EXISTS (SELECT 1 FROM Farm.Tanks AS t WHERE t.TankId = i.TankId);

    INSERT INTO Farm.Batches (TankId, Species, FishCount)
    SELECT i.TankId, i.Species, i.TotalFish FROM inserted AS i;
END;
GO
CREATE TRIGGER Farm.trg_Batches_AuditA
ON Farm.Batches
AFTER INSERT
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO Farm.EventLog (Source, Detail)
    SELECT 'AFTER Batches A', CONCAT(COUNT(*), N' batch row(s)') FROM inserted;
END;
GO
CREATE TRIGGER Farm.trg_Batches_AuditB
ON Farm.Batches
AFTER INSERT
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO Farm.EventLog (Source, Detail)
    SELECT 'AFTER Batches B', CONCAT(COUNT(*), N' batch row(s)') FROM inserted;
END;
GO
CREATE TRIGGER Farm.trg_Batches_Cull
ON Farm.Batches
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    IF NOT EXISTS (SELECT 1 FROM inserted WHERE FishCount > 1000) RETURN;

    INSERT INTO Farm.EventLog (Source, Detail)
    SELECT 'AFTER Batches Cull', CONCAT(N'batch ', BatchId, N' at ', FishCount)
    FROM inserted WHERE FishCount > 1000;

    UPDATE b SET b.FishCount = b.FishCount / 2
    FROM Farm.Batches AS b
    JOIN inserted AS i ON i.BatchId = b.BatchId
    WHERE i.FishCount > 1000;
END;
GO
CREATE TRIGGER trg_ProtectSchema
ON DATABASE
FOR DROP_TABLE, ALTER_TABLE
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @e XML = EVENTDATA();
    DECLARE @what NVARCHAR(200) =
        @e.value('(/EVENT_INSTANCE/EventType)[1]',  'NVARCHAR(50)') + N' '
      + @e.value('(/EVENT_INSTANCE/SchemaName)[1]', 'NVARCHAR(50)') + N'.'
      + @e.value('(/EVENT_INSTANCE/ObjectName)[1]', 'NVARCHAR(50)');

    INSERT INTO Farm.EventLog (Source, Detail) VALUES ('DDL attempt', @what);
    ROLLBACK;
    INSERT INTO Farm.EventLog (Source, Detail) VALUES ('DDL blocked', @what);
END;
GO
```

The eleven statements, each in its own batch:

```sql
-- S1
EXEC sp_settriggerorder @triggername = 'Farm.trg_Batches_AuditB', @order = 'First', @stmttype = 'INSERT';

-- S2
EXEC sp_settriggerorder @triggername = 'Farm.trg_TankStock_Insert', @order = 'First', @stmttype = 'INSERT';

-- S3
INSERT INTO Farm.TankStock (TankId, TankName, Species, TotalFish)
VALUES (3, N'North Pond', 'Trout', 300),
       (1, N'East Pond',  'Trout', 120);

-- S4
UPDATE Farm.Batches SET FishCount = 5000 WHERE BatchId = 1;

-- S5
ALTER DATABASE AquaFarm SET RECURSIVE_TRIGGERS ON;

-- S6
UPDATE Farm.Batches SET FishCount = 5000 WHERE BatchId = 2;

-- S7
DROP TABLE Farm.Batches;

-- S8
DISABLE TRIGGER trg_ProtectSchema ON DATABASE;

-- S9
ALTER TABLE Farm.Batches ADD Notes NVARCHAR(50) NULL;

-- S10
DISABLE TRIGGER Farm.trg_Batches_AuditA ON Farm.Batches;

-- S11
INSERT INTO Farm.Batches (TankId, Species, FishCount) VALUES (2, 'Carp', 90);
```

The two final queries:

```sql
SELECT LogId, Source, Detail FROM Farm.EventLog ORDER BY LogId;
SELECT BatchId, TankId, Species, FishCount FROM Farm.Batches ORDER BY BatchId;
```

## 4. The question (ask exactly this)

"For each statement, S1 to S11, tell me whether it succeeds or raises an error. For the DML statements that succeed, also tell me the rows-affected count reported to the client. Let's go one at a time, starting with S1."

After all eleven: "Now give me the exact result of the first final query: every row of Farm dot EventLog, ordered by LogId, with LogId, Source and Detail."

Then: "And the exact result of the second final query: every row of Farm dot Batches, ordered by BatchId, with BatchId, TankId, Species and FishCount."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

| Stmt | Outcome | Detail |
|---|---|---|
| S1 | Succeeds | `trg_Batches_AuditB` becomes the first AFTER INSERT trigger on `Farm.Batches` |
| S2 | Fails, error 15133 | "INSTEAD OF trigger 'Farm.trg_TankStock_Insert' cannot be associated with an order." |
| S3 | Succeeds, 2 rows affected | INSTEAD OF trigger logs row 1, creates tank 3, inserts batches 3 and 4; then the two AFTER INSERT triggers on Batches fire once each, B before A, logging rows 2 and 3 |
| S4 | Succeeds, 1 row affected | `trg_Batches_Cull` fires once, logs row 4, halves batch 1 to 2500; its own UPDATE does not re-fire it because RECURSIVE_TRIGGERS is OFF |
| S5 | Succeeds | direct recursion is now enabled for the database |
| S6 | Succeeds, 1 row affected | `trg_Batches_Cull` fires and re-fires: 5000 logs row 5 and halves to 2500; 2500 logs row 6 and halves to 1250; 1250 logs row 7 and halves to 625; the next invocation sees 625 and the guard returns |
| S7 | Fails, error 3609 | "The transaction ended in the trigger. The batch has been aborted." Table not dropped. The 'DDL attempt' row (LogId 8) is rolled back; the 'DDL blocked' row written after ROLLBACK persists as LogId 9 |
| S8 | Succeeds | DDL trigger disabled, `is_disabled = 1`, not dropped |
| S9 | Succeeds | column Notes added; nothing logged |
| S10 | Succeeds | `trg_Batches_AuditA` disabled |
| S11 | Succeeds, 1 row affected | only `trg_Batches_AuditB` fires, logs row 10 |

Final contents of `Farm.EventLog` (note the gap at LogId 8):

| LogId | Source | Detail |
|---|---|---|
| 1 | INSTEAD OF TankStock | 2 view row(s) |
| 2 | AFTER Batches B | 2 batch row(s) |
| 3 | AFTER Batches A | 2 batch row(s) |
| 4 | AFTER Batches Cull | batch 1 at 5000 |
| 5 | AFTER Batches Cull | batch 2 at 5000 |
| 6 | AFTER Batches Cull | batch 2 at 2500 |
| 7 | AFTER Batches Cull | batch 2 at 1250 |
| 9 | DDL blocked | DROP_TABLE Farm.Batches |
| 10 | AFTER Batches B | 1 batch row(s) |

Final contents of `Farm.Batches`:

| BatchId | TankId | Species | FishCount |
|---|---|---|---|
| 1 | 1 | Salmon | 2500 |
| 2 | 2 | Trout | 625 |
| 3 | 3 | Trout | 300 |
| 4 | 1 | Trout | 120 |
| 5 | 2 | Carp | 90 |

## 6. Hint ladder (one hint per attempt, in order)

**S1**
1. "What does sp underscore settriggerorder do, and for which kind of trigger does it make sense?"
2. "There are two AFTER INSERT triggers on Batches. Can one of them be pinned as First?"

**S2**
1. "Which kind of trigger is trg underscore TankStock underscore Insert: AFTER or INSTEAD OF?"
2. "How many INSTEAD OF triggers can an object have per event? If only one, does an order mean anything?"
3. "The call is syntactically perfect. The engine still refuses it, with an error saying that this kind of trigger cannot be associated with an order."

**S3**
1. "The view is grouped and joins two tables. Normally you cannot insert into it. What changes that?"
2. "Walk through the INSTEAD OF body. What goes into EventLog first? How many rows are in inserted?"
3. "Does tank one already exist? Does tank three? Which one does the NOT EXISTS clause let through?"
4. "The body inserts two rows into Batches. Are there AFTER INSERT triggers on Batches? Does a trigger's own DML fire other triggers with nested triggers at one?"
5. "Two AFTER INSERT triggers fire. S1 pinned one of them first. Which log row comes before which?"
6. "The rows-affected count the client sees: is it the trigger's inner inserts, or the two rows of the original statement?"

**S4**
1. "Which trigger fires on UPDATE of Batches? Does five thousand pass its guard?"
2. "The trigger updates its own table. Does that inner UPDATE fire the same trigger again? Which database option controls that, and what is its default?"
3. "So the trigger runs once. What does it log, and what is batch one's FishCount afterwards?"

**S5**
1. "This is an ALTER DATABASE. Is there anything blocking it? Check which events the DDL trigger covers."
2. "It succeeds. What has changed for the next UPDATE?"

**S6**
1. "Same trigger as in S4, but now RECURSIVE underscore TRIGGERS is ON. What happens when the trigger halves batch two?"
2. "Each halving is a new UPDATE on Batches, and the trigger fires again with the new value in inserted. Keep halving until the guard stops it. Which values are over one thousand?"
3. "Five thousand, twenty-five hundred, twelve fifty. Then six twenty-five. Which of those get a log row, and which one makes the guard return?"
4. "What rows-affected count does the client see for the original UPDATE?"

**S7**
1. "Which trigger catches a DROP TABLE? Walk its body in order: log, rollback, log."
2. "What does ROLLBACK inside a DDL trigger undo? Just the DROP, or also what the trigger wrote before it?"
3. "After the ROLLBACK, the trigger keeps running. Under what transaction does the second INSERT run, and does it persist?"
4. "What does the client see? Think of the message about the transaction ending in the trigger and the batch being aborted."
5. "The rolled-back row consumed an identity value. Is that value reused?"

**S8**
1. "DISABLE TRIGGER on a database-level trigger. Does that need the DROP or ALTER TABLE events? Does the DDL trigger see it?"
2. "Is the trigger dropped, or just switched off?"

**S9**
1. "ALTER TABLE is one of the events the DDL trigger covers. But what is the trigger's state after S8?"
2. "So nothing blocks it and nothing logs it. Does the table Batches still exist after S7?"

**S10**
1. "This disables one of the two AFTER INSERT triggers on Batches. Which one remains active?"

**S11**
1. "A direct insert into Batches. Which AFTER INSERT triggers are still enabled?"
2. "Does the AFTER UPDATE trigger fire on an INSERT? Does the INSTEAD OF trigger on the view fire on a direct table insert?"
3. "How many rows in inserted, and what is the Detail text?"

**EventLog**
1. "Count the log rows written by S3: one from the INSTEAD OF trigger and one from each AFTER INSERT trigger. In which order?"
2. "S4 writes one Cull row. S6 writes three. What are the batch numbers and values in the Detail text?"
3. "S7 wrote two rows but rolled one back. Which identity value is missing, and which survives?"
4. "S11 writes the last row. Which trigger wrote it, and how many batch rows does it mention?"

**Batches**
1. "Start from batches one and two. What did S4 and S6 do to their FishCount?"
2. "S3 added two batches through the view. Which BatchId did each get, and which tank is each in?"
3. "S7 did not drop the table. S11 added one more batch. What BatchId does it get?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "S2 succeeds; the call looks correct" | Thinks order applies to any trigger | "What kind of trigger is it? How many of that kind can exist per event?" |
| "S3 fails; you cannot insert into a grouped view" | Forgets INSTEAD OF makes any view writable | "Is there something on the view that replaces the INSERT entirely?" |
| "S3 logs A before B" | Ignores S1 | "Go back to S1. Which trigger was pinned First?" |
| "S3, the AFTER triggers do not fire because the insert came from a trigger" | Confuses nested triggers with recursion | "Nested triggers is at one. Does a trigger's DML fire triggers on other tables?" |
| "S3 reports four rows affected" | Counts the trigger's inner inserts | "Which statement's rowcount does the client receive: the original, or the trigger's inner DML?" |
| "S4 keeps halving down to six twenty-five" | Assumes recursion is on by default | "Which database option governs a trigger firing itself, and what is its default?" |
| "S6 halves only once, like S4" | Missed S5 | "What did S5 change?" |
| "S6 logs four Cull rows" | Counts the invocation that returns via the guard | "The fourth time the trigger runs, what is in inserted? Does it pass the guard?" |
| "S7 succeeds; the table is dropped" | Forgets the ROLLBACK | "Walk the DDL trigger body. What does the middle statement do?" |
| "S7 keeps both DDL attempt and DDL blocked rows" | Thinks ROLLBACK only undoes the DDL | "ROLLBACK undoes everything in the transaction. Was the first INSERT inside that transaction?" |
| "S7 keeps neither row" | Thinks the batch abort stops the trigger | "Does the trigger body stop at ROLLBACK, or does the next statement still run?" |
| "The EventLog has LogId 8" | Thinks identity values are reused after rollback | "The rolled-back row consumed an identity value. Does SQL Server give it back?" |
| "S9 is blocked and logged" | Missed that S8 disabled the trigger | "What is the state of trg underscore ProtectSchema after S8?" |
| "S11 logs both A and B" | Missed S10 | "Which trigger did S10 disable?" |
| "S11 also fires the Cull trigger" | Confuses INSERT with UPDATE events | "Cull is an AFTER UPDATE trigger. Is S11 an update?" |
| "Final Batches has no batch five" or "batch five is in tank one" | Misreads S11 | "S11 inserted a Carp batch. Which tank, and which identity value comes next after four?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the three questions to ask per statement: which triggers can fire, how many times, and what survives a ROLLBACK.

- **INSTEAD OF versus AFTER.** An INSTEAD OF trigger replaces the statement. It is the only way to INSERT, UPDATE or DELETE through a grouped or multi-table view: the engine never tries to write the view, it just fires the trigger with inserted populated from the VALUES list, including a value for the aggregate column TotalFish, which the trigger is free to interpret as the batch size. There is only one INSTEAD OF trigger per event per object, so `sp_settriggerorder` refuses it with error 15133. That is S2 and S3.
- **`sp_settriggerorder`.** Among several AFTER triggers for the same event, the order is undefined unless you pin one as First and one as Last, with `@stmttype` INSERT, UPDATE or DELETE, or a DDL event name with `@namespace` DATABASE or SERVER. S1 pins B first, so S3 logs B then A, and `sys.trigger_events` shows `is_first = 1` for B.
- **Nested triggers versus recursive triggers.** `nested triggers` is a server option, default one: a trigger's DML fires triggers on other tables, which is why the INSTEAD OF body's insert into Batches fires A and B in S3. `RECURSIVE_TRIGGERS` is a database option, default OFF: it governs direct recursion, a trigger firing itself. S4 halves batch one once, to 2500, and stops. After S5 turns it on, S6 recurses: 5000, 2500, 1250, each logged and halved, until inserted holds 625 and the guard returns. Without the guard, recursion would hit the 32-level limit with error 217 and the whole statement would be rolled back. The client sees one row affected in both S4 and S6: the count of the original statement, not the trigger's inner DML.
- **DDL triggers, EVENTDATA and ROLLBACK.** A DDL trigger ON DATABASE fires after the DDL statement, inside its transaction. `EVENTDATA()` is XML with EventType, SchemaName, ObjectName, LoginName, TSQLCommand and more; it needs `QUOTED_IDENTIFIER ON` at creation because of the XML methods. ROLLBACK undoes the DROP TABLE and everything the trigger wrote before it, so the 'DDL attempt' row vanishes. Statements after ROLLBACK still run in their own auto-commit transactions, so 'DDL blocked' survives as LogId 9. The client gets error 3609 and the batch is aborted. The identity value 8 was consumed by the rolled-back row and is never reused. That is S7.
- **DISABLE TRIGGER.** It flips `is_disabled` to 1 and keeps the object; ENABLE TRIGGER restores it. S8 disables the DDL trigger, so S9's ALTER TABLE runs unlogged and unblocked. S10 disables AuditA, so S11's direct insert fires only AuditB, logging row 10. The added Notes column does not invalidate the triggers on Batches.

Memory hook: "INSTEAD OF replaces, AFTER follows, only AFTER can be ordered. Nested is server, recursive is database. ROLLBACK eats what came before it, and burns the identity."

## 9. Follow-up oral questions (optional)

1. "If the Cull trigger had no IF NOT EXISTS guard and RECURSIVE underscore TRIGGERS were ON, what would S6 do?" (Recurse until the 32-level nesting limit, error 217, and the entire UPDATE is rolled back.)
2. "If the DDL trigger had used RAISERROR or PRINT before the ROLLBACK instead of an INSERT, would the message reach the client?" (Yes. Messages are not transactional, so they survive the ROLLBACK.)
3. "To order a DDL trigger with sp underscore settriggerorder, what extra parameter is required?" (`@namespace`, DATABASE or SERVER; omitting it raises error 15600.)

## 10. References

- CREATE TRIGGER, INSTEAD OF and AFTER DML triggers, DDL triggers: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-trigger-transact-sql
- sp_settriggerorder, First and Last, `@namespace` for DDL triggers: https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-settriggerorder-transact-sql
- Nested triggers and recursive triggers, `nested triggers` server option: https://learn.microsoft.com/en-us/sql/relational-databases/triggers/create-nested-triggers
- ALTER DATABASE SET options, RECURSIVE_TRIGGERS: https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-database-transact-sql-set-options
- EVENTDATA function: https://learn.microsoft.com/en-us/sql/t-sql/functions/eventdata-transact-sql
- DISABLE TRIGGER: https://learn.microsoft.com/en-us/sql/t-sql/statements/disable-trigger-transact-sql
- Modify data through a view: https://learn.microsoft.com/en-us/sql/relational-databases/views/modify-data-through-a-view
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
