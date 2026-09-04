# Instructor-Examiner guide — Error Handling 1

Companion to [error_handling_1.md](error_handling_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a five-part prediction question, batch 1 to batch 5, taken strictly in order. For batches 1 to 4 ask for every PRINT line and every message, in order, and which rows are left in the table after that batch. Batch 5 is the final result set. The learner may have run the script already; if so, ask what they observed and still make them explain each line.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Write advanced T-SQL code.
- Task bullet: Implement error handling with TRY CATCH, THROW and RAISERROR, and transaction control.
- What is tested: how XACT_ABORT changes what an error does to a transaction, the three values of XACT_STATE, why severity 10 is not caught, THROW versus RAISERROR, and autocommit behaviour.

## 2. Scenario to read aloud

**Piece 1, the story.** "A parcel shipping company validates shipping labels in a database called ParcelFlow before handing them to the carrier. The validation pipeline runs as a sequence of T-SQL batches. Some of them are expected to hit errors, bad weights and duplicate label IDs, and the team wants to know exactly what each error-handling construct does. The script runs top to bottom, in one session, in Management Studio, with NOCOUNT on."

**Piece 2, the table.** "There is a schema called Ship and one table, Ship dot Labels, with three columns. LabelID, an integer, not null, primary key, constraint PK underscore Labels. Barcode, a char of ten, not null. And WeightKg, a decimal six comma two, not null, with a check constraint named CK underscore Labels underscore WeightKg that requires WeightKg greater than zero. The table starts empty."

**Piece 3, batch 1.** "Batch 1 starts with SET XACT underscore ABORT OFF. Then BEGIN TRY. Inside: BEGIN TRANSACTION. Insert label 1, barcode PKG one, weight 2.50. Insert label 2, barcode PKG two, weight minus 1.00. Insert label 3, barcode PKG three, weight 5.00. COMMIT TRANSACTION. END TRY. Then BEGIN CATCH. If XACT underscore STATE equals 1, print Batch 1 colon committable, committing, and COMMIT TRANSACTION. If XACT underscore STATE equals minus 1, print Batch 1 colon doomed, rolling back, and ROLLBACK TRANSACTION. END CATCH. GO."

**Piece 4, batch 2.** "Batch 2 is the same shape, but it starts with SET XACT underscore ABORT ON. BEGIN TRY, BEGIN TRANSACTION. Insert label 4, PKG four, weight 1.25. Insert label 5, PKG five, weight minus 2.00. Insert label 6, PKG six, weight 3.75. COMMIT. END TRY. The CATCH has three IFs. If XACT underscore STATE equals 1, print Batch 2 colon committable, committing, and commit. If it equals minus 1, print Batch 2 colon doomed, rolling back, and roll back. And if it equals 0, print Batch 2 colon no open transaction remains. Note these are three separate IFs, not an IF ELSE chain. GO."

**Piece 5, batch 3.** "Batch 3 starts with SET XACT underscore ABORT OFF. BEGIN TRY. RAISERROR with the message Routine label audit notice, severity 10, state 1. Then PRINT Batch 3 colon after RAISERROR, ending with a semicolon. Then THROW 50001, message Label audit failed, state 1. Then PRINT Batch 3 colon after THROW. END TRY. The CATCH prints the concatenation of Batch 3 colon caught error, then ERROR underscore NUMBER, then a comma, the word severity, and ERROR underscore SEVERITY. GO."

**Piece 6, batch 4.** "Batch 4 has no BEGIN TRANSACTION. BEGIN TRY. Insert label 7, barcode PKG seven, weight 4.10. Insert label 7 again, barcode PKG eight, weight 6.00. END TRY. The CATCH prints Batch 4 colon caught error and ERROR underscore NUMBER, then runs THROW with no arguments. After END CATCH, still in the same batch, PRINT Batch 4 colon completed. GO."

**Piece 7, batch 5.** "Batch 5 is SELECT LabelID, Barcode and WeightKg from Ship dot Labels, ordered by LabelID."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE ParcelFlow;
GO
USE ParcelFlow;
GO
SET NOCOUNT ON;
GO
CREATE SCHEMA Ship;
GO
CREATE TABLE Ship.Labels
(
    LabelID  int          NOT NULL CONSTRAINT PK_Labels PRIMARY KEY,
    Barcode  char(10)     NOT NULL,
    WeightKg decimal(6,2) NOT NULL CONSTRAINT CK_Labels_WeightKg CHECK (WeightKg > 0)
);
GO
-- Batch 1
SET XACT_ABORT OFF;
BEGIN TRY
    BEGIN TRANSACTION;
    INSERT INTO Ship.Labels VALUES (1, 'PKG0000001', 2.50);
    INSERT INTO Ship.Labels VALUES (2, 'PKG0000002', -1.00);
    INSERT INTO Ship.Labels VALUES (3, 'PKG0000003', 5.00);
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF XACT_STATE() = 1
    BEGIN
        PRINT 'Batch 1: committable, committing';
        COMMIT TRANSACTION;
    END
    IF XACT_STATE() = -1
    BEGIN
        PRINT 'Batch 1: doomed, rolling back';
        ROLLBACK TRANSACTION;
    END
END CATCH;
GO
-- Batch 2
SET XACT_ABORT ON;
BEGIN TRY
    BEGIN TRANSACTION;
    INSERT INTO Ship.Labels VALUES (4, 'PKG0000004', 1.25);
    INSERT INTO Ship.Labels VALUES (5, 'PKG0000005', -2.00);
    INSERT INTO Ship.Labels VALUES (6, 'PKG0000006', 3.75);
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF XACT_STATE() = 1
    BEGIN
        PRINT 'Batch 2: committable, committing';
        COMMIT TRANSACTION;
    END
    IF XACT_STATE() = -1
    BEGIN
        PRINT 'Batch 2: doomed, rolling back';
        ROLLBACK TRANSACTION;
    END
    IF XACT_STATE() = 0
        PRINT 'Batch 2: no open transaction remains';
END CATCH;
GO
-- Batch 3
SET XACT_ABORT OFF;
BEGIN TRY
    RAISERROR('Routine label audit notice.', 10, 1);
    PRINT 'Batch 3: after RAISERROR';
    THROW 50001, 'Label audit failed.', 1;
    PRINT 'Batch 3: after THROW';
END TRY
BEGIN CATCH
    PRINT CONCAT('Batch 3: caught error ', ERROR_NUMBER(), ', severity ', ERROR_SEVERITY());
END CATCH;
GO
-- Batch 4
BEGIN TRY
    INSERT INTO Ship.Labels VALUES (7, 'PKG0000007', 4.10);
    INSERT INTO Ship.Labels VALUES (7, 'PKG0000008', 6.00);
END TRY
BEGIN CATCH
    PRINT CONCAT('Batch 4: caught error ', ERROR_NUMBER());
    THROW;
END CATCH;
PRINT 'Batch 4: completed';
GO
-- Batch 5
SELECT LabelID, Barcode, WeightKg
FROM Ship.Labels
ORDER BY LabelID;
GO
```

## 4. The question (ask exactly this)

"Predict the complete output of batches 1 through 5: every PRINT line, every message, and the exact result set of the final SELECT, including which rows physically survive in Ship dot Labels. One batch at a time. Batch 1: what is printed, and which rows are in the table afterwards?"

Then batch 2, batch 3, batch 4 in turn, and finally: "Batch 5: what rows does the SELECT return?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

Messages, in this exact order:

```text
Batch 1: committable, committing
Batch 2: doomed, rolling back
Batch 2: no open transaction remains
Routine label audit notice.
Batch 3: after RAISERROR
Batch 3: caught error 50001, severity 16
Batch 4: caught error 2627
Msg 2627, Level 14, State 1, Line 4
Violation of PRIMARY KEY constraint 'PK_Labels'. Cannot insert duplicate key in object 'Ship.Labels'. The duplicate key value is (7).
```

"Batch 3: after THROW" and "Batch 4: completed" are never printed.

Per batch:

- **Batch 1.** Label 2 violates the check constraint, error 547. With XACT_ABORT OFF only the statement dies; control jumps to CATCH, so label 3 and the COMMIT in the TRY never run. XACT_STATE is 1; the CATCH prints "committable, committing" and commits. Label 1 alone survives.
- **Batch 2.** Same error, but XACT_ABORT ON dooms the transaction. XACT_STATE is minus 1; the CATCH prints "doomed, rolling back" and rolls back label 4 too. After the rollback XACT_STATE is 0, so the third IF also prints "no open transaction remains". Nothing from batch 2 survives.
- **Batch 3.** RAISERROR severity 10 is informational: the text is printed, the CATCH is not entered, "after RAISERROR" prints. THROW 50001 raises severity 16, is caught, and the CATCH prints "caught error 50001, severity 16". "after THROW" never prints.
- **Batch 4.** Autocommit. Label 7's first insert commits immediately. The second violates the primary key, error 2627, severity 14. The CATCH prints "caught error 2627", then THROW with no arguments re-raises the original error to the client with its original number, level 14 and state 1, and terminates the batch. "Batch 4: completed" never prints. Label 7 with barcode PKG0000007 survives.
- **Batch 5.** Exactly two rows: (1, PKG0000001, 2.50) and (7, PKG0000007, 4.10).

## 6. Hint ladder (one hint per attempt, in order)

**Batch 1**
1. "Label 2 has a negative weight. Which constraint does it hit, and is that error statement-terminating or batch-terminating when XACT underscore ABORT is OFF?"
2. "Only the failing INSERT is undone. But where does control go right after the error? Does label 3 ever get inserted? Does the COMMIT inside the TRY ever run?"
3. "In the CATCH, the transaction is still alive and committable. What does XACT underscore STATE return, and what does that branch do with label 1?"

**Batch 2**
1. "Same error, but XACT underscore ABORT is ON. What does an error inside a TRY block do to the transaction under that setting?"
2. "The transaction is doomed. What does XACT underscore STATE return for a doomed transaction, and what is the only thing you can do with it?"
3. "After the ROLLBACK runs, there are still two IFs left to evaluate. What is XACT underscore STATE now? Does the third IF fire?"
4. "Label 4 was inserted successfully before the error. Does it survive a ROLLBACK?"

**Batch 3**
1. "TRY CATCH catches errors with a severity higher than what number? What severity does the RAISERROR use?"
2. "Severity 10 is a message, not an error. So what is printed, and does execution continue to the next PRINT?"
3. "THROW with a number and a message: what severity does it always use? Is that caught?"

**Batch 4**
1. "Is there a BEGIN TRANSACTION in this batch? So what happens to the first insert of label 7 the moment it finishes?"
2. "The second insert duplicates the primary key. What error number is that? The CATCH prints it, then runs THROW with no arguments. What does a bare THROW do?"
3. "A re-throw sends the original error to the client with the original severity, and then what happens to the rest of the batch? Is the last PRINT reached?"

**Batch 5**
1. "Go batch by batch. What did batch 1 commit? What did batch 2 leave behind? What was already durable in batch 4 before the failure?"
2. "Two rows. One from batch 1, one from batch 4. Which barcode does label 7 carry?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "Batch 1 keeps labels 1 and 3" | Thinks execution continues in the TRY after a caught error | "After the error, where does control go? Is the third INSERT ever reached?" |
| "Batch 1 rolls back everything" | Thinks any error dooms a transaction | "What is XACT underscore ABORT set to in batch 1? What does a constraint violation do under that setting?" |
| "Batch 2 keeps label 4 because it succeeded" | Ignores the rollback | "What does the minus 1 branch do? Does a ROLLBACK spare earlier successful statements?" |
| "Batch 2 prints only the doomed line" | Misses that the IFs are independent | "After the rollback, XACT underscore STATE is re-evaluated by the next IF. What is it now?" |
| "Batch 3 goes straight to the CATCH on the RAISERROR" | Does not know severity 10 is informational | "What is the minimum severity TRY CATCH catches?" |
| "Batch 3 prints severity 10 or the severity of RAISERROR" | Confuses the two constructs | "Which statement was actually caught, and what severity does a new THROW always carry?" |
| "Batch 4 prints completed at the end" | Thinks a re-throw is just another PRINT | "What does THROW do to the batch after it fires?" |
| "The re-thrown error shows Level 16" | Assumes THROW always means 16 | "Does a parameterless re-throw invent a severity, or keep the original one?" |
| "Batch 5 has no label 7 because batch 4 failed" | Forgets autocommit | "Without BEGIN TRANSACTION, when does the first INSERT of label 7 commit?" |

## 8. Teaching notes (after the answer is complete or revealed)

- **Batch 1, XACT_ABORT OFF.** A constraint violation, error 547 severity 16, is statement-terminating: the failing INSERT is rolled back, the transaction stays alive and committable. Control jumps to the CATCH immediately, so label 3 and the COMMIT in the TRY never execute. XACT_STATE is 1; the CATCH commits what is left, label 1 alone. Two traps in one batch: a caught error still skips the rest of the TRY, and committing in the CATCH keeps the pre-error work.
- **Batch 2, XACT_ABORT ON.** Per the TRY CATCH docs, an error that would normally terminate only the statement makes the transaction uncommittable, doomed, when it happens inside a TRY with XACT_ABORT ON. XACT_STATE is minus 1: the transaction still exists and holds locks but can only read or ROLLBACK; a write or COMMIT raises error 3930. The rollback undoes label 4 as well. Afterwards XACT_STATE is 0, so the third IF also fires. All three XACT_STATE values, 1, minus 1 and 0, seen in two batches.
- **Batch 3, RAISERROR versus THROW.** TRY CATCH catches errors with severity higher than 10. Severity 10 and below are informational, printed to the messages stream, CATCH not entered. THROW with arguments always raises severity 16; it has no severity parameter. The statement before a THROW must end with a semicolon or THROW can be misparsed, for example as a column alias after a SELECT.
- **Batch 4, autocommit and bare THROW.** With no BEGIN TRANSACTION each INSERT is its own autocommit transaction, so label 7's first insert is durable before anything fails. The duplicate hits PK_Labels, error 2627 severity 14. THROW with no arguments is legal only inside a CATCH and re-raises the original error with its original number, severity and state, level 14 here, not 16. The re-throw terminates the batch, so the final PRINT never runs. The reported Line 4 is the line of the failing INSERT within the batch, counting from the comment line, because a comment after GO belongs to the next batch.
- **Batch 5.** Label 1 from batch 1, label 7 from batch 4. Labels 2, 3, 4, 5, 6 never persisted.

THROW versus RAISERROR: RAISERROR lets the caller choose severity, cannot re-throw, needs no terminator, ignores XACT_ABORT, and execution can continue after it in a CATCH. THROW is always 16 when initiating, re-throws with a bare THROW inside CATCH only, needs the previous statement terminated with a semicolon, honours XACT_ABORT, and terminates the batch.

Memory hook: "ABORT OFF: statement dies, state 1. ABORT ON: transaction doomed, state minus 1. Severity 10 is not an error. Bare THROW keeps the original severity and ends the batch. Autocommit means it is already saved."

## 9. Follow-up oral questions (optional)

1. "In batch 2, what would happen if the CATCH tried to COMMIT while XACT_STATE is minus 1?" (Error 3930: the current transaction cannot be committed and cannot support operations that write to the log.)
2. "How would you re-raise the caught error using RAISERROR instead of THROW?" (You cannot re-throw directly; you must rebuild it from ERROR_NUMBER, ERROR_MESSAGE, ERROR_SEVERITY and ERROR_STATE.)
3. "If batch 3's PRINT before the THROW had no semicolon, what could go wrong?" (THROW may be misparsed, for example as an alias; the docs require the previous statement to be terminated.)

## 10. References

- TRY...CATCH: https://learn.microsoft.com/en-us/sql/t-sql/language-elements/try-catch-transact-sql
- THROW: https://learn.microsoft.com/en-us/sql/t-sql/language-elements/throw-transact-sql
- RAISERROR: https://learn.microsoft.com/en-us/sql/t-sql/language-elements/raiserror-transact-sql
- SET XACT_ABORT: https://learn.microsoft.com/en-us/sql/t-sql/statements/set-xact-abort-transact-sql
- XACT_STATE: https://learn.microsoft.com/en-us/sql/t-sql/functions/xact-state-transact-sql
- Database engine error severities: https://learn.microsoft.com/en-us/sql/relational-databases/errors-events/database-engine-error-severities
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
