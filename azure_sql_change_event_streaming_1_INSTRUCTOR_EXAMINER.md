# Instructor-Examiner guide — Azure SQL Change Event Streaming 1

Companion to [azure_sql_change_event_streaming_1.md](azure_sql_change_event_streaming_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.
7. **This is a hands-on Azure lab question.** Before reading the scenario, ask: "Have you already run this lab in your Azure subscription?" If **yes**, do not quiz right away. Walk through the steps D1 to D7 and ask what the learner observed at each one: how many events the consumer printed, what the envelope looked like, which statements failed and with which message number. Then ask the question. If **no**, read section 2 in full, including the provisioning and consumer pieces, so the question can be answered from the documented facts alone. The learner does not need Azure to answer.
8. **This is a multiple-choice question.** Read all four options, pieces 12 to 15, before taking an answer. Take one letter as the answer. Do not accept "a or b".

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Implement data change tracking and event-driven patterns.
- Task bullet: Configure change event streaming from Azure SQL Database to Azure Event Hubs.
- What is tested: what CES emits and what it does not, the shape of a CES CloudEvent, ordering inside a transaction, which DDL is blocked, and which destination type Azure SQL Database accepts.

## 2. Scenario to read aloud

**Piece 1, the story.** "A ski resort sells lift passes from an Azure SQL Database called LiftPass. They want every change to a table named Resort dot Passes pushed to Azure Event Hubs, using change event streaming, C E S, which is in preview. The database authenticates to the event hub with the logical server's managed identity. A small Python script consumes the events. This is a hands-on lab: you provision, enable the stream, run seven steps D1 to D7, read the events, and then answer a multiple-choice question about what the consumer sees."

**Piece 2, provisioning with the Azure CLI, part one.** "A bash script uses the Azure CLI. It picks the region westeurope and a random suffix. It names a resource group rg dash dp800 dash liftpass, a logical server sql dash liftpass, a database LiftPass, an Event Hubs namespace ehns dash liftpass, and an event hub called passes. It reads your own signed-in user principal name and object id, and your public IP address. It creates the resource group. Then it creates the Event Hubs namespace with SKU Standard, capacity one, and Kafka enabled. A comment says: Standard tier, because CES publishes with the Kafka protocol on port 9093, which the Basic tier does not offer. It creates the event hub passes with two partitions, cleanup policy Delete, and twenty-four hours retention."

**Piece 3, provisioning, part two.** "Next it creates the SQL logical server with Entra-only authentication, a system-assigned managed identity, and you as the external admin. It adds a firewall rule for your IP. It creates the database LiftPass as General Purpose, Gen5, one vCore, serverless, auto-pause after sixty minutes, minimum capacity zero point five, local backup redundancy. It reads the server's fully qualified domain name and the principal id of the server's managed identity. Then two role assignments, both scoped to the one event hub: the server identity gets Azure Event Hubs Data Sender, and you get Azure Event Hubs Data Receiver. Finally it echoes the destination: the namespace dot servicebus dot windows dot net, colon 9093, slash passes."

**Piece 4, the table and the pre-existing rows.** "You connect with sqlcmd using Entra authentication. You create a schema Resort and a table Resort dot Passes with four columns. PassId, an integer, the primary key. Holder, an NVARCHAR of forty. Days, a TINYINT. Price, a DECIMAL seven comma two. Then you insert two rows before streaming is enabled: pass 1, Ana Roig, one day, sixty-two euros. Pass 2, Tomas Vik, six days, two hundred ninety."

**Piece 5, enabling the stream.** "You create a database master key with a password. You create a database scoped credential called LiftHubCred with identity equal to the string Managed Identity. Then three procedures. First, sys dot sp underscore enable underscore event underscore stream. Second, sys dot sp underscore create underscore event underscore stream underscore group with these parameters: stream group name PassesGroup, destination type AzureEventHubs, destination location the namespace dot servicebus dot windows dot net colon 9093 slash passes, destination credential LiftHubCred, max message size 256 kilobytes, and partition key scheme None. Third, sys dot sp underscore add underscore object underscore to underscore event underscore stream underscore group, with PassesGroup and Resort dot Passes."

**Piece 6, the catalog checks.** "Three checks follow. A select of name and is underscore event underscore stream underscore enabled from sys dot databases for the current database. A select of name and is underscore replicated from sys dot tables. And exec sys dot sp underscore help underscore change underscore feed."

**Piece 7, the consumer.** "Before the DML, you start a Python script. It creates an EventHubConsumerClient for the namespace, hub passes, consumer group dollar Default, using DefaultAzureCredential, which picks up your az login. For every event it parses the body as JSON; that is the CloudEvent envelope. Then it parses the data attribute again, because data is a JSON string. From data it takes eventsource and eventrow. It prints the partition id, the envelope type, the envelope operation, schema dot table, the pkkey, the transaction sequence number, and the list of column names from cols. Then it prints eventrow old and eventrow current. It receives from starting position minus one, meaning from the beginning, with a thirty-second max wait."

**Piece 8, steps D1 and D2.** "Now the seven steps, each in its own batch. D1 inserts two rows: pass 3, Ines Sola, three days, one hundred fifty; and pass 4, Piet Bos, two days, one hundred ten. D2 is one explicit transaction: begin tran, update Passes set Price to one hundred forty-five where PassId is 3, delete from Passes where PassId is 4, commit."

**Piece 9, steps D3, D4 and D5.** "D3 is TRUNCATE TABLE Resort dot Passes. D4 alters the table to add a column Lane, TINYINT, nullable. D5 runs sp underscore rename to rename the column Holder to Guest."

**Piece 10, steps D6 and D7.** "D6 inserts pass 5, Lea Kurz, one day, sixty-two, with Lane equal to 2, naming the columns PassId, Holder, Days, Price and Lane. D7 calls sp underscore create underscore event underscore stream underscore group again, with a new group called KafkaGroup, destination type AzureEventHubsApacheKafka, the same destination location on port 9093, and the same credential LiftHubCred."

**Piece 11, cost and cleanup.** "For the record: one Standard throughput unit costs about two cents an hour, plus a negligible per-million-events charge, and the serverless database bills while active. Cleanup removes the table from the group, drops the group, disables the event stream so the log can truncate, and deletes the resource group."

**Piece 12, option a.** "Option a says: the consumer prints five events, insert, insert from D1, update, delete from D2, and insert from D6, and never anything for passes 1 and 2. Each envelope has type com dot microsoft dot SQL dot CES dot DML dot V1, source equal to a slash, the operation attribute at envelope level, and a data attribute that is a JSON string. Inside data, eventsource carries db, schema, tbl, the cols array, pkkey as an array with columnname PassId and value 3, and transaction with commitlsn, beginlsn, sequencenumber and committime. Eventrow old is an empty object string for inserts, and eventrow current is an empty object string for deletes. D2's two events share a commitlsn and have sequencenumber 1 and 2. D3 fails with message 23663. D4 succeeds and D6's cols then includes Lane. D5 fails with message 4928. And D7 fails with message 23626 because AzureEventHubs is the only destination type Azure SQL Database accepts."

**Piece 13, option b.** "Option b says: the consumer prints seven events, the five DML events plus one insert per pre-existing row, passes 1 and 2, emitted as a snapshot when sp underscore add underscore object ran. And D3 emits one delete per row, so the count is actually nine."

**Piece 14, option c.** "Option c says: D4 and D5 each produce a CloudEvent of type com dot microsoft dot SQL dot CES dot DDL dot V1 describing the schema change, which consumers must apply before reading the next insert. D3 is allowed and produces no event."

**Piece 15, option d.** "Option d says: because Azure SQL Database publishes CES over AMQP on port 5671, a Basic-tier namespace and a destination location on port 5671 would have worked, and D7's AzureEventHubsApacheKafka is the value to use when the namespace has Kafka enabled."

## 3. Setup script (reference only; do not read verbatim unless asked)

```bash
LOCATION="westeurope"
SUFFIX=$RANDOM
RG="rg-dp800-liftpass-$SUFFIX"
SQL="sql-liftpass-$SUFFIX"; DB="LiftPass"
EHNS="ehns-liftpass-$SUFFIX"; EH="passes"
ADMIN_UPN=$(az ad signed-in-user show --query userPrincipalName -o tsv)
ADMIN_OID=$(az ad signed-in-user show --query id -o tsv)
MYIP=$(curl -s https://api.ipify.org)

az group create -n $RG -l $LOCATION
# Standard tier: CES publishes with the Kafka protocol on port 9093, which the Basic tier does not offer
az eventhubs namespace create -g $RG -n $EHNS -l $LOCATION --sku Standard --capacity 1 --enable-kafka true
az eventhubs eventhub create -g $RG --namespace-name $EHNS -n $EH --partition-count 2 --cleanup-policy Delete --retention-time 24
EH_ID=$(az eventhubs eventhub show -g $RG --namespace-name $EHNS -n $EH --query id -o tsv)

az sql server create -g $RG -n $SQL -l $LOCATION --enable-ad-only-auth --assign-identity --identity-type SystemAssigned \
  --external-admin-principal-type User --external-admin-name "$ADMIN_UPN" --external-admin-sid $ADMIN_OID
az sql server firewall-rule create -g $RG -s $SQL -n client --start-ip-address $MYIP --end-ip-address $MYIP
az sql db create -g $RG -s $SQL -n $DB -e GeneralPurpose -f Gen5 -c 1 --compute-model Serverless \
  --auto-pause-delay 60 --min-capacity 0.5 --backup-storage-redundancy Local
SQL_FQDN=$(az sql server show -g $RG -n $SQL --query fullyQualifiedDomainName -o tsv)
SQL_MI=$(az sql server show -g $RG -n $SQL --query identity.principalId -o tsv)

# the server identity may SEND to this one event hub; you may RECEIVE from it
az role assignment create --assignee-object-id $SQL_MI --assignee-principal-type ServicePrincipal \
  --role "Azure Event Hubs Data Sender" --scope $EH_ID
az role assignment create --assignee-object-id $ADMIN_OID --assignee-principal-type User \
  --role "Azure Event Hubs Data Receiver" --scope $EH_ID
echo "Destination: $EHNS.servicebus.windows.net:9093/$EH"
```

```sql
CREATE SCHEMA Resort;
GO
CREATE TABLE Resort.Passes
(
    PassId   INT           NOT NULL PRIMARY KEY,
    Holder   NVARCHAR(40)  NOT NULL,
    Days     TINYINT       NOT NULL,
    Price    DECIMAL(7,2)  NOT NULL
);
INSERT INTO Resort.Passes VALUES (1, N'Ana Roig', 1, 62.00), (2, N'Tomas Vik', 6, 290.00);   -- rows that exist BEFORE streaming
GO
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'Str0ng!Passw0rd#2026';
CREATE DATABASE SCOPED CREDENTIAL LiftHubCred WITH IDENTITY = 'Managed Identity';
GO
EXEC sys.sp_enable_event_stream;
EXEC sys.sp_create_event_stream_group
     @stream_group_name = N'PassesGroup', @destination_type = N'AzureEventHubs',
     @destination_location = N'<EHNS>.servicebus.windows.net:9093/passes',
     @destination_credential = LiftHubCred, @max_message_size_kb = 256, @partition_key_scheme = N'None';
EXEC sys.sp_add_object_to_event_stream_group N'PassesGroup', N'Resort.Passes';
GO
SELECT name, is_event_stream_enabled FROM sys.databases WHERE name = DB_NAME();
SELECT name, is_replicated FROM sys.tables;
EXEC sys.sp_help_change_feed;
GO
```

```python
import json, os
from azure.eventhub import EventHubConsumerClient
from azure.identity import DefaultAzureCredential

client = EventHubConsumerClient(
    fully_qualified_namespace=os.environ["EHNS"] + ".servicebus.windows.net",
    eventhub_name="passes", consumer_group="$Default", credential=DefaultAzureCredential())

def on_event(ctx, event):
    if event is None:
        return
    ce = json.loads(event.body_as_str())           # the CloudEvent envelope
    data = json.loads(ce["data"])                  # the data attribute is a JSON *string*
    src, row = data["eventsource"], data["eventrow"]
    print(f'p{ctx.partition_id} {ce["type"]} {ce["operation"]} {src["schema"]}.{src["tbl"]} '
          f'pk={src["pkkey"]} seq={src["transaction"]["sequencenumber"]} cols={[c["name"] for c in src["cols"]]}')
    print("   old    :", row["old"]); print("   current:", row["current"])

with client:
    client.receive(on_event=on_event, starting_position="-1", max_wait_time=30)
```

```sql
-- D1
INSERT INTO Resort.Passes VALUES (3, N'Ines Sola', 3, 150.00), (4, N'Piet Bos', 2, 110.00);
-- D2
BEGIN TRAN; UPDATE Resort.Passes SET Price = 145.00 WHERE PassId = 3; DELETE Resort.Passes WHERE PassId = 4; COMMIT;
-- D3
TRUNCATE TABLE Resort.Passes;
-- D4
ALTER TABLE Resort.Passes ADD Lane TINYINT NULL;
-- D5
EXEC sp_rename 'Resort.Passes.Holder', 'Guest', 'COLUMN';
-- D6
INSERT INTO Resort.Passes (PassId, Holder, Days, Price, Lane) VALUES (5, N'Lea Kurz', 1, 62.00, 2);
-- D7
EXEC sys.sp_create_event_stream_group @stream_group_name = N'KafkaGroup', @destination_type = N'AzureEventHubsApacheKafka',
     @destination_location = N'<EHNS>.servicebus.windows.net:9093/passes', @destination_credential = LiftHubCred;
```

Cleanup:

```sql
EXEC sys.sp_remove_object_from_event_stream_group N'PassesGroup', N'Resort.Passes';
EXEC sys.sp_drop_event_stream_group N'PassesGroup';
EXEC sys.sp_disable_event_stream;
```

```bash
az group delete -n $RG --yes --no-wait
```

## 4. The question (ask exactly this)

"Which statement about the lab is correct? Option a, option b, option c or option d?"

- **a.** The consumer prints five events, `INS`, `INS` (D1), `UPD`, `DEL` (D2) and `INS` (D6), never anything for passes 1 and 2. Each envelope has `type = com.microsoft.SQL.CES.DML.V1`, `source = "/"`, the `operation` attribute at envelope level, and a `data` attribute that is a JSON string whose `eventsource` carries `db`, `schema`, `tbl`, the `cols` array, `pkkey` (`[{"columnname":"PassId","value":"3"}]`) and `transaction` (`commitlsn`, `beginlsn`, `sequencenumber`, `committime`); `eventrow.old` is `"{}"` for inserts and `eventrow.current` is `"{}"` for deletes. D2's two events share a `commitlsn` and have `sequencenumber` 1 and 2. D3 fails with `Msg 23663`, D4 succeeds and D6's `cols` then includes `Lane`, D5 fails with `Msg 4928`, and D7 fails with `Msg 23626` because `AzureEventHubs` is the only destination type Azure SQL Database accepts.
- **b.** The consumer prints seven events: the five DML events plus one `INS` per pre-existing row (passes 1 and 2), emitted as a snapshot when `sp_add_object_to_event_stream_group` ran; D3 emits one `DEL` per row, so the count is actually nine.
- **c.** D4 and D5 each produce a CloudEvent of type `com.microsoft.SQL.CES.DDL.V1` describing the schema change, which consumers must apply before reading the next `INS`; D3 is allowed and produces no event.
- **d.** Because Azure SQL Database publishes CES over AMQP on port 5671, a Basic-tier namespace and `@destination_location = N'<EHNS>.servicebus.windows.net:5671/passes'` would have worked, and D7's `AzureEventHubsApacheKafka` is the value to use when the namespace has Kafka enabled.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

| Item | Outcome |
|---|---|
| Events printed | Five: INS, INS (D1), UPD, DEL (D2), INS (D6). Passes 1 and 2 are never streamed. |
| Envelope | `type` com.microsoft.SQL.CES.DML.V1, `source` "/", `operation` INS/UPD/DEL at envelope level, `data` is a JSON string parsed a second time. |
| data.eventsource | db, schema, tbl, cols (name, type, index), pkkey (columnname/value), transaction (commitlsn, beginlsn, sequencenumber, finalevent, committime). |
| data.eventrow | old and current, both strings. Insert: old = "{}". Delete: current = "{}". Update: both images, so the UPD carries 150.00 in old and 145.00 in current. |
| D2 | Two events, same commitlsn, sequencenumber 1 (UPD) and 2 (DEL). |
| D3 | Fails, Msg 23663: cannot truncate a table used for Change Streams. |
| D4 | Succeeds; no event. D6's cols then lists Lane. |
| D5 | Fails, Msg 4928: cannot alter column Holder because it is enabled for Change Streams. |
| D6 | Succeeds, one INS event with five columns. |
| D7 | Fails, Msg 23626: @destination_type failed validation. AzureEventHubs is the only value accepted on Azure SQL Database. |

Why the wrong options are wrong:

- **b.** CES never seeds or snapshots pre-existing rows, and TRUNCATE is rejected, not expanded into deletes.
- **c.** There is no DDL event type. DDL emits nothing; the next DML event just shows the new cols. And D3 is not allowed.
- **d.** AzureEventHubs on Azure SQL Database uses the Kafka protocol on port 9093, which needs Standard tier or higher. AMQP on 5671 is the deprecated AzureEventHubsAmqp path on Managed Instance and SQL Server only. AzureEventHubsApacheKafka is rejected on Azure SQL Database.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with a single fact: does CES send anything for rows that already existed before the stream was enabled? Think about whether there is a snapshot or seeding step."
2. "Now think about TRUNCATE TABLE. Is it a logged row-by-row delete? Would a change stream fed from the transaction log be happy with it?"
3. "Next, DDL. Does CES have a separate event type for schema changes, or does it only know about inserts, updates and deletes? What does a consumer see after a column is added?"
4. "Look at the ports. The provisioning script chose Standard tier for a stated reason, and the destination location ends with colon 9093. Which protocol uses 9093, and which uses 5671?"
5. "Two options can now be eliminated because they claim events that CES does not produce: a snapshot in one, DDL events in another. Two remain."
6. "Between the last two, one says Azure SQL Database accepts a Kafka-named destination type and AMQP on port 5671. Does the script's own comment agree with that? The other option lists the envelope fields and error numbers. Which one is consistent with everything you have heard?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "b, because the existing rows are sent when the table is added to the group" | Assumes CES seeds like a snapshot | "Is there any seeding or snapshot in CES? Compare with how CDC starts." |
| "b, TRUNCATE gives one delete per row" | Thinks TRUNCATE is logged per row | "Does TRUNCATE even run on a table enabled for CES?" |
| "c, there must be a DDL event so the consumer can adapt" | Invents a DDL event type | "How does the consumer learn the new column list? Look at what each DML event already carries." |
| "c, D3 is allowed" | Forgets the TRUNCATE restriction | "Recall the message number mentioned for D3 in another option." |
| "d, AMQP is the classic Event Hubs protocol" | Confuses the deprecated AMQP path with Azure SQL Database | "Why did the provisioning script insist on Standard tier and port 9093?" |
| "d, D7 succeeds because Kafka is enabled on the namespace" | Thinks the destination type follows the namespace feature | "Which destination types are valid on Azure SQL Database specifically, as opposed to SQL Server 2025?" |
| "a, but I think D4 also produces an event" | Mixes DDL with DML | "Option a says D4 succeeds and D6's cols includes Lane. Does that require a separate DDL event?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain what CES streams and what it does not:

- **Only DML, only after enabling.** CES emits events for INSERT, UPDATE and DELETE. It does not seed or snapshot rows that existed before enabling, so passes 1 and 2 are never seen. If the initial state is needed, load it separately, exactly as with CDC.
- **The envelope.** Every CES CloudEvent has eleven attributes: specversion, type (com.microsoft.SQL.CES.DML.V1), source (always a slash), id, logicalid, time, datacontenttype, operation (INS, UPD, DEL), segmentindex, finalsegment, and data. Data is a byte array. In JSON encoding it is a string you parse a second time. Logicalid, segmentindex and finalsegment exist because a change bigger than max message size (128 to 1024 KB, default 256) is split into chunks. Column values over one megabyte are silently truncated first.
- **The data attribute.** eventsource holds db, schema, tbl, cols (name, type, index), pkkey (array of columnname and value; the primary key is mandatory for a streamed table) and transaction (commitlsn, beginlsn, sequencenumber, finalevent which is always false, committime). eventrow holds old and current, both objects wrapped in strings. Insert: old is "{}". Delete: current is "{}". Update: both images. CES is a before-and-after stream like CDC, not a key-only stream like change tracking.
- **Ordering.** sequencenumber orders operations inside one transaction; D2's UPD and DEL share a commitlsn and are numbered 1 and 2. With partition key scheme None, events are spread round-robin across partitions, so global order across partitions is not guaranteed. StreamGroup, Table or Column, with a partition key column name, pin related events to one partition.
- **D3, D4, D5.** TRUNCATE TABLE is not supported on a CES table, Msg 23663. DDL is not blocked but produces no event; the next DML event reflects the new schema, so D6's cols includes Lane. Renaming a streamed column fails with Msg 4928; table renames fail too; database renames are allowed.
- **D7.** On Azure SQL Database and SQL database in Fabric, AzureEventHubs is the only accepted destination type and it uses Kafka on port 9093. AzureEventHubsApacheKafka is for SQL Server 2025 and Managed Instance. AzureEventHubsAmqp is deprecated. A wrong value fails validation with Msg 23626.
- **CES versus CDC.** CES keeps no capture tables and needs no LSN polling; it pushes CloudEvents to Event Hubs or Fabric Eventstream. CES and CDC cannot coexist on the same database; change tracking can. Both need full recovery, both use the transaction log, which is why the log cannot truncate while events are pending, and why disabling the stream is part of cleanup.

Memory hook: "CES streams DML only, from now on, as before-and-after JSON. No snapshot, no DDL event, no TRUNCATE, no rename. Azure SQL DB speaks Kafka on 9093."

## 9. Follow-up oral questions (optional)

1. "If the lab had used partition key scheme Table instead of None, what changes for the consumer?" (All events for Resort.Passes land on one partition, so their order is preserved end to end.)
2. "Could you enable CDC on LiftPass while the CES group exists?" (No. CES does not support databases configured with CDC. Change tracking is allowed.)
3. "Why must the stream be disabled during cleanup, not just the group dropped?" (While the stream is enabled, the transaction log cannot truncate past pending events. Disabling releases the log.)

## 10. References

- Change event streaming overview: https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/change-event-streaming/overview
- Configure change event streaming (Azure SQL Database, destination types and parameters): https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/change-event-streaming/configure
- CES event message format (CloudEvent envelope and data attribute): https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/change-event-streaming/message-format
- sp_create_event_stream_group: https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-create-event-stream-group-transact-sql
- Azure Event Hubs quotas and tier features (Kafka availability per tier): https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-quotas
- azure-eventhub Python package: https://learn.microsoft.com/en-us/python/api/overview/azure/eventhub-readme
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
