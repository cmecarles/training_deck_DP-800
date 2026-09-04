# SQL Server question — Column-Level Encryption 1

## Statement

A courier company stores customers' cards on file in a SQL Server 2025 database named `CourierVault`. The card number is protected with the classic **engine-side** encryption hierarchy (database master key → certificate → symmetric key) and the `ENCRYPTBYKEY` / `DECRYPTBYKEY` functions.

The complete setup is:

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

Rows 2 and 3 were encrypted **with an authenticator** (the customer id, `CONVERT(sysname, CustomerID)`); row 4 was encrypted **without** one. Every `INSERT` above succeeded.

The following queries are then executed in order, **in a single session**, each in its own batch:

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

Give the exact result of **each** query (Q1–Q6). Assume no query raises an error.

## Correct Answer

Every query succeeds; none raises an error. Results (all engine output):

| Query | CardID 1 | CardID 2 | CardID 3 | CardID 4 |
|-------|----------|----------|----------|----------|
| Q1 (key closed, with authenticator) | NULL | NULL | NULL | NULL |
| Q2 (key open, with authenticator) | NULL | 5500111122223333 | 4000999988887777 | NULL |
| Q3 (key open, no authenticator) | NULL | NULL | NULL | 3400555566667777 |
| Q4 | `Hits = 0` (single value) | | | |
| Q5 (after the ciphertext swap) | NULL | 5500111122223333 | NULL | NULL |
| Q6 (key closed, `DECRYPTBYKEYAUTOCERT`) | NULL | 5500111122223333 | NULL | NULL |

Row 1's `CardNoEnc` is **NULL in the table itself** (`DATALENGTH` = NULL; rows 2 and 3 hold 84 bytes, row 4 holds 68), so it decrypts to NULL in every query.

## Explanation

The whole question turns on one design decision of the engine-side crypto functions: **they never raise an error for a failed decryption or encryption — they return NULL.** Verified against SQL Server 2025 (RTM 17.0.1000.7); every value above is the engine's literal output.

### L1 — ENCRYPTBYKEY with a closed key silently stores NULL

`ENCRYPTBYKEY` requires the symmetric key to be **open in the current session**. In L1 nothing has been opened, so the function returns NULL and the `INSERT` succeeds anyway, writing `CardNoEnc = NULL` for Nadia Ferrer. No error, no warning: the application "successfully" lost the card number. This is the classic trap of column-level encryption — always `OPEN SYMMETRIC KEY` (or check `sys.openkeys`) before encrypting, and consider a `NOT NULL` constraint on the ciphertext column so that the insert fails instead.

### Q1 — DECRYPTBYKEY without an open key returns NULL for every row

Per the documentation, `DECRYPTBYKEY` "returns NULL if the symmetric key used for data encryption isn't open or if ciphertext is NULL". The session has not opened `PayKey` (the `OPEN` in L2 belonged to an earlier batch/session state that was explicitly closed), so all four rows come back NULL — including rows 2–4 whose ciphertext is perfectly valid.

### Q2 — the key is open and the authenticator matches for rows 2 and 3

`OPEN SYMMETRIC KEY PayKey DECRYPTION BY CERTIFICATE PayCert` uses the certificate's private key (itself protected by the database master key, which the service master key opens automatically) to decrypt `PayKey` into the session. Rows 2 and 3 were encrypted with `add_authenticator = 1` and authenticator `'502'` / `'503'`; the same values are supplied now, so decryption succeeds. Row 4 was encrypted **without** an authenticator; decrypting it *with* one fails the integrity check → NULL, not an error.

### Q3 — the opposite mismatch

Now no authenticator is passed. Row 4 (encrypted without one) decrypts; rows 2 and 3 (encrypted with one) return NULL. The `add_authenticator` and `authenticator` arguments must match the values used at encryption time, in both directions.

### Q4 — you cannot search by re-encrypting the value

`ENCRYPTBYKEY` uses a random initialization vector, so encrypting the same plaintext twice produces **different** ciphertext (`Hits = 0` even though row 2 holds exactly that card with exactly that authenticator). Equality searches on engine-side encrypted columns require a separate deterministic token (e.g. an `HASHBYTES`/HMAC column), or Always Encrypted with deterministic encryption. Note also the storage size: AES-256 ciphertext of a 16-character string is 68 bytes, and 84 bytes with an authenticator — the `VARBINARY` column must be sized for that, not for the plaintext.

### Q5 — the authenticator defeats ciphertext substitution

Copying row 2's ciphertext into row 3 is a **whole-value substitution attack**: without authenticators the copied blob would decrypt fine under row 3 and Lena Voss would "own" Pau Roca's card. Because the ciphertext was bound to `'502'` and row 3 supplies `'503'`, `DECRYPTBYKEY` returns NULL for row 3. The authenticator is exactly the mechanism the documentation recommends "to prevent a malicious user from substituting values in an encrypted column". Row 2 still decrypts; row 4 still returns NULL under an authenticator.

### Q6 — DECRYPTBYKEYAUTOCERT opens the key for you

`DECRYPTBYKEYAUTOCERT(cert_id, cert_password, ciphertext, add_authenticator, authenticator)` decrypts the symmetric key with the certificate **and** decrypts the data in one call, so it works even though the session closed `PayKey` (`sys.openkeys` reports 0 open keys afterwards). `NULL` is passed as the certificate password because `PayCert`'s private key is protected by the database master key, not by a password. Results match Q5: row 2 decrypts, rows 3 and 4 fail their authenticator check, row 1 has no ciphertext.

### Equivalent alternatives and two side notes

- `CONVERT(VARCHAR(20), ...)` is needed because `DECRYPTBYKEY` returns `varbinary(8000)`; the cast must target the **original** type. Encrypt `N'4111'` (nvarchar) and cast the result to `varchar` and you get `'4'` — the first byte of the UTF-16 pair — while `CONVERT(NVARCHAR(20), ...)` returns `'4111'` (8 bytes of plaintext). Verified.
- `ENCRYPTBYPASSPHRASE`/`DECRYPTBYPASSPHRASE` skip the hierarchy entirely (the passphrase is the key) and share the NULL-on-failure behaviour; `ENCRYPTBYCERT`/`ENCRYPTBYASYMKEY` are for small payloads and are much slower — the documented advice is "for best performance, encrypt data using symmetric keys instead of certificates or asymmetric keys".
- `DECRYPTBYKEY` must run in the context of the database that holds the key: the same call from `master` against `CourierVault.Pay.CardOnFile` returns NULL.

## DP-800 Exam Rule to Remember

```text
Engine-side hierarchy (each layer protects the one below):
  Windows DPAPI -> Service Master Key -> Database Master Key (CREATE MASTER KEY ... PASSWORD)
                -> Certificate / asymmetric key -> Symmetric key (AES_256) -> data (varbinary)

ENCRYPTBYKEY / DECRYPTBYKEY NEVER throw for crypto failures - they return NULL when:
  * the symmetric key is not OPEN in this session (OPEN SYMMETRIC KEY ... DECRYPTION BY ...)
  * the authenticator / add_authenticator flag does not match the one used to encrypt
  * the ciphertext was tampered with or swapped
  * you are not in the database that owns the key
ENCRYPTBYKEY with a closed key -> NULL is INSERTED silently.

Same plaintext -> different ciphertext every time (random IV): no WHERE col = ENCRYPTBYKEY(...).
Authenticator (typically the row's primary key) = protection against ciphertext substitution.
DECRYPTBYKEYAUTOCERT / DECRYPTBYKEYAUTOASYMKEY = decrypt without a prior OPEN (cert opens the key).
Cast the varbinary result back to the ORIGINAL type (nvarchar vs varchar matters).
Keys/certs live in the database: DROP DATABASE removes them; the DBA who can OPEN the key sees
plaintext - use Always Encrypted when the DBA must not.
```
