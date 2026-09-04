# Instructor-Examiner guide — Object-Level Permissions 1

Companion to [object_level_permissions_1.md](object_level_permissions_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is an eight-statement permissions question. The six GRANT and DENY statements in piece 6 and the two owners in piece 5 are the whole puzzle, so repeat them whenever asked, and offer to repeat them before each statement. For each statement, require "succeeds" or "fails", and on failure ask whether the error is object-level, 229, or column-level, 230; accept the reason in words if the learner does not know the numbers. Say which user runs each statement every time.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Implement data security and compliance.
- Task bullet: Implement object-level permissions.
- What is tested: GRANT, DENY and REVOKE precedence including the column-level GRANT over table-level DENY quirk, DENY inherited through a role, ownership chaining through views and procedures, how dynamic SQL and a different owner break the chain, and error 229 versus 230.

## 2. Scenario to read aloud

**Piece 1, the story.** "An agricultural cooperative keeps its finance and operations data in a SQL Server 2025 database named HarvestCoop. Access is granted to database users without logins, so the whole scenario can be tested with EXECUTE AS USER. Everything is set up by dbo, the database owner."

**Piece 2, schemas, users and role.** "Two schemas: Fin and Ops. Three users without login: Analyst, Auditor and OpsOwner. One role, Reporting, with two members: Analyst and Auditor. OpsOwner is not in the role."

**Piece 3, the tables.** "Fin dot Invoices has InvoiceID, an integer primary key; Member, a VARCHAR forty; Amount, a DECIMAL ten comma two; and Margin, a DECIMAL ten comma two. Three rows: invoice 1, Mas Oliva, amount twelve hundred, margin one hundred eighty. Invoice 2, Can Roig, eight hundred fifty, margin ninety-five fifty. Invoice 3, Vall Verda, twenty-one hundred, margin three hundred ten. Ops dot Deliveries has DeliveryID, InvoiceID and Tons. Three rows: delivery 10 for invoice 1, twelve point five tons; 11 for invoice 2, eight tons; 12 for invoice 3, twenty-one tons."

**Piece 4, the views and procedures.** "Four objects in schema Fin. The view Fin dot vInvoiceMargin selects InvoiceID, Member and Margin from Fin dot Invoices. The view Fin dot vDeliveries selects DeliveryID, InvoiceID and Tons from Ops dot Deliveries. The procedure Fin dot GetMarginStatic takes an InvoiceID and runs a plain SELECT Margin from Fin dot Invoices for that id. The procedure Fin dot GetMarginDynamic takes an InvoiceID and runs the same SELECT through sp underscore executesql, that is, as dynamic SQL with a parameter."

**Piece 5, the owners.** "Before the views are created, dbo runs ALTER AUTHORIZATION ON SCHEMA Ops TO OpsOwner. So schema Ops, and Ops dot Deliveries, are owned by OpsOwner. Schema Fin, and therefore every object in it, the table, the two views and the two procedures, is owned by dbo."

**Piece 6, the six permission statements, listen carefully.** "One: GRANT SELECT ON SCHEMA Fin TO Reporting. Two: GRANT EXECUTE ON SCHEMA Fin TO Reporting. Three: DENY SELECT ON Fin dot Invoices TO Analyst. Four: GRANT SELECT ON Fin dot Invoices, columns InvoiceID and Amount only, TO Analyst. Five: GRANT SELECT ON Ops dot Deliveries TO Auditor. Six: DENY SELECT ON SCHEMA Ops TO Reporting."

**Piece 7, S1 to S4, all as Analyst.** "S1 selects InvoiceID and Amount from Fin dot Invoices. S2 selects InvoiceID and Margin from Fin dot Invoices. S3 selects InvoiceID, Member and Margin from the view Fin dot vInvoiceMargin. S4 executes Fin dot GetMarginStatic with InvoiceID 3."

**Piece 8, S5 to S8.** "S5, as Analyst, executes Fin dot GetMarginDynamic with InvoiceID 3. S6, as Analyst, selects DeliveryID and Tons from the view Fin dot vDeliveries. S7, as Auditor, selects DeliveryID and Tons from Ops dot Deliveries directly. S8 has two steps: first dbo runs REVOKE SELECT ON Fin dot Invoices FROM Analyst; then, as Analyst, the S2 query runs again: InvoiceID and Margin from Fin dot Invoices."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE HarvestCoop;
GO
USE HarvestCoop;
GO
CREATE SCHEMA Fin;
GO
CREATE SCHEMA Ops;
GO
CREATE USER Analyst  WITHOUT LOGIN;
CREATE USER Auditor  WITHOUT LOGIN;
CREATE USER OpsOwner WITHOUT LOGIN;
CREATE ROLE Reporting;
ALTER ROLE Reporting ADD MEMBER Analyst;
ALTER ROLE Reporting ADD MEMBER Auditor;
GO
CREATE TABLE Fin.Invoices
(
    InvoiceID INT           NOT NULL PRIMARY KEY,
    Member    VARCHAR(40)   NOT NULL,
    Amount    DECIMAL(10,2) NOT NULL,
    Margin    DECIMAL(10,2) NOT NULL
);
INSERT INTO Fin.Invoices VALUES
  (1, 'Mas Oliva',  1200.00, 180.00),
  (2, 'Can Roig',    850.00,  95.50),
  (3, 'Vall Verda', 2100.00, 310.00);
CREATE TABLE Ops.Deliveries
(
    DeliveryID INT          NOT NULL PRIMARY KEY,
    InvoiceID  INT          NOT NULL,
    Tons       DECIMAL(6,1) NOT NULL
);
INSERT INTO Ops.Deliveries VALUES (10, 1, 12.5), (11, 2, 8.0), (12, 3, 21.0);
GO
ALTER AUTHORIZATION ON SCHEMA::Ops TO OpsOwner;      -- Ops.Deliveries is now owned by OpsOwner
GO
CREATE VIEW Fin.vInvoiceMargin AS
SELECT InvoiceID, Member, Margin FROM Fin.Invoices;
GO
CREATE VIEW Fin.vDeliveries AS
SELECT d.DeliveryID, d.InvoiceID, d.Tons FROM Ops.Deliveries AS d;
GO
CREATE PROCEDURE Fin.GetMarginStatic @InvoiceID INT AS
    SELECT Margin FROM Fin.Invoices WHERE InvoiceID = @InvoiceID;
GO
CREATE PROCEDURE Fin.GetMarginDynamic @InvoiceID INT AS
    EXEC sp_executesql N'SELECT Margin FROM Fin.Invoices WHERE InvoiceID = @id',
                       N'@id INT', @InvoiceID;
GO
GRANT SELECT  ON SCHEMA::Fin TO Reporting;
GRANT EXECUTE ON SCHEMA::Fin TO Reporting;
DENY  SELECT  ON Fin.Invoices TO Analyst;
GRANT SELECT  ON Fin.Invoices (InvoiceID, Amount) TO Analyst;
GRANT SELECT  ON Ops.Deliveries TO Auditor;
DENY  SELECT  ON SCHEMA::Ops TO Reporting;
GO
```

The statements:

```sql
-- S1  (as Analyst)
SELECT InvoiceID, Amount FROM Fin.Invoices ORDER BY InvoiceID;

-- S2  (as Analyst)
SELECT InvoiceID, Margin FROM Fin.Invoices ORDER BY InvoiceID;

-- S3  (as Analyst)
SELECT InvoiceID, Member, Margin FROM Fin.vInvoiceMargin ORDER BY InvoiceID;

-- S4  (as Analyst)
EXEC Fin.GetMarginStatic @InvoiceID = 3;

-- S5  (as Analyst)
EXEC Fin.GetMarginDynamic @InvoiceID = 3;

-- S6  (as Analyst)
SELECT DeliveryID, Tons FROM Fin.vDeliveries ORDER BY DeliveryID;

-- S7  (as Auditor)
SELECT DeliveryID, Tons FROM Ops.Deliveries ORDER BY DeliveryID;

-- S8  (as dbo, then as Analyst)
REVOKE SELECT ON Fin.Invoices FROM Analyst;
EXECUTE AS USER = 'Analyst';
SELECT InvoiceID, Margin FROM Fin.Invoices ORDER BY InvoiceID;
REVERT;
```

## 4. The question (ask exactly this)

"For each statement, S1 to S8, tell me whether it succeeds or fails, and if it fails, with which error. Let's go one at a time. S1, as Analyst: InvoiceID and Amount from Fin dot Invoices."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

| Stmt | Outcome | Detail |
|---|---|---|
| S1 | Succeeds, 3 rows | (1, 1200.00), (2, 850.00), (3, 2100.00). Column-level GRANT wins over the table-level DENY |
| S2 | Fails, error 230 | The SELECT permission was denied on the column Margin of the object Invoices |
| S3 | Succeeds, 3 rows | Including Margin: 180.00, 95.50, 310.00. Unbroken ownership chain, dbo view over dbo table; the DENY on the table is never checked |
| S4 | Succeeds, 1 row | 310.00. Ownership chain through the procedure |
| S5 | Fails, error 230 | Dynamic SQL breaks the chain, so Analyst's own denied permission on Margin is checked |
| S6 | Fails, error 229 | The SELECT permission was denied on the object Deliveries, schema Ops. Chain broken by the owner change: dbo view over an OpsOwner table |
| S7 | Fails, error 229 | The DENY inherited through role Reporting overrides Auditor's direct GRANT |
| S8 | Succeeds, 3 rows | Including Margin. REVOKE removed the table-level DENY and the column grants; the schema-level GRANT through Reporting now applies |

HAS_PERMS_BY_NAME as Analyst before S8: Fin.Invoices SELECT equals 0; column Amount equals 1; column Margin equals 0; Fin.vInvoiceMargin SELECT equals 1; Ops.Deliveries SELECT equals 0.

## 6. Hint ladder (one hint per attempt, in order)

**S1**
1. "Analyst has a DENY on the whole table and a GRANT on two specific columns. Which two columns does S1 read?"
2. "There is one documented exception to DENY beats GRANT, and it is about columns. What is it?"

**S2**
1. "S2 reads Margin. Is Margin one of the columns in the column-level GRANT?"
2. "So the table-level DENY applies to that column. Is the error the object-level number or the column-level number?"

**S3**
1. "Forget the table for a moment. Who owns the view, and who owns the table? Same owner or different?"
2. "When the owner is the same all the way down, which object's permissions does the engine check: the entry object only, or every object?"
3. "Does Analyst have SELECT on the view? Look at the schema-level GRANT to Reporting. And is the DENY on the table ever consulted?"

**S4**
1. "Same mechanism as S3, through a procedure. Does Analyst have EXECUTE on it? Look at the schema-level grant."
2. "Is the SELECT inside the procedure static T-SQL? Then the chain covers it."

**S5**
1. "The only difference from S4 is how the SELECT runs. What does sp underscore executesql do to the security context?"
2. "Dynamic SQL runs as a new batch under the caller's own permissions. What does Analyst have on the Margin column?"

**S6**
1. "The view is in schema Fin, owned by dbo. The table it reads is in schema Ops. Who owns Ops after the ALTER AUTHORIZATION?"
2. "When the owner changes along the path, the chain breaks and the caller's permission on the table is checked. What does Analyst inherit on schema Ops through Reporting?"

**S7**
1. "Auditor has a direct GRANT on the table. But is Auditor in any role, and does that role have a DENY that covers Ops dot Deliveries?"
2. "A DENY at schema scope covers every object in the schema. Does a direct GRANT ever beat a DENY, apart from the column quirk?"

**S8**
1. "REVOKE is not the opposite of GRANT. What exactly does it remove: the GRANT, the DENY, or whichever explicit entry exists?"
2. "After the table-level DENY and the column grants are gone, what does Analyst still inherit from Reporting on schema Fin?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "S1 fails, DENY always wins" | Does not know the column-level exception | "That is the general rule. There is one documented exception involving columns. Which columns does S1 touch?" |
| "S2 fails with 229" | Mixes up object-level and column-level errors | "Is the denied thing the whole object or one column?" |
| "S3 fails, Margin is denied" | Ignores ownership chaining | "Who owns the view and who owns the table? What does the engine skip when they match?" |
| "S4 fails" | Same as S3 | "Does the chain apply to procedures as well as views?" |
| "S5 succeeds like S4" | Does not know dynamic SQL breaks the chain | "In whose security context does sp underscore executesql run its batch?" |
| "S6 succeeds, the view is in Fin" | Misses the owner change | "Follow the path: dbo view, then a table owned by whom?" |
| "S7 succeeds, Auditor has a direct GRANT" | Forgets DENY through a role | "What is Auditor a member of, and what does that role have on schema Ops?" |
| "S8 fails, Margin is still denied" | Thinks REVOKE only removes GRANTs | "What did REVOKE remove? Is there any DENY left?" |
| "S8 fails, Analyst now has nothing" | Forgets the schema-level GRANT via the role | "What does Reporting have on schema Fin?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the four rules in order.

- **Rule 1, DENY beats GRANT, except the column quirk.** If any applicable DENY exists, direct or through any role, at any covering scope, access is refused regardless of GRANTs. S7 is the textbook case: Auditor's direct GRANT on Ops dot Deliveries loses to the schema-scoped DENY inherited through Reporting, error 229. The one documented exception: a table-level DENY does not take precedence over a column-level GRANT. Analyst's column GRANT on InvoiceID and Amount makes S1 succeed; Margin has no column grant, so S2 hits the deny with error 230, the column-specific number. Microsoft calls this legacy behaviour that may change, but it is what the engine does today.
- **Rule 2, ownership chaining skips permission checks on referenced objects.** When a view or procedure references an object owned by the same principal, only the entry object is checked; the referenced objects are not checked at all, not even for DENYs. The view vInvoiceMargin and the table Invoices are both owned by dbo through schema Fin. Analyst has SELECT on the view via the schema grant, so S3 returns every column including the denied Margin. A DENY on a base table is not a guarantee that the data is unreachable; any same-owner view or module that exposes the column is a legal path. That is why "grant EXECUTE on procedures, nothing on tables" works, and why you must audit which views and modules expose sensitive columns. S4 is the same through the procedure.
- **Rule 3, the chain breaks with dynamic SQL or a different owner.** Dynamic SQL, sp underscore executesql or EXEC of a string, is compiled and run as a new batch in the caller's context, outside the module's chain. In S5 the inner SELECT Margin is checked against Analyst directly and the column deny fires, error 230. To make such a procedure work without table permissions, use EXECUTE AS OWNER on the module or sign it with a certificate. With different owners, vDeliveries by dbo and Ops dot Deliveries by OpsOwner, the chain breaks at the owner change and the caller's permission on the table is checked. Analyst inherits DENY on schema Ops through Reporting, so S6 fails with 229. Even without the deny, an explicit GRANT on the table would be needed. Cross-database chaining is off by default for the same reason.
- **Rule 4, REVOKE removes a DENY too.** REVOKE deletes whatever explicit entry exists for that permission on that securable, GRANT or DENY. After S8's REVOKE, sys dot database underscore permissions lists only CONNECT for Analyst; the column grants went too, because they hang off the same table-level permission. Nothing now blocks the schema-scoped GRANT through Reporting, so S8 reads Margin. To remove access, do not revoke, DENY. To restore the default, REVOKE.

Then testing: CREATE USER WITHOUT LOGIN plus EXECUTE AS USER and REVERT is the standard way to test a permission model on one connection. HAS_PERMS_BY_NAME evaluates the effective permission of the current context, including roles and denies, and fn_my_permissions lists everything the current context holds on a securable.

Memory hook: "Any DENY wins, except column GRANT over table DENY. Same owner, only the entry is checked. Dynamic SQL or a new owner breaks the chain. REVOKE erases a DENY too. 229 object, 230 column."

## 9. Follow-up oral questions (optional)

1. "How would you make GetMarginDynamic work for Analyst without granting anything on the table?" (Add WITH EXECUTE AS OWNER to the procedure, or sign the module with a certificate.)
2. "dbo wants Analyst to lose access to Margin for good, even after future grants. REVOKE or DENY?" (DENY. REVOKE only removes the existing entry and does not block later grants.)
3. "Would S6 succeed if the DENY on schema Ops were removed?" (No. The chain is still broken by the owner change, so Analyst would need an explicit GRANT SELECT on Ops dot Deliveries.)

## 10. References

- Permissions hierarchy and precedence (Database Engine): https://learn.microsoft.com/en-us/sql/relational-databases/security/permissions-hierarchy-database-engine
- DENY object permissions, including the column-level note: https://learn.microsoft.com/en-us/sql/t-sql/statements/deny-object-permissions-transact-sql
- GRANT object permissions: https://learn.microsoft.com/en-us/sql/t-sql/statements/grant-object-permissions-transact-sql
- REVOKE object permissions: https://learn.microsoft.com/en-us/sql/t-sql/statements/revoke-object-permissions-transact-sql
- Ownership chains and context switching: https://learn.microsoft.com/en-us/sql/relational-databases/security/authentication-access/ownership-chains
- EXECUTE AS clause for modules: https://learn.microsoft.com/en-us/sql/t-sql/statements/execute-as-clause-transact-sql
- HAS_PERMS_BY_NAME: https://learn.microsoft.com/en-us/sql/t-sql/functions/has-perms-by-name-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
