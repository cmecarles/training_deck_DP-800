# Instructor-Examiner guide — Triggers 1

Companion to [triggers_1.md](triggers_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Each option is a pair of result sets: the audit table and the books table. Read all four options before taking an answer. The order of the first two lines of the trigger body matters a great deal; read piece 4 slowly and repeat it if asked.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Implement programmability objects.
- Task bullet: Create triggers.
- What is tested: that a DML AFTER trigger fires once per statement and not once per row, that at at ROWCOUNT must be captured in the trigger's first statement, and that UPDATE FROM JOIN touches each target row only once even when several inserted rows match it.

## 2. Scenario to read aloud

**Piece 1, the story.** "The city public library runs its lending system on SQL Server. The script I will describe is the complete history of the database, run top to bottom in a single session, with no errors. There is a trigger that logs each batch of loans and reduces the number of copies available."

**Piece 2, the tables.** "Three tables in a schema called Lending. Books has BookId, integer, primary key; Title, text up to eighty; and CopiesAvailable, an integer. Loans has LoanId, an integer IDENTITY primary key; BookId, a foreign key to Books; MemberId, integer; and LoanDate, a date. AuditTrail has AuditId, an integer IDENTITY primary key; EventType, text up to twenty; and LoanCount, an integer."

**Piece 3, the trigger, header.** "A trigger named Lending dot trg underscore Loans underscore AfterInsert, on Lending dot Loans, AFTER INSERT. Its body has five parts, in this exact order. I will read them one by one."

**Piece 4, the trigger, body.** "Part one, the very first statement: DECLARE at RowsInserted, integer, equals at at ROWCOUNT. Part two: SET NOCOUNT ON. Part three: IF at RowsInserted equals zero, RETURN. Part four: INSERT INTO AuditTrail, columns EventType and LoanCount, VALUES the string LOAN underscore BATCH and at RowsInserted. One VALUES row. Part five: an UPDATE with a FROM clause. UPDATE b SET b dot CopiesAvailable equals b dot CopiesAvailable minus one, FROM Books as b INNER JOIN inserted as i ON i dot BookId equals b dot BookId."

**Piece 5, the books.** "Three books are inserted. Book 101, The Sea Atlas, four copies. Book 202, Clockwork Cities, two copies. Book 303, Paper Lanterns, five copies. There is no trigger on Books."

**Piece 6, the two loan statements.** "Statement S1 is one INSERT statement with three rows into Loans: book 101 for member 11, book 101 again for member 12, and book 202 for member 13, all on August first 2026. Note that book 101 appears twice in the same statement. Statement S2 is one INSERT statement with one row: book 303 for member 14, August second."

**Piece 7, the two queries.** "After that, two queries. The first selects AuditId, EventType and LoanCount from AuditTrail ordered by AuditId. The second selects BookId and CopiesAvailable from Books ordered by BookId."

**Piece 8, option a.** "Option a: the audit table has four rows, each LOAN underscore BATCH with LoanCount one. Books: 101 has two copies, 202 has one, 303 has four."

**Piece 9, option b.** "Option b: the audit table has two rows, LoanCount three and then one. Books: 101 has two, 202 has one, 303 has four."

**Piece 10, option c.** "Option c: the audit table has two rows, LoanCount three and then one. Books: 101 has three, 202 has one, 303 has four."

**Piece 11, option d.** "Option d: the audit table has two rows, both with LoanCount zero. Books: 101 has three, 202 has one, 303 has four."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE CityLibrary;
GO
USE CityLibrary;
GO
CREATE SCHEMA Lending;
GO

CREATE TABLE Lending.Books
(
    BookId          int          NOT NULL PRIMARY KEY,
    Title           nvarchar(80) NOT NULL,
    CopiesAvailable int          NOT NULL
);
GO

CREATE TABLE Lending.Loans
(
    LoanId   int  IDENTITY(1,1) PRIMARY KEY,
    BookId   int  NOT NULL REFERENCES Lending.Books(BookId),
    MemberId int  NOT NULL,
    LoanDate date NOT NULL
);
GO

CREATE TABLE Lending.AuditTrail
(
    AuditId   int          IDENTITY(1,1) PRIMARY KEY,
    EventType nvarchar(20) NOT NULL,
    LoanCount int          NOT NULL
);
GO

CREATE TRIGGER Lending.trg_Loans_AfterInsert
ON Lending.Loans
AFTER INSERT
AS
BEGIN
    DECLARE @RowsInserted int = @@ROWCOUNT;

    SET NOCOUNT ON;

    IF @RowsInserted = 0
        RETURN;

    INSERT INTO Lending.AuditTrail (EventType, LoanCount)
    VALUES (N'LOAN_BATCH', @RowsInserted);

    UPDATE b
       SET b.CopiesAvailable = b.CopiesAvailable - 1
      FROM Lending.Books AS b
     INNER JOIN inserted AS i
             ON i.BookId = b.BookId;
END;
GO

INSERT INTO Lending.Books (BookId, Title, CopiesAvailable) VALUES
    (101, N'The Sea Atlas',     4),
    (202, N'Clockwork Cities',  2),
    (303, N'Paper Lanterns',    5);
GO

-- Statement S1: one INSERT statement, three rows
INSERT INTO Lending.Loans (BookId, MemberId, LoanDate) VALUES
    (101, 11, '2026-08-01'),
    (101, 12, '2026-08-01'),
    (202, 13, '2026-08-01');
GO

-- Statement S2: one INSERT statement, one row
INSERT INTO Lending.Loans (BookId, MemberId, LoanDate) VALUES
    (303, 14, '2026-08-02');
GO

SELECT AuditId, EventType, LoanCount
FROM Lending.AuditTrail
ORDER BY AuditId;

SELECT BookId, CopiesAvailable
FROM Lending.Books
ORDER BY BookId;
```

Options as shown to the learner:

```text
a. Audit: 1/LOAN_BATCH/1, 2/LOAN_BATCH/1, 3/LOAN_BATCH/1, 4/LOAN_BATCH/1   Books: 101/2, 202/1, 303/4
b. Audit: 1/LOAN_BATCH/3, 2/LOAN_BATCH/1                                   Books: 101/2, 202/1, 303/4
c. Audit: 1/LOAN_BATCH/3, 2/LOAN_BATCH/1                                   Books: 101/3, 202/1, 303/4
d. Audit: 1/LOAN_BATCH/0, 2/LOAN_BATCH/0                                   Books: 101/3, 202/1, 303/4
```

## 4. The question (ask exactly this)

"Which option shows exactly the two result sets returned? Option a: four audit rows of LoanCount one, and book 101 at two copies. Option b: two audit rows, three and one, and book 101 at two copies. Option c: two audit rows, three and one, and book 101 at three copies. Option d: two audit rows, both zero, and book 101 at three copies. In every option, book 202 has one copy and book 303 has four. Which one, a, b, c or d?"

If the learner prefers to reason first: "Two questions. How many times does the trigger fire, and what is LoanCount each time? Then, how many copies does each book lose?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: c.**

Trace:

- S1 inserts three rows. The trigger fires once. The DECLARE runs first and captures at at ROWCOUNT equals 3, before SET NOCOUNT ON resets it. The guard does not fire. One audit row is written: AuditId 1, LOAN underscore BATCH, 3. The UPDATE joins Books to inserted, which holds 101, 101, 202. Two target rows qualify, 101 and 202, and each is decremented once. Book 101: 4 minus 1 equals 3, not 2, even though it was borrowed twice. Book 202: 2 minus 1 equals 1. Book 303 untouched at 5.
- S2 inserts one row. The trigger fires once more. At RowsInserted equals 1. Audit row 2, LOAN underscore BATCH, 1. Book 303: 5 minus 1 equals 4.

Final:

| AuditId | EventType | LoanCount |
|---|---|---|
| 1 | LOAN_BATCH | 3 |
| 2 | LOAN_BATCH | 1 |

| BookId | CopiesAvailable |
|---|---|
| 101 | 3 |
| 202 | 1 |
| 303 | 4 |

Why each wrong option is wrong:

- **a** — The row-level trigger myth. SQL Server DML triggers are statement-level only; there is no FOR EACH ROW. Two statements ran, so two firings and two audit rows, with LoanCount 3 and 1. And book 101 loses only one copy.
- **b** — Audit table right, but assumes the UPDATE JOIN inserted decrements once per matching inserted row, giving 101 the value 2. UPDATE FROM updates each target row at most once per statement, however many source rows join to it. Book 101 is 3.
- **d** — Books right, but claims LoanCount zero because SET NOCOUNT ON resets at at ROWCOUNT. SET does reset it, but the DECLARE is the first statement of the body and captures 3 and 1 before the SET runs. Had the two lines been swapped, option d's audit column would be correct.

## 6. Hint ladder (one hint per attempt, in order)

1. "First question: in SQL Server, does a DML trigger fire once per row or once per statement? How many INSERT statements ran against Loans?"
2. "Two firings means two audit rows. That eliminates one option. Now, what value does at at ROWCOUNT hold at the very start of the trigger, and which statement of the body reads it first?"
3. "SET NOCOUNT ON resets at at ROWCOUNT to zero. Does that happen before or after the DECLARE in this trigger? Order matters."
4. "So LoanCount is three, then one. Two options remain, and they differ only in book 101. In S1, inserted holds book 101 twice. Does UPDATE FROM JOIN subtract once per matching inserted row, or once per target row?"
5. "Each target row of Books is updated at most once per UPDATE statement. Book 101 started at four. What is it now?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, four loans, four firings" | Thinks triggers are row-level | "SQL Server has no FOR EACH ROW. What is the unit that fires a DML trigger?" |
| "b, book 101 was borrowed twice so it loses two" | Thinks UPDATE JOIN accumulates per source row | "The join finds book 101 twice. But how many times can one UPDATE statement modify the same target row?" |
| "d, SET NOCOUNT ON wipes at at ROWCOUNT" | Right fact, wrong order | "You are right that SET resets it. Now check which line runs first in the body." |
| "The guard IF equals zero returns, so nothing is logged" | Misreads the guard | "The guard returns only when the count is zero. What is the count after S1?" |
| "The audit INSERT writes three rows in S1" | Thinks the VALUES insert scales with inserted | "The audit INSERT has one VALUES row. How many rows does a one-row VALUES clause write?" |
| "The result is nondeterministic because 101 matches two inserted rows" | Knows the multi-match rule but misses the SET clause | "Which inserted row feeds the SET is undefined, yes. Does the SET clause reference i at all?" |
| "The trigger fires on the Books insert too" | Wrong table | "Which table is the trigger on?" |

## 8. Teaching notes (after the answer is complete or revealed)

Three documented facts decide the question:

- **A DML AFTER trigger fires once per statement, not once per row.** A single INSERT with many rows causes one invocation, and the inserted logical table holds all the new rows at once. There is no FOR EACH ROW in SQL Server, unlike Oracle or PostgreSQL. Two statements ran, so two firings, two audit rows.
- **At the moment the trigger starts, at at ROWCOUNT holds the number of rows affected by the triggering statement.** It must be captured by the very first statement of the body, because almost any later statement, including SET NOCOUNT ON, since SET options reset at at ROWCOUNT to zero, overwrites it. Here the DECLARE is first, so it captures 3 and then 1. Swapping the two lines would give option d. That fragility is why the capture-first pattern is the documented convention.
- **UPDATE FROM JOIN updates each qualifying target row exactly once**, even when it matches several source rows. Book 101 appears twice in inserted but loses one copy. This is the classic single-row-assumption trigger bug that Microsoft's own multirow trigger documentation demonstrates and warns against. When a target row matches multiple source rows, the engine does not define which one feeds the SET clause; here the SET never references i, so the outcome is deterministic anyway.

Then the fix, if the learner asks: aggregate inserted before joining. Something like SET b dot CopiesAvailable equals b dot CopiesAvailable minus i dot Cnt, FROM Books JOIN a subquery that selects BookId and COUNT star as Cnt from inserted GROUP BY BookId.

Then the checklist for any trigger on the exam: how many times does it fire, once per statement, even for a statement that affects zero rows, hence the guard. What is in inserted and deleted, every row the statement touched, as a set; any code that assumes one row, such as SELECT at x equals Col FROM inserted, silently misbehaves on multirow statements. Capture at at ROWCOUNT first. And UPDATE FROM touches each target row once; to accumulate per key, GROUP BY inserted first.

Memory hook: "One statement, one firing, inserted holds them all. Grab ROWCOUNT first. UPDATE JOIN hits each target once."

## 9. Follow-up oral questions (optional)

1. "If the DECLARE and the SET NOCOUNT ON lines were swapped, what would the audit table contain?" (Nothing. The variable would read zero, the guard would return at once, and the trigger would write no audit rows and decrement no counts. Option d's two zero rows would be wrong too.)
2. "How would you rewrite the UPDATE so that book 101 loses two copies in S1?" (Join Books to a subquery over inserted grouped by BookId with COUNT star, and subtract that count.)
3. "Does an AFTER trigger fire for an INSERT that affects zero rows?" (Yes. That is why the guard IF at RowsInserted equals zero RETURN exists.)

## 10. References

- CREATE TRIGGER: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-trigger-transact-sql
- DML triggers overview: https://learn.microsoft.com/en-us/sql/relational-databases/triggers/dml-triggers
- Create DML triggers to handle multiple rows of data: https://learn.microsoft.com/en-us/sql/relational-databases/triggers/create-dml-triggers-to-handle-multiple-rows-of-data
- Use the inserted and deleted tables: https://learn.microsoft.com/en-us/sql/relational-databases/triggers/use-the-inserted-and-deleted-tables
- @@ROWCOUNT: https://learn.microsoft.com/en-us/sql/t-sql/functions/rowcount-transact-sql
- UPDATE, including the FROM clause and multiple-match behaviour: https://learn.microsoft.com/en-us/sql/t-sql/queries/update-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
