# Instructor-Examiner guide — Scalar Functions 1

Companion to [scalar_functions_1.md](scalar_functions_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a long multi-part question: eight statements plus a final catalog query. Take them strictly one at a time, S1 to S8, then the catalog query. For statements that produce rows, ask for the values row by row. The catalog query has five rows and six columns; take it function by function, and accept "yes or no" for the one-and-zero columns.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Implement programmability objects.
- Task bullet: Create user-defined functions.
- What is tested: how scalar user-defined functions are called, NULL-input handling, determinism and SCHEMABINDING for persisted computed columns, the side-effect rule, and scalar UDF inlining as shown in sys dot sql underscore modules.

## 2. Scenario to read aloud

**Piece 1, the story.** "TollRoad is the billing database of a motorway operator, running SQL Server 2025 at compatibility level 170. Toll fees depend on the vehicle class. Vehicles with no class pay a base rate. The developers write several scalar user-defined functions, each with different options, and then try to use them in various ways."

**Piece 2, the tables.** "Schema Toll. Table Toll dot Rate has two columns: VehicleClass, a single character, primary key, and PerKm, a decimal with six digits and three decimals. Three rows: class A at 0.050 per kilometre, class B at 0.120, class C at 0.300. Table Toll dot Trip has three columns: TripID, integer, primary key; VehicleClass, a single character that allows NULL; and Km, a decimal with seven digits and one decimal, not null. Four trips: trip 1, class A, one hundred kilometres. Trip 2, class C, forty kilometres. Trip 3, class NULL, one hundred kilometres. Trip 4, class B, ten point five kilometres."

**Piece 3, the function Fee.** "Function Toll dot Fee takes a class and a kilometre count and returns a decimal with nine digits and two decimals. It has no options at all. Its body declares a rate variable initialised to 0.010, the base rate for unclassified vehicles. Then it does SELECT rate equals PerKm FROM Toll dot Rate WHERE VehicleClass equals the class parameter. Then it returns kilometres times rate."

**Piece 4, the functions FeeStrict and FeeNoInline.** "Function Toll dot FeeStrict has exactly the same body as Fee, but it is created WITH SCHEMABINDING and RETURNS NULL ON NULL INPUT. Function Toll dot FeeNoInline also has exactly the same body, and is created WITH INLINE equals OFF. So there are three functions with identical bodies and different options."

**Piece 5, the functions RushSurcharge and RoundKm.** "Function Toll dot RushSurcharge takes only kilometres, is created WITH SCHEMABINDING, and returns kilometres times 0.01 when the current hour, taken from GETDATE, is between 7 and 9, otherwise zero. Function Toll dot RoundKm takes kilometres, is created WITH SCHEMABINDING, returns an integer, and simply returns CEILING of the kilometres."

**Piece 6, statements S1 to S4.** "Eight statements then run in order, each in its own batch.
- S1 is SELECT Fee open paren, quote A, comma 10.0, close paren AS Fee. Note: just Fee, with no schema in front.
- S2 selects, from Toll dot Trip ordered by TripID, the TripID, Toll dot Fee of VehicleClass and Km as Fee, and Toll dot FeeStrict of VehicleClass and Km as FeeStrict.
- S3 alters Toll dot Trip to add a computed column Surcharge, defined as Toll dot RushSurcharge of Km, PERSISTED.
- S4 alters Toll dot Trip to add a computed column FeeDue, defined as Toll dot Fee of VehicleClass and Km, PERSISTED."

**Piece 7, statements S5 to S8.** "
- S5 alters Toll dot Trip to add a computed column KmUp, defined as Toll dot RoundKm of Km, PERSISTED, and then creates a nonclustered index IX underscore Trip underscore KmUp on that column.
- S6 alters table Toll dot Rate, ALTER COLUMN PerKm to decimal eight comma three, not null. That is a widening from six digits to eight.
- S7 creates a new function Toll dot LogFee, taking class and kilometres, whose body first inserts a row into Toll dot Trip with TripID 99, and then returns Toll dot Fee of the same arguments.
- S8 declares a decimal variable r, then runs EXEC r equals Toll dot Fee, quote B, comma 10.0, and selects r as FeeB. That is EXEC syntax on a function, not SELECT."

**Piece 8, the final catalog query.** "Finally a query joins sys dot sql underscore modules to sys dot objects for scalar functions, type FN, in schema Toll, ordered by function name. It returns, per function: is underscore inlineable, inline underscore type, is underscore schema underscore bound, null underscore on underscore null underscore input, and OBJECTPROPERTY IsDeterministic. All are one or zero."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE TollRoad;
GO
USE TollRoad;
GO
CREATE SCHEMA Toll;
GO
CREATE TABLE Toll.Rate
(
    VehicleClass CHAR(1)      NOT NULL PRIMARY KEY,
    PerKm        DECIMAL(6,3) NOT NULL
);
INSERT INTO Toll.Rate (VehicleClass, PerKm) VALUES ('A', 0.050), ('B', 0.120), ('C', 0.300);
CREATE TABLE Toll.Trip
(
    TripID       INT          NOT NULL PRIMARY KEY,
    VehicleClass CHAR(1)      NULL,
    Km           DECIMAL(7,1) NOT NULL
);
INSERT INTO Toll.Trip (TripID, VehicleClass, Km)
VALUES (1, 'A', 100.0), (2, 'C', 40.0), (3, NULL, 100.0), (4, 'B', 10.5);
GO
CREATE FUNCTION Toll.Fee (@Class CHAR(1), @Km DECIMAL(7,1))
RETURNS DECIMAL(9,2)
AS
BEGIN
    DECLARE @rate DECIMAL(6,3) = 0.010;   -- base rate for unclassified vehicles
    SELECT @rate = PerKm FROM Toll.Rate WHERE VehicleClass = @Class;
    RETURN @Km * @rate;
END;
GO
CREATE FUNCTION Toll.FeeStrict (@Class CHAR(1), @Km DECIMAL(7,1))
RETURNS DECIMAL(9,2)
WITH SCHEMABINDING, RETURNS NULL ON NULL INPUT
AS
BEGIN
    DECLARE @rate DECIMAL(6,3) = 0.010;
    SELECT @rate = PerKm FROM Toll.Rate WHERE VehicleClass = @Class;
    RETURN @Km * @rate;
END;
GO
CREATE FUNCTION Toll.FeeNoInline (@Class CHAR(1), @Km DECIMAL(7,1))
RETURNS DECIMAL(9,2)
WITH INLINE = OFF
AS
BEGIN
    DECLARE @rate DECIMAL(6,3) = 0.010;
    SELECT @rate = PerKm FROM Toll.Rate WHERE VehicleClass = @Class;
    RETURN @Km * @rate;
END;
GO
CREATE FUNCTION Toll.RushSurcharge (@Km DECIMAL(7,1))
RETURNS DECIMAL(9,2)
WITH SCHEMABINDING
AS
BEGIN
    RETURN CASE WHEN DATEPART(HOUR, GETDATE()) BETWEEN 7 AND 9 THEN @Km * 0.01 ELSE 0 END;
END;
GO
CREATE FUNCTION Toll.RoundKm (@Km DECIMAL(7,1))
RETURNS INT
WITH SCHEMABINDING
AS
BEGIN
    RETURN CEILING(@Km);
END;
GO
-- S1
SELECT Fee('A', 10.0) AS Fee;
-- S2
SELECT TripID, Toll.Fee(VehicleClass, Km) AS Fee, Toll.FeeStrict(VehicleClass, Km) AS FeeStrict
FROM Toll.Trip ORDER BY TripID;
-- S3
ALTER TABLE Toll.Trip ADD Surcharge AS Toll.RushSurcharge(Km) PERSISTED;
-- S4
ALTER TABLE Toll.Trip ADD FeeDue AS Toll.Fee(VehicleClass, Km) PERSISTED;
-- S5
ALTER TABLE Toll.Trip ADD KmUp AS Toll.RoundKm(Km) PERSISTED;
CREATE NONCLUSTERED INDEX IX_Trip_KmUp ON Toll.Trip (KmUp);
-- S6
ALTER TABLE Toll.Rate ALTER COLUMN PerKm DECIMAL(8,3) NOT NULL;
-- S7
CREATE FUNCTION Toll.LogFee (@Class CHAR(1), @Km DECIMAL(7,1))
RETURNS DECIMAL(9,2)
AS
BEGIN
    INSERT INTO Toll.Trip (TripID, VehicleClass, Km) VALUES (99, @Class, @Km);
    RETURN Toll.Fee(@Class, @Km);
END;
-- S8
DECLARE @r DECIMAL(9,2);
EXEC @r = Toll.Fee 'B', 10.0;
SELECT @r AS FeeB;
-- Final catalog query
SELECT OBJECT_NAME(m.object_id) AS FunctionName, m.is_inlineable, m.inline_type,
       m.is_schema_bound, m.null_on_null_input,
       OBJECTPROPERTY(m.object_id, 'IsDeterministic') AS IsDeterministic
FROM sys.sql_modules AS m
JOIN sys.objects AS o ON o.object_id = m.object_id
WHERE o.type = 'FN' AND o.schema_id = SCHEMA_ID('Toll')
ORDER BY FunctionName;
```

## 4. The question (ask exactly this)

"For each statement S1 to S8, tell me whether it succeeds or raises an error, and give the exact result set where one is produced. Let's go one at a time, starting with S1."

After all eight: "Now the final catalog query. For each function in schema Toll, in name order, tell me the values of is underscore inlineable, inline underscore type, is underscore schema underscore bound, null underscore on underscore null underscore input, and IsDeterministic. Start with the first function alphabetically."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

| Stmt | Outcome | Detail |
|---|---|---|
| S1 | Fails, error 195 | "'Fee' is not a recognized built-in function name." A scalar UDF needs at least a two-part name |
| S2 | Succeeds, 4 rows | See below. Trip 3: Fee 1.00, FeeStrict NULL |
| S3 | Fails, error 4936 | Computed column Surcharge cannot be persisted because it is non-deterministic (GETDATE inside RushSurcharge) |
| S4 | Fails, error 4936 | Same message for FeeDue: Fee is not schema-bound, so it is not deterministic |
| S5 | Succeeds | Persisted column KmUp created with values 100, 40, 100, 11, and the index is created |
| S6 | Fails, errors 5074 then 4922 | "The object 'FeeStrict' is dependent on column 'PerKm'." then "ALTER TABLE ALTER COLUMN PerKm failed because one or more objects access this column." |
| S7 | Fails, error 443 | "Invalid use of a side-effecting operator 'INSERT' within a function." |
| S8 | Succeeds | FeeB = 1.20 |

S2 result:

| TripID | Fee | FeeStrict |
|---|---|---|
| 1 | 5.00 | 5.00 |
| 2 | 12.00 | 12.00 |
| 3 | 1.00 | NULL |
| 4 | 1.26 | 1.26 |

Final catalog query:

| FunctionName | is_inlineable | inline_type | is_schema_bound | null_on_null_input | IsDeterministic |
|---|---|---|---|---|---|
| Fee | 1 | 1 | 0 | 0 | 0 |
| FeeNoInline | 1 | 0 | 0 | 0 | 0 |
| FeeStrict | 1 | 1 | 1 | 1 | 1 |
| RoundKm | 1 | 1 | 1 | 0 | 1 |
| RushSurcharge | 0 | 0 | 1 | 0 | 0 |

LogFee is absent because S7 failed.

## 6. Hint ladder (one hint per attempt, in order)

**S1**
1. "Look at how the function is named in the call. Is there a schema in front of it?"
2. "When the parser sees a one-part name with parentheses, what kind of function does it assume you mean?"
3. "There is no built-in function called Fee. Is that a success or an error?"

**S2**
1. "Trips 1, 2 and 4 have a class. Multiply kilometres by the class rate. For trip 4, that is ten point five times 0.120."
2. "Trip 3 has a NULL class. Walk through the body of Fee: the SELECT finds no row, so what value does the rate variable keep?"
3. "FeeStrict has the same body but one extra option about NULL input. What does that option do before the body even runs?"

**S3**
1. "A PERSISTED computed column has one requirement about its expression. What must the expression be?"
2. "RushSurcharge is schema-bound. But look inside it: what function does it call to know the hour?"
3. "GETDATE is non-deterministic. Does schema binding fix that?"

**S4**
1. "Fee calls only deterministic code. So why might the engine still refuse? Compare the options of Fee with those of FeeStrict."
2. "A user-defined function is only considered deterministic when it is created with a specific option. Which one is missing from Fee?"

**S5**
1. "RoundKm: is it schema-bound? Does CEILING depend on anything but its input?"
2. "If the column can be persisted, can it also be indexed? Then compute CEILING for 100.0, 40.0, 100.0 and 10.5."

**S6**
1. "This alters the Rate table, not a function. Which function was created with an option that ties it to Rate dot PerKm?"
2. "Does SCHEMABINDING care that the change is only a widening?"
3. "It is an error with two messages, the same pair you get when a schemabound view blocks an ALTER COLUMN."

**S7**
1. "Read the first statement inside LogFee. What does it do to a real table?"
2. "May a user-defined function change database state? What is the rule?"
3. "The error is raised at CREATE time, so the function is never created. Keep that in mind for the catalog query."

**S8**
1. "This is EXEC syntax, the way you would call a stored procedure. Is that allowed for a scalar function?"
2. "It is allowed. So compute ten times the class B rate."

**Catalog query**
1. "Which functions exist at the end? Remember whether LogFee was created."
2. "is underscore schema underscore bound and null underscore on underscore null underscore input come straight from the CREATE options. Fill those in first."
3. "IsDeterministic is one only when the function is schema-bound AND calls nothing non-deterministic. Which two functions satisfy both?"
4. "For inlining: is underscore inlineable asks whether the body qualifies; inline underscore type asks whether inlining is actually enabled. Which function has a body that qualifies but was created with INLINE equals OFF? Which function has a body that does not qualify at all?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "S1 returns 0.50" | Forgets the two-part name rule | "Read the call again. Is Fee qualified with its schema?" |
| "S2 trip 3: both columns 1.00" | Ignores RETURNS NULL ON NULL INPUT | "FeeStrict has an extra option. Does the body even run when an argument is NULL?" |
| "S2 trip 3: both columns NULL" | Thinks any NULL argument gives NULL | "Fee has the default option, CALLED ON NULL INPUT. Trace the body with a NULL class." |
| "S4 succeeds, Fee is deterministic, no GETDATE" | Does not know determinism needs SCHEMABINDING | "The engine judges determinism by two conditions, not one. Compare Fee's options with FeeStrict's." |
| "S6 succeeds, widening is safe" | Thinks SCHEMABINDING only blocks narrowing | "SCHEMABINDING does not judge whether the change is safe. What does it forbid?" |
| "S7 succeeds, then S8 works" | Thinks functions may do DML | "What is the rule about side effects inside a function? And when is it checked?" |
| "S8 fails, functions cannot be EXECed" | Confuses functions with procedures | "It is unusual, but is it invalid? Think of EXEC with a return variable." |
| "FeeNoInline: is_inlineable 0" | Confuses body eligibility with enabled state | "There are two columns. One is about the body, one about the option. Which one does INLINE equals OFF change?" |
| "RushSurcharge: is_inlineable 1, schema-bound after all" | Does not know GETDATE blocks inlining | "Time-dependent intrinsics are on the documented exclusion list for inlining." |

## 8. Teaching notes (after the answer is complete or revealed)

Go rule by rule:

- **Calling convention.** A scalar UDF is always called as schema dot function. A one-part name is parsed as a built-in function, hence error 195 in S1. The EXEC form in S8, EXEC variable equals schema dot function arguments, is also valid, positional or named.
- **NULL handling.** RETURNS NULL ON NULL INPUT returns NULL without running the body whenever any argument is NULL. The default is CALLED ON NULL INPUT, where the body decides. That is why Fee and FeeStrict, with identical bodies, differ only on trip 3: Fee falls back to the base rate 0.010 and gives 1.00; FeeStrict gives NULL.
- **Determinism and PERSISTED.** A computed column can be PERSISTED or indexed only if its expression is deterministic. A UDF is deterministic only when it is created WITH SCHEMABINDING and calls no non-deterministic function. RushSurcharge fails on GETDATE, Fee fails on missing SCHEMABINDING, both with error 4936. RoundKm passes both tests, so S5 succeeds and the index can be created.
- **SCHEMABINDING freezes columns.** FeeStrict is bound to Rate dot PerKm, so even a widening ALTER COLUMN fails with 5074 and 4922. Fee and FeeNoInline read the same column without binding and would not block the change.
- **No side effects.** A function may not INSERT, UPDATE or DELETE a permanent table, may not EXEC a procedure, and may not use TRY CATCH or RAISERROR or THROW. Error 443 is raised at CREATE time, so LogFee never exists. Table variables inside the function may be modified.
- **Inlining.** Since compatibility level 150 the optimizer can inline a scalar UDF into the calling query. is underscore inlineable says whether the body qualifies; inline underscore type says whether inlining is enabled for that function. FeeNoInline qualifies but is switched off by WITH INLINE equals OFF, so one and zero. RushSurcharge does not qualify because it calls GETDATE, so zero and zero. Inlining can also be disabled database-wide with the scoped configuration TSQL underscore SCALAR underscore UDF underscore INLINING, or per query with the hint DISABLE underscore TSQL underscore SCALAR underscore UDF underscore INLINING.

Memory hook: "Two-part name to call. Null-on-null skips the body. Deterministic means schemabound plus no clock. No side effects. Inlineable is the body, inline type is the switch."

## 9. Follow-up oral questions (optional)

1. "How would you make S4 succeed without changing the body of Fee?" (Recreate Fee WITH SCHEMABINDING, as FeeStrict is; then IsDeterministic becomes 1.)
2. "Could you add Surcharge as a non-persisted computed column?" (Yes. Non-deterministic expressions are allowed in non-persisted computed columns; they just cannot be persisted or indexed.)
3. "Name one thing on the list of constructs that make a scalar UDF not inlineable, other than GETDATE." (Any of: at-at-ROWCOUNT, table variables or table-valued parameters, CTEs, an EXECUTE AS clause other than CALLER, NEWSEQUENTIALID, use in a computed column or CHECK constraint.)

## 10. References

- CREATE FUNCTION, including SCHEMABINDING, RETURNS NULL ON NULL INPUT, INLINE and the side-effect rules: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-function-transact-sql
- Scalar UDF inlining: https://learn.microsoft.com/en-us/sql/relational-databases/user-defined-functions/scalar-udf-inlining
- Deterministic and nondeterministic functions: https://learn.microsoft.com/en-us/sql/relational-databases/user-defined-functions/deterministic-and-nondeterministic-functions
- Indexes on computed columns: https://learn.microsoft.com/en-us/sql/relational-databases/indexes/indexes-on-computed-columns
- sys.sql_modules: https://learn.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-sql-modules-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
