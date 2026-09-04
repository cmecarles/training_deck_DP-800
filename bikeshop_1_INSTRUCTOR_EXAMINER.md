# Instructor-Examiner guide — BikeShop 1

Companion to [bikeshop_1.md](bikeshop_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a write-the-query question, not a predict-the-output question. The learner dictates a T-SQL query by voice. Take it in five parts, in the order given in section 4: the starting table and joins, where the status filter goes, the three aggregate columns, the grouping, and the sort. Accept any of the correct shapes in section 5 (single-pass join, CTE, OUTER APPLY, or scalar subqueries), as long as it produces the four rows exactly. Judge each part by the six checks listed at the end of section 5. The learner may ask you to repeat any row of data; the decoded order table in section 3 is there for that.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Write and optimize Transact-SQL queries.
- Task bullet: Write queries that combine data from multiple tables with joins, aggregates and grouping.
- What is tested: choosing the starting table for an outer join, placing a filter in `ON` versus `WHERE`, counting distinct parents over a child join, pricing from the transactional column rather than the catalog, replacing an empty sum with zero while leaving an empty max as NULL, and sorting by a computed column.

## 2. Scenario to read aloud

**Piece 1, the story.** "A bike shop's database, called BikeShop, was created in a single sqlcmd session, and that session is the complete history of the database. After it ended, one T-SQL query was run and returned exactly four rows. Your job is to write that query. I will describe the tables, the data, and then the expected result."

**Piece 2, the tables.** "There is a schema Sales with four tables. Sales dot Customers has CustomerID, an identity integer starting at one, primary key; FullName; City; and JoinedOn, a date. Sales dot Products has ProductID, identity starting at one; ProductName; Category; and ListPrice, a decimal with a check that it is positive. Sales dot Orders has OrderID, identity starting at one thousand; CustomerID, a foreign key to Customers; OrderDate; and OrderStatus, which must be Pending, Shipped or Cancelled. Sales dot OrderLines has OrderID, a foreign key to Orders; LineNum; ProductID, a foreign key to Products; Quantity, checked positive; and UnitPrice, a decimal. Its primary key is OrderID plus LineNum."

**Piece 3, customers and products.** "Four customers are inserted, so they get IDs one to four. One, Ana Ruiz, Madrid. Two, Ben Okafor, Lagos. Three, Chloe Dubois, Lyon. Four, Dev Patel, Pune. Five products, IDs one to five. One, Trail Helmet, Accessories, list price fifty-nine ninety. Two, Road Bike R200, Bikes, eight hundred ninety-nine. Three, Gravel Bike G1, Bikes, twelve hundred forty-nine. Four, Bottle Cage, Accessories, nine fifty. Five, Repair Kit, Tools, twenty-four."

**Piece 4, orders.** "Six orders are inserted, so they get IDs one thousand to one thousand five. Order 1000, customer 1 Ana, April 2 2024, Shipped. Order 1001, customer 2 Ben, April 5, Shipped. Order 1002, customer 1 Ana, April 20, Cancelled. Order 1003, customer 3 Chloe, May 3, Shipped. Order 1004, customer 2 Ben, May 14, Pending. Order 1005, customer 3 Chloe, May 21, Shipped. Note that customer 4, Dev Patel, has no orders at all."

**Piece 5, order lines.** "Ten order lines, each as order, line number, product, quantity, unit price.

- Order 1000, line 1, product 2 Road Bike, quantity 1, price eight ninety-nine.
- Order 1000, line 2, product 1 Trail Helmet, quantity 1, price fifty-nine ninety.
- Order 1000, line 3, product 4 Bottle Cage, quantity 2, price nine fifty.
- Order 1001, line 1, product 5 Repair Kit, quantity 3, price twenty-four.
- Order 1002, line 1, product 3 Gravel Bike, quantity 1, price twelve forty-nine.
- Order 1003, line 1, product 4 Bottle Cage, quantity 4, price nine fifty.
- Order 1003, line 2, product 1 Trail Helmet, quantity 2, price fifty-four ninety. Note: below the list price of fifty-nine ninety.
- Order 1004, line 1, product 2 Road Bike, quantity 1, price eight ninety-nine.
- Order 1005, line 1, product 5 Repair Kit, quantity 1, price twenty-four.
- Order 1005, line 2, product 3 Gravel Bike, quantity 1, price eleven ninety-nine. Note: below the list price of twelve forty-nine."

**Piece 6, the expected result.** "The query returns four rows with five columns: FullName, City, ShippedOrders, Revenue, LastShippedOn. Row 1: Chloe Dubois, Lyon, ShippedOrders 2, Revenue thirteen hundred seventy point eight zero, LastShippedOn May 21 2024. Row 2: Ana Ruiz, Madrid, 1, nine hundred seventy-seven point nine zero, April 2 2024. Row 3: Ben Okafor, Lagos, 1, seventy-two point zero zero, April 5 2024. Row 4: Dev Patel, Pune, 0, zero point zero zero, and LastShippedOn is null. Same column names, same values, same row order."

## 3. Setup script (reference only; do not read verbatim unless asked)

```powershell
PS C:\Users\dba> sqlcmd -S localhost -E
1> CREATE DATABASE BikeShop;
2> GO
1> USE BikeShop;
2> GO
Changed database context to 'BikeShop'.
1> CREATE SCHEMA Sales;
2> GO
1> CREATE TABLE Sales.Customers (
2>     CustomerID  INT IDENTITY(1,1) PRIMARY KEY,
3>     FullName    NVARCHAR(80) NOT NULL,
4>     City        NVARCHAR(60) NOT NULL,
5>     JoinedOn    DATE         NOT NULL
6> );
7> GO
1> CREATE TABLE Sales.Products (
2>     ProductID   INT IDENTITY(1,1) PRIMARY KEY,
3>     ProductName NVARCHAR(80)  NOT NULL,
4>     Category    NVARCHAR(40)  NOT NULL,
5>     ListPrice   DECIMAL(10,2) NOT NULL CHECK (ListPrice > 0)
6> );
7> GO
1> CREATE TABLE Sales.Orders (
2>     OrderID     INT IDENTITY(1000,1) PRIMARY KEY,
3>     CustomerID  INT  NOT NULL REFERENCES Sales.Customers(CustomerID),
4>     OrderDate   DATE NOT NULL,
5>     OrderStatus VARCHAR(10) NOT NULL
6>                 CHECK (OrderStatus IN ('Pending','Shipped','Cancelled'))
7> );
8> GO
1> CREATE TABLE Sales.OrderLines (
2>     OrderID     INT      NOT NULL REFERENCES Sales.Orders(OrderID),
3>     LineNum     SMALLINT NOT NULL,
4>     ProductID   INT      NOT NULL REFERENCES Sales.Products(ProductID),
5>     Quantity    SMALLINT NOT NULL CHECK (Quantity > 0),
6>     UnitPrice   DECIMAL(10,2) NOT NULL,
7>     CONSTRAINT PK_OrderLines PRIMARY KEY (OrderID, LineNum)
8> );
9> GO
1> INSERT INTO Sales.Customers (FullName, City, JoinedOn) VALUES
2>   (N'Ana Ruiz',     N'Madrid', '2023-02-11'),
3>   (N'Ben Okafor',   N'Lagos',  '2023-05-30'),
4>   (N'Chloe Dubois', N'Lyon',   '2024-01-08'),
5>   (N'Dev Patel',    N'Pune',   '2024-03-19');
6> GO

(4 rows affected)
1> INSERT INTO Sales.Products (ProductName, Category, ListPrice) VALUES
2>   (N'Trail Helmet',   N'Accessories',   59.90),
3>   (N'Road Bike R200', N'Bikes',        899.00),
4>   (N'Gravel Bike G1', N'Bikes',       1249.00),
5>   (N'Bottle Cage',    N'Accessories',    9.50),
6>   (N'Repair Kit',     N'Tools',         24.00);
7> GO

(5 rows affected)
1> INSERT INTO Sales.Orders (CustomerID, OrderDate, OrderStatus) VALUES
2>   (1, '2024-04-02', 'Shipped'),
3>   (2, '2024-04-05', 'Shipped'),
4>   (1, '2024-04-20', 'Cancelled'),
5>   (3, '2024-05-03', 'Shipped'),
6>   (2, '2024-05-14', 'Pending'),
7>   (3, '2024-05-21', 'Shipped');
8> GO

(6 rows affected)
1> INSERT INTO Sales.OrderLines (OrderID, LineNum, ProductID, Quantity, UnitPrice) VALUES
2>   (1000, 1, 2, 1,  899.00),
3>   (1000, 2, 1, 1,   59.90),
4>   (1000, 3, 4, 2,    9.50),
5>   (1001, 1, 5, 3,   24.00),
6>   (1002, 1, 3, 1, 1249.00),
7>   (1003, 1, 4, 4,    9.50),
8>   (1003, 2, 1, 2,   54.90),
9>   (1004, 1, 2, 1,  899.00),
10>   (1005, 1, 5, 1,   24.00),
11>   (1005, 2, 3, 1, 1199.00);
12> GO

(10 rows affected)
1> EXIT
PS C:\Users\dba>
```

Expected result set:

```json
[
  { "FullName": "Chloe Dubois", "City": "Lyon",   "ShippedOrders": 2, "Revenue": 1370.80, "LastShippedOn": "2024-05-21" },
  { "FullName": "Ana Ruiz",     "City": "Madrid", "ShippedOrders": 1, "Revenue": 977.90,  "LastShippedOn": "2024-04-02" },
  { "FullName": "Ben Okafor",   "City": "Lagos",  "ShippedOrders": 1, "Revenue": 72.00,   "LastShippedOn": "2024-04-05" },
  { "FullName": "Dev Patel",    "City": "Pune",   "ShippedOrders": 0, "Revenue": 0.00,    "LastShippedOn": null }
]
```

Decoded order table, for reading any row on request:

| Order | Customer | Status | Line | Product | Qty | Paid each | List price | Line total |
|---|---|---|---|---|---|---|---|---|
| 1000 | Ana Ruiz | Shipped | 1 | Road Bike R200 | 1 | 899.00 | 899.00 | 899.00 |
| 1000 | Ana Ruiz | Shipped | 2 | Trail Helmet | 1 | 59.90 | 59.90 | 59.90 |
| 1000 | Ana Ruiz | Shipped | 3 | Bottle Cage | 2 | 9.50 | 9.50 | 19.00 |
| 1001 | Ben Okafor | Shipped | 1 | Repair Kit | 3 | 24.00 | 24.00 | 72.00 |
| 1002 | Ana Ruiz | Cancelled | 1 | Gravel Bike G1 | 1 | 1249.00 | 1249.00 | excluded |
| 1003 | Chloe Dubois | Shipped | 1 | Bottle Cage | 4 | 9.50 | 9.50 | 38.00 |
| 1003 | Chloe Dubois | Shipped | 2 | Trail Helmet | 2 | 54.90 | 59.90 | 109.80 |
| 1004 | Ben Okafor | Pending | 1 | Road Bike R200 | 1 | 899.00 | 899.00 | excluded |
| 1005 | Chloe Dubois | Shipped | 1 | Repair Kit | 1 | 24.00 | 24.00 | 24.00 |
| 1005 | Chloe Dubois | Shipped | 2 | Gravel Bike G1 | 1 | 1199.00 | 1249.00 | 1199.00 |

## 4. The question (ask exactly this)

"After the session ends, a single T-SQL query is run against BikeShop and returns exactly the result set I described: same column names, same values, same row order. Write that query. Let's build it one part at a time."

- Part 1: "Which table do you start from, which tables do you join, and what kind of join?"
- Part 2: "Where do you put the condition that only shipped orders count?"
- Part 3: "How do you compute ShippedOrders, Revenue and LastShippedOn? Give me each expression."
- Part 4: "What do you group by?"
- Part 5: "How do you sort?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

Reference query:

```sql
SELECT
    c.FullName,
    c.City,
    COUNT(DISTINCT o.OrderID)                   AS ShippedOrders,
    ISNULL(SUM(ol.Quantity * ol.UnitPrice), 0)  AS Revenue,
    MAX(o.OrderDate)                            AS LastShippedOn
FROM Sales.Customers AS c
LEFT JOIN Sales.Orders AS o
       ON o.CustomerID  = c.CustomerID
      AND o.OrderStatus = 'Shipped'
LEFT JOIN Sales.OrderLines AS ol
       ON ol.OrderID = o.OrderID
GROUP BY c.CustomerID, c.FullName, c.City
ORDER BY Revenue DESC;
```

Per part:

- Part 1: start from Sales.Customers, LEFT JOIN Orders on CustomerID, LEFT JOIN OrderLines on OrderID. No join to Products is needed.
- Part 2: inside the ON clause of the Orders join, o.OrderStatus equals Shipped. In WHERE it would remove Dev Patel.
- Part 3: COUNT DISTINCT o.OrderID; ISNULL of SUM of ol.Quantity times ol.UnitPrice, zero; MAX of o.OrderDate, left NULL for Dev.
- Part 4: GROUP BY c.CustomerID, c.FullName, c.City.
- Part 5: ORDER BY Revenue DESC.

Also correct, any of these shapes:

- A. A CTE that inner-joins Orders to OrderLines, filters WHERE OrderStatus equals Shipped, groups per OrderID with the order total; then Customers LEFT JOIN the CTE, COUNT of OrderID without DISTINCT, ISNULL SUM of the totals, MAX date, GROUP BY customer, ORDER BY Revenue DESC. The filter may sit in WHERE here because it runs before the outer join.
- B. Customers OUTER APPLY a subquery that computes COUNT DISTINCT OrderID, SUM of Quantity times UnitPrice and MAX OrderDate for the current customer's shipped orders; ISNULL on the sum outside; ORDER BY Revenue DESC. No GROUP BY needed.
- C. Three correlated scalar subqueries, one per aggregate, each filtered on the current CustomerID and Shipped; COUNT star is fine there because it counts orders, not lines; ISNULL on the sum; ORDER BY Revenue DESC.

To emit the JSON literally, append FOR JSON PATH, INCLUDE_NULL_VALUES; without INCLUDE_NULL_VALUES Dev's LastShippedOn key is omitted.

Six checks any answer must pass:

1. Every customer appears, including Dev Patel: an outer join, OUTER APPLY or scalar subqueries starting from Customers.
2. Both hops preserve the customer: the OrderLines join is also LEFT in the single-pass form.
3. Count orders, not lines: COUNT DISTINCT OrderID, or count at a level where one row is one order. Plain COUNT star gives 4, 3, 1, 1.
4. Price from OrderLines.UnitPrice, not Products.ListPrice. ListPrice gives Chloe 1430.80 instead of 1370.80.
5. Empty sum becomes 0.00 with ISNULL or COALESCE; empty max stays NULL. One, not both.
6. ORDER BY Revenue DESC.

## 6. Hint ladder (one hint per attempt, in order)

**Part 1, start table and joins**
1. "Look at the result. Dev Patel is there with zeros and a null. Which table is the only one that knows about Dev?"
2. "If you start from OrderLines or Orders and walk to the customer, can you ever reach a customer who has no lines?"
3. "Start from Customers and walk outward. What kind of join keeps a customer who matches nothing?"
4. "Do you actually need Products? Where does the money live?"

**Part 2, the status filter**
1. "Ana has a cancelled order and Ben has a pending one, and neither counts. But Dev has no orders. What is Dev's status after a LEFT JOIN to Orders?"
2. "If you filter status equals Shipped after the join, in WHERE, does a NULL status pass that test?"
3. "The filter must run while matching, not afterwards. Which clause is evaluated while matching?"

**Part 3, the three aggregates**
1. "Chloe has two shipped orders but four order lines. If you count rows, what number do you get for her?"
2. "You need to count different order numbers. Which aggregate does that?"
3. "For Revenue, there are two prices for the same product: one in Products and one on the line. Chloe's revenue is 1370.80. Which price gives that number?"
4. "Dev's revenue is 0.00 but his LastShippedOn is null. What does SUM return over no rows, and how do you turn that into zero without also touching the date?"

**Part 4, grouping**
1. "One row per customer. What identifies a customer uniquely, and which non-aggregated columns are in the select list?"
2. "Group by the key and the two displayed columns."

**Part 5, sorting**
1. "The buffer inserted Ana first, but the result starts with Chloe. Which column runs monotonically down the four rows?"
2. "Largest first."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| Starts from Orders or OrderLines with inner joins | Walks the arrows in reading order | "Trace Dev Patel through that walk. Does he ever appear?" |
| LEFT JOIN Orders but INNER JOIN OrderLines | Forgets the second hop can also drop the customer | "After the first LEFT JOIN, Dev's OrderID is NULL. What does an inner join to OrderLines do with that?" |
| Status filter in WHERE | Filters after the outer join | "What is Dev's OrderStatus after the outer join, and does WHERE keep it?" |
| COUNT star or COUNT of o.OrderID | Counts lines, not orders | "Chloe's count comes out as four that way. What is the expected value?" |
| Joins Products and uses ListPrice | Uses the catalog price | "Compute Chloe's revenue with list prices. Does it match 1370.80?" |
| ISNULL on both the sum and the max | Overcorrects | "Look at Dev's row again. Is LastShippedOn zero, or null?" |
| ORDER BY FullName or by CustomerID | Guesses the sort | "Read the four revenues top to bottom. What order are they in?" |
| Grouping only by FullName | Omits the key | "It works on this data, but which column truly identifies a customer?" |

## 8. Teaching notes (after the answer is complete or revealed)

Read the buffer as a map, not as code. Every REFERENCES is an arrow from a child column to a parent key: OrderLines points at Orders and Products, Orders points at Customers. Nothing points the other way. The buffer never types an ID; IDENTITY assigns customers 1 to 4, products 1 to 5, and orders 1000 to 1005 because of the seed. That decoder ring turns the last INSERT into a story: order 1003, line 2, is Chloe's trail helmet, two units at 54.90 instead of 59.90.

Then let the result tell you where to start. Four rows, four customers, including one with nothing. So start at Customers and walk outward with LEFT JOINs. Walking inward from the lines can only reach customers that some line points to, and Dev has none.

The subtle step is the filter. Attach only shipped orders while matching, in the ON clause. If you attach everything first and delete non-shipped rows afterwards in WHERE, Dev's placeholder has a blank status, blank is not Shipped, and Dev is deleted too. Filtering while matching keeps him; filtering afterwards loses him. A CTE that pre-filters before the outer join sidesteps this, which is why alternative A may use WHERE.

Price each line with the line's own UnitPrice. Products is never needed. Two lines were sold below list, the helmet on 1003 and the gravel bike on 1005; pricing with ListPrice gives Chloe 1430.80, and the JSON says 1370.80, so the result itself tells you which column was used.

Fold back to one row per customer with GROUP BY. Count distinct orders, because Chloe is spread over four line rows but has two order numbers. Sum the line totals and replace the empty sum with zero, because the JSON shows 0.00. Take the latest date and leave the empty one alone, because the JSON shows null. Sort by Revenue descending, the only column that runs monotonically down the result.

The method in one breath: column names say which tables are involved; the row with zeros says where to start and that nothing may be discarded; the figures say which price column was used; the row order says how to sort. Everything else is walking the arrows.

Memory hook: "Start where the empty row lives. Filter in ON, not WHERE. Count distinct parents. Price from the line. ISNULL the sum, not the max."

## 9. Follow-up oral questions (optional)

1. "If the status condition were moved from ON to WHERE, which rows would the query return?" (Three rows: Chloe, Ana, Ben. Dev Patel disappears.)
2. "What would ShippedOrders show for each customer if COUNT star were used instead of COUNT DISTINCT OrderID?" (Chloe 4, Ana 3, Ben 1, Dev 1, because Dev's placeholder row also counts.)
3. "What would Chloe's revenue be if the lines were priced with Products.ListPrice?" (1430.80, sixty more than 1370.80: five more per helmet times two, and fifty more for the gravel bike.)

## 10. References

- FROM clause with JOIN syntax, including LEFT OUTER JOIN and the ON clause: https://learn.microsoft.com/en-us/sql/t-sql/queries/from-transact-sql
- Joins in SQL Server, outer joins and filter placement: https://learn.microsoft.com/en-us/sql/relational-databases/performance/joins
- COUNT (Transact-SQL), including COUNT DISTINCT: https://learn.microsoft.com/en-us/sql/t-sql/functions/count-transact-sql
- ISNULL (Transact-SQL): https://learn.microsoft.com/en-us/sql/t-sql/functions/isnull-transact-sql
- FOR JSON PATH and INCLUDE_NULL_VALUES: https://learn.microsoft.com/en-us/sql/relational-databases/json/format-query-results-as-json-with-for-json-sql-server
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
