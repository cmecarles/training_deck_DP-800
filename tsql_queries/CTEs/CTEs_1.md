# SQL Server question — CTEs 1

## Statement

A mountain-rescue organization tracks its incident command structure in a SQL Server database named `OrgAtlas`. Members are identified by call sign; the chain of command is stored as edges in `Org.CommandLink` (`LeaderID` directly commands `MemberID`).

```sql
CREATE DATABASE OrgAtlas;
GO
USE OrgAtlas;
GO
CREATE SCHEMA Org;
GO
CREATE TABLE Org.Member
(
    MemberID INT         NOT NULL PRIMARY KEY,
    CallSign VARCHAR(20) NOT NULL UNIQUE,
    Duty     VARCHAR(30) NOT NULL
);
GO
CREATE TABLE Org.CommandLink
(
    LeaderID INT NOT NULL REFERENCES Org.Member(MemberID),
    MemberID INT NOT NULL REFERENCES Org.Member(MemberID),
    CONSTRAINT PK_CommandLink PRIMARY KEY (LeaderID, MemberID)
);
GO
INSERT INTO Org.Member (MemberID, CallSign, Duty) VALUES
  (1, 'Cortina', 'Incident Commander'),
  (2, 'Sella',   'Operations Lead'),
  (3, 'Brenta',  'Medical Officer'),
  (4, 'Marmol',  'Field Team Lead'),
  (5, 'Pordoi',  'Drone Pilot'),
  (6, 'Falzar',  'Base Medic'),
  (7, 'Tofana',  'Rope Specialist'),
  (8, 'Civetta', 'Field Medic');
GO
INSERT INTO Org.CommandLink (LeaderID, MemberID) VALUES
  (1, 2),
  (1, 3),
  (2, 4),
  (2, 5),
  (3, 6),
  (3, 8),
  (4, 7),
  (4, 8);
GO
```

Note that the field medic **Civetta (8)** appears on two edges: she reports both to the medical officer Brenta (3) and to the field team lead Marmol (4).

The duty officer runs the following query:

```sql
WITH Chain AS
(
    SELECT m.MemberID,
           m.CallSign,
           0 AS Lvl,
           CAST('/' + m.CallSign AS VARCHAR(200)) AS Path
    FROM Org.Member AS m
    WHERE m.MemberID = 1

    UNION ALL

    SELECT m.MemberID,
           m.CallSign,
           c.Lvl + 1,
           CAST(c.Path + '/' + m.CallSign AS VARCHAR(200))
    FROM Chain AS c
    JOIN Org.CommandLink AS l ON l.LeaderID = c.MemberID
    JOIN Org.Member      AS m ON m.MemberID = l.MemberID
)
SELECT Lvl, MemberID, CallSign, Path
FROM Chain
ORDER BY Path
OPTION (MAXRECURSION 3);
```

What does the query return?

### a.

The statement fails with:

```text
Msg 530, Level 16, State 1
The statement terminated. The maximum recursion 3 has been exhausted before statement completion.
```

because the anchor member counts as the first recursion level, so a hierarchy whose deepest member sits three joins below the commander needs `MAXRECURSION 4`.

### b.

Nine rows:

| Lvl | MemberID | CallSign | Path |
|-----|----------|----------|------|
| 0 | 1 | Cortina | /Cortina |
| 1 | 3 | Brenta | /Cortina/Brenta |
| 2 | 8 | Civetta | /Cortina/Brenta/Civetta |
| 2 | 6 | Falzar | /Cortina/Brenta/Falzar |
| 1 | 2 | Sella | /Cortina/Sella |
| 2 | 4 | Marmol | /Cortina/Sella/Marmol |
| 3 | 8 | Civetta | /Cortina/Sella/Marmol/Civetta |
| 3 | 7 | Tofana | /Cortina/Sella/Marmol/Tofana |
| 2 | 5 | Pordoi | /Cortina/Sella/Pordoi |

### c.

Eight rows — as in option b, but **without** the row `(3, 8, Civetta, /Cortina/Sella/Marmol/Civetta)`, because member 8 has already been produced at level 2 and a recursive CTE emits each row only once.

### d.

Seven rows — the rows of option b with `Lvl` 0, 1 and 2 only. `MAXRECURSION 3` means the recursion is silently stopped after producing level 2, so the two `Lvl = 3` rows are omitted but no error is raised.

## Correct Answer

**b**

## Explanation

The query and both `MAXRECURSION` variants were executed on SQL Server 2025 (RTM); the row set in option b and the error text in option a are the engine's literal output (the error is real — it just occurs at `MAXRECURSION 2`, not 3).

### How the recursion actually unfolds

- **Anchor** (recursion count 0): the commander — `(0, 1, Cortina, /Cortina)`.
- **Recursive execution 1**: joins level-0 rows to their outgoing edges → Sella and Brenta (`Lvl = 1`).
- **Recursive execution 2**: from Sella → Marmol, Pordoi; from Brenta → Falzar and **Civetta** (`Lvl = 2`).
- **Recursive execution 3**: from Marmol → Tofana and **Civetta again** (`Lvl = 3`). Pordoi, Falzar and Civetta-as-level-2 lead nowhere.
- **Recursive execution 4**: the level-3 rows (Tofana, Civetta) have no outgoing edges, so this execution produces **zero rows** and the recursion terminates normally.

Nine rows total, sorted by `Path` (which is unique even for the duplicated member, because the two Civetta rows carry different paths). `ORDER BY Path` makes the displayed order deterministic: the `/Cortina/Brenta...` subtree sorts before `/Cortina/Sella...`.

### The twist 1 — a member that joins twice

The recursive member is a plain join between the previous level and the edge table, combined with **`UNION ALL`**. `UNION ALL` performs no duplicate elimination, and the recursion has no memory of which `MemberID`s it has already visited — it only feeds *the rows produced by the previous execution* into the next one. Civetta is reachable through two different edges at two different depths, so she legitimately appears twice, with different `Lvl` and `Path` values. (This is also why a *cyclic* graph makes a recursive CTE spin forever: nothing stops a row from being reached again. `MAXRECURSION` is the safety net, and plain `UNION` is not even permitted between the anchor and the recursive member — the engine requires `UNION ALL`.)

### The twist 2 — the MAXRECURSION boundary

`MAXRECURSION n` caps the number of **recursive-member executions**, and the anchor does not count. This hierarchy needs three productive recursive executions (levels 1, 2, 3); the fourth execution finds no matching edges, returns an empty set, and ends the recursion **without tripping the limit**. So `OPTION (MAXRECURSION 3)` is exactly enough and the statement succeeds.

Rerunning the identical query with `OPTION (MAXRECURSION 2)` does fail, with the engine's message quoted in option a:

```text
Msg 530, Level 16, State 1
The statement terminated. The maximum recursion 3 has been exhausted before statement completion.
```

(with `3` replaced by `2`). Per the Microsoft documentation for query hints, `MAXRECURSION` accepts 0 to 32,767, `0` means no limit, the server default is 100, and when the limit is reached **the query ends with an error and the statement's effects are rolled back** — a `SELECT` may return partial or no results, but it never succeeds silently.

### Why option a is wrong

It miscounts what `MAXRECURSION` limits. The anchor is not a recursion; only executions of the recursive member count. Three productive recursive executions reach `Lvl = 3`, and the final empty execution does not count against the limit. The quoted error is genuine engine output — but for `MAXRECURSION 2`, not 3.

### Why option c is wrong (the subtle one)

It assumes the CTE deduplicates members. It does not: the required set operator is `UNION ALL`, and cycle/duplicate detection is entirely the query author's problem. Because Civetta is reachable via Brenta (depth 2) and via Marmol (depth 3), she is emitted twice with two distinct paths. If a single row per member were wanted, the query would need explicit logic (e.g., keep the minimum `Lvl` per `MemberID` in an outer `SELECT`).

### Why option d is wrong

It describes `MAXRECURSION` as a silent truncation. Exceeding the limit is an **error** (530), never a quiet stop — and in this query the limit is not even reached. There is no setting that makes a recursive CTE silently stop at a chosen depth; to truncate cleanly you add a filter such as `WHERE c.Lvl < 3` (or `c.Lvl + 1 <= 2`) to the recursive member's join, which ends the recursion by producing an empty set.

### Equivalent alternatives

- `ORDER BY Path` could be replaced by `ORDER BY Path COLLATE Latin1_General_BIN` or by `ORDER BY Lvl, Path` only if the intended order changed; as written, `Path` alone is a deterministic total order because paths are unique.
- The two `CAST(... AS VARCHAR(200))` calls are required as written: anchor and recursive member must produce **identical types**, and without the casts the concatenated types/lengths differ between the two branches and the statement fails with `Msg 240 — Types don't match between the anchor and the recursive part in column "Path" of recursive query "Chain".`

Verified against SQL Server 2025 (RTM 17.0.1000.7); every message above is the engine's literal output.

## DP-800 Exam Rule to Remember

For a recursive CTE:

```text
anchor member            runs once            (recursion count: not counted)
UNION ALL                mandatory, no dedup  (duplicates and cycles are YOUR problem)
recursive member         runs repeatedly, each pass consuming
                         only the previous pass's rows
termination              a pass that returns 0 rows (does not count against the limit)
```

`OPTION (MAXRECURSION n)`:

- counts **productive recursive executions only** — a tree whose deepest node is n joins below the anchor succeeds with `MAXRECURSION n`;
- range 0–32,767, `0` = unlimited, default 100;
- exceeding it raises **error 530 and terminates/rolls back the statement** — it never silently truncates.

And remember the shape of a graph traversal: a node reachable by two paths appears **twice**, with two levels and two paths — deduplicate explicitly if the business question is "which members", not "which chains of command".
