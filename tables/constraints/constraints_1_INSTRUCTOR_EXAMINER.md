# Instructor-Examiner guide — Constraints 1

Companion to [constraints_1.md](constraints_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a long trace question with fourteen statements. Keep a running note of the table contents as the learner goes, because a failed statement changes nothing and later statements depend on earlier ones. Accept "error 547" or "a CHECK violation" as a correct outcome; the exact message text is a bonus, not a requirement.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Design and implement database objects.
- Task bullet: Implement constraints (primary key, unique, foreign key, check, default) and referential actions.
- What is tested: which constraint blocks each statement, how NULL behaves in UNIQUE and CHECK, what ON DELETE actions do, and what WITH NOCHECK means for trust.

## 2. Scenario to read aloud

**Piece 1, the story.** "A dog-boarding kennel keeps its bookings in a SQL Server database called KennelBook. Owners register their dogs, dogs are booked for stays, and owners receive invoices. The whole schema relies on declarative constraints: keys, unique constraints, check constraints, defaults and foreign keys. No triggers, no procedures."

**Piece 2, the Owner table.** "There is a schema called Board. The first table is Board dot Owner. It has three columns. OwnerID, an integer, the primary key, named PK underscore Owner. FullName, text up to sixty characters, not null. And Email, a text column up to eighty characters, which allows NULL, and has a UNIQUE constraint called UQ underscore Owner underscore Email."

**Piece 3, the Dog table.** "The second table is Board dot Dog. DogID, an integer, primary key. OwnerID, an integer that allows NULL. Name, text up to forty characters, not null. WeightKg, a decimal that allows NULL, with a CHECK constraint called CK underscore Dog underscore Weight that requires the weight to be greater than zero and less than one hundred. Vaccinated, a bit, not null, with a DEFAULT of zero. And a foreign key, FK underscore Dog underscore Owner, from OwnerID to Owner, declared ON DELETE SET NULL. On top of that, there is a separate unique nonclustered index on the Name column, called UX underscore Dog underscore Name. Note that it is an index, not a constraint."

**Piece 4, the Stay and Invoice tables.** "The third table is Board dot Stay. StayID, integer, primary key. DogID, integer, not null. CheckIn, a date, not null. CheckOut, a date, that allows NULL. A foreign key FK underscore Stay underscore Dog from DogID to Dog, declared ON DELETE CASCADE. And a CHECK constraint CK underscore Stay underscore Dates that requires CheckOut greater than or equal to CheckIn. The fourth table is Board dot Invoice. InvoiceID, primary key. OwnerID, integer, not null, with a foreign key FK underscore Invoice underscore Owner to Owner, with no ON DELETE action specified. And Total, a decimal, not null."

**Piece 5, the data.** "Three owners. Owner 1, Ana Ruiz, email ana at example dot com. Owner 2, Ben Cole, email NULL. Owner 3, Cy Dorn, email cy at example dot com. Three dogs. Dog 10, owner 1, named Rex, thirty-two point five kilos, vaccinated. Dog 11, owner 2, named Mia, weight NULL, vaccinated. Dog 12, owner 3, named Tor, eight kilos, not vaccinated. Three stays. Stay 100, dog 10, check-in first of September 2026, check-out fifth of September. Stay 101, dog 11, check-in second of September, check-out NULL. Stay 102, dog 12, check-in third of September, check-out fourth of September. And one invoice, number 500, for owner 1, total one hundred twenty."

**Piece 6, statements S1 to S7.** "Fourteen statements run in order, each in its own batch, in one session. Here are the first seven.

- S1 inserts owner 4, Di Eng, with email NULL.
- S2 inserts dog 13, owner 3, named Rex, twelve kilos.
- S3 inserts dog 13, owner 3, named Zed, with no weight and no Vaccinated value given.
- S4 inserts dog 14, owner 3, named Bruno, one hundred fifty kilos.
- S5 inserts dog 15, owner 3, weight ten kilos, and gives no Name at all.
- S6 inserts dog 16, owner 99, named Ghost.
- S7 deletes from Owner where OwnerID equals 1."

**Piece 7, statements S8 to S14.** "And the last seven.

- S8 deletes from Owner where OwnerID equals 2.
- S9 deletes from Dog where DogID equals 12.
- S10 alters table Dog, WITH NOCHECK, and adds a CHECK constraint called CK underscore Dog underscore Name that requires the length of Name to be at least four characters.
- S11 inserts dog 17, owner 3, named Bo.
- S12 alters table Dog, WITH CHECK, CHECK CONSTRAINT CK underscore Dog underscore Name. That is, it asks the engine to re-validate that constraint.
- S13 updates Stay, sets CheckIn to the tenth of September 2026, where StayID is 101.
- S14 updates Stay, sets CheckOut to the thirtieth of August 2026, where StayID is 100."

**Piece 8, the final query.** "At the end there is a query against sys dot check underscore constraints. It returns the name and the column is underscore not underscore trusted for every check constraint on Board dot Dog, ordered by name."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE KennelBook;
GO
USE KennelBook;
GO
CREATE SCHEMA Board;
GO
CREATE TABLE Board.Owner
(
    OwnerID  INT          NOT NULL CONSTRAINT PK_Owner PRIMARY KEY,
    FullName NVARCHAR(60) NOT NULL,
    Email    VARCHAR(80)  NULL     CONSTRAINT UQ_Owner_Email UNIQUE
);
CREATE TABLE Board.Dog
(
    DogID      INT          NOT NULL CONSTRAINT PK_Dog PRIMARY KEY,
    OwnerID    INT          NULL,
    Name       NVARCHAR(40) NOT NULL,
    WeightKg   DECIMAL(5,1) NULL     CONSTRAINT CK_Dog_Weight CHECK (WeightKg > 0 AND WeightKg < 100),
    Vaccinated BIT          NOT NULL CONSTRAINT DF_Dog_Vaccinated DEFAULT (0),
    CONSTRAINT FK_Dog_Owner FOREIGN KEY (OwnerID) REFERENCES Board.Owner (OwnerID) ON DELETE SET NULL
);
CREATE UNIQUE NONCLUSTERED INDEX UX_Dog_Name ON Board.Dog (Name);
CREATE TABLE Board.Stay
(
    StayID   INT  NOT NULL CONSTRAINT PK_Stay PRIMARY KEY,
    DogID    INT  NOT NULL,
    CheckIn  DATE NOT NULL,
    CheckOut DATE NULL,
    CONSTRAINT FK_Stay_Dog FOREIGN KEY (DogID) REFERENCES Board.Dog (DogID) ON DELETE CASCADE,
    CONSTRAINT CK_Stay_Dates CHECK (CheckOut >= CheckIn)
);
CREATE TABLE Board.Invoice
(
    InvoiceID INT          NOT NULL CONSTRAINT PK_Invoice PRIMARY KEY,
    OwnerID   INT          NOT NULL CONSTRAINT FK_Invoice_Owner FOREIGN KEY REFERENCES Board.Owner (OwnerID),
    Total     DECIMAL(8,2) NOT NULL
);
GO
INSERT INTO Board.Owner (OwnerID, FullName, Email) VALUES
  (1, N'Ana Ruiz', 'ana@example.com'),
  (2, N'Ben Cole', NULL),
  (3, N'Cy Dorn',  'cy@example.com');
INSERT INTO Board.Dog (DogID, OwnerID, Name, WeightKg, Vaccinated) VALUES
  (10, 1, N'Rex', 32.5, 1),
  (11, 2, N'Mia', NULL, 1),
  (12, 3, N'Tor',  8.0, 0);
INSERT INTO Board.Stay (StayID, DogID, CheckIn, CheckOut) VALUES
  (100, 10, '2026-09-01', '2026-09-05'),
  (101, 11, '2026-09-02', NULL),
  (102, 12, '2026-09-03', '2026-09-04');
INSERT INTO Board.Invoice (InvoiceID, OwnerID, Total) VALUES (500, 1, 120.00);
GO
-- S1
INSERT INTO Board.Owner (OwnerID, FullName, Email) VALUES (4, N'Di Eng', NULL);
-- S2
INSERT INTO Board.Dog (DogID, OwnerID, Name, WeightKg) VALUES (13, 3, N'Rex', 12.0);
-- S3
INSERT INTO Board.Dog (DogID, OwnerID, Name) VALUES (13, 3, N'Zed');
-- S4
INSERT INTO Board.Dog (DogID, OwnerID, Name, WeightKg) VALUES (14, 3, N'Bruno', 150.0);
-- S5
INSERT INTO Board.Dog (DogID, OwnerID, WeightKg) VALUES (15, 3, 10.0);
-- S6
INSERT INTO Board.Dog (DogID, OwnerID, Name) VALUES (16, 99, N'Ghost');
-- S7
DELETE FROM Board.Owner WHERE OwnerID = 1;
-- S8
DELETE FROM Board.Owner WHERE OwnerID = 2;
-- S9
DELETE FROM Board.Dog WHERE DogID = 12;
-- S10
ALTER TABLE Board.Dog WITH NOCHECK ADD CONSTRAINT CK_Dog_Name CHECK (LEN(Name) >= 4);
-- S11
INSERT INTO Board.Dog (DogID, OwnerID, Name) VALUES (17, 3, N'Bo');
-- S12
ALTER TABLE Board.Dog WITH CHECK CHECK CONSTRAINT CK_Dog_Name;
-- S13
UPDATE Board.Stay SET CheckIn = '2026-09-10' WHERE StayID = 101;
-- S14
UPDATE Board.Stay SET CheckOut = '2026-08-30' WHERE StayID = 100;
-- Final query
SELECT name, is_not_trusted FROM sys.check_constraints
WHERE parent_object_id = OBJECT_ID('Board.Dog') ORDER BY name;
```

## 4. The question (ask exactly this)

"For each of the fourteen statements, S1 to S14, tell me whether it succeeds or raises an error. For the ones that succeed, tell me how many rows the statement reports. Let's go one at a time, starting with S1."

After all fourteen: "Now give me the final contents of Owner, Dog and Stay, ordered by key. And finally, what does the query against sys dot check underscore constraints return for Board dot Dog: each constraint name and its is underscore not underscore trusted value?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

| Stmt | Outcome | Detail |
|---|---|---|
| S1 | Fails, error 2627 | UNIQUE constraint UQ_Owner_Email; owner 2 already holds a NULL email, and SQL Server allows only one NULL in a UNIQUE constraint |
| S2 | Fails, error 2601 | Duplicate key in unique index UX_Dog_Name, value Rex. 2601 because it is an index, not a constraint |
| S3 | Succeeds, 1 row | Dog 13 Zed; WeightKg NULL passes the CHECK (UNKNOWN is not FALSE); Vaccinated filled by default 0 |
| S4 | Fails, error 547 | CHECK constraint CK_Dog_Weight, 150 is not less than 100 |
| S5 | Fails, error 515 | Name is NOT NULL with no default, and it was omitted |
| S6 | Fails, error 547 | FOREIGN KEY FK_Dog_Owner, owner 99 does not exist; message names table Board.Owner |
| S7 | Fails, error 547 | REFERENCE constraint FK_Invoice_Owner, owner 1 has invoice 500 and the FK has the default NO ACTION |
| S8 | Succeeds, 1 row | Owner 2 deleted; FK_Dog_Owner sets Mia's OwnerID to NULL; not counted in the message |
| S9 | Succeeds, 1 row | Dog 12 deleted; ON DELETE CASCADE removes stay 102; not counted in the message |
| S10 | Succeeds | Constraint CK_Dog_Name added without checking existing rows; is_not_trusted = 1 |
| S11 | Fails, error 547 | CHECK constraint CK_Dog_Name; Bo is two characters; NOCHECK only skips existing rows |
| S12 | Fails, error 547 | ALTER TABLE conflicts with CK_Dog_Name; Rex, Mia and Zed still have three letters; constraint stays untrusted |
| S13 | Succeeds, 1 row | Stay 101 has CheckOut NULL; NULL >= date is UNKNOWN, so the CHECK passes |
| S14 | Fails, error 547 | CHECK constraint CK_Stay_Dates; thirtieth of August is before first of September; message names no column |

Final Owner table:

| OwnerID | FullName | Email |
|---|---|---|
| 1 | Ana Ruiz | ana@example.com |
| 3 | Cy Dorn | cy@example.com |

Final Dog table:

| DogID | OwnerID | Name | WeightKg | Vaccinated |
|---|---|---|---|---|
| 10 | 1 | Rex | 32.5 | 1 |
| 11 | NULL | Mia | NULL | 1 |
| 13 | 3 | Zed | NULL | 0 |

Final Stay table:

| StayID | DogID | CheckIn | CheckOut |
|---|---|---|---|
| 100 | 10 | 2026-09-01 | 2026-09-05 |
| 101 | 11 | 2026-09-10 | NULL |

Final query:

| name | is_not_trusted |
|---|---|
| CK_Dog_Name | 1 |
| CK_Dog_Weight | 0 |

## 6. Hint ladder (one hint per attempt, in order)

**S1**
1. "The new owner has a NULL email. Is there another owner with a NULL email already?"
2. "In SQL Server, how many NULLs does a UNIQUE constraint accept? Think of NULL as a value here."
3. "It accepts exactly one. A second NULL is a duplicate key."

**S2**
1. "Look at the dog's name. Is there a dog with that name already?"
2. "What protects the Name column: a constraint, or a unique index? The error number differs."
3. "A unique index reports error 2601. A PRIMARY KEY or UNIQUE constraint reports 2627."

**S3**
1. "Two columns are omitted: WeightKg and Vaccinated. Check each: does it allow NULL, or does it have a default?"
2. "WeightKg becomes NULL. Evaluate the CHECK: NULL greater than zero. Is that TRUE, FALSE, or UNKNOWN?"
3. "A CHECK only rejects FALSE. And DogID 13 is free, because S2 failed."

**S4**
1. "Compare the weight with the CHECK constraint on WeightKg."
2. "One hundred fifty is not below one hundred. Which error number is a CHECK violation?"

**S5**
1. "Which columns did the insert leave out? Look at Name."
2. "Name is NOT NULL. Does it have a DEFAULT to fall back on?"
3. "No default, so the engine must put NULL into a NOT NULL column. That is error 515."

**S6**
1. "Does owner 99 exist?"
2. "Which foreign key checks that, and what error number does a foreign key violation raise?"

**S7**
1. "Owner 1 is referenced by more than one table. List them."
2. "FK underscore Dog underscore Owner would set Rex's owner to NULL. But what about FK underscore Invoice underscore Owner? What is its delete action?"
3. "The default action is NO ACTION. One blocking reference is enough to refuse the whole delete."

**S8**
1. "Who references owner 2? Check Dog and Invoice."
2. "Only Mia references owner 2, and the Dog foreign key is ON DELETE SET NULL. What happens to Mia's OwnerID?"
3. "Does the rows-affected message count Mia, or only the deleted owner?"

**S9**
1. "Dog 12 has a stay. What is the delete action on FK underscore Stay underscore Dog?"
2. "CASCADE deletes stay 102 as well. How many rows does the DELETE statement report?"

**S10**
1. "Three existing dogs have three-letter names. Does WITH NOCHECK look at existing rows?"
2. "It does not. So the statement succeeds. But what does that do to the constraint's trust flag?"

**S11**
1. "Bo has two letters. Is the new constraint enforced for new rows, even though it was added WITH NOCHECK?"
2. "NOCHECK only skips existing rows at creation time. New inserts are checked."

**S12**
1. "WITH CHECK CHECK CONSTRAINT re-validates every existing row. Are Rex, Mia and Zed still there?"
2. "They still have three letters. What happens to an ALTER TABLE that finds violating rows?"

**S13**
1. "Stay 101 has CheckOut NULL. Evaluate NULL greater than or equal to the tenth of September."
2. "That is UNKNOWN, not FALSE. Does a CHECK reject UNKNOWN?"

**S14**
1. "Stay 100 checks in on the first of September. Is the thirtieth of August greater than or equal to that?"
2. "That is a definite FALSE. Which constraint fires, and with which error number?"

**Final tables**
1. "Only three statements changed data: S3, S8 and S9, plus S13 changed a date. Start from the original rows and apply just those."
2. "Remember the side effects: S8 nulled Mia's owner, S9 cascaded to stay 102."
3. "For the trust query: which constraint was added WITH NOCHECK and never successfully re-validated?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "S1 succeeds, NULLs are never equal" | Applies the ANSI rule; SQL Server treats NULL as a value for uniqueness | "That is the ANSI standard. Does SQL Server follow it for UNIQUE constraints?" |
| "S2 fails with 2627" | Does not distinguish unique index from unique constraint | "Look at how UX underscore Dog underscore Name was created. Constraint, or index?" |
| "S3 fails, weight NULL violates the CHECK" | Thinks a CHECK rejects UNKNOWN | "Which truth values does a CHECK reject? Only one of the three." |
| "S3 fails, Vaccinated is NOT NULL" | Forgets the DEFAULT | "Read the Vaccinated column definition once more." |
| "S7 succeeds and Rex's owner becomes NULL" | Looks at only one of the two foreign keys | "How many tables reference Owner? Check every one of them." |
| "S8 reports 2 rows affected" | Counts the SET NULL side effect | "Does the message count rows changed by a referential action?" |
| "S9 reports 2 rows affected" | Counts the cascade | "Same idea as S8. The cascade is silent." |
| "S10 fails, existing names are too short" | Does not know what WITH NOCHECK does | "What does WITH NOCHECK skip when the constraint is added?" |
| "S11 succeeds because the constraint is untrusted" | Confuses untrusted with disabled | "Untrusted is not the same as disabled. Is the constraint enabled for new rows?" |
| "S12 succeeds and makes the constraint trusted" | Forgets the violating rows are still present | "What does WITH CHECK do with the existing rows? Are any of them still short?" |
| "S13 fails" | Treats NULL comparison as FALSE | "What is NULL compared to a date? And does a CHECK reject that?" |
| Final Dog table still shows Mia with owner 2 | Forgot the SET NULL from S8 | "What did S8 do to the dogs of owner 2?" |
| Final Stay table still shows stay 102 | Forgot the cascade from S9 | "Where did stay 102 go when dog 12 was deleted?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the four error numbers first:

- **515** is NULL into a NOT NULL column that has no DEFAULT. That is S5.
- **2627** is a duplicate in a PRIMARY KEY or UNIQUE constraint. That is S1.
- **2601** is a duplicate in a unique index created with CREATE UNIQUE INDEX. That is S2. Same protection, different metadata: a constraint appears in sys dot key underscore constraints, an index only in sys dot indexes.
- **547** is a CHECK or FOREIGN KEY violation, on INSERT, UPDATE, DELETE or ALTER TABLE WITH CHECK. That is S4, S6, S7, S11, S12 and S14.

Then the rules:

- **UNIQUE allows exactly one NULL.** SQL Server treats NULL as a value for uniqueness, unlike the ANSI standard. To allow many NULL emails, use a filtered unique index with WHERE Email IS NOT NULL.
- **CHECK rejects only FALSE.** TRUE and UNKNOWN both pass. That is why S3 (NULL weight) and S13 (NULL check-out) succeed, and why S4 and S14 fail. The message for a multi-column check such as CK underscore Stay underscore Dates does not name a column.
- **DEFAULT** fills an omitted column. That is why Vaccinated is zero for Zed.
- **Foreign keys on delete.** NO ACTION, the default, refuses the parent delete. CASCADE deletes the children. SET NULL rewrites the child column, which must be nullable. One blocking reference is enough to stop the whole delete, even if another FK on the same parent would have been fine. That is S7. The rows-affected message counts only the rows the statement targeted, never rows changed by a cascade or a SET NULL. That is S8 and S9.
- **WITH NOCHECK** adds the constraint without validating existing rows, and marks it is underscore not underscore trusted equals 1. It is still enabled for new rows, so S11 fails. The optimizer ignores untrusted constraints. To make it trusted, run ALTER TABLE WITH CHECK CHECK CONSTRAINT, which re-validates every row and fails with 547 if bad rows remain. That is S12, and it is why CK underscore Dog underscore Name stays untrusted at the end.

Memory hook: "One NULL per unique. Check rejects only FALSE. Cascades are silent. NOCHECK means untrusted, not disabled."

## 9. Follow-up oral questions (optional)

1. "How would you let many owners have a NULL email and still keep non-NULL emails unique?" (A filtered unique index: CREATE UNIQUE INDEX on Email WHERE Email IS NOT NULL.)
2. "What single change would make S7 succeed, and what would happen to invoice 500?" (Declare FK underscore Invoice underscore Owner ON DELETE CASCADE; invoice 500 is deleted and Rex's owner becomes NULL.)
3. "After S12 failed, what must you do before WITH CHECK CHECK CONSTRAINT can succeed?" (Fix or remove the rows with names shorter than four characters: Rex, Mia and Zed.)

## 10. References

- Unique constraints and check constraints: https://learn.microsoft.com/en-us/sql/relational-databases/tables/unique-constraints-and-check-constraints
- Primary and foreign key constraints, including cascading referential integrity: https://learn.microsoft.com/en-us/sql/relational-databases/tables/primary-and-foreign-key-constraints
- ALTER TABLE, WITH CHECK and WITH NOCHECK: https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-table-transact-sql
- Disable check constraints with INSERT and UPDATE: https://learn.microsoft.com/en-us/sql/relational-databases/tables/disable-check-constraints-with-insert-and-update-statements
- sys.check_constraints: https://learn.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-check-constraints-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
