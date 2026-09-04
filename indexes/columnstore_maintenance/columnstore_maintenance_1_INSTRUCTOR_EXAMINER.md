# Instructor-Examiner guide — Columnstore Maintenance 1

Companion to [columnstore_maintenance_1.md](columnstore_maintenance_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

**Specific to this question.** This is an eight-step prediction question about rowgroup states. Take the steps one at a time, S1 to S8. For each step the learner must describe the rowgroups: their state, total rows, deleted rows, and for compressed rowgroups the trim reason and the transition reason. Numbers are large; say them slowly, for example "one hundred two thousand three hundred ninety-nine". Accept the state names in any casing.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Design and implement database objects.
- Task bullet: Tables, data types, columns, indexes, column store indexes.
- What is tested: the 102,400-row bulk-load threshold, delta rowgroups versus compressed rowgroups, how deletes are recorded in each store, what `REORGANIZE`, `REORGANIZE WITH (COMPRESS_ALL_ROW_GROUPS = ON)` and `REBUILD` each do, and how to read `sys.dm_db_column_store_row_group_physical_stats`.

## 2. Scenario to read aloud

**Piece 1, the story.** "A wind farm logs one telemetry reading per second from every turbine into a SQL Server 2025 database called WindFarmLog. The readings live in a table with a clustered columnstore index. We will load, delete, and then maintain that index, and after every step we look at the rowgroups."

**Piece 2, the table.** "Schema Telemetry, table Telemetry dot TurbineReadings. Five columns: ReadingId, a bigint. TurbineId, an integer. ReadAt, a datetime2 with zero decimals. WindSpeed, decimal five comma two. PowerKw, decimal eight comma two. There is no primary key. The table has a clustered columnstore index named CCI underscore TurbineReadings."

**Piece 3, the helper view.** "A view, Telemetry dot RowGroups, selects from the DMV sys dot dm underscore db underscore column underscore store underscore row underscore group underscore physical underscore stats, filtered to this table. It shows six columns: row underscore group underscore id, state underscore desc, total underscore rows, deleted underscore rows, trim underscore reason underscore desc, and transition underscore to underscore compressed underscore state underscore desc. After every step we run SELECT star from this view, ordered by row group id."

**Piece 4, how rows are generated.** "Each load builds numbers with a CTE of ten digits cross-joined six times, takes TOP n with ROW underscore NUMBER, and inserts in a single INSERT dot dot dot SELECT statement. ReadingId is the number n. TurbineId is n modulo twenty plus one, so the rows are spread evenly over turbines one to twenty: each turbine owns exactly one row in twenty of every load. The other columns are derived from n and do not matter."

**Piece 5, S1 and S2.** "S1 inserts, in one statement, one hundred two thousand three hundred ninety-nine rows. That is 102,400 minus one. S2 inserts, in one statement, exactly one hundred two thousand four hundred rows, with ReadingId starting at two hundred thousand and one."

**Piece 6, S3 and S4.** "S3 deletes every row where TurbineId equals 7. Remember, that touches one row in twenty of both loads. S4 deletes rows where TurbineId is 8 or 9, and ReadingId is greater than two hundred thousand. So S4 only touches rows from the second load."

**Piece 7, S5 to S8.** "S5 runs ALTER INDEX CCI underscore TurbineReadings REORGANIZE, with no options. S6 runs the same REORGANIZE WITH open paren COMPRESS underscore ALL underscore ROW underscore GROUPS equals ON close paren. S7 runs a plain REORGANIZE again, no options. S8 runs ALTER INDEX REBUILD. The instance is otherwise idle, so no background process interferes between steps."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE WindFarmLog;
GO
USE WindFarmLog;
GO
CREATE SCHEMA Telemetry;
GO
CREATE TABLE Telemetry.TurbineReadings
(
    ReadingId  BIGINT       NOT NULL,
    TurbineId  INT          NOT NULL,
    ReadAt     DATETIME2(0) NOT NULL,
    WindSpeed  DECIMAL(5,2) NOT NULL,
    PowerKw    DECIMAL(8,2) NOT NULL
);
GO
CREATE CLUSTERED COLUMNSTORE INDEX CCI_TurbineReadings ON Telemetry.TurbineReadings;
GO
CREATE VIEW Telemetry.RowGroups AS
SELECT row_group_id, state_desc, total_rows, deleted_rows,
       trim_reason_desc, transition_to_compressed_state_desc
FROM sys.dm_db_column_store_row_group_physical_stats
WHERE object_id = OBJECT_ID('Telemetry.TurbineReadings');
GO
-- S1: one INSERT ... SELECT of 102,399 rows
WITH Digits AS (SELECT d FROM (VALUES (0),(1),(2),(3),(4),(5),(6),(7),(8),(9)) AS t(d)),
     N AS (SELECT TOP (102399) ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n
           FROM Digits a CROSS JOIN Digits b CROSS JOIN Digits c
                CROSS JOIN Digits d CROSS JOIN Digits e CROSS JOIN Digits f)
INSERT INTO Telemetry.TurbineReadings (ReadingId, TurbineId, ReadAt, WindSpeed, PowerKw)
SELECT n, n % 20 + 1, DATEADD(SECOND, n, '2026-09-01'), 5 + (n % 100) / 10.0, 100 + n % 2000
FROM N;
-- S2: one INSERT ... SELECT of 102,400 rows (ReadingId 200001 ... 302400)
WITH Digits AS (SELECT d FROM (VALUES (0),(1),(2),(3),(4),(5),(6),(7),(8),(9)) AS t(d)),
     N AS (SELECT TOP (102400) 200000 + ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n
           FROM Digits a CROSS JOIN Digits b CROSS JOIN Digits c
                CROSS JOIN Digits d CROSS JOIN Digits e CROSS JOIN Digits f)
INSERT INTO Telemetry.TurbineReadings (ReadingId, TurbineId, ReadAt, WindSpeed, PowerKw)
SELECT n, n % 20 + 1, DATEADD(SECOND, n, '2026-09-01'), 5 + (n % 100) / 10.0, 100 + n % 2000
FROM N;
-- S3
DELETE FROM Telemetry.TurbineReadings WHERE TurbineId = 7;
-- S4
DELETE FROM Telemetry.TurbineReadings WHERE TurbineId IN (8, 9) AND ReadingId > 200000;
-- S5
ALTER INDEX CCI_TurbineReadings ON Telemetry.TurbineReadings REORGANIZE;
-- S6
ALTER INDEX CCI_TurbineReadings ON Telemetry.TurbineReadings REORGANIZE WITH (COMPRESS_ALL_ROW_GROUPS = ON);
-- S7
ALTER INDEX CCI_TurbineReadings ON Telemetry.TurbineReadings REORGANIZE;
-- S8
ALTER INDEX CCI_TurbineReadings ON Telemetry.TurbineReadings REBUILD;
-- after each step
SELECT * FROM Telemetry.RowGroups ORDER BY row_group_id;
```

## 4. The question (ask exactly this)

"After each of the eight statements, S1 to S8, describe what the RowGroups view shows: for every rowgroup, its id, its state, total rows, deleted rows, and, if it is compressed, the trim reason and the transition-to-compressed reason. Let's go one step at a time, starting with S1."

After S3 and again after S8: "And how many rows does SELECT COUNT star return now?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

| After | Rowgroup | State | total_rows | deleted_rows | trim_reason | transition |
|---|---|---|---|---|---|---|
| S1 | 0 | OPEN | 102,399 | 0 | NULL | NULL |
| S2 | 0 | OPEN | 102,399 | 0 | NULL | NULL |
| | 1 | COMPRESSED | 102,400 | 0 | BULKLOAD | BULKLOAD |
| S3 | 0 | OPEN | 97,279 | 0 | NULL | NULL |
| | 1 | COMPRESSED | 102,400 | 5,120 | BULKLOAD | BULKLOAD |
| S4 | 0 | OPEN | 97,279 | 0 | NULL | NULL |
| | 1 | COMPRESSED | 102,400 | 15,360 | BULKLOAD | BULKLOAD |
| S5 | unchanged from S4 | | | | | |
| S6 | 0 | TOMBSTONE | 97,279 | 0 | NULL | NULL |
| | 1 | COMPRESSED | 102,400 | 15,360 | BULKLOAD | BULKLOAD |
| | 2 | COMPRESSED | 97,279 | 0 | REORG | REORG_FORCED |
| S7 | 1 | TOMBSTONE | 102,400 | 15,360 | NULL | NULL |
| | 2 | TOMBSTONE | 97,279 | 0 | NULL | NULL |
| | 3 | COMPRESSED | 184,319 | 0 | REORG | MERGE |
| S8 | 0 | COMPRESSED | 184,319 | 0 | RESIDUAL_ROW_GROUP | INDEX_BUILD |

COUNT star: 194,559 after S3 (97,279 plus 102,400 minus 5,120). 184,319 after S8, and in fact already after S4; S5 to S8 never change the logical row count.

Key facts per step: S1 is one row below the threshold, so all rows go to an OPEN delta rowgroup. S2 is exactly at the threshold, so the batch is compressed directly. S3 removes 5,120 rows physically from the delta rowgroup, and marks 5,120 rows in the compressed rowgroup. S4 marks 10,240 more, for 15 percent. S5 changes nothing. S6 forces the open delta rowgroup into a new compressed rowgroup and leaves a tombstone; the deleted rows remain. S7 merges the two small compressed rowgroups into one, dropping the deleted rows on the way. S8 rebuilds into a single rowgroup with no tombstones.

## 6. Hint ladder (one hint per attempt, in order)

**S1**
1. "There is a magic number for bulk loads into a columnstore. Is 102,399 above or below it?"
2. "The threshold is 102,400 rows per batch. Below it, where do the rows go?"
3. "They go to the delta store. What is the state of a delta rowgroup that still accepts rows, and what are its trim and transition reasons?"

**S2**
1. "This batch is exactly 102,400. Is the threshold at least or more than?"
2. "At least. So the batch is compressed directly. What does the DMV say about why the rowgroup is smaller than a million rows, and how it got compressed?"
3. "Both reasons are the same word, and it describes a bulk load. And rowgroup 0 is untouched."

**S3**
1. "Turbine 7 owns one row in twenty in each rowgroup. How many rows is that in each?"
2. "Now think about the two stores separately. A delta rowgroup is a regular row-format structure. A compressed rowgroup is immutable. Which one can really lose rows?"
3. "In the delta rowgroup total rows falls by 5,120. In the compressed rowgroup total rows does not change; which column records the delete?"
4. "For the count, add the live rows: delta total, plus compressed total minus deleted."

**S4**
1. "ReadingId greater than 200,000 restricts the delete to which rowgroup?"
2. "Two more turbines, one in twenty each, in the compressed rowgroup only. Add them to the deleted rows counter."

**S5**
1. "What does a plain REORGANIZE do to a CLOSED delta rowgroup? And is rowgroup 0 CLOSED?"
2. "It is OPEN, not CLOSED. Plain REORGANIZE does not touch open delta rowgroups. What about the deleted rows in rowgroup 1? Is there another compressed rowgroup to merge with?"
3. "There is none. On this build, with nothing to merge and nothing closed, the statement changes nothing at all."

**S6**
1. "What does the COMPRESS_ALL_ROW_GROUPS option add to REORGANIZE?"
2. "It compresses OPEN delta rowgroups too. Is the delta rowgroup compressed in place, or does a new rowgroup appear?"
3. "A new rowgroup 2 appears, and rowgroup 0 gets a state that means 'formerly in the delta store, no longer used'. What reasons does the new rowgroup carry? Think 'forced by REORG'."
4. "And rowgroup 1 with its 15,360 deleted rows: does this option touch it?"

**S7**
1. "Now there are two compressed rowgroups, both far below 1,048,576 rows. What does REORGANIZE do with small compressed rowgroups?"
2. "It merges them. Only the live rows are copied. Compute 102,400 minus 15,360 plus 97,279."
3. "The merged rowgroup's transition reason is the operation itself. And the two sources become tombstones."

**S8**
1. "REBUILD drops and re-creates the index from all the data. How many rowgroups do 184,319 rows need?"
2. "One. Its trim reason describes the last rowgroup of an index build with fewer than a million rows, and its transition reason describes an index build. Tombstones are gone."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "S1 is compressed, it is a bulk insert" | Thinks INSERT SELECT always compresses | "Bulk loads compress directly only from a certain batch size. Is 102,399 there yet?" |
| "S2 goes to the delta store, the threshold is more than 102,400" | Off by one on the rule | "The rule says at least. Check the boundary again." |
| "S3 leaves total rows at 102,399 with 5,120 deleted in rowgroup 0" | Applies delete-bitmap logic to the delta store | "Rowgroup 0 is row format, not compressed segments. Can the engine simply remove those rows?" |
| "S3 reduces total rows of rowgroup 1 to 97,280" | Thinks compressed rows are removed | "Compressed segments are immutable. Where does the engine record a deleted compressed row?" |
| "S5 removes the deleted rows because 15 percent is above 10 percent" | Trusts the documented rule unconditionally | "The engine removes deleted rows while merging into a new rowgroup. Is there anything to merge with here?" |
| "S5 compresses rowgroup 0" | Confuses OPEN with CLOSED | "Plain REORGANIZE compresses closed delta rowgroups. Is rowgroup 0 closed?" |
| "S6 removes the 15,360 deleted rows" | Thinks COMPRESS_ALL_ROW_GROUPS cleans everything | "That option is about the delta store. Which rowgroup is in the delta store?" |
| "S6 compresses rowgroup 0 in place" | Does not know about tombstones | "Compression produces a new rowgroup. What becomes of the old delta rowgroup?" |
| "S7 does nothing, like S5" | Misses the new merge partner | "What is different now compared with S5? Count the compressed rowgroups." |
| "S8 keeps the tombstones" | Thinks REBUILD is just another merge | "REBUILD re-creates the whole index. Is anything from the old structure kept?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the two stores:

- **Compressed rowgroups** hold up to 1,048,576 rows as immutable column segments. **Delta rowgroups** hold rows in row format until they are compressed. A bulk batch of at least 102,400 rows is compressed directly, with trim reason and transition BULKLOAD; a smaller batch, or any trickle insert, goes to an OPEN delta rowgroup. That is S1 versus S2. A delta rowgroup closes at 1,048,576 rows and the tuple mover compresses closed rowgroups in the background. Two batches of 51,200 do not add up to a compressed rowgroup: the threshold is per batch.
- **Deletes.** In the delta store rows are physically removed and deleted underscore rows stays zero. In a compressed rowgroup the row is only flagged in the delete bitmap: deleted underscore rows grows, total underscore rows stays. An UPDATE on a compressed row is a flag plus an insert into the delta store. Fragmentation of a columnstore is deleted rows over total rows. That is S3 and S4.

Then the three maintenance commands:

- **REORGANIZE** is online. It compresses CLOSED delta rowgroups, and merges small compressed rowgroups into bigger ones; deleted rows are dropped as part of a merge, transition MERGE. It does not touch OPEN delta rowgroups, and with a single compressed rowgroup and nothing to merge it changed nothing, even at 60 percent deleted rows in a side test. That is S5 versus S7.
- **REORGANIZE WITH COMPRESS_ALL_ROW_GROUPS = ON** additionally forces OPEN delta rowgroups into the columnstore: a new compressed rowgroup with trim reason REORG and transition REORG_FORCED, the old delta rowgroup becomes a TOMBSTONE. It does nothing about deleted rows. Use it right after a load. That is S6.
- **REBUILD** re-creates the index from scratch, one rowgroup per 1,048,576 rows, purging every deleted row and every tombstone; trim reason RESIDUAL_ROW_GROUP and transition INDEX_BUILD. It is offline unless ONLINE equals ON, needs double space, and updates statistics. Rebuild only the affected partition on partitioned tables. That is S8.

Then the rule of thumb: compute one hundred times deleted rows over total rows per compressed rowgroup; act at roughly 10 to 20 percent. REORGANIZE first because it is online and cheap; REBUILD when a rowgroup will not merge or fragmentation is heavy. Since SQL Server 2019 a background merge task does much of this automatically, with trim reason AUTO_MERGE.

Memory hook: "102,400 or delta. Delete only marks. Reorganize merges, compress-all forces, rebuild purges."

## 9. Follow-up oral questions (optional)

1. "A single INSERT SELECT of 1,100,000 rows into an empty columnstore table: what rowgroups result?" (One COMPRESSED rowgroup of 1,048,576 rows with trim reason NO_TRIM, and an OPEN delta rowgroup of 51,424 rows, because the remainder is below 102,400.)
2. "After S8, a single INSERT VALUES adds one reading. Where does it go?" (A new OPEN delta rowgroup with total rows 1.)
3. "Which state means 'a delta rowgroup that reached its maximum and is waiting for the tuple mover'?" (CLOSED.)

## 10. References

- Columnstore indexes, data loading guidance, including the 102,400-row rule: https://learn.microsoft.com/en-us/sql/relational-databases/indexes/columnstore-indexes-data-loading-guidance
- Reorganize and rebuild indexes, including the columnstore REORGANIZE and REBUILD rules: https://learn.microsoft.com/en-us/sql/relational-databases/indexes/reorganize-and-rebuild-indexes
- sys.dm_db_column_store_row_group_physical_stats, with all state, trim and transition values: https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-objects/sys-dm-db-column-store-row-group-physical-stats-transact-sql
- ALTER INDEX, columnstore examples: https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-index-transact-sql
- Columnstore indexes overview: https://learn.microsoft.com/en-us/sql/relational-databases/indexes/columnstore-indexes-overview
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
