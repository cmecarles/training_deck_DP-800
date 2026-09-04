# Instructor-Examiner guide — Partitioning 1

Companion to [partitioning_1.md](partitioning_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a, b, c and d. Each option is a small result table. Read all four options before taking an answer. The ten dates in piece 4 are the whole puzzle; read them slowly, and repeat the three boundary values and the words RANGE LEFT whenever asked. Encourage the learner to place each row in a partition before choosing. Accept the answer as a letter.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked. Say dates as "the first of February twenty twenty-five".

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Design and implement database objects.
- Task bullet: Implement partitioning for tables and indexes.
- What is tested: that with RANGE LEFT a row equal to a boundary value goes to the lower partition, that n boundaries make n plus one partitions, what `$PARTITION` returns, and that `TRUNCATE TABLE WITH (PARTITIONS (...))` names the partitions to empty, with commas for a list and TO for a range.

## 2. Scenario to read aloud

**Piece 1, the story.** "An energy utility stores electricity-grid sensor telemetry in a SQL Server 2022 database named GridSense. Readings are partitioned by month so that old months can be purged instantly. A batch runs on the brand-new, empty database, and every statement in it succeeds."

**Piece 2, the partition function and scheme.** "A schema Ops is created. Then a partition function PF underscore ReadingMonth on the date type, declared AS RANGE LEFT, FOR VALUES with three boundaries: the first of February twenty twenty-five, the first of March twenty twenty-five, and the first of April twenty twenty-five. Then a partition scheme PS underscore ReadingMonth on that function, ALL TO PRIMARY."

**Piece 3, the table.** "The table Ops dot Readings has four columns: ReadingId, a bigint; SensorId, an integer; ReadingDate, a date; and kWh, a decimal nine comma three. The clustered primary key is on ReadingId and ReadingDate together, and the table is created ON the partition scheme PS underscore ReadingMonth, partitioned by ReadingDate."

**Piece 4, the ten rows, listen carefully to the dates.** "Ten readings are inserted. I will give the id and the date; the sensor and kWh values do not matter. Reading 1, the fifteenth of January. Reading 2, the thirty-first of January. Reading 3, the first of February. Reading 4, the second of February. Reading 5, the twenty-eighth of February. Reading 6, the first of March. Reading 7, the fifteenth of March. Reading 8, the thirty-first of March. Reading 9, the first of April. Reading 10, the second of April. All in twenty twenty-five."

**Piece 5, the truncate.** "Then this statement runs: TRUNCATE TABLE Ops dot Readings WITH open paren PARTITIONS open paren two comma four close paren close paren. Two comma four, with a comma."

**Piece 6, the query.** "After the batch, a query selects dollar PARTITION dot PF underscore ReadingMonth of ReadingDate, aliased PartitionNumber, and COUNT star aliased ReadingCount, from Ops dot Readings, grouped by that same partition expression, ordered by PartitionNumber."

**Piece 7, option a.** "Option a: two rows. Partition 1 with 2 readings. Partition 3 with 3 readings."

**Piece 8, option b.** "Option b: two rows. Partition 1 with 3 readings. Partition 3 with 3 readings."

**Piece 9, option c.** "Option c: two rows. Partition 2 with 3 readings. Partition 4 with 1 reading."

**Piece 10, option d.** "Option d: one row. Partition 1 with 3 readings."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
USE GridSense;
GO

CREATE SCHEMA Ops;
GO

CREATE PARTITION FUNCTION PF_ReadingMonth (date)
AS RANGE LEFT
FOR VALUES ('20250201', '20250301', '20250401');
GO

CREATE PARTITION SCHEME PS_ReadingMonth
AS PARTITION PF_ReadingMonth
ALL TO ([PRIMARY]);
GO

CREATE TABLE Ops.Readings
(
    ReadingId    bigint        NOT NULL,
    SensorId     int           NOT NULL,
    ReadingDate  date          NOT NULL,
    kWh          decimal(9,3)  NOT NULL,
    CONSTRAINT PK_Readings
        PRIMARY KEY CLUSTERED (ReadingId, ReadingDate)
) ON PS_ReadingMonth (ReadingDate);
GO

INSERT INTO Ops.Readings (ReadingId, SensorId, ReadingDate, kWh) VALUES
    ( 1, 101, '20250115',  42.500),
    ( 2, 101, '20250131',  40.250),
    ( 3, 102, '20250201',  38.900),
    ( 4, 102, '20250202',  39.100),
    ( 5, 103, '20250228',  41.000),
    ( 6, 103, '20250301',  37.750),
    ( 7, 104, '20250315',  44.300),
    ( 8, 104, '20250331',  43.600),
    ( 9, 105, '20250401',  36.200),
    (10, 105, '20250402',  35.800);
GO

TRUNCATE TABLE Ops.Readings
WITH (PARTITIONS (2, 4));
GO
```

The query:

```sql
SELECT
    $PARTITION.PF_ReadingMonth(ReadingDate) AS PartitionNumber,
    COUNT(*)                                AS ReadingCount
FROM Ops.Readings
GROUP BY $PARTITION.PF_ReadingMonth(ReadingDate)
ORDER BY PartitionNumber;
```

## 4. The question (ask exactly this)

"Which result set does the query return? Choose one option.

- a. Partition 1 with 2 readings, partition 3 with 3 readings.
- b. Partition 1 with 3 readings, partition 3 with 3 readings.
- c. Partition 2 with 3 readings, partition 4 with 1 reading.
- d. Partition 1 with 3 readings, and nothing else."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: b.**

With RANGE LEFT each boundary belongs to the partition on its left, the lower one. Three boundaries make four partitions:

| Partition | Condition |
|---|---|
| 1 | ReadingDate less than or equal to 2025-02-01 |
| 2 | after 2025-02-01, up to and including 2025-03-01 |
| 3 | after 2025-03-01, up to and including 2025-04-01 |
| 4 | after 2025-04-01 |

Row placement: readings 1, 2, 3 in partition 1 (reading 3 on the first of February equals boundary 1, so it goes left). Readings 4, 5, 6 in partition 2 (reading 6 on the first of March goes left). Readings 7, 8, 9 in partition 3 (reading 9 on the first of April goes left). Reading 10 in partition 4.

Counts before the truncate: 3, 3, 3, 1. The truncate empties partitions 2 and 4, removing readings 4, 5, 6 and 10. Survivors: 1, 2, 3 in partition 1 and 7, 8, 9 in partition 3. GROUP BY shows only nonempty partitions.

| PartitionNumber | ReadingCount |
|---|---|
| 1 | 3 |
| 3 | 3 |

Why the wrong options are wrong:

- a: RANGE RIGHT semantics applied to a RANGE LEFT function. Under RANGE RIGHT the first of February would go to partition 2, leaving partition 1 with only 2 rows.
- c: inverts the WITH PARTITIONS clause, treating 2 and 4 as the partitions to keep. It even reports exactly the rows the truncate removed.
- d: reads 2 comma 4 as the range 2 TO 4, truncating partition 3 as well. A range needs the keyword TO; a comma-separated list names individual partitions.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with the function. Three boundary values: how many partitions does that make?"
2. "The function is RANGE LEFT. When a row's date is exactly equal to a boundary, does it go to the partition below the boundary or above it?"
3. "Three of the ten readings fall exactly on a boundary: readings 3, 6 and 9, the first of February, March and April. Place those three first."
4. "Now count each partition before the truncate. You should have four numbers."
5. "The truncate says PARTITIONS open paren two comma four close paren. Does the list name the partitions to empty or the partitions to keep? And is a comma a list or a range?"
6. "Two partitions survive. Does GROUP BY show a zero row for an emptied partition, or does it just leave it out?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, because February starts on the first of February" | Applies RANGE RIGHT intuition to RANGE LEFT | "That is how it works with one of the two range options. Which one is declared here, and which side does a boundary value belong to under it?" |
| "c, partitions 2 and 4 are kept" | Inverts the meaning of the PARTITIONS list | "Does WITH PARTITIONS name what is removed or what is kept?" |
| "d, partitions 2 through 4 are truncated" | Reads the comma as a range | "How do you write a range of partitions in that clause? What keyword?" |
| "There should be four rows, two of them with zero" | Expects empty partitions to be reported | "GROUP BY groups existing rows. Can a partition with no rows produce a group?" |
| "Reading 9 on the first of April is in partition 4" | Forgets the equality rule | "Is the first of April greater than the boundary, or equal to it? Where does equal go under RANGE LEFT?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the two independent facts:

- **RANGE LEFT boundary placement.** For CREATE PARTITION FUNCTION AS RANGE LEFT or RIGHT FOR VALUES with n boundaries, you get n plus one partitions. With RANGE LEFT, a row equal to a boundary goes to the lower partition, column less than or equal to boundary. With RANGE RIGHT, it goes to the higher partition, column greater than or equal to boundary. LEFT is the default when neither is specified. The trap here: the DDL uses first-of-month boundaries, which look like the start of each month's partition, but with RANGE LEFT the first of a month is the last day of the previous partition. First-of-month boundaries behave the intuitive way only with RANGE RIGHT. That is why readings 3, 6 and 9 land one partition lower than intuition suggests, and why option a is wrong.
- **TRUNCATE TABLE WITH PARTITIONS.** Available since SQL Server 2016, it empties only the listed partitions: individual numbers separated by commas, ranges with the keyword TO, such as 6 TO 8, or a mix. The list names what is removed, never what is kept, which is option c's mistake. Two comma four is two individual partitions, not a range, which is option d's mistake. It requires a partitioned table whose indexes are aligned with the table's partition function; here the only index, the clustered primary key, is created on the partition scheme, so the table is aligned.

Then the query:

- `$PARTITION.function_name(value)` returns the one-based partition number for any value, usable in SELECT, WHERE or GROUP BY to audit where rows actually live. Grouping by it reports only nonempty partitions; emptied partitions do not appear as zero. The yyyymmdd date literals and the explicit ORDER BY make the result independent of language, DATEFORMAT and plan choices.

Memory hook: "LEFT: equal goes lower. RIGHT: equal goes higher. First-of-month wants RIGHT. PARTITIONS lists what you empty; commas are a list, TO is a range."

## 9. Follow-up oral questions (optional)

1. "If the function had been declared RANGE RIGHT with the same boundaries, what would the query return after the same truncate?" (Partition 1 with 2 readings and partition 3 with 3 readings, that is option a.)
2. "How would you truncate partitions 2, 3 and 4 in one statement?" (TRUNCATE TABLE Ops.Readings WITH open paren PARTITIONS open paren 2 TO 4 close paren close paren.)
3. "What is the partition number of a reading dated the tenth of May twenty twenty-five?" (4, the last partition, everything after the first of April.)

## 10. References

- CREATE PARTITION FUNCTION, RANGE LEFT and RIGHT: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-partition-function-transact-sql
- CREATE PARTITION SCHEME: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-partition-scheme-transact-sql
- Partitioned tables and indexes: https://learn.microsoft.com/en-us/sql/relational-databases/partitions/partitioned-tables-and-indexes
- $PARTITION: https://learn.microsoft.com/en-us/sql/t-sql/functions/partition-transact-sql
- TRUNCATE TABLE, WITH PARTITIONS clause: https://learn.microsoft.com/en-us/sql/t-sql/statements/truncate-table-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
