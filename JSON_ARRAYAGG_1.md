# SQL Server question — JSON_ARRAYAGG 1

## Statement

A bistro exposes its menu through an API that returns, per category, several JSON arrays built with the `JSON_ARRAYAGG` aggregate: the dish names from most to least expensive, the allergen labels, and an array of dish "cards" (one JSON object per dish).

The following script is run on SQL Server 2025 and completes without errors:

```sql
CREATE DATABASE BistroMenu;
GO
ALTER DATABASE BistroMenu SET COMPATIBILITY_LEVEL = 170;
GO
USE BistroMenu;
GO
CREATE SCHEMA Menu;
GO
CREATE TABLE Menu.Categories (
    CategoryId   INT          PRIMARY KEY,
    CategoryName NVARCHAR(30) NOT NULL
);
GO
CREATE TABLE Menu.Dishes (
    DishId     INT           PRIMARY KEY,
    CategoryId INT           NOT NULL REFERENCES Menu.Categories(CategoryId),
    DishName   NVARCHAR(40)  NOT NULL,
    Price      DECIMAL(6,2)  NOT NULL,
    Allergen   NVARCHAR(20)  NULL
);
GO
INSERT INTO Menu.Categories (CategoryId, CategoryName) VALUES
  (1, N'Desserts'),
  (2, N'Mains'),
  (3, N'Specials'),
  (4, N'Starters');
GO
INSERT INTO Menu.Dishes (DishId, CategoryId, DishName, Price, Allergen) VALUES
  (1, 1, N'Chocolate Pot', 6.50, N'dairy'),
  (2, 1, N'Lemon Tart',    6.00, N'gluten'),
  (3, 2, N'Coq au Vin',   19.90, N'sulphites'),
  (4, 2, N'Duck Confit',  22.50, NULL),
  (5, 2, N'Ratatouille',  16.00, NULL),
  (6, 4, N'Baked Brie',    9.50, N'dairy'),
  (7, 4, N'Olive Trio',    6.00, NULL),
  (8, 4, N'Soup du Jour',  7.25, N'gluten');
GO
```

Note that category 3, `Specials`, has no dishes. Then this query is executed:

```sql
SELECT c.CategoryName,
       JSON_ARRAYAGG(d.DishName ORDER BY d.Price DESC, d.DishName)               AS DishesByPrice,
       JSON_ARRAYAGG(d.Allergen ORDER BY d.Price DESC, d.DishName)               AS Allergens,
       JSON_ARRAYAGG(d.Allergen ORDER BY d.Price DESC, d.DishName NULL ON NULL)  AS AllergensAll,
       JSON_ARRAYAGG(JSON_OBJECT('dish':     d.DishName,
                                 'eur':      d.Price,
                                 'allergen': d.Allergen)
                     ORDER BY d.Price DESC, d.DishName)                          AS Cards
FROM Menu.Categories AS c
LEFT JOIN Menu.Dishes AS d
       ON d.CategoryId = c.CategoryId
GROUP BY c.CategoryId, c.CategoryName
ORDER BY c.CategoryName;
```

Write the **exact** value of `DishesByPrice`, `Allergens`, `AllergensAll`, and `Cards` for each of the four result rows — every character, including array lengths, `null` elements, and element order. Pay particular attention to the `Specials` row.

## Correct Answer

Four rows are returned, ordered by `CategoryName`. All JSON strings below are copied verbatim from the engine.

**Row 1 — `Desserts`**

```json
DishesByPrice:  ["Chocolate Pot","Lemon Tart"]
Allergens:      ["dairy","gluten"]
AllergensAll:   ["dairy","gluten"]
Cards:          [{"dish":"Chocolate Pot","eur":6.50,"allergen":"dairy"},{"dish":"Lemon Tart","eur":6.00,"allergen":"gluten"}]
```

**Row 2 — `Mains`**

```json
DishesByPrice:  ["Duck Confit","Coq au Vin","Ratatouille"]
Allergens:      ["sulphites"]
AllergensAll:   [null,"sulphites",null]
Cards:          [{"dish":"Duck Confit","eur":22.50,"allergen":null},{"dish":"Coq au Vin","eur":19.90,"allergen":"sulphites"},{"dish":"Ratatouille","eur":16.00,"allergen":null}]
```

**Row 3 — `Specials`**

```json
DishesByPrice:  []
Allergens:      []
AllergensAll:   [null]
Cards:          [{"dish":null,"eur":null,"allergen":null}]
```

**Row 4 — `Starters`**

```json
DishesByPrice:  ["Baked Brie","Soup du Jour","Olive Trio"]
Allergens:      ["dairy","gluten"]
AllergensAll:   ["dairy","gluten",null]
Cards:          [{"dish":"Baked Brie","eur":9.50,"allergen":"dairy"},{"dish":"Soup du Jour","eur":7.25,"allergen":"gluten"},{"dish":"Olive Trio","eur":6.00,"allergen":null}]
```

## Explanation

### 1. `ORDER BY` inside the aggregate orders the array elements

`JSON_ARRAYAGG (value_expression [ order_by_clause ] [ json_null_clause ])` accepts an `ORDER BY` *inside* the parentheses; it orders the input rows of the aggregate and therefore the elements of the produced array. Here `ORDER BY d.Price DESC, d.DishName` sorts each category's dishes from most to least expensive — visibly different from insertion order: `Starters` comes out `["Baked Brie","Soup du Jour","Olive Trio"]` (9.50, 7.25, 6.00), not in the alphabetical order the rows were inserted. The outer query's `ORDER BY c.CategoryName` orders the *rows*; without both `ORDER BY`s the output would be nondeterministic.

(Engine note, verified on SQL Server 2025: all ordered aggregates in the same `SELECT` scope must use mutually compatible orderings, otherwise the query fails with error 8711 — *"Multiple ordered aggregate functions in the same scope have mutually incompatible orderings."* That is why all four aggregates here share `ORDER BY d.Price DESC, d.DishName`; the same restriction applies to `STRING_AGG ... WITHIN GROUP`.)

### 2. The default is `ABSENT ON NULL` — arrays silently shrink

Per the Microsoft Learn reference for `JSON_ARRAYAGG` (Transact-SQL): *"If omitted, `ABSENT ON NULL` is default."*

`Allergens` has no `json_null_clause`, so every SQL `NULL` input row contributes **nothing**:

- `Mains` has 3 dishes but only Coq au Vin has an allergen → `["sulphites"]`, length 1.
- `Starters` has 3 dishes, one `NULL` → `["dairy","gluten"]`, length 2.

The array length no longer matches the group's row count, and the surviving elements can no longer be correlated positionally with `DishesByPrice`.

`AllergensAll` adds `NULL ON NULL` (written after the `ORDER BY`, before the closing parenthesis), so every row contributes an element and positions line up with `DishesByPrice`: `Mains` → `[null,"sulphites",null]` — Duck Confit (22.50, null), Coq au Vin (19.90, sulphites), Ratatouille (16.00, null).

### 3. The `Specials` row: what an empty group produces

`Specials` has no dishes; the `LEFT JOIN` keeps the category as a single all-`NULL` placeholder row. In that group:

- `DishesByPrice` and `Allergens` (default `ABSENT ON NULL`): the only input is `NULL`, it is absented, and the aggregate returns an **empty array** `[]` — not SQL `NULL` (contrast with `SUM`/`MAX`, which return `NULL` for an all-`NULL` group).
- `AllergensAll` (`NULL ON NULL`): the placeholder's `NULL` becomes an element → `[null]`, an array of length **1** for a category with **0** dishes.
- `Cards`: the trap. The argument is `JSON_OBJECT(...)`, and a `JSON_OBJECT` call **never returns SQL `NULL`** — for the placeholder row it returns the string `{"dish":null,"eur":null,"allergen":null}` (keys kept, because `JSON_OBJECT`'s own default is `NULL ON NULL`). Since the value fed to `JSON_ARRAYAGG` is not `NULL`, `ABSENT ON NULL` has nothing to absent, and the "empty" category yields a **one-element array containing a ghost card**: `[{"dish":null,"eur":null,"allergen":null}]`. Filtering with an inner join, or a `HAVING COUNT(d.DishId) > 0`, or building cards only `WHERE d.DishId IS NOT NULL`, would be needed to avoid it.

### 4. Character-level rendering inside `Cards`

- `JSON_OBJECT` keeps keys in call order: `dish`, `eur`, `allergen`.
- `decimal(6,2)` renders unquoted with its exact scale: `6.50`, `22.50`, `7.25` — the trailing zero of `6.50` is preserved, and no quotes are added.
- `NULL` allergens inside the object render as `"allergen":null` (`JSON_OBJECT` default `NULL ON NULL`) — so `Cards` never loses a dish and never loses a key, while the bare `Allergens` aggregate loses elements. Same data, three different shapes, purely from null-handling defaults.
- No whitespace is emitted anywhere; objects are concatenated with a single comma.

### Equivalent alternatives

- Spelling out the default — `JSON_ARRAYAGG(d.Allergen ORDER BY d.Price DESC, d.DishName ABSENT ON NULL)` — is exactly equivalent to `Allergens` as written.
- `Cards` could equivalently be produced per category with a correlated `(SELECT ... FOR JSON PATH, INCLUDE_NULL_VALUES)` subquery, but note `FOR JSON PATH` *without* `INCLUDE_NULL_VALUES` drops the null-valued **keys** (`{"dish":"Duck Confit","eur":22.50}`), which `JSON_OBJECT` does not.
- Adding `RETURNING json` to any of the aggregates returns the native `json` type instead of `nvarchar(max)`; the text is unchanged.

## DP-800 Exam Rule to Remember

`JSON_ARRAYAGG` defaults to **`ABSENT ON NULL`**: `NULL` input rows vanish, so the array length can differ from `COUNT(*)` of the group, and an all-`NULL` (or empty-after-`LEFT JOIN`) group returns `[]`, never SQL `NULL`.

The clause order inside the aggregate is fixed:

```text
JSON_ARRAYAGG ( value  [ ORDER BY ... ]  [ NULL ON NULL | ABSENT ON NULL ]  [ RETURNING json ] )
```

Three traps stacked in this question:

1. **Determinism** — element order needs `ORDER BY` *inside* the aggregate; row order needs `ORDER BY` *outside*. All ordered aggregates in one scope must share compatible orderings (error 8711).
2. **Shrinking arrays** — default `ABSENT ON NULL` breaks positional correlation between two aggregated arrays; use `NULL ON NULL` to keep positions.
3. **The ghost element** — wrapping rows in `JSON_OBJECT(...)` defeats `ABSENT ON NULL`, because the object is never SQL `NULL`: a `LEFT JOIN` placeholder row becomes `[{"k":null,...}]`, not `[]`.
