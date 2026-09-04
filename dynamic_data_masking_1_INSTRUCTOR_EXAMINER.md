# Instructor-Examiner guide — Dynamic Data Masking 1

Companion to [dynamic_data_masking_1.md](dynamic_data_masking_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Three of the options are result tables that differ in only one or two columns, so read them slowly, column by column, and repeat any row on request. A good way to run it: before reading the options, ask the learner how many rows they expect and what each column looks like; then read the options and take one letter.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Implement data protection and security.
- Task bullet: Implement dynamic data masking.
- What is tested: that predicates run on real values, the output of each masking function, column-level UNMASK, and that any expression over a masked column is masked with the default function.

## 2. Scenario to read aloud

**Piece 1, the story.** "A dental clinic stores patient records in an Azure SQL Database called ClinicCare. The complete history of the database is one script, executed by the database owner from creation onward. Nothing else has ever been run against it."

**Piece 2, the table.** "There is a schema called Med and one table, Med dot Patients, with six columns. PatientID, an integer identity, the primary key. Then five columns, every one of them with a mask. FullName, an NVARCHAR of sixty, masked with partial, open paren, 2, comma, a string of five hash signs, comma, 2, close paren. Email, a varchar of eighty, masked with the email function. NationalID, a char of nine, masked with partial, open paren, 0, comma, a string of five capital X, comma, 4, close paren. MonthlyFee, a decimal eight comma two, masked with default. And NextVisit, a date, masked with default."

**Piece 3, the data.** "Four patients are inserted. Patient 1, Marta Vidal, email marta dot vidal at dentalia dot example, national ID 4 8 2 9 1 7 3 6 M, fee 300, next visit third of September 2026. Patient 2, Joan Serra, email j dot serra at clinicare dot example, ID 1 0 4 5 7 8 2 9 K, fee 180, next visit tenth of September. Patient 3, Aina Pons, email aina dot pons at mail dot example, ID 3 9 6 8 4 2 1 5 T, fee 300, next visit seventeenth of September. Patient 4, Omar Haddad, email omar dot haddad at dentalia dot example, ID 5 2 1 7 0 3 4 8 Z, fee 245, next visit first of October."

**Piece 4, the user and grants.** "Then a user called Reception is created, without login. Two grants: GRANT SELECT ON SCHEMA Med TO Reception. And GRANT UNMASK ON Med dot Patients, open paren, Email, close paren, TO Reception. Reception is in no role other than public, holds no other permission, and is not db underscore owner."

**Piece 5, the query.** "The owner then runs EXECUTE AS USER Reception, and a SELECT of seven columns from Med dot Patients: PatientID, FullName, UPPER of FullName aliased as NameKey, Email, NationalID, MonthlyFee and NextVisit. WHERE MonthlyFee is greater than or equal to 200. ORDER BY PatientID. Then REVERT. The question is what exact result set the SELECT returns."

**Piece 6, option a.** "Option a. The query returns zero rows, because for Reception every MonthlyFee is masked to zero, and zero is not greater than or equal to 200."

**Piece 7, option b.** "Option b. Three rows, patients 1, 3 and 4. FullName shows Ma, five hashes, al; Ai, five hashes, ns; and Om, five hashes, ad. NameKey shows four lowercase x on every row. Email shows the real addresses. NationalID shows five capital X followed by 736M, 215T and 348Z. MonthlyFee shows 0.00 on every row. NextVisit shows the first of January 1900 on every row."

**Piece 8, option c.** "Option c. Same three rows as b, same values in every column except NameKey. Here NameKey is the partial mask in uppercase: capital M A, five hashes, capital A L; A I, five hashes, N S; O M, five hashes, A D."

**Piece 9, option d.** "Option d. Same three rows, patients 1, 3 and 4, but nothing is masked. Full names in clear, NameKey in uppercase, real emails, full national IDs, real fees 300, 300 and 245, and real next visit dates."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE SCHEMA Med;
GO

CREATE TABLE Med.Patients
(
    PatientID   INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    FullName    NVARCHAR(60) NOT NULL
                MASKED WITH (FUNCTION = 'partial(2, "#####", 2)'),
    Email       VARCHAR(80)  NOT NULL
                MASKED WITH (FUNCTION = 'email()'),
    NationalID  CHAR(9)      NOT NULL
                MASKED WITH (FUNCTION = 'partial(0, "XXXXX", 4)'),
    MonthlyFee  DECIMAL(8,2) NOT NULL
                MASKED WITH (FUNCTION = 'default()'),
    NextVisit   DATE         NOT NULL
                MASKED WITH (FUNCTION = 'default()')
);
GO

INSERT INTO Med.Patients (FullName, Email, NationalID, MonthlyFee, NextVisit) VALUES
(N'Marta Vidal',  'marta.vidal@dentalia.example',  '48291736M', 300.00, '2026-09-03'),
(N'Joan Serra',   'j.serra@clinicare.example',     '10457829K', 180.00, '2026-09-10'),
(N'Aina Pons',    'aina.pons@mail.example',        '39684215T', 300.00, '2026-09-17'),
(N'Omar Haddad',  'omar.haddad@dentalia.example',  '52170348Z', 245.00, '2026-10-01');
GO

CREATE USER Reception WITHOUT LOGIN;
GO

GRANT SELECT ON SCHEMA::Med TO Reception;
GRANT UNMASK ON Med.Patients(Email) TO Reception;
GO
```

```sql
EXECUTE AS USER = 'Reception';

SELECT PatientID,
       FullName,
       UPPER(FullName) AS NameKey,
       Email,
       NationalID,
       MonthlyFee,
       NextVisit
FROM Med.Patients
WHERE MonthlyFee >= 200
ORDER BY PatientID;

REVERT;
```

Option b:

| PatientID | FullName | NameKey | Email | NationalID | MonthlyFee | NextVisit |
|---|---|---|---|---|---|---|
| 1 | Ma#####al | xxxx | marta.vidal@dentalia.example | XXXXX736M | 0.00 | 1900-01-01 |
| 3 | Ai#####ns | xxxx | aina.pons@mail.example | XXXXX215T | 0.00 | 1900-01-01 |
| 4 | Om#####ad | xxxx | omar.haddad@dentalia.example | XXXXX348Z | 0.00 | 1900-01-01 |

Option c: same as b, but NameKey is MA#####AL, AI#####NS, OM#####AD.

Option d: same three rows with every column unmasked.

## 4. The question (ask exactly this)

"What exact result set does the SELECT return? Option a, zero rows. Option b, three rows with partial-masked names, NameKey four x's, real emails, masked IDs, fee 0.00 and date 1900. Option c, the same but NameKey is the uppercase partial mask. Option d, three rows fully unmasked. Give me one letter, and tell me why."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct: b.** The WHERE clause runs on the real fees, so rows 1, 3 and 4 survive and row 2 with fee 180 is filtered out. FullName shows the partial mask: Ma#####al, Ai#####ns, Om#####ad. NameKey is an expression over a masked column, so it is masked with the default function, which is xxxx for strings. Email is unmasked because Reception holds column-level UNMASK on it. NationalID shows XXXXX736M, XXXXX215T, XXXXX348Z. MonthlyFee shows 0.00 and NextVisit shows 1900-01-01, the default masks for numeric and date.

- **a is wrong.** Masking happens at presentation, after the engine has filtered on stored values. If masks fed predicates, the documented inference attack could not exist.
- **c is wrong.** The engine does not build a masked string and then apply UPPER. Any projected expression over a masked column is masked as a whole, always with the default function, so the result is xxxx.
- **d is wrong.** UNMASK is grantable at column scope since SQL Server 2022 and in Azure SQL Database. The grant names only Email; the other four columns stay masked.

## 6. Hint ladder (one hint per attempt, in order)

1. "First decide how many rows come back. Does the WHERE clause see the masked fee or the stored fee? Think about what dynamic data masking actually changes: the data, or its presentation?"
2. "The engine filters on real values. Three patients pay 200 or more. So how many rows? That settles one option."
3. "Now the Email column. Reception has one UNMASK grant. What scope does it cover: the whole database, the table, or one column? What does that mean for the other four masked columns?"
4. "Two options left, and they differ only in NameKey, which is UPPER of FullName. Does the engine mask the column first and then run UPPER, or does it treat the whole expression differently? Which masking function applies to an expression over a masked column?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, the fee is zero for Reception so nothing passes the filter" | Thinks masks feed predicates | "If that were true, could a masked user ever filter on a number? What does the documentation call the inference attack?" |
| "d, Reception has UNMASK so it sees everything" | Pre-2022 single-scope UNMASK | "Read the grant again. Which object, and which column, does it name?" |
| "c, UPPER of Ma hash hash al is MA hash hash AL" | Assumes masking produces an intermediate string | "Does the engine apply the column's own mask inside an expression, or something else? Which function masks expressions?" |
| "The email shows m X X X at X X X X dot com" | Forgets the column-level grant | "Reception has one UNMASK grant. On which column?" |
| "NationalID shows XXXXX plus the last five characters" | Misreads the partial suffix argument | "The third argument of partial on NationalID is 4. How many trailing characters does that expose?" |
| "MonthlyFee shows random numbers" | Confuses default with random | "Which function is on MonthlyFee? What does default return for a numeric type?" |

## 8. Teaching notes (after the answer is complete or revealed)

Three independent DDM behaviours decide this result:

1. **Predicates run against real stored values.** Masking happens only when data is presented. WHERE, JOIN, GROUP BY and ORDER BY see real data. Rows 1, 3 and 4 have fees of 300, 300 and 245; row 2 has 180 and is dropped. This is the documented inference leak: the query already proves to Reception that three patients pay at least 200. By repeating with narrower predicates, Reception can brute-force every exact fee while only ever seeing 0.00. DDM is a presentation-layer control, not an authorization boundary; Microsoft recommends pairing it with auditing, least privilege, row-level security or encryption.
2. **UNMASK is grantable at column granularity** in Azure SQL Database and SQL Server 2022 and later, at database, schema, table or column scope. The grant names Med.Patients(Email), so only Email is unmasked. Only principals with UNMASK at a covering scope, or CONTROL such as db_owner or sysadmin, see the other columns.
3. **Any expression over a masked column is masked with the default function**, whatever the column's own mask. So UPPER(FullName) collapses to xxxx, not to an uppercase partial mask.

Deriving each value:

- FullName, partial(2, "#####", 2): first two characters, five hashes, last two. Marta Vidal gives Ma#####al. Aina Pons gives Ai#####ns. Omar Haddad gives Om#####ad.
- Email, email(): would show first character plus XXX@XXXX.com, but it is unmasked for Reception, so the real addresses appear.
- NationalID, partial(0, "XXXXX", 4): zero leading characters, five X, last four. 48291736M gives XXXXX736M. 39684215T gives XXXXX215T. 52170348Z gives XXXXX348Z.
- MonthlyFee, default() on a numeric type: zero, rendered with the column's scale as 0.00.
- NextVisit, default() on a date: 1900-01-01.

Default masks by type: string xxxx, fewer x's if the column is shorter than four; numeric 0; date and time 1900-01-01 00:00:00.0000000; binary a single byte 0. The random(low, high) function is valid only on numeric types and is the one non-deterministic mask. Copies stay masked: SELECT INTO or INSERT SELECT by a user without UNMASK persists masked text. A mask cannot be defined on an Always Encrypted column; and with Always Encrypted the engine could not have evaluated MonthlyFee >= 200 at all, which is the signature difference: with DDM the server sees everything and only the presentation is masked.

Memory hook: "Filters see the truth, results see the mask. UNMASK is per column. Expressions over masked columns always get the default mask."

## 9. Follow-up oral questions (optional)

1. "Without the UNMASK grant, what would Marta's email look like?" (mXXX@XXXX.com: first character plus the constant XXX@XXXX.com.)
2. "How could Reception discover Omar's exact fee without ever seeing it?" (Repeat the query with narrower predicates, for example WHERE MonthlyFee BETWEEN 240 AND 250, and see which rows come back.)
3. "Which masking function is non-deterministic, and on which types is it allowed?" (random(low, high), numeric types only.)

## 10. References

- Dynamic data masking: https://learn.microsoft.com/en-us/sql/relational-databases/security/dynamic-data-masking
- GRANT UNMASK, column-level granularity: https://learn.microsoft.com/en-us/sql/relational-databases/security/dynamic-data-masking#granular-permission-examples
- CREATE TABLE, MASKED WITH: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-table-transact-sql
- ALTER TABLE ALTER COLUMN, ADD MASKED: https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-table-column-definition-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
