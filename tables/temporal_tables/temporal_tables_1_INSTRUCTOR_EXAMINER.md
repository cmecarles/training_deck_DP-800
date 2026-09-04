# Instructor-Examiner guide — Temporal Tables 1

Companion to [temporal_tables_1.md](temporal_tables_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Each option is a pair of result sets, so read all four options before taking an answer. A good way to run it by voice: first let the learner build the version ledger themselves, then let them say how many rows Query A and Query B return, then match to an option. Nothing depends on actual clock values; only the set of row versions matters.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Design and implement database objects.
- Task bullet: Implement system-versioned temporal tables.
- What is tested: which DML statements write history rows, that a no-change UPDATE still versions the row, that two updates in one transaction leave a zero-duration version, and that FOR SYSTEM underscore TIME filters zero-duration rows while the history table itself does not.

## 2. Scenario to read aloud

**Piece 1, the story.** "The revenue team of a seaside hotel tracks nightly room rates in a database called HarborHotel. Rates change during the season, and the team must be able to audit every price a room has ever had. So the rate table is a system-versioned temporal table. The script I will describe is the complete history of the database, run top to bottom in one session. Every batch succeeds."

**Piece 2, the table.** "One table, in a schema called Rooms, named RoomRate. RoomNumber, integer, primary key. RoomType, text up to thirty. NightlyRate, decimal with two decimals. Then two period columns, both datetime2, both HIDDEN, both NOT NULL: ValidFrom, generated always as row start, and ValidTo, generated always as row end. PERIOD FOR SYSTEM underscore TIME uses those two. And the table is created WITH SYSTEM underscore VERSIONING ON, with the history table named Rooms dot RoomRateHistory."

**Piece 3, the insert.** "Season opening. One INSERT of four rows, giving only RoomNumber, RoomType and NightlyRate. Room 101, Standard, one hundred twenty. Room 102, Standard, one hundred twenty. Room 201, Sea View, one hundred eighty. Room 301, Suite, three hundred twenty."

**Piece 4, the rate changes.** "Then four data modifications, each in its own batch. First, UPDATE room 101 to one hundred thirty-five. Second, UPDATE room 102 to one hundred twenty. Yes, the same value it already has. Third, a BEGIN TRANSACTION, then UPDATE room 201 to one hundred ninety-nine, then UPDATE room 201 again to two hundred ten, then COMMIT. Both updates in one transaction. Fourth, DELETE room 301, because the suite is out for renovation."

**Piece 5, the two queries.** "After the script, an auditor runs two queries. Query A selects RoomNumber and NightlyRate from Rooms dot RoomRate FOR SYSTEM underscore TIME ALL, ordered by RoomNumber and NightlyRate. Query B selects RoomNumber and NightlyRate directly from the history table, Rooms dot RoomRateHistory, same ordering."

**Piece 6, option a.** "Option a says Query A returns eight rows: 101 at 120, 101 at 135, 102 at 120 twice, 201 at 180, 201 at 199, 201 at 210, and 301 at 320. Query B returns five rows: 101 at 120, 102 at 120, 201 at 180, 201 at 199, and 301 at 320."

**Piece 7, option b.** "Option b says Query A returns seven rows: 101 at 120, 101 at 135, 102 at 120 twice, 201 at 180, 201 at 210, and 301 at 320. Query B returns four rows: 101 at 120, 102 at 120, 201 at 180, and 301 at 320. So no 199 anywhere."

**Piece 8, option c.** "Option c says Query A returns seven rows, exactly the same seven as option b: 101 at 120, 101 at 135, 102 at 120 twice, 201 at 180, 201 at 210, and 301 at 320. But Query B returns five rows: 101 at 120, 102 at 120, 201 at 180, 201 at 199, and 301 at 320."

**Piece 9, option d.** "Option d says Query A returns six rows: 101 at 120, 101 at 135, 102 at 120 once only, 201 at 180, 201 at 210, and 301 at 320. Query B returns four rows: 101 at 120, 201 at 180, 201 at 199, and 301 at 320. So room 102 is absent from the history."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE HarborHotel;
GO
USE HarborHotel;
GO
CREATE SCHEMA Rooms;
GO
CREATE TABLE Rooms.RoomRate
(
    RoomNumber  int           NOT NULL PRIMARY KEY,
    RoomType    nvarchar(30)  NOT NULL,
    NightlyRate decimal(8,2)  NOT NULL,
    ValidFrom   datetime2 GENERATED ALWAYS AS ROW START HIDDEN NOT NULL,
    ValidTo     datetime2 GENERATED ALWAYS AS ROW END   HIDDEN NOT NULL,
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
)
WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = Rooms.RoomRateHistory));
GO
-- Season opening: initial rate card
INSERT INTO Rooms.RoomRate (RoomNumber, RoomType, NightlyRate) VALUES
  (101, N'Standard', 120.00),
  (102, N'Standard', 120.00),
  (201, N'Sea View', 180.00),
  (301, N'Suite',    320.00);
GO
-- Rate maintenance during the season
UPDATE Rooms.RoomRate SET NightlyRate = 135.00 WHERE RoomNumber = 101;
GO
UPDATE Rooms.RoomRate SET NightlyRate = 120.00 WHERE RoomNumber = 102;
GO
BEGIN TRANSACTION;
UPDATE Rooms.RoomRate SET NightlyRate = 199.00 WHERE RoomNumber = 201;
UPDATE Rooms.RoomRate SET NightlyRate = 210.00 WHERE RoomNumber = 201;
COMMIT;
GO
-- The suite is taken out of inventory for renovation
DELETE Rooms.RoomRate WHERE RoomNumber = 301;
GO

-- Query A
SELECT RoomNumber, NightlyRate
FROM Rooms.RoomRate FOR SYSTEM_TIME ALL
ORDER BY RoomNumber, NightlyRate;

-- Query B
SELECT RoomNumber, NightlyRate
FROM Rooms.RoomRateHistory
ORDER BY RoomNumber, NightlyRate;
```

Options as shown to the learner:

```text
a. Query A (8 rows): 101/120, 101/135, 102/120, 102/120, 201/180, 201/199, 201/210, 301/320
   Query B (5 rows): 101/120, 102/120, 201/180, 201/199, 301/320
b. Query A (7 rows): 101/120, 101/135, 102/120, 102/120, 201/180, 201/210, 301/320
   Query B (4 rows): 101/120, 102/120, 201/180, 301/320
c. Query A (7 rows): 101/120, 101/135, 102/120, 102/120, 201/180, 201/210, 301/320
   Query B (5 rows): 101/120, 102/120, 201/180, 201/199, 301/320
d. Query A (6 rows): 101/120, 101/135, 102/120, 201/180, 201/210, 301/320
   Query B (4 rows): 101/120, 201/180, 201/199, 301/320
```

## 4. The question (ask exactly this)

"What do Query A and Query B return? Option a: eight rows and five rows, with 199 in both. Option b: seven rows and four rows, with 199 in neither. Option c: seven rows and five rows, with 199 only in Query B. Option d: six rows and four rows, with room 102 missing from the history. Which one, a, b, c or d?"

If the learner prefers to reason first: "Take it in two steps. First, how many rows does the history table hold, and which? Then, how many rows does FOR SYSTEM underscore TIME ALL return?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: c.**

Version ledger, using symbolic transaction start times T0 to T4:

| Statement | Current table afterward | Rows written to history |
|---|---|---|
| INSERT four rows, T0 | 101 at 120, 102 at 120, 201 at 180, 301 at 320 | None. An INSERT only opens a row |
| UPDATE 101 to 135, T1 | 101 at 135 | (101, 120.00), period T0 to T1 |
| UPDATE 102 to 120, T2 | 102 at 120, unchanged | (102, 120.00), period T0 to T2. A history row is written even though no value changed |
| UPDATE 201 to 199 then to 210 in one transaction, T3 | 201 at 210 | (201, 180.00), period T0 to T3, and (201, 199.00), period T3 to T3, a zero-duration row, because both updates carry the same transaction start time |
| DELETE 301, T4 | Row removed | (301, 320.00), period T0 to T4 |

Query B, the history table queried directly, returns every stored version: (101, 120), (102, 120), (201, 180), (201, 199), (301, 320). Five rows.

Query A, FOR SYSTEM underscore TIME ALL, returns current plus history minus zero-duration rows: current (101, 135), (102, 120), (201, 210), plus the five history rows minus (201, 199). Ordered: (101, 120), (101, 135), (102, 120), (102, 120), (201, 180), (201, 210), (301, 320). Seven rows. Room 102 appears twice with the same rate: once current, once from the no-op update.

Why each wrong option is wrong:

- **a** — Treats FOR SYSTEM underscore TIME ALL as a plain UNION ALL of current and history, giving eight rows including (201, 199). Every FOR SYSTEM underscore TIME sub-clause, including ALL, discards versions whose ValidFrom equals ValidTo. Its Query B is correct, which makes it the subtle distractor.
- **b** — Assumes the engine collapses multiple updates in one transaction and keeps only the pre-transaction version. It does not: each UPDATE moves the version it replaces into history; the intermediate (201, 199) is stored with a zero-duration period. Query B returns five rows, not four.
- **d** — Assumes updating room 102 to the value it already has writes nothing. System-versioning does not compare old and new values; any UPDATE versions the row. So (102, 120) is missing from its Query B and the duplicate pair is missing from its Query A.

## 6. Hint ladder (one hint per attempt, in order)

1. "Forget the clock. Build a ledger: for each of the five data modifications, what goes into the history table? Start with the INSERT. Does an INSERT write any history?"
2. "Now the update of room 102 to one hundred twenty, the value it already has. Does system-versioning compare old and new values before writing history, or does it version every UPDATE?"
3. "Now the two updates of room 201 inside one transaction. Each UPDATE moves the version it replaces into history. How many history rows for room 201? And what timestamps do both updates get, given that the engine uses the transaction start time?"
4. "You should now have five history rows, one of them with ValidFrom equal to ValidTo. Query B reads the history table directly, so it sees all of them. That eliminates the options with four rows in Query B."
5. "Two options left. They differ in whether Query A shows 201 at 199. FOR SYSTEM underscore TIME has a documented filter about rows with a period of zero duration. What does it do with them?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, because ALL means current plus history" | Unaware of the zero-duration filter | "Almost. Read the definition of ALL once more. Is there any kind of row that every FOR SYSTEM underscore TIME clause drops?" |
| "b, the transaction only records the before and after" | Thinks versioning happens per transaction | "Versioning happens per statement. How many UPDATE statements touched room 201?" |
| "d, updating to the same value is a no-op" | Thinks the engine compares values | "The documentation says a history row is added on any data modification, even if no column values change. Rethink room 102." |
| "The INSERT writes four rows to history" | Thinks history records creation | "An INSERT opens a row in the current table. There is no previous version to move. What does the history table hold after the INSERT?" |
| "The DELETE removes the row from history too" | Confuses current and history | "The DELETE closes the current version and moves it to history. Where is room 301 afterwards?" |
| "Query A cannot show 102 at 120 twice" | Expects deduplication | "One is the current row, one is the history version from the no-op update. Does ALL remove duplicates?" |
| "The answer depends on the actual timestamps" | Misses that only the set of versions matters | "The question never asks when. Only which versions exist. Can you decide that without a clock?" |

## 8. Teaching notes (after the answer is complete or revealed)

Give the three DML rules first:

- **INSERT** touches the current table only. The new row gets ValidFrom equal to the transaction start time and ValidTo equal to 9999-12-31. History is untouched.
- **UPDATE** moves the old version into history, closing it with ValidTo equal to the transaction start time, and writes the new version in the current table. This happens even if no column value changes. That is room 102.
- **DELETE** removes the row from the current table and moves the closed version into history. That is room 301.

Then the timestamp rule: the period columns are stamped with the **start time of the transaction, in UTC**. So N updates to the same row within one transaction leave N minus 1 zero-duration versions in history, with ValidFrom equal to ValidTo. That is room 201 at 199.

Then the query-side rule that decides the question: **FOR SYSTEM underscore TIME, with any sub-clause including ALL, filters out rows with a zero-duration period.** The documentation says the engine generates those rows when you perform multiple updates on the same primary key within the same transaction, and every temporal clause hides them. A plain SELECT from the history table is just a SELECT from a table and returns every stored version. So Query A is seven rows, Query B is five.

Then the side notes:

- HIDDEN period columns disappear from SELECT star and can be omitted from INSERT column lists, but they are still there. Name them explicitly to see them. HIDDEN has no effect on which rows the two queries return.
- Never trust a temporal question whose answer depends on actual datetime2 values. They are system-generated and nondeterministic. Only the set of versions is predictable, and that is what this question asks.

Memory hook: "Every UPDATE versions, even a no-op. Same transaction, same timestamp, zero-duration row. FOR SYSTEM underscore TIME hides zero-duration; the history table shows everything."

## 9. Follow-up oral questions (optional)

1. "If the two updates of room 201 had been in separate batches without an explicit transaction, how many rows would Query A show for room 201?" (Three: 180, 199 and 210, because 199 would then have a real, non-zero period.)
2. "What does SELECT star from Rooms dot RoomRate return for columns?" (Only RoomNumber, RoomType and NightlyRate, because the period columns are HIDDEN.)
3. "Which time zone are the period columns stamped in?" (UTC, from the transaction start time.)

## 10. References

- Temporal tables overview: https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables
- Querying data in a system-versioned temporal table, including the zero-duration filter: https://learn.microsoft.com/en-us/sql/relational-databases/tables/querying-data-in-a-system-versioned-temporal-table
- Modifying data in a system-versioned temporal table: https://learn.microsoft.com/en-us/sql/relational-databases/tables/modifying-data-in-a-system-versioned-temporal-table
- Creating a system-versioned temporal table: https://learn.microsoft.com/en-us/sql/relational-databases/tables/creating-a-system-versioned-temporal-table
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
