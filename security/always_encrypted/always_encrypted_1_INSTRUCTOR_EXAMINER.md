# Instructor-Examiner guide — Always Encrypted 1

Companion to [always_encrypted_1.md](always_encrypted_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Read all four requirements and all four options before taking an answer. Ask the learner for one letter, and ask them to say in one sentence why the other three fail. If the learner picks a wrong letter, say only "not that one" and go to the hints.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Implement data security and compliance.
- Task bullet: Implement Always Encrypted, including secure enclaves.
- What is tested: which operations deterministic encryption supports, which operations need a secure enclave with randomized encryption and enclave-enabled keys, how in-place encryption works, and why TDE and Dynamic Data Masking do not hide data from a DBA.

## 2. Scenario to read aloud

**Piece 1, the story.** "A consumer-lending company keeps borrower records in a SQL Server 2025 database named LoanLedger. It runs on a Windows host at compatibility level one hundred seventy. The DBA team administers the instance, but the DBAs are not allowed to see two columns: the borrower's tax identifier and the salary. The security team has already provisioned the Always Encrypted key metadata and the table."

**Piece 2, the column master key.** "First, a column master key named CMK underscore Loans. It lives outside the engine, in the Windows certificate store of the application servers. The CREATE COLUMN MASTER KEY statement gives two things: the key store provider name, which is MSSQL underscore CERTIFICATE underscore STORE, and the key path, which is CurrentUser slash My slash a long certificate thumbprint. Note that there is no ENCLAVE underscore COMPUTATIONS clause on this key."

**Piece 3, the column encryption key.** "Second, a column encryption key named CEK underscore Loans. It is stored inside the database, but only as a value encrypted by the column master key. The statement says WITH VALUES, column master key equals CMK underscore Loans, algorithm RSA underscore OAEP, and a long hexadecimal encrypted value."

**Piece 4, the table.** "Third, the table dbo dot Borrowers, with three columns. BorrowerID, an integer, the primary key. TaxID, a fixed eleven-character string with the collation Latin1 underscore General underscore BIN2, encrypted with CEK underscore Loans, encryption type DETERMINISTIC, algorithm AEAD AES 256 CBC HMAC SHA 256, not null. And Salary, a decimal ten comma two, encrypted with the same key, but encryption type RANDOMIZED, same algorithm, and it allows nulls."

**Piece 5, the four requirements.** "The application team must build a loan-origination service against this table. Four requirements. One: the service must find a borrower by exact TaxID, a point lookup, and must be able to join Borrowers to a second table on TaxID. Two: an underwriting report must filter borrowers with WHERE Salary BETWEEN at lo AND at hi, and sort by Salary, inside the database. It cannot pull every row to the client and filter there. Three: neither the plaintext values nor the column encryption key may ever be available in plaintext to the Database Engine outside a trusted execution environment. DBAs with sysadmin must keep seeing only ciphertext. Four: the existing plaintext Salary data in a legacy table must be encrypted without exporting it to a client tool and re-importing it."

**Piece 6, option a.** "Option a. Keep the table exactly as created. In the application connection string set Column Encryption Setting equals Enabled. Implement requirement two with parameterized queries such as SELECT from Borrowers WHERE Salary BETWEEN at lo AND at hi ORDER BY Salary, because the driver transparently encrypts the parameters and rewrites the query. For requirement four, run ALTER TABLE, ALTER COLUMN Salary, ENCRYPTED WITH, from SQL Server Management Studio."

**Piece 7, option b.** "Option b. Change Salary to encryption type DETERMINISTIC so the engine can compare ciphertexts. Keep TaxID deterministic. Set Column Encryption Setting equals Enabled in the application. Range predicates and ORDER BY then work on the deterministic ciphertext, and the DBA still sees only ciphertext."

**Piece 8, option c.** "Option c. Enable a secure enclave for the instance: run sp underscore configure, column encryption enclave type, value one, which is a VBS enclave, then restart. Re-create CMK underscore Loans with ENCLAVE underscore COMPUTATIONS and a signed metadata value. Re-create CEK underscore Loans encrypted by that enclave-enabled master key. Keep TaxID deterministic and Salary randomized. In the application connection string set Column Encryption Setting equals Enabled, plus the enclave attestation settings of the driver. Encrypt the legacy Salary column in place with ALTER TABLE, ALTER COLUMN, ENCRYPTED WITH."

**Piece 9, option d.** "Option d. Replace Always Encrypted with Transparent Data Encryption on LoanLedger, plus Dynamic Data Masking on TaxID and Salary, using MASKED WITH function default. Grant UNMASK only to the application user. The claim is that TDE encrypts the data files, so DBAs cannot read the columns, and the engine can evaluate every predicate normally."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE LoanLedger;
GO
USE LoanLedger;
GO
-- Column master key: lives OUTSIDE the engine (Windows certificate store of the app servers).
CREATE COLUMN MASTER KEY CMK_Loans
WITH (KEY_STORE_PROVIDER_NAME = 'MSSQL_CERTIFICATE_STORE',
      KEY_PATH = 'CurrentUser/My/2A6F3B9C1D4E5F60718293A4B5C6D7E8F9A0B1C2');
GO
-- Column encryption key: stored in the database, but only as a value ENCRYPTED BY the CMK.
CREATE COLUMN ENCRYPTION KEY CEK_Loans
WITH VALUES (COLUMN_MASTER_KEY = CMK_Loans, ALGORITHM = 'RSA_OAEP',
             ENCRYPTED_VALUE = 0x016E000001630075007200720065006E00740075 /* ... */);
GO
CREATE TABLE dbo.Borrowers
(
    BorrowerID INT NOT NULL PRIMARY KEY,
    TaxID  CHAR(11) COLLATE Latin1_General_BIN2
           ENCRYPTED WITH (COLUMN_ENCRYPTION_KEY = CEK_Loans, ENCRYPTION_TYPE = DETERMINISTIC,
                           ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256') NOT NULL,
    Salary DECIMAL(10,2)
           ENCRYPTED WITH (COLUMN_ENCRYPTION_KEY = CEK_Loans, ENCRYPTION_TYPE = RANDOMIZED,
                           ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256') NULL
);
GO
```

Engine messages observed without an enclave (SQL Server 2025, no client driver), for reading on request:

```text
INSERT INTO dbo.Borrowers VALUES (1, '123-45-6789', 52000.00);        -- Msg 206 operand type clash
SELECT BorrowerID FROM dbo.Borrowers WHERE TaxID LIKE '123%';          -- Msg 402 incompatible in LIKE
SELECT BorrowerID FROM dbo.Borrowers WHERE Salary > 50000;             -- Msg 33277 encryption scheme mismatch
SELECT Salary * 12 FROM dbo.Borrowers;                                 -- Msg 206
SELECT TaxID, COUNT(*) FROM dbo.Borrowers GROUP BY TaxID;              -- succeeds (deterministic)
SELECT Salary, COUNT(*) FROM dbo.Borrowers GROUP BY Salary;            -- Msg 33277 (randomized)
SELECT ... FROM dbo.Borrowers a JOIN dbo.Borrowers b ON a.TaxID = b.TaxID;   -- succeeds
SELECT BorrowerID FROM dbo.Borrowers ORDER BY TaxID;                   -- Msg 33277 (needs enclave)
CREATE INDEX IX_TaxID  ON dbo.Borrowers (TaxID);                       -- succeeds (deterministic)
CREATE INDEX IX_Salary ON dbo.Borrowers (Salary);                      -- Msg 33573
ALTER TABLE dbo.Borrowers ALTER COLUMN Salary DECIMAL(10,2) NULL;      -- Msg 33543 (key not enclave-enabled)
```

## 4. The question (ask exactly this)

"Which approach satisfies all four requirements?

a. Keep the table exactly as created. In the application connection string set Column Encryption Setting equals Enabled. Implement requirement two with parameterized queries such as SELECT from dbo dot Borrowers WHERE Salary BETWEEN at lo AND at hi ORDER BY Salary, because the driver transparently encrypts the parameters and rewrites the query. For requirement four, run ALTER TABLE, ALTER COLUMN Salary, ENCRYPTED WITH, from SSMS.

b. Change Salary to ENCRYPTION TYPE equals DETERMINISTIC so that the engine can compare ciphertexts, keep TaxID deterministic, and set Column Encryption Setting equals Enabled in the application. Range predicates and ORDER BY then work on the deterministic ciphertext, and the DBA still sees only ciphertext.

c. Enable a secure enclave for the instance, sp underscore configure column encryption enclave type equals one for a VBS enclave, then restart. Re-create CMK underscore Loans with ENCLAVE underscore COMPUTATIONS and a signed metadata value. Re-create CEK underscore Loans encrypted by that enclave-enabled CMK. Keep TaxID deterministic and Salary randomized. In the application connection string set Column Encryption Setting equals Enabled plus the enclave attestation settings of the driver. Encrypt the legacy Salary column in place with ALTER TABLE, ALTER COLUMN, ENCRYPTED WITH.

d. Replace Always Encrypted with Transparent Data Encryption on LoanLedger plus Dynamic Data Masking, MASKED WITH function default, on TaxID and Salary, and grant UNMASK only to the application user. TDE encrypts the data files, so DBAs cannot read the columns, and the engine can evaluate every predicate normally.

Which letter, and why do the other three fail?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: c.**

| Option | Verdict | Why |
|---|---|---|
| a | Wrong | The driver can encrypt a parameter for an equality on a deterministic column, but the engine cannot compare randomized ciphertexts. BETWEEN and ORDER BY on Salary raise Msg 33277 without an enclave. ALTER TABLE ENCRYPTED WITH on a non-enclave key raises Msg 33543; SSMS would have to download, encrypt and re-upload the data, which requirement 4 forbids. |
| b | Wrong | Deterministic encryption preserves equality only. Ciphertexts sort by ciphertext, not by plaintext, so BETWEEN and ORDER BY still fail. It also weakens confidentiality because equal salaries give equal ciphertexts. Even an enclave supports rich computations only on randomized columns. |
| c | Correct | VBS enclave on Windows SQL Server 2019 and later, CMK with ENCLAVE_COMPUTATIONS, enclave-enabled CEK, deterministic TaxID for equality and joins, randomized Salary for range and sort inside the enclave, compatibility level at least 160, driver with Column Encryption Setting=Enabled plus attestation settings, in-place encryption via ALTER TABLE ALTER COLUMN. |
| d | Wrong | TDE encrypts files at rest; the engine decrypts pages in memory and any sysadmin reads plaintext. DDM is a presentation mask that UNMASK and CONTROL holders bypass and that predicates ignore, so a DBA can brute-force values with WHERE. Requirement 3 fails. A masked column cannot be an Always Encrypted column. |

Key messages: Msg 33277 lists the three enclave conditions, RANDOMIZED, BIN2 collation for strings, enclave-enabled CEK. Msg 33543 refuses in-place encryption when a key is not enclave-enabled. Msg 33573 refuses an index on a randomized column with a non-enclave key. Msg 33289 says deterministic strings need a BIN2 collation.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with requirement three. Which of the four options gives up on Always Encrypted entirely? Think about who can read a TDE-encrypted database once it is open in memory."
2. "Now requirement two, a range predicate and a sort on Salary. Deterministic encryption gives you one thing only: equal plaintexts become equal ciphertexts. Does equality help you sort or compare with BETWEEN?"
3. "Option a keeps Salary randomized and relies on the driver alone. The driver encrypts the parameters, yes. But who has to evaluate BETWEEN? The engine. Can the engine compare two randomized ciphertexts without help?"
4. "Requirement four asks for encryption without moving the data out. Without a certain feature, the engine refuses in-place encryption with message 33543. What feature does that message mention?"
5. "One option enables a trusted execution environment inside the engine and re-creates both keys so they are enclave-enabled. That is the only path to range queries and in-place encryption. Which letter is that?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "Option a, the driver rewrites the query so BETWEEN works" | Thinks client-side parameter encryption makes any predicate evaluable | "The driver encrypts the parameter. The comparison still happens in the engine. Can the engine order two randomized ciphertexts?" |
| "Option a, SSMS can encrypt in place" | Confuses the SSMS wizard with true in-place encryption | "Where does the SSMS wizard do the encryption work, and where does the data travel? Read requirement four again." |
| "Option b, deterministic lets the engine compare" | Believes deterministic encryption preserves order | "Deterministic preserves one relation only. Which one? Is BETWEEN that relation?" |
| "Option b is safer for salaries" | Does not know deterministic leaks equality patterns | "If two borrowers earn the same salary, what does a DBA see in the two ciphertexts?" |
| "Option d, TDE hides data from DBAs" | Confuses at-rest encryption with client-side encryption | "Who is dbo in every database? What does that person see when they run SELECT on a TDE database?" |
| "Option d, masking blocks the DBA" | Thinks DDM is a security boundary | "Can a masked column still be used in a WHERE clause? What could a DBA learn by filtering with BETWEEN?" |
| "Option c, but Salary should be deterministic too" | Does not know enclave computations require randomized | "Message 33277 lists three conditions for rich computations. What encryption type is the first one?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the client-side model first:

- **Always Encrypted is client-side encryption.** The driver encrypts parameters and decrypts results. The engine stores ciphertext and never holds the keys in plaintext. The column master key lives outside SQL, in a Windows certificate store, Azure Key Vault or an HSM. The column encryption key lives inside the database, but only as a value encrypted by the master key. The driver switch is Column Encryption Setting equals Enabled.
- **Deterministic encryption** gives the same ciphertext for the same plaintext. That supports equality, IN, GROUP BY, DISTINCT, equality joins and indexes. Strings must use a BIN2 collation, otherwise Msg 33289. It leaks equality patterns, so use it only for lookup columns. That is requirement 1 and TaxID.
- **Randomized encryption** gives no computations at all, unless a secure enclave is used.

Then the enclave rules:

- Without an enclave, literals and T-SQL variables against encrypted columns, LIKE, less than, greater than, BETWEEN, arithmetic, ORDER BY, and INSERT SELECT between plaintext and encrypted all fail with Msg 206, 402 or 33277. In-place ALTER COLUMN fails with Msg 33543. An index on a randomized column fails with Msg 33573.
- With a secure enclave, the driver sends the CEK into a protected memory region inside the engine process over a secure channel. Plaintext exists only inside that region. Everyone outside, including sysadmin, sees ciphertext. That is requirement 3.
- The chain is: CMK created with ENCLAVE underscore COMPUTATIONS and a signed metadata value, then an enclave-enabled CEK, then a RANDOMIZED column, and compatibility level 160 or higher. That unlocks LIKE, comparisons, BETWEEN, ORDER BY, JOIN and GROUP BY since SQL Server 2022, indexes on randomized columns, and in-place encryption, key rotation and type changes through ALTER TABLE ALTER COLUMN. That is requirements 2 and 4.
- Platform: VBS enclaves on SQL Server 2019 and later on Windows and in Azure SQL Database. Intel SGX only on Azure SQL Database DC-series hardware, with Microsoft Azure Attestation. SQL Server does not support SGX. The driver needs Column Encryption Setting equals Enabled plus the attestation protocol setting, Host Guardian Service, or no attestation for VBS on SQL Server 2019 and later.

Why each wrong option fails:

- Option a: never enables an enclave, so BETWEEN and ORDER BY on randomized Salary raise 33277, and in-place ALTER raises 33543.
- Option b: deterministic preserves equality only, not order. Range and sort still fail, and salary patterns leak.
- Option d: TDE decrypts pages in memory for every reader; DDM is bypassed by UNMASK and CONTROL and ignored by predicates. DBAs still reach plaintext.

Memory hook: "DBA must not see it: Always Encrypted. Range, LIKE or sort in the database, or encrypt in place: add a secure enclave, randomized column, enclave-enabled keys."

## 9. Follow-up oral questions (optional)

1. "Which collation family must a deterministic string column use, and what error do you get otherwise?" (A BIN2 collation such as Latin1_General_BIN2; Msg 33289.)
2. "With an enclave in place, can Salary stay deterministic and still support BETWEEN?" (No. Enclave rich computations require randomized encryption.)
3. "Where does sys dot column underscore master underscore keys store the master key itself?" (It does not. It stores only the provider name and key path, the location of the key.)

## 10. References

- Always Encrypted overview: https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/always-encrypted-database-engine
- Always Encrypted with secure enclaves: https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/always-encrypted-enclaves
- Configure the secure enclave in SQL Server: https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/configure-sql-server-always-encrypted-enclaves
- Configure column encryption in-place using Always Encrypted with secure enclaves: https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/always-encrypted-enclaves-configure-encryption
- CREATE COLUMN MASTER KEY: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-column-master-key-transact-sql
- CREATE COLUMN ENCRYPTION KEY: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-column-encryption-key-transact-sql
- Transparent Data Encryption: https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/transparent-data-encryption
- Dynamic Data Masking: https://learn.microsoft.com/en-us/sql/relational-databases/security/dynamic-data-masking
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
