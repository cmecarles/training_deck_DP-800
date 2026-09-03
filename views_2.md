# SQL Server question — Views 2

## Statement

A university stores its course-registration catalog in a SQL Server database named `CampusReg`. The registrar's office exposes the Computer Science part of the catalog to advisors through a view, and advisors modify data **only through that view**.

The complete setup is:

```sql
CREATE DATABASE CampusReg;
GO
USE CampusReg;
GO
CREATE SCHEMA Acad;
GO
CREATE TABLE Acad.Course
(
    CourseID INT          NOT NULL PRIMARY KEY,
    Dept     CHAR(4)      NOT NULL,
    Title    NVARCHAR(60) NOT NULL,
    Credits  TINYINT      NOT NULL,
    Seats    SMALLINT     NOT NULL CONSTRAINT DF_Course_Seats DEFAULT (30)
);
GO
INSERT INTO Acad.Course (CourseID, Dept, Title, Credits, Seats) VALUES
  (101, 'CS',   N'Programming I',  3, 120),
  (102, 'CS',   N'Databases',      4,  80),
  (205, 'MATH', N'Linear Algebra', 3,  60),
  (301, 'CS',   N'Compilers',      6,  40);
GO
CREATE VIEW Acad.CoreCatalog
WITH SCHEMABINDING
AS
SELECT CourseID, Dept, Title, Credits
FROM Acad.Course
WHERE Dept = 'CS' AND Credits <= 4
WITH CHECK OPTION;
GO
CREATE VIEW Acad.DeptStats
AS
SELECT Dept, COUNT(*) AS CourseCount
FROM Acad.Course
GROUP BY Dept;
GO

-- S1
INSERT INTO Acad.CoreCatalog (CourseID, Dept, Title, Credits)
VALUES (110, 'CS', N'Operating Systems', 4);
```

Each option below describes what would happen to S1 **if** the table had instead been defined with `CourseID INT IDENTITY(1,1) PRIMARY KEY`. Every option contains zero or more factual errors. **Choose the option with the highest count of wrong statements.**

### a.

If the table had been defined as `CourseID INT IDENTITY(1,1) PRIMARY KEY`, S1 would fail with error 544 because it is not possible to insert an explicit value for an identity column in table 'Course' when `IDENTITY_INSERT` is set to OFF. That applies through a view and a base table. Running `SET IDENTITY_INSERT Acad.Course ON` first on the base table would avoid the problem.

### b.

If the table had been defined as `CourseID INT IDENTITY(1,1) PRIMARY KEY`, S1 would fail with error 544 because it is not possible to insert an explicit value for an identity column in table 'Course' when `IDENTITY_INSERT` is set to OFF. That applies through a view and a base table. Leaving `CourseID` out of the column list and letting the engine generate it would avoid the problem.

### c.

If the table had been defined as `CourseID INT IDENTITY(1,1) PRIMARY KEY`, S1 would fail with error 544 because it is not possible to insert an explicit value for an identity column in table 'Course' when `IDENTITY_INSERT` is set to ON. That applies through a view and a base table. Running `SET IDENTITY_INSERT Acad.CoreCatalog ON` first on the view would avoid the problem.

### d.

If the table had been defined as `CourseID INT IDENTITY(1,1) PRIMARY KEY`, S1 would fail with error 544 because it is not possible to insert an explicit value for an identity column in table 'Course' when `IDENTITY_INSERT` is set to ON. That applies through a view and a base table. Leaving `CourseID` out of the column list and letting the engine generate it would avoid the problem.

## Correct Answer

**c** — it contains two wrong statements; every other option contains at most one.

## Explanation

The four options share the same skeleton and differ only in two places: the state of `IDENTITY_INSERT` named in the error explanation, and the proposed workaround. Count the errors in each.

### Why option c is correct (two errors)

- **Error 1:** "when `IDENTITY_INSERT` is set to **ON**". Error 544 (`Cannot insert explicit value for identity column in table 'Course' when IDENTITY_INSERT is set to OFF.`) is raised precisely because the setting is **OFF**, which is the default. Setting it ON is what allows explicit values.
- **Error 2:** "Running `SET IDENTITY_INSERT Acad.CoreCatalog ON` first on the view". `IDENTITY_INSERT` can only be set on a **table**. Against a view the statement itself fails with error 8105: `'Acad.CoreCatalog' is not a user table. Cannot perform SET operation.` The correct fix is `SET IDENTITY_INSERT Acad.Course ON` on the base table (and only one table per session may have it ON at a time).

### Why option a is wrong (zero errors)

Every statement is true: error 544 fires because `IDENTITY_INSERT` is OFF; the rule applies whether the insert goes through the view or directly against the table, because the view is just a pass-through to the base table for updatable inserts; and enabling `IDENTITY_INSERT` on the base table `Acad.Course` allows the explicit value 110.

### Why option b is wrong (zero errors)

Same first two sentences as option a, which are true. The workaround is also true: omitting `CourseID` from the column list lets the engine generate the identity value, which is the normal way to insert into an identity table, through a view or not.

### Why option d is wrong (one error)

It has the same "set to ON" mistake as option c (error 544 arises when the setting is **OFF**), but its workaround, omitting `CourseID` and letting the engine generate it, is correct. One error, fewer than option c.

Verified against SQL Server 2025 (RTM 17.0.1000.7); the error 544 and error 8105 messages above are the engine's literal output.

## DP-800 Exam Rule to Remember

- Error 544 means "explicit value for an identity column while `IDENTITY_INSERT` is **OFF**". The fix is either `SET IDENTITY_INSERT <table> ON` for the session, or omit the identity column from the insert list.
- `SET IDENTITY_INSERT` targets a **table**, never a view; against a view it fails with error 8105. When you insert through an updatable view, enable the setting on the underlying base table.
- Only one table per session can have `IDENTITY_INSERT` ON at a time; turn it OFF when done.
