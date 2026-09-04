# Instructor-Examiner guide — Column-Level Encryption 1

Companion to [column_level_encryption_1.md](column_level_encryption_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a six-part prediction question, Q1 to Q6, one query per part. Take them strictly in order, because the session state (key open or closed, and the row swap in Q5) carries forward. For each query, ask the learner for the value of every one of the four rows. Do not hint about row 1 until the learner has committed to an answer for it; the L1 trap is the heart of the question.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Implement data security and compliance.
- Task bullet: Implement column-level encryption.
- What is tested: the engine-side key hierarchy, that ENCRYPTBYKEY and DECRYPTBYKEY return NULL rather than errors, how authenticators work in both directions, why re-encrypting cannot be used for equality search, and what DECRYPTBYKEYAUTOCERT does.

## 2. Scenario to read aloud

**Piece 1, the story.** "A courier company stores customers' cards on file in a SQL Server 2025 database called CourierVault. The card number is protected with the classic engine-side encryption hierarchy: a database master key, then a certificate, then a symmetric key. Encryption and decryption use the functions ENCRYPTBYKEY and DECRYPTBYKEY."

**Piece 2, the table.** "There is a schema called Pay and one table, Pay dot CardOnFile. Four columns. CardID, an integer, the primary key. CustomerID, an integer, not null. Holder, a varchar of forty, not null. And CardNoEnc, a VARBINARY of 256, which allows null. That last column holds the ciphertext."

**Piece 3, the key hierarchy.** "Three statements build the hierarchy. CREATE MASTER KEY ENCRYPTION BY PASSWORD, with a long password. CREATE CERTIFICATE PayCert WITH SUBJECT CourierVault card protection. And CREATE SYMMETRIC KEY PayKey WITH ALGORITHM AES underscore 256, ENCRYPTION BY CERTIFICATE PayCert. Note that the certificate has no password of its own; it is protected by the database master key."

**Piece 4, load step L1.** "Now the first load step, called L1. Important: no key has been opened yet. An INSERT puts row 1 into CardOnFile: CardID 1, CustomerID 501, holder Nadia Ferrer, and CardNoEnc set to ENCRYPTBYKEY of KEY underscore GUID of PayKey, the card number 4111 2222 3333 4444, the flag 1, and an authenticator which is CustomerID 501 converted to sysname. The insert succeeds."

**Piece 5, load step L2.** "The second load step, L2, is one batch. First, OPEN SYMMETRIC KEY PayKey DECRYPTION BY CERTIFICATE PayCert. Then three inserts. Row 2: CardID 2, customer 502, Pau Roca, card 5500 1111 2222 3333, encrypted with flag 1 and authenticator 502. Row 3: CardID 3, customer 503, Lena Voss, card 4000 9999 8888 7777, flag 1 and authenticator 503. Row 4: CardID 4, customer 504, Iker Lasa, card 3400 5555 6666 7777, encrypted with no flag and no authenticator. Then CLOSE SYMMETRIC KEY PayKey. Every insert succeeds. So rows 2 and 3 use an authenticator, and row 4 does not."

**Piece 6, the queries.** "Then six queries run in order, in a single session, each in its own batch. I will describe them one at a time. All of them return CardID and a CardNo column, ordered by CardID, unless I say otherwise. CardNo is DECRYPTBYKEY converted to varchar of twenty."

**Piece 7, Q1.** "Q1. No key has been opened in this session. SELECT CardID and DECRYPTBYKEY of CardNoEnc, with flag 1 and authenticator CustomerID converted to sysname, from CardOnFile, ordered by CardID."

**Piece 8, Q2.** "Q2. First, OPEN SYMMETRIC KEY PayKey DECRYPTION BY CERTIFICATE PayCert. Then the same SELECT as Q1: DECRYPTBYKEY with flag 1 and the CustomerID authenticator."

**Piece 9, Q3.** "Q3. The key is still open. SELECT CardID and DECRYPTBYKEY of CardNoEnc with no flag and no authenticator, converted to varchar of twenty."

**Piece 10, Q4.** "Q4. SELECT COUNT star as Hits from CardOnFile WHERE CardNoEnc equals ENCRYPTBYKEY of KEY underscore GUID of PayKey, the string 5500 1111 2222 3333, flag 1, and authenticator 502. That is exactly the card and authenticator used for row 2."

**Piece 11, Q5.** "Q5. A rogue admin runs an UPDATE that copies row 2's CardNoEnc into row 3, where CardID equals 3. Then the same SELECT as Q2, DECRYPTBYKEY with flag 1 and the CustomerID authenticator. Then CLOSE SYMMETRIC KEY PayKey."

**Piece 12, Q6.** "Q6. The key is closed again. SELECT CardID and DECRYPTBYKEYAUTOCERT, with arguments CERT underscore ID of PayCert, then NULL for the certificate password, then CardNoEnc, then flag 1 and the CustomerID authenticator. Converted to varchar of twenty, ordered by CardID. And you may assume no query raises an error."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE CourierVault;
GO
USE CourierVault;
GO
CREATE SCHEMA Pay;
GO
CREATE TABLE Pay.CardOnFile
(
    CardID     INT            NOT NULL PRIMARY KEY,
    CustomerID INT            NOT NULL,
    Holder     VARCHAR(40)    NOT NULL,
    CardNoEnc  VARBINARY(256) NULL
);
GO
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'Cv#2026!MasterKey_Long';
GO
CREATE CERTIFICATE PayCert WITH SUBJECT = 'CourierVault card protection';
GO
CREATE SYMMETRIC KEY PayKey
    WITH ALGORITHM = AES_256
    ENCRYPTION BY CERTIFICATE PayCert;
GO
-- L1: the key is NOT open yet
INSERT INTO Pay.CardOnFile (CardID, CustomerID, Holder, CardNoEnc)
VALUES (1, 501, 'Nadia Ferrer',
        ENCRYPTBYKEY(KEY_GUID('PayKey'), '4111222233334444', 1, CONVERT(sysname, 501)));
GO
-- L2: open the key, insert three more rows, close it
OPEN SYMMETRIC KEY PayKey DECRYPTION BY CERTIFICATE PayCert;
INSERT INTO Pay.CardOnFile (CardID, CustomerID, Holder, CardNoEnc)
VALUES (2, 502, 'Pau Roca',
        ENCRYPTBYKEY(KEY_GUID('PayKey'), '5500111122223333', 1, CONVERT(sysname, 502)));
INSERT INTO Pay.CardOnFile (CardID, CustomerID, Holder, CardNoEnc)
VALUES (3, 503, 'Lena Voss',
        ENCRYPTBYKEY(KEY_GUID('PayKey'), '4000999988887777', 1, CONVERT(sysname, 503)));
INSERT INTO Pay.CardOnFile (CardID, CustomerID, Holder, CardNoEnc)
VALUES (4, 504, 'Iker Lasa',
        ENCRYPTBYKEY(KEY_GUID('PayKey'), '3400555566667777'));
CLOSE SYMMETRIC KEY PayKey;
GO
```

```sql
-- Q1  (no key has been opened in this session)
SELECT CardID,
       CONVERT(VARCHAR(20), DECRYPTBYKEY(CardNoEnc, 1, CONVERT(sysname, CustomerID))) AS CardNo
FROM Pay.CardOnFile ORDER BY CardID;

-- Q2
OPEN SYMMETRIC KEY PayKey DECRYPTION BY CERTIFICATE PayCert;
SELECT CardID,
       CONVERT(VARCHAR(20), DECRYPTBYKEY(CardNoEnc, 1, CONVERT(sysname, CustomerID))) AS CardNo
FROM Pay.CardOnFile ORDER BY CardID;

-- Q3
SELECT CardID, CONVERT(VARCHAR(20), DECRYPTBYKEY(CardNoEnc)) AS CardNo
FROM Pay.CardOnFile ORDER BY CardID;

-- Q4
SELECT COUNT(*) AS Hits
FROM Pay.CardOnFile
WHERE CardNoEnc = ENCRYPTBYKEY(KEY_GUID('PayKey'), '5500111122223333', 1, CONVERT(sysname, 502));

-- Q5  (a rogue admin copies row 2's ciphertext into row 3, then decrypts)
UPDATE Pay.CardOnFile
SET CardNoEnc = (SELECT CardNoEnc FROM Pay.CardOnFile WHERE CardID = 2)
WHERE CardID = 3;
SELECT CardID,
       CONVERT(VARCHAR(20), DECRYPTBYKEY(CardNoEnc, 1, CONVERT(sysname, CustomerID))) AS CardNo
FROM Pay.CardOnFile ORDER BY CardID;
CLOSE SYMMETRIC KEY PayKey;

-- Q6  (key closed again)
SELECT CardID,
       CONVERT(VARCHAR(20), DECRYPTBYKEYAUTOCERT(CERT_ID('PayCert'), NULL, CardNoEnc,
                                                 1, CONVERT(sysname, CustomerID))) AS CardNo
FROM Pay.CardOnFile ORDER BY CardID;
```

## 4. The question (ask exactly this)

"Give the exact result of each query, Q1 to Q6. Assume no query raises an error. For each query, tell me the CardNo value for CardID 1, 2, 3 and 4, and for Q4 the single Hits value. Let's start with Q1."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

Every query succeeds; none raises an error.

| Query | CardID 1 | CardID 2 | CardID 3 | CardID 4 |
|---|---|---|---|---|
| Q1 (key closed, with authenticator) | NULL | NULL | NULL | NULL |
| Q2 (key open, with authenticator) | NULL | 5500111122223333 | 4000999988887777 | NULL |
| Q3 (key open, no authenticator) | NULL | NULL | NULL | 3400555566667777 |
| Q4 | Hits = 0, a single value | | | |
| Q5 (after the ciphertext swap) | NULL | 5500111122223333 | NULL | NULL |
| Q6 (key closed, DECRYPTBYKEYAUTOCERT) | NULL | 5500111122223333 | NULL | NULL |

Row 1's CardNoEnc is NULL in the table itself: in L1 the key was not open, so ENCRYPTBYKEY returned NULL and the INSERT stored NULL. Rows 2 and 3 hold 84 bytes of ciphertext, row 4 holds 68 bytes. Row 1 therefore decrypts to NULL in every query.

## 6. Hint ladder (one hint per attempt, in order)

**Q1**
1. "Before decrypting, what does DECRYPTBYKEY need to be true in this session? Look at the comment on Q1."
2. "The key is not open. Does DECRYPTBYKEY throw an error in that case, or does it return something?"
3. "It returns NULL for every row, whatever the ciphertext. That includes the good rows."

**Q2**
1. "Now the key is open. Compare how each row was encrypted with how this query decrypts. Which rows used an authenticator at insert time?"
2. "Row 4 was encrypted without an authenticator, and this query supplies one. Does the integrity check pass?"
3. "Now think about row 1 more carefully. Go back to L1. Was the key open when row 1 was inserted? What did ENCRYPTBYKEY return, and what did the INSERT store?"
4. "The INSERT in L1 succeeded. But what value sits in CardNoEnc for row 1? Decrypting that gives what?"

**Q3**
1. "This query passes no authenticator. Which row was encrypted without one?"
2. "The mismatch works in both directions. Rows encrypted with an authenticator and decrypted without one give what?"

**Q4**
1. "You are comparing stored ciphertext to freshly computed ciphertext of the same plaintext and same authenticator. Does ENCRYPTBYKEY produce the same bytes each time?"
2. "ENCRYPTBYKEY uses a random initialization vector. So how many rows can match?"

**Q5**
1. "Row 3 now holds row 2's ciphertext. That ciphertext was bound to which authenticator? What authenticator does row 3 supply?"
2. "This is exactly the attack authenticators are designed to stop. What does DECRYPTBYKEY return when the authenticator does not match? And what about rows 2 and 4, unchanged from Q2?"

**Q6**
1. "The key is closed, but this function is not DECRYPTBYKEY. Read its name again. What does the AUTOCERT part do?"
2. "It opens the symmetric key with the certificate and decrypts in one call, so a closed session key does not matter. Same authenticator rules as Q5. Same table contents as after Q5."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "Q1 raises an error because the key is not open" | Expects crypto functions to throw | "The question says no query raises an error. What do these functions return instead?" |
| "Q2 row 1 is 4111 2222 3333 4444" | Assumes the L1 insert stored real ciphertext | "Was the key open during L1? What does ENCRYPTBYKEY return when it is not? And does the INSERT still succeed?" |
| "Q2 row 4 decrypts too, the key is open" | Ignores the authenticator mismatch | "Row 4 was encrypted without an authenticator. This query supplies one. Does that pass?" |
| "Q3 all four rows decrypt" | Thinks the authenticator is optional at decrypt time | "The flag and authenticator must match what was used at encryption time. Which rows used one?" |
| "Q4 Hits is 1, row 2 matches" | Assumes deterministic encryption | "Does ENCRYPTBYKEY give identical bytes for identical input? Think about the initialization vector." |
| "Q5 row 3 shows Pau Roca's card" | Forgets the authenticator binds ciphertext to the row | "Which CustomerID was baked into that ciphertext, and which one does row 3 pass?" |
| "Q6 all NULL because the key is closed" | Reads DECRYPTBYKEYAUTOCERT as DECRYPTBYKEY | "What extra work does the AUTOCERT variant do before decrypting the data?" |

## 8. Teaching notes (after the answer is complete or revealed)

The whole question turns on one design decision: the engine-side crypto functions never raise an error for a failed encryption or decryption. They return NULL.

- **L1, the silent loss.** ENCRYPTBYKEY requires the symmetric key to be open in the current session. Nothing was open in L1, so the function returned NULL and the INSERT succeeded with CardNoEnc NULL. The application "successfully" lost the card number. Always OPEN SYMMETRIC KEY first, or check sys.openkeys, and consider a NOT NULL constraint on the ciphertext column so the insert fails instead.
- **Q1.** DECRYPTBYKEY returns NULL if the key is not open or the ciphertext is NULL. The OPEN in L2 was explicitly closed, so all four rows are NULL, including the three good ones.
- **Q2.** OPEN SYMMETRIC KEY uses the certificate's private key, itself protected by the database master key, which the service master key opens automatically. Rows 2 and 3 were encrypted with add_authenticator 1 and authenticators 502 and 503; the same values are supplied, so they decrypt. Row 4 has no authenticator; decrypting with one fails the integrity check and returns NULL.
- **Q3.** The opposite mismatch. No authenticator passed: row 4 decrypts, rows 2 and 3 return NULL. The flag and authenticator must match in both directions.
- **Q4.** ENCRYPTBYKEY uses a random initialization vector, so the same plaintext gives different ciphertext each time. Hits is 0. Equality search on engine-side encrypted columns needs a separate deterministic token, such as an HMAC column via HASHBYTES, or Always Encrypted with deterministic encryption. Storage note: AES-256 ciphertext of a 16-character string is 68 bytes, 84 with an authenticator; size the VARBINARY for that, not for the plaintext.
- **Q5.** Copying row 2's ciphertext into row 3 is a whole-value substitution attack. Without authenticators the blob would decrypt under row 3 and Lena Voss would own Pau Roca's card. The ciphertext was bound to 502; row 3 supplies 503; DECRYPTBYKEY returns NULL. The documentation recommends the authenticator precisely "to prevent a malicious user from substituting values in an encrypted column".
- **Q6.** DECRYPTBYKEYAUTOCERT takes cert id, cert password, ciphertext, flag and authenticator. It decrypts the symmetric key with the certificate and decrypts the data in one call, so it works with the session key closed. NULL is the certificate password because PayCert is protected by the database master key, not by a password. Results match Q5.

Side notes: DECRYPTBYKEY returns varbinary(8000), so the cast must target the original type; encrypt an nvarchar and cast to varchar and you get only the first character. ENCRYPTBYPASSPHRASE skips the hierarchy and shares the NULL-on-failure behaviour. ENCRYPTBYCERT and ENCRYPTBYASYMKEY are for small payloads and much slower; the documented advice is to encrypt data with symmetric keys. DECRYPTBYKEY must run in the database that holds the key; from master it returns NULL.

Memory hook: "Closed key, wrong authenticator, swapped blob, wrong database: all NULL, never an error. Same plaintext, different bytes. AUTOCERT opens the key for you."

## 9. Follow-up oral questions (optional)

1. "How could the team have made the L1 insert fail instead of storing NULL?" (A NOT NULL constraint on CardNoEnc, or checking sys.openkeys before encrypting.)
2. "If the card had been stored as NVARCHAR and encrypted as N'4111', what would CONVERT to VARCHAR return after decryption?" (Just '4', the first byte of the UTF-16 pair; you must cast to NVARCHAR.)
3. "Who can read the plaintext in this design, and what should you use when even the DBA must not?" (Anyone who can OPEN the key, including the DBA. Use Always Encrypted when the DBA must not see plaintext.)

## 10. References

- ENCRYPTBYKEY: https://learn.microsoft.com/en-us/sql/t-sql/functions/encryptbykey-transact-sql
- DECRYPTBYKEY: https://learn.microsoft.com/en-us/sql/t-sql/functions/decryptbykey-transact-sql
- DECRYPTBYKEYAUTOCERT: https://learn.microsoft.com/en-us/sql/t-sql/functions/decryptbykeyautocert-transact-sql
- OPEN SYMMETRIC KEY: https://learn.microsoft.com/en-us/sql/t-sql/statements/open-symmetric-key-transact-sql
- CREATE SYMMETRIC KEY: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-symmetric-key-transact-sql
- Encryption hierarchy: https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/encryption-hierarchy
- Encrypt a column of data: https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/encrypt-a-column-of-data
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
