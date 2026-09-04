# Instructor-Examiner guide — JSON_ARRAYAGG 1

Companion to [JSON_ARRAYAGG_1.md](JSON_ARRAYAGG_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is an exact-output question with four result rows and four JSON columns per row, sixteen values in all. Take it one row at a time, and within a row one column at a time. The learner answers by speaking JSON, so accept spoken forms such as "open bracket, quote Duck Confit quote, comma, ..." and "an empty array". Be strict about element count, element order, and whether a null appears. Save the Specials row for last; it holds the main trap.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Write advanced T-SQL code.
- Task bullet: Work with JSON data using the native JSON functions.
- What is tested: how `JSON_ARRAYAGG` orders its elements, what its default `ABSENT ON NULL` does to array length, how `NULL ON NULL` differs, and why wrapping each row in `JSON_OBJECT` produces a "ghost" element for an empty group.

## 2. Scenario to read aloud

**Piece 1, the story.** "A bistro publishes its menu through an API. For each menu category the API returns several JSON arrays. One array holds the dish names, from most to least expensive. Another holds the allergen labels. And another holds one JSON object per dish, which the bistro calls a card. All of these arrays are built in SQL Server 2025 with the aggregate function JSON underscore ARRAYAGG."

**Piece 2, the tables.** "The database is called BistroMenu, and it runs at compatibility level one hundred seventy. There is a schema called Menu with two tables. Menu dot Categories has CategoryId, an integer primary key, and CategoryName, text up to thirty characters. Menu dot Dishes has DishId, an integer primary key; CategoryId, a foreign key to Categories; DishName, text up to forty characters; Price, a decimal with six digits and two decimals; and Allergen, text up to twenty characters, which allows null."

**Piece 3, the categories.** "Four categories are inserted. One, Desserts. Two, Mains. Three, Specials. Four, Starters. Note that category three, Specials, will have no dishes at all."

**Piece 4, the dishes.** "Eight dishes are inserted, in this order. Dish 1, category 1 Desserts, Chocolate Pot, six fifty, allergen dairy. Dish 2, Desserts, Lemon Tart, six euros exactly, allergen gluten. Dish 3, category 2 Mains, Coq au Vin, nineteen ninety, allergen sulphites. Dish 4, Mains, Duck Confit, twenty-two fifty, allergen null. Dish 5, Mains, Ratatouille, sixteen euros, allergen null. Dish 6, category 4 Starters, Baked Brie, nine fifty, allergen dairy. Dish 7, Starters, Olive Trio, six euros, allergen null. Dish 8, Starters, Soup du Jour, seven twenty-five, allergen gluten."

**Piece 5, the query, shape.** "One query is run. It selects from Categories, with a LEFT JOIN to Dishes on CategoryId. It groups by CategoryId and CategoryName, and the outer ORDER BY is on CategoryName. The select list has CategoryName followed by four aggregate columns. Every aggregate contains the same inner ORDER BY: Price descending, then DishName."

**Piece 6, the four aggregate columns.** "Column DishesByPrice is JSON ARRAYAGG of DishName, with that inner ORDER BY and nothing else. Column Allergens is JSON ARRAYAGG of Allergen, with the inner ORDER BY and nothing else. Column AllergensAll is JSON ARRAYAGG of Allergen, with the inner ORDER BY, followed by the clause NULL ON NULL. Column Cards is JSON ARRAYAGG of a JSON underscore OBJECT call, with the inner ORDER BY. That JSON OBJECT has three keys in this order: dish, taken from DishName; eur, taken from Price; and allergen, taken from Allergen."

**Piece 7, what is asked.** "You will be asked for the exact text of the four JSON columns in each of the four result rows. Every character matters: how many elements, in which order, and whether the word null appears. I can read any line of the script or the query on request."

## 3. Setup script (reference only; do not read verbatim unless asked)

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

## 4. The question (ask exactly this)

"Write the exact value of DishesByPrice, Allergens, AllergensAll, and Cards for each of the four result rows. Every character, including array lengths, null elements, and element order. Pay particular attention to the Specials row. Let's go one row at a time. Start with the Desserts row: first DishesByPrice, then Allergens, then AllergensAll, then Cards."

Then do the same for Mains, then Starters, and finally Specials. One column at a time.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Row 1, Desserts**

```json
DishesByPrice:  ["Chocolate Pot","Lemon Tart"]
Allergens:      ["dairy","gluten"]
AllergensAll:   ["dairy","gluten"]
Cards:          [{"dish":"Chocolate Pot","eur":6.50,"allergen":"dairy"},{"dish":"Lemon Tart","eur":6.00,"allergen":"gluten"}]
```

**Row 2, Mains**

```json
DishesByPrice:  ["Duck Confit","Coq au Vin","Ratatouille"]
Allergens:      ["sulphites"]
AllergensAll:   [null,"sulphites",null]
Cards:          [{"dish":"Duck Confit","eur":22.50,"allergen":null},{"dish":"Coq au Vin","eur":19.90,"allergen":"sulphites"},{"dish":"Ratatouille","eur":16.00,"allergen":null}]
```

**Row 3, Specials**

```json
DishesByPrice:  []
Allergens:      []
AllergensAll:   [null]
Cards:          [{"dish":null,"eur":null,"allergen":null}]
```

**Row 4, Starters**

```json
DishesByPrice:  ["Baked Brie","Soup du Jour","Olive Trio"]
Allergens:      ["dairy","gluten"]
AllergensAll:   ["dairy","gluten",null]
Cards:          [{"dish":"Baked Brie","eur":9.50,"allergen":"dairy"},{"dish":"Soup du Jour","eur":7.25,"allergen":"gluten"},{"dish":"Olive Trio","eur":6.00,"allergen":null}]
```

Key points to check in the learner's answer:

- Element order is by Price descending, then DishName. Starters is Baked Brie, Soup du Jour, Olive Trio, not alphabetical.
- Allergens drops every null, so Mains has one element and Starters has two.
- AllergensAll keeps nulls in position: Mains is null, sulphites, null.
- Prices are unquoted and keep two decimals: 6.50, 6.00, 22.50.
- Specials: two empty arrays, then an array with a single null, then an array with a single object whose three values are all null. Not SQL NULL anywhere.
- No spaces anywhere in the JSON text.

## 6. Hint ladder (one hint per attempt, in order)

**DishesByPrice, any row**
1. "There is an ORDER BY inside the aggregate parentheses. What does it order?"
2. "The inner ORDER BY is Price descending, then DishName. Sort the dishes of this category by price, highest first."
3. "For Starters the prices are nine fifty, seven twenty-five and six. Which name goes first?"

**Allergens, Mains and Starters**
1. "This aggregate has no null clause at all. Which null behaviour does JSON ARRAYAGG use by default?"
2. "The default is ABSENT ON NULL. What happens to a row whose Allergen is null?"
3. "Count only the dishes that have an allergen. How many elements survive in Mains? And in Starters?"

**AllergensAll, Mains and Starters**
1. "This one says NULL ON NULL. How is that different from the default?"
2. "Every row contributes an element now. Keep the price order and put the word null where the allergen is missing."
3. "Mains in price order is Duck Confit, Coq au Vin, Ratatouille. Which of those has an allergen?"

**Cards, any non-empty row**
1. "Each element is a JSON OBJECT with keys dish, eur and allergen, in that order. How does a decimal price render in JSON: quoted or bare?"
2. "Price is decimal six comma two, so the scale is kept. Six fifty renders as six point five zero. Is the trailing zero kept?"
3. "When Allergen is null, JSON OBJECT still writes the key. What value does it write?"

**Specials, DishesByPrice and Allergens**
1. "Specials has no dishes, but the LEFT JOIN still keeps the category. What does the single joined row look like?"
2. "The only input to the aggregate is a null. With ABSENT ON NULL, what remains?"
3. "Nothing remains. Does JSON ARRAYAGG return SQL NULL like SUM would, or an empty array?"

**Specials, AllergensAll**
1. "Same placeholder row, but this time NULL ON NULL. Does the null become an element?"
2. "So the array has one element. What is that element?"

**Specials, Cards**
1. "This is the trap. Before the aggregate sees anything, JSON OBJECT runs on the placeholder row. Does JSON OBJECT ever return SQL NULL?"
2. "JSON OBJECT returns a string, an object with three keys whose values are null. Is that string itself a SQL NULL?"
3. "ABSENT ON NULL only removes SQL NULL inputs. The object is not NULL. So how many elements does the array have?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| Starters DishesByPrice as Baked Brie, Olive Trio, Soup du Jour | Uses insertion or alphabetical order, ignores the inner ORDER BY | "Look at the ORDER BY inside the parentheses. Which column comes first in it?" |
| Mains Allergens as null, sulphites, null | Assumes nulls are kept by default | "Which null clause is the default for JSON ARRAYAGG when none is written?" |
| Mains AllergensAll as sulphites only | Confuses NULL ON NULL with the default | "Read the clause at the end of that aggregate again. What does it promise for every row?" |
| Prices as 6.5 or as quoted strings | Does not know decimal renders unquoted with full scale | "The column is decimal six comma two. Does JSON keep the scale, and does a number get quotes?" |
| Specials DishesByPrice is NULL | Treats the aggregate like SUM or MAX | "Does JSON ARRAYAGG behave like SUM on an all-null group, or does it still produce an array?" |
| Specials AllergensAll is an empty array | Forgets the LEFT JOIN placeholder row is a real input | "How many rows does the Specials group contain after the LEFT JOIN? Zero, or one?" |
| Specials Cards is an empty array | Thinks ABSENT ON NULL removes the null card | "What exactly is fed to the aggregate: a SQL NULL, or a string produced by JSON OBJECT?" |
| Cards without the allergen key when null | Confuses JSON OBJECT with FOR JSON PATH | "FOR JSON PATH drops null keys by default. Does JSON OBJECT do the same?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the three stacked rules:

- **Rule 1, two ORDER BYs, two jobs.** The ORDER BY inside the aggregate orders the array elements. The ORDER BY at the end of the query orders the result rows. Without both, the output is nondeterministic. Also, all ordered aggregates in the same SELECT must use compatible orderings, otherwise error 8711, "Multiple ordered aggregate functions in the same scope have mutually incompatible orderings." That is why all four aggregates share the same ORDER BY. The same restriction applies to STRING AGG with WITHIN GROUP.
- **Rule 2, the default is ABSENT ON NULL.** A SQL NULL input contributes nothing, so the array shrinks. Mains has three dishes but Allergens has one element. That breaks positional correlation with DishesByPrice. NULL ON NULL, written after the ORDER BY and before the closing paren, keeps every position. For an all-null group, ABSENT ON NULL yields an empty array, never SQL NULL; NULL ON NULL yields an array with a single null.
- **Rule 3, the ghost element.** JSON OBJECT never returns SQL NULL. Its own default is NULL ON NULL, so a placeholder row becomes an object with all keys present and all values null. That string is not NULL, so ABSENT ON NULL cannot remove it, and the empty category gets a one-element array with a ghost card. To avoid it, use an inner join, or HAVING COUNT of DishId greater than zero, or build the card only where DishId is not null.

Rendering details worth saying once: keys keep call order; decimal renders unquoted with its exact scale, so six point five zero keeps the trailing zero; no whitespace is emitted; objects are separated by a single comma. Adding RETURNING json changes the return type to native json but not the text. A FOR JSON PATH subquery without INCLUDE_NULL_VALUES would drop the null-valued keys entirely, which JSON OBJECT never does.

Clause order inside the aggregate is fixed: value, then ORDER BY, then NULL ON NULL or ABSENT ON NULL, then RETURNING json.

Memory hook: "Arrayagg absents nulls by default. Objects are never null, so they never vanish."

## 9. Follow-up oral questions (optional)

1. "If the LEFT JOIN were changed to an INNER JOIN, how many rows would the query return?" (Three. Specials disappears entirely.)
2. "If the Allergens aggregate were written with ABSENT ON NULL spelled out, would anything change?" (No. ABSENT ON NULL is the default, so the result is identical.)
3. "What happens if one of the four aggregates used ORDER BY DishName only, while the others kept Price descending?" (The query fails with error 8711, incompatible orderings among ordered aggregates in the same scope.)

## 10. References

- JSON_ARRAYAGG (Transact-SQL), including the null clause default: https://learn.microsoft.com/en-us/sql/t-sql/functions/json-arrayagg-transact-sql
- JSON_OBJECT (Transact-SQL): https://learn.microsoft.com/en-us/sql/t-sql/functions/json-object-transact-sql
- JSON data in SQL Server: https://learn.microsoft.com/en-us/sql/relational-databases/json/json-data-sql-server
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
