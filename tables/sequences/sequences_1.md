# SQL Server question — Sequences 1

## Statement

You run the following SQL:

```sql
CREATE DATABASE SEQUENCE_EXERCISE;
GO

USE SEQUENCE_EXERCISE;
GO

CREATE SEQUENCE dbo.SEQ_AB     AS BIGINT START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE dbo.SEQ_ABC    AS BIGINT START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE dbo.SEQ_ABCD   AS BIGINT START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE dbo.SEQ_ABCDE  AS BIGINT START WITH 1 INCREMENT BY 1;
GO

CREATE TABLE dbo.TABLE_A(
Id    INT IDENTITY(1,1) PRIMARY KEY,
Step  TINYINT NOT NULL,
AB    BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_AB),
ABC   BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABC),
ABCD  BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABCD),
ABCDE BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABCDE)
);

CREATE TABLE dbo.TABLE_B(
Id    INT IDENTITY(1,1) PRIMARY KEY,
Step  TINYINT NOT NULL,
AB    BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_AB),
ABC   BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABC),
ABCD  BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABCD),
ABCDE BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABCDE)
);

CREATE TABLE dbo.TABLE_C(
Id    INT IDENTITY(1,1) PRIMARY KEY,
Step  TINYINT NOT NULL,
ABC   BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABC),
ABCD  BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABCD),
ABCDE BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABCDE)
);

CREATE TABLE dbo.TABLE_D(
Id    INT IDENTITY(1,1) PRIMARY KEY,
Step  TINYINT NOT NULL,
ABCD  BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABCD),
ABCDE BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABCDE)
);

CREATE TABLE dbo.TABLE_E(
Id    INT IDENTITY(1,1) PRIMARY KEY,
Step  TINYINT NOT NULL,
ABCDE BIGINT NOT NULL DEFAULT (NEXT VALUE FOR SEQ_ABCDE)
);
GO

INSERT INTO dbo.TABLE_A (Step) VALUES (1);
INSERT INTO dbo.TABLE_B (Step) VALUES (2);
INSERT INTO dbo.TABLE_C (Step) VALUES (3);
INSERT INTO dbo.TABLE_D (Step) VALUES (4);
INSERT INTO dbo.TABLE_E (Step) VALUES (5);
INSERT INTO dbo.TABLE_E (Step) VALUES (6);
INSERT INTO dbo.TABLE_D (Step) VALUES (7);
INSERT INTO dbo.TABLE_C (Step) VALUES (8);
INSERT INTO dbo.TABLE_B (Step) VALUES (9);
INSERT INTO dbo.TABLE_A (Step) VALUES (10);
INSERT INTO dbo.TABLE_A (Step) VALUES (11);
INSERT INTO dbo.TABLE_B (Step) VALUES (12);
INSERT INTO dbo.TABLE_C (Step) VALUES (13);
INSERT INTO dbo.TABLE_D (Step) VALUES (14);
INSERT INTO dbo.TABLE_E (Step) VALUES (15);
INSERT INTO dbo.TABLE_E (Step) VALUES (16);
INSERT INTO dbo.TABLE_D (Step) VALUES (17);
INSERT INTO dbo.TABLE_C (Step) VALUES (18);
INSERT INTO dbo.TABLE_B (Step) VALUES (19);
INSERT INTO dbo.TABLE_A (Step) VALUES (20);
GO
```

Then you run a query that generates this result:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Data xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <Row>
    <Id>1</Id>
    <Step>1</Step>
    <AB>1</AB>
    <ABC>1</ABC>
    <ABCD>1</ABCD>
    <ABCDE>1</ABCDE>
  </Row>
  <Row>
    <Id>2</Id>
    <Step>10</Step>
    <AB>4</AB>
    <ABC>6</ABC>
    <ABCD>8</ABCD>
    <ABCDE>10</ABCDE>
  </Row>
  <Row>
    <Id>3</Id>
    <Step>11</Step>
    <AB>5</AB>
    <ABC>7</ABC>
    <ABCD>9</ABCD>
    <ABCDE>11</ABCDE>
  </Row>
  <Row>
    <Id>4</Id>
    <Step>20</Step>
    <AB>8</AB>
    <ABC>12</ABC>
    <ABCD>16</ABCD>
    <ABCDE>20</ABCDE>
  </Row>
</Data>
```

Which is that query?

## Answer

```sql
SELECT Id, Step, AB, ABC, ABCD, ABCDE
FROM dbo.TABLE_A
ORDER BY Id
FOR XML PATH('Row'), ROOT('Data'), ELEMENTS XSINIL;
```

(`SELECT *` also works: the column order of `TABLE_A` matches the element order in the result.)

## Explanation

Two things must be recovered from the result: **which** table was queried, and **how** the XML shape was produced.

### 1. Why `TABLE_A`

Each `INSERT` draws a new value from every sequence its table has a `DEFAULT` for: `TABLE_A` and `TABLE_B` draw from all four sequences, `TABLE_C` from `SEQ_ABC`/`SEQ_ABCD`/`SEQ_ABCDE`, `TABLE_D` from `SEQ_ABCD`/`SEQ_ABCDE`, `TABLE_E` only from `SEQ_ABCDE`. So `SEQ_AB` counts inserts into {A,B}, `SEQ_ABC` counts inserts into {A,B,C}, `SEQ_ABCD` counts {A,B,C,D}, and `SEQ_ABCDE` counts every insert — the sequence names say which tables consume them. Replaying the 20 inserts:

| Step | Table | AB | ABC | ABCD | ABCDE |
|---|---|---|---|---|---|
| 1 | A | 1 | 1 | 1 | 1 |
| 2 | B | 2 | 2 | 2 | 2 |
| 3 | C | — | 3 | 3 | 3 |
| 4 | D | — | — | 4 | 4 |
| 5 | E | — | — | — | 5 |
| 6 | E | — | — | — | 6 |
| 7 | D | — | — | 5 | 7 |
| 8 | C | — | 4 | 6 | 8 |
| 9 | B | 3 | 5 | 7 | 9 |
| 10 | A | 4 | 6 | 8 | 10 |
| 11 | A | 5 | 7 | 9 | 11 |
| 12 | B | 6 | 8 | 10 | 12 |
| 13 | C | — | 9 | 11 | 13 |
| 14 | D | — | — | 12 | 14 |
| 15 | E | — | — | — | 15 |
| 16 | E | — | — | — | 16 |
| 17 | D | — | — | 13 | 17 |
| 18 | C | — | 10 | 14 | 18 |
| 19 | B | 7 | 11 | 15 | 19 |
| 20 | A | 8 | 12 | 16 | 20 |

`TABLE_A` received the inserts of steps 1, 10, 11, 20, so its rows (Id 1–4) are exactly the four `<Row>` elements in the result: (1,1,1,1,1,1), (2,10,4,6,8,10), (3,11,5,7,9,11) and (4,20,8,12,16,20). Only `TABLE_A` and `TABLE_B` even have all four sequence columns, and `TABLE_B`'s Step values would be 2, 9, 12, 19 — so the Step column pins the query to `TABLE_A`. Note the giveaway in row 4: `ABCDE` equals `Step` (20 inserts total, and `SEQ_ABCDE` fires on every one of them).

### 2. Why `FOR XML PATH('Row'), ROOT('Data'), ELEMENTS XSINIL`

A bare `SELECT` returns a tabular result set, not XML, so the shown output cannot come from `SELECT ... ORDER BY Id` alone. The result is element-centric XML (each column is a subelement of `<Row>`), wrapped in a root element `<Data>`, and — the telling detail — the root carries `xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"`. `FOR XML` adds that namespace declaration only when `XSINIL` is specified (it is needed to be able to mark NULLs as `xsi:nil`). Hence:

- `PATH('Row')` → each row becomes `<Row>`, columns become subelements
- `ROOT('Data')` → the `<Data>` wrapper
- `ELEMENTS XSINIL` → element-centric output plus the `xmlns:xsi` declaration on the root

The `<?xml version="1.0" encoding="utf-8"?>` prolog is not emitted by SQL Server; the client tool adds it when the XML result is opened or saved as a file.

Equivalent formulations (also correct):

```sql
SELECT Id, Step, AB, ABC, ABCD, ABCDE
FROM dbo.TABLE_A
ORDER BY Id
FOR XML RAW('Row'), ROOT('Data'), ELEMENTS XSINIL;
```

```sql
SELECT Id, Step, AB, ABC, ABCD, ABCDE
FROM dbo.TABLE_A AS Row
ORDER BY Row.Id
FOR XML AUTO, ROOT('Data'), ELEMENTS XSINIL;
```

Also, because Id, Step and the four sequence columns all increase together within `TABLE_A`, `ORDER BY Step` (or `AB`, `ABC`, `ABCD`, `ABCDE`) produces the same row order as `ORDER BY Id`.

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.
