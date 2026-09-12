# 💻 Write Advanced T-SQL Code

> **DP-800 Exam Domain:** Design and Develop Database Solutions (35–40%)  
> **Subtopic:** Write Advanced T-SQL Code  
> **Skills Measured (as of March 12, 2026)**

---

## 📋 Table of Contents

- [Exam Objectives Breakdown](#-exam-objectives-breakdown)
- [1. Common Table Expressions (CTEs)](#1-write-common-table-expressions-ctes)
- [2. Window Functions](#2-write-queries-that-include-window-functions)
- [3. JSON Functions](#3-write-queries-that-include-json-functions)
- [4. Regular Expressions](#4-write-queries-that-include-regular-expressions)
- [5. Fuzzy String Matching](#5-write-queries-that-include-fuzzy-string-matching-functions)
- [6. Graph Queries with MATCH](#6-write-graph-queries-that-use-the-match-operator)
- [7. Correlated Queries](#7-write-correlated-queries)
- [8. Error Handling](#8-implement-error-handling)
- [Practice Questions](#-practice-questions)
- [Resources & References](#-resources--references)

---

## 📌 Exam Objectives Breakdown

| # | Objective | Complexity |
|---|-----------|------------|
| 1 | Write common table expressions (CTEs) | ⭐⭐⭐ |
| 2 | Write queries that include window functions | ⭐⭐⭐⭐ |
| 3 | Write queries with JSON functions (JSON_OBJECT, JSON_ARRAY, JSON_ARRAYAGG, OPENJSON, JSON_VALUE, JSON_CONTAINS) | ⭐⭐⭐⭐ |
| 4 | Write queries with regular expressions (REGEXP_LIKE, REGEXP_REPLACE, REGEXP_SUBSTR, REGEXP_MATCHES, etc.) | ⭐⭐⭐ |
| 5 | Write queries with fuzzy string matching (EDIT_DISTANCE, EDIT_DISTANCE_SIMILARITY, JARO_WINKLER_DISTANCE) | ⭐⭐⭐ |
| 6 | Write graph queries using the MATCH operator | ⭐⭐⭐ |
| 7 | Write correlated queries | ⭐⭐⭐ |
| 8 | Implement error handling | ⭐⭐⭐ |

> 🎯 **Platform Context:** DP-800 covers **SQL Server 2022+**, **Azure SQL Database**, **Azure SQL Managed Instance**, and **SQL databases in Microsoft Fabric**. Feature availability varies — the exam tests awareness of these differences.

---

## 1. Write Common Table Expressions (CTEs)

### 🔑 What is a CTE?

A **Common Table Expression (CTE)** is a temporary, named result set defined within a `WITH` clause that exists for the duration of a single query. CTEs improve readability, enable recursion, and allow referencing derived results multiple times.

```sql
WITH cte_name (column1, column2) AS (
    SELECT column1, column2
    FROM source_table
    WHERE condition
)
SELECT * FROM cte_name;
```

### 📐 Types of CTEs

#### Simple (Non-Recursive) CTE

```sql
-- Top 5 customers by total sales
WITH CustomerSales AS (
    SELECT
        c.CustomerID,
        c.CustomerName,
        SUM(o.OrderTotal) AS TotalSales
    FROM Customers c
    JOIN Orders o ON c.CustomerID = o.CustomerID
    GROUP BY c.CustomerID, c.CustomerName
)
SELECT TOP 5 CustomerName, TotalSales
FROM CustomerSales
ORDER BY TotalSales DESC;
```

#### Multiple CTEs in One Query

```sql
WITH
MonthlySales AS (
    SELECT
        YEAR(OrderDate) AS SalesYear,
        MONTH(OrderDate) AS SalesMonth,
        SUM(Amount) AS MonthlyTotal
    FROM Orders
    GROUP BY YEAR(OrderDate), MONTH(OrderDate)
),
YearlySales AS (
    SELECT SalesYear, SUM(MonthlyTotal) AS YearlyTotal
    FROM MonthlySales
    GROUP BY SalesYear
)
SELECT
    ms.SalesYear,
    ms.SalesMonth,
    ms.MonthlyTotal,
    ys.YearlyTotal,
    ROUND(ms.MonthlyTotal * 100.0 / ys.YearlyTotal, 2) AS PctOfYear
FROM MonthlySales ms
JOIN YearlySales ys ON ms.SalesYear = ys.SalesYear
ORDER BY ms.SalesYear, ms.SalesMonth;
```

#### Recursive CTE

Recursive CTEs consist of two parts joined by `UNION ALL`:
1. **Anchor member** — the starting/base case
2. **Recursive member** — references the CTE itself, adding one level per iteration

```sql
-- Organizational hierarchy traversal
WITH OrgHierarchy AS (
    -- Anchor: start from the CEO (no manager)
    SELECT
        EmployeeID,
        EmployeeName,
        ManagerID,
        0 AS Level,
        CAST(EmployeeName AS VARCHAR(1000)) AS HierarchyPath
    FROM Employees
    WHERE ManagerID IS NULL

    UNION ALL

    -- Recursive: find direct reports of each level
    SELECT
        e.EmployeeID,
        e.EmployeeName,
        e.ManagerID,
        oh.Level + 1,
        CAST(oh.HierarchyPath + ' > ' + e.EmployeeName AS VARCHAR(1000))
    FROM Employees e
    INNER JOIN OrgHierarchy oh ON e.ManagerID = oh.EmployeeID
)
SELECT
    REPLICATE('  ', Level) + EmployeeName AS OrgChart,
    Level,
    HierarchyPath
FROM OrgHierarchy
ORDER BY HierarchyPath;
```

> ⚠️ **MAXRECURSION:** Default is 100 levels. Override with `OPTION (MAXRECURSION 0)` for unlimited, or `OPTION (MAXRECURSION 365)` for date series.

```sql
-- Generate a date series using recursive CTE
WITH DateSeries AS (
    SELECT CAST('2024-01-01' AS DATE) AS DateValue
    UNION ALL
    SELECT DATEADD(DAY, 1, DateValue)
    FROM DateSeries
    WHERE DateValue < '2024-01-31'
)
SELECT DateValue FROM DateSeries
OPTION (MAXRECURSION 365);
```

### 🆚 CTE vs Subquery vs Temp Table

| Feature | CTE | Subquery | Temp Table |
|---------|-----|----------|------------|
| **Readability** | ✅ High | ⚠️ Medium | ✅ High |
| **Reuse in same query** | ✅ Yes | ❌ No | ✅ Yes |
| **Recursion** | ✅ Yes | ❌ No | ❌ No |
| **Indexes** | ❌ No | ❌ No | ✅ Yes |
| **Persists across queries** | ❌ No | ❌ No | ✅ Yes |
| **Best for** | Readability, recursion | Simple lookups | Large intermediate results |

---

## 2. Write Queries That Include Window Functions

### 🔑 Window Function Syntax

```sql
function_name() OVER (
    [PARTITION BY column(s)]    -- Divides rows into groups
    [ORDER BY column(s)]        -- Defines row order within partition
    [ROWS/RANGE frame_spec]     -- Defines the window frame
)
```

### 📊 Ranking Functions

```sql
SELECT
    ProductName,
    Category,
    Price,
    -- ROW_NUMBER: unique sequential number — no ties (1,2,3,4)
    ROW_NUMBER() OVER (PARTITION BY Category ORDER BY Price DESC) AS RowNum,
    -- RANK: same rank for ties, gaps after ties (1,2,2,4)
    RANK()       OVER (PARTITION BY Category ORDER BY Price DESC) AS RankNum,
    -- DENSE_RANK: same rank for ties, no gaps (1,2,2,3)
    DENSE_RANK() OVER (PARTITION BY Category ORDER BY Price DESC) AS DenseRank,
    -- NTILE: distributes rows into N equal buckets
    NTILE(4)     OVER (PARTITION BY Category ORDER BY Price DESC) AS Quartile
FROM Products;
```

### 📊 Aggregate Window Functions

```sql
SELECT
    SaleDate,
    Region,
    Amount,
    -- Running total within each region
    SUM(Amount) OVER (
        PARTITION BY Region
        ORDER BY SaleDate
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS RunningTotal,
    -- 3-day moving average
    AVG(Amount) OVER (
        PARTITION BY Region
        ORDER BY SaleDate
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS MovingAvg3Day,
    -- Region total (for percentage calculation)
    SUM(Amount) OVER (PARTITION BY Region) AS RegionTotal,
    ROUND(Amount * 100.0 / SUM(Amount) OVER (PARTITION BY Region), 2) AS PctOfRegion
FROM Sales;
```

### 📊 Window Frame Options

| Frame Clause | Description |
|-------------|-------------|
| `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | All rows from partition start to current |
| `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` | Current row + 2 rows before |
| `ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING` | Current row to end of partition |
| `ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING` | Current row ± 1 row |
| `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | All rows with same ORDER value as current |

### 📊 Analytic Functions (LAG, LEAD, FIRST_VALUE, LAST_VALUE)

```sql
SELECT
    SaleDate,
    Amount,
    LAG(Amount, 1, 0)  OVER (ORDER BY SaleDate) AS PrevDayAmount,
    LEAD(Amount, 1, 0) OVER (ORDER BY SaleDate) AS NextDayAmount,
    Amount - LAG(Amount, 1, 0) OVER (ORDER BY SaleDate) AS DayOverDayChange,
    FIRST_VALUE(Amount) OVER (ORDER BY SaleDate
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS FirstDaySale,
    LAST_VALUE(Amount)  OVER (ORDER BY SaleDate
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS LastDaySale
FROM DailySales;
```

> ⚠️ `LAST_VALUE` requires the explicit `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` frame — otherwise the default frame stops at the current row.

#### Practical Pattern: Deduplicate with ROW_NUMBER

```sql
-- Remove duplicates, keeping most recent record per CustomerID
WITH Deduped AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY CustomerID
            ORDER BY UpdatedAt DESC
        ) AS rn
    FROM Customers
)
DELETE FROM Deduped WHERE rn > 1;
```

---

## 3. Write Queries That Include JSON Functions

> **Platform note:** JSON functions vary across platforms. The exam covers SQL Server 2022+, Azure SQL, and Fabric SQL database.

### 🔑 Core JSON Functions

| Function | Purpose | Available |
|----------|---------|-----------|
| `JSON_VALUE(json, path)` | Extract scalar value | All platforms |
| `JSON_QUERY(json, path)` | Extract object/array | All platforms |
| `JSON_OBJECT(key: value, ...)` | Construct JSON object | SQL Server 2022+, Azure SQL, Fabric |
| `JSON_ARRAY(val1, val2, ...)` | Construct JSON array | SQL Server 2022+, Azure SQL, Fabric |
| `JSON_ARRAYAGG(expr)` | Aggregate rows into JSON array | SQL Server 2022+, Azure SQL, Fabric |
| `OPENJSON(json [, path])` | Parse JSON into rows | All platforms |
| `JSON_CONTAINS(json, val, path)` | Check if JSON contains value | Azure SQL, Fabric |
| `ISJSON(json)` | Validate JSON format | All platforms |
| `FOR JSON` | Convert rows to JSON | All platforms |

### 📐 Reading JSON Data

```sql
-- Sample column ProductData: {"name":"Widget","specs":{"weight":1.5},"tags":["sale","new"]}

-- JSON_VALUE: extracts a scalar (returns NULL for objects/arrays)
SELECT
    JSON_VALUE(ProductData, '$.name')           AS ProductName,
    JSON_VALUE(ProductData, '$.specs.weight')   AS Weight,
    JSON_VALUE(ProductData, '$.tags[0]')        AS FirstTag
FROM Products;

-- JSON_QUERY: extracts objects or arrays (returns NULL for scalars)
SELECT
    JSON_QUERY(ProductData, '$.specs')  AS SpecsObject,
    JSON_QUERY(ProductData, '$.tags')   AS TagsArray
FROM Products;

-- JSON_CONTAINS: check if JSON array contains a value (Azure SQL / Fabric)
SELECT ProductName
FROM Products
WHERE JSON_CONTAINS(ProductData, '"sale"', '$.tags') = 1;
```

### 📐 Constructing JSON

```sql
-- JSON_OBJECT: build a JSON object
SELECT JSON_OBJECT(
    'id':     CustomerID,
    'name':   CustomerName,
    'email':  Email,
    'active': IsActive
) AS CustomerJSON
FROM Customers;

-- JSON_ARRAY: build an inline array
SELECT JSON_ARRAY(1, 'hello', TRUE, NULL) AS SimpleArray;
-- Result: [1,"hello",true,null]

-- JSON_ARRAYAGG: aggregate rows into a JSON array
SELECT
    c.CustomerID,
    c.CustomerName,
    JSON_ARRAYAGG(
        JSON_OBJECT(
            'orderId': o.OrderID,
            'total':   o.OrderTotal,
            'date':    o.OrderDate
        )
    ) AS OrdersJSON
FROM Customers c
LEFT JOIN Orders o ON c.CustomerID = o.CustomerID
GROUP BY c.CustomerID, c.CustomerName;
```

### 📐 OPENJSON: Parse JSON into Tabular Format

```sql
DECLARE @json NVARCHAR(MAX) = '
[
    {"id": 1, "name": "Alice", "score": 95.5},
    {"id": 2, "name": "Bob",   "score": 88.0}
]';

-- WITH schema: maps JSON properties to typed columns
SELECT StudentID, StudentName, Score
FROM OPENJSON(@json)
WITH (
    StudentID   INT           '$.id',
    StudentName NVARCHAR(100) '$.name',
    Score       DECIMAL(5,2)  '$.score'
);
```

```sql
-- OPENJSON with nested array: unnest order items
DECLARE @order NVARCHAR(MAX) = '{
    "orderId": 1001,
    "items": [
        {"sku": "A1", "qty": 2, "price": 9.99},
        {"sku": "B3", "qty": 1, "price": 24.99}
    ]
}';

SELECT
    JSON_VALUE(@order, '$.orderId') AS OrderId,
    items.sku,
    items.qty,
    items.price,
    items.qty * items.price AS LineTotal
FROM OPENJSON(@order, '$.items')
WITH (
    sku   NVARCHAR(10) '$.sku',
    qty   INT          '$.qty',
    price DECIMAL(8,2) '$.price'
) AS items;
```

### 📐 FOR JSON: Convert Query Results to JSON

```sql
-- FOR JSON PATH: controlled structure via column aliases
SELECT
    CustomerID      AS 'customer.id',
    CustomerName    AS 'customer.name',
    Email           AS 'customer.contact.email'
FROM Customers
FOR JSON PATH, ROOT('customers');
-- Result: {"customers":[{"customer":{"id":1,"name":"Alice","contact":{"email":"..."}}}]}
```

### 🔍 JSON Indexes (SQL Server 2022+ / Azure SQL)

```sql
-- Create computed column + index for fast JSON property access
ALTER TABLE Products
    ADD ProductName AS JSON_VALUE(ProductData, '$.name') PERSISTED;

CREATE INDEX idx_products_name ON Products(ProductName);
```

---

## 4. Write Queries That Include Regular Expressions

> **Platform note:** `REGEXP_*` functions are available in **Azure SQL Database** and **SQL databases in Fabric**. NOT available in SQL Server on-premises.

### 🔑 REGEXP Functions Reference

| Function | Description |
|----------|-------------|
| `REGEXP_LIKE(value, pattern)` | Returns 1 if value matches pattern |
| `REGEXP_REPLACE(value, pattern, replacement)` | Replace matched text |
| `REGEXP_SUBSTR(value, pattern [, pos, occurrence])` | Extract matched substring |
| `REGEXP_INSTR(value, pattern [, pos, occurrence])` | Position of first match |
| `REGEXP_COUNT(value, pattern)` | Count of non-overlapping matches |
| `REGEXP_MATCHES(value, pattern)` | Returns all matches as a table |
| `REGEXP_SPLIT_TO_TABLE(value, pattern)` | Split string on regex delimiter, returns table |

### 📐 Examples

```sql
-- REGEXP_LIKE: filter rows with valid email format
SELECT CustomerName, Email
FROM Customers
WHERE REGEXP_LIKE(Email, '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$') = 1;

-- REGEXP_REPLACE: normalize phone numbers (remove non-digits)
SELECT
    Phone AS OriginalPhone,
    REGEXP_REPLACE(Phone, '[^0-9]', '') AS NormalizedPhone
FROM Customers;
-- '(555) 123-4567' → '5551234567'

-- REGEXP_COUNT: count word occurrences in text
SELECT
    ReviewText,
    REGEXP_COUNT(ReviewText, '\bgreat\b') AS CountOfGreat
FROM ProductReviews;

-- REGEXP_SPLIT_TO_TABLE: split a CSV-like string into rows
SELECT value AS Tag
FROM REGEXP_SPLIT_TO_TABLE('python,sql,azure,ai', ',');
-- Returns: python | sql | azure | ai (4 rows)

-- REGEXP_MATCHES: return ALL matches as a result set
SELECT match_value
FROM REGEXP_MATCHES('Call 555-1234 or 555-9876 for info', '[0-9]{3}-[0-9]{4}');
-- Returns: 555-1234, 555-9876
```

### 🎯 Common Regex Patterns

| Pattern | Matches |
|---------|---------|
| `^[A-Z]` | Starts with uppercase letter |
| `[0-9]{5}` | Exactly 5 digits (ZIP code) |
| `[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}` | Email format |
| `^\+?[0-9]{10,15}$` | Phone number (10–15 digits) |
| `[^a-zA-Z0-9 ]` | Special/non-alphanumeric characters |
| `\bword\b` | Whole word match |

---

## 5. Write Queries That Include Fuzzy String Matching Functions

> **Platform note:** Fuzzy string functions are available in **Azure SQL Database** and **SQL databases in Fabric**. NOT in SQL Server on-premises.

### 🔑 Fuzzy String Matching Functions

| Function | Description | Range | Best Use |
|----------|-------------|-------|----------|
| `EDIT_DISTANCE(s1, s2)` | Levenshtein distance — minimum character edits to transform s1 → s2 | 0 to len(s) | Typo detection, deduplication |
| `EDIT_DISTANCE_SIMILARITY(s1, s2)` | Normalized similarity score | 0–100 | Threshold-based matching |
| `JARO_WINKLER_DISTANCE(s1, s2)` | Jaro-Winkler similarity (favors matching prefixes) | 0.0–1.0 | Name matching, short strings |

### 📐 Examples

```sql
-- Find potential duplicate customer records
SELECT
    a.CustomerID AS ID1,
    a.CustomerName AS Name1,
    b.CustomerID AS ID2,
    b.CustomerName AS Name2,
    EDIT_DISTANCE(a.CustomerName, b.CustomerName) AS EditDist,
    EDIT_DISTANCE_SIMILARITY(a.CustomerName, b.CustomerName) AS Similarity,
    JARO_WINKLER_DISTANCE(a.CustomerName, b.CustomerName) AS JaroWinkler
FROM Customers a
CROSS JOIN Customers b
WHERE a.CustomerID < b.CustomerID
  AND EDIT_DISTANCE_SIMILARITY(a.CustomerName, b.CustomerName) > 80
ORDER BY Similarity DESC;

-- Fuzzy product search (tolerates typos)
DECLARE @search NVARCHAR(100) = 'Wirelss Moues'; -- "Wireless Mouse" with typos

SELECT
    ProductName,
    EDIT_DISTANCE_SIMILARITY(ProductName, @search) AS Similarity
FROM Products
WHERE EDIT_DISTANCE_SIMILARITY(ProductName, @search) > 60
ORDER BY Similarity DESC;

-- Jaro-Winkler: best for human names (rewards matching prefixes)
SELECT
    FullName,
    JARO_WINKLER_DISTANCE(FullName, 'Robert Smith') AS Score
FROM Employees
WHERE JARO_WINKLER_DISTANCE(FullName, 'Robert Smith') > 0.85
ORDER BY Score DESC;
```

### 🎯 Choosing the Right Function

```
Short strings (names, codes)?   → JARO_WINKLER_DISTANCE (better for prefixes)
Longer text, typo detection?    → EDIT_DISTANCE / EDIT_DISTANCE_SIMILARITY
Need a % similarity score?      → EDIT_DISTANCE_SIMILARITY (0-100 scale)
Need raw character edit count?  → EDIT_DISTANCE
```

---

## 6. Write Graph Queries That Use the MATCH Operator

### 🔑 What are SQL Graph Tables?

SQL Graph (SQL Server 2017+, Azure SQL, Fabric SQL) stores **nodes** (entities) and **edges** (relationships) as special table types, enabling relationship traversal queries.

```sql
-- Create NODE table (entities)
CREATE TABLE Person (
    PersonID    INT PRIMARY KEY,
    PersonName  NVARCHAR(100)
) AS NODE;

-- Create EDGE table (relationships)
CREATE TABLE Knows (
    Relationship NVARCHAR(50)
) AS EDGE;

-- Insert nodes
INSERT INTO Person VALUES (1,'Alice'),(2,'Bob'),(3,'Carol'),(4,'Dave');

-- Insert edges using $node_id
INSERT INTO Knows ($from_id, $to_id, Relationship)
SELECT p1.$node_id, p2.$node_id, 'colleague'
FROM Person p1, Person p2
WHERE (p1.PersonName = 'Alice' AND p2.PersonName = 'Bob')
   OR (p1.PersonName = 'Bob'   AND p2.PersonName = 'Carol');
```

### 📐 MATCH Operator Syntax

```sql
-- Basic: Who does Alice know?
SELECT p1.PersonName AS Person, p2.PersonName AS Knows
FROM Person p1, Knows k, Person p2
WHERE MATCH(p1-(k)->p2)
  AND p1.PersonName = 'Alice';

-- 2-hop: Friends of friends
SELECT p1.PersonName, p3.PersonName AS FriendOfFriend
FROM Person p1, Knows k1, Person p2, Knows k2, Person p3
WHERE MATCH(p1-(k1)->p2-(k2)->p3)
  AND p1.PersonName = 'Alice'
  AND p3.PersonName <> 'Alice';

-- Undirected (either direction)
SELECT p1.PersonName, p2.PersonName AS Connected
FROM Person p1, Knows k, Person p2
WHERE MATCH(p1-(k)-p2);

-- SHORTEST_PATH: find shortest connection between two people
SELECT
    src.PersonName AS Source,
    dst.PersonName AS Destination,
    STRING_AGG(mid.PersonName, ' -> ') WITHIN GROUP (GRAPH PATH) AS Path,
    COUNT(mid.PersonName) WITHIN GROUP (GRAPH PATH) AS Hops
FROM
    Person AS src,
    SHORTEST_PATH(Person AS mid -(Knows AS k)->+) AS sp,
    Person AS dst
WHERE src.PersonName = 'Alice'
  AND dst.PersonName = 'Dave';
```

### 🎯 Graph Use Cases in DP-800

| Scenario | Node Tables | Edge Tables |
|----------|------------|-------------|
| Social network | Person | Follows, Likes |
| Supply chain | Product, Supplier, Warehouse | Supplies, Ships |
| Org hierarchy | Employee | ReportsTo |
| Recommendation engine | User, Product | Purchased, Viewed |

---

## 7. Write Correlated Queries

### 🔑 What is a Correlated Subquery?

A **correlated subquery** references columns from the outer query — it executes **once per row** of the outer query.

```sql
-- Pattern
SELECT outer_col
FROM outer_table o
WHERE outer_col = (
    SELECT inner_col
    FROM inner_table i
    WHERE i.fk_col = o.pk_col  -- ← references outer query column
);
```

### 📐 Examples

```sql
-- Customers with orders above their own average order value
SELECT c.CustomerName, o.OrderID, o.OrderTotal
FROM Customers c
JOIN Orders o ON c.CustomerID = o.CustomerID
WHERE o.OrderTotal > (
    SELECT AVG(o2.OrderTotal)
    FROM Orders o2
    WHERE o2.CustomerID = c.CustomerID  -- correlated
);

-- EXISTS: customers with at least one order this year
SELECT CustomerName
FROM Customers c
WHERE EXISTS (
    SELECT 1 FROM Orders o
    WHERE o.CustomerID = c.CustomerID
      AND YEAR(o.OrderDate) = YEAR(GETDATE())
);

-- NOT EXISTS: customers with NO orders (preferred over NOT IN)
SELECT CustomerName
FROM Customers c
WHERE NOT EXISTS (
    SELECT 1 FROM Orders o
    WHERE o.CustomerID = c.CustomerID
);

-- Correlated UPDATE: update each product with its category avg price
UPDATE p
SET p.CategoryAvgPrice = (
    SELECT AVG(p2.Price) FROM Products p2
    WHERE p2.CategoryID = p.CategoryID  -- correlated
)
FROM Products p;
```

### 🆚 EXISTS vs IN vs JOIN Performance

| Pattern | Performance | Use When |
|---------|-------------|----------|
| `EXISTS` | ✅ Fast — stops at first match | Checking existence only |
| `NOT EXISTS` | ✅ Fast | Checking non-existence |
| `IN` | ⚠️ Medium — materializes subquery | Small subquery result set |
| `NOT IN` | ❌ Slow + NULL trap | Avoid — use `NOT EXISTS` instead |
| `JOIN` | ✅ Fast | Need columns from both tables |

> ⚠️ **NULL trap with NOT IN:** If the subquery returns ANY NULL, `NOT IN` returns no rows. Always use `NOT EXISTS` instead.

---

## 8. Implement Error Handling

### 🔑 TRY...CATCH Block

```sql
BEGIN TRY
    -- Code that might fail
    INSERT INTO Orders (CustomerID, Amount) VALUES (999, -100);
END TRY
BEGIN CATCH
    SELECT
        ERROR_NUMBER()    AS ErrorNumber,
        ERROR_SEVERITY()  AS Severity,
        ERROR_STATE()     AS State,
        ERROR_PROCEDURE() AS Procedure,
        ERROR_LINE()      AS Line,
        ERROR_MESSAGE()   AS Message;
END CATCH;
```

### 📐 Error Handling with Transactions

```sql
CREATE PROCEDURE usp_TransferFunds
    @FromAccountID INT,
    @ToAccountID   INT,
    @Amount        DECIMAL(18,2)
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        BEGIN TRANSACTION;

        UPDATE BankAccounts
        SET Balance = Balance - @Amount
        WHERE AccountID = @FromAccountID;

        IF (SELECT Balance FROM BankAccounts WHERE AccountID = @FromAccountID) < 0
            THROW 50001, 'Insufficient funds.', 1;

        UPDATE BankAccounts
        SET Balance = Balance + @Amount
        WHERE AccountID = @ToAccountID;

        COMMIT TRANSACTION;

    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;

        INSERT INTO ErrorLog (ErrorNumber, ErrorMessage, ErrorTime, Procedure)
        VALUES (ERROR_NUMBER(), ERROR_MESSAGE(), GETDATE(), ERROR_PROCEDURE());

        THROW;  -- Re-raise to caller
    END CATCH;
END;
```

### 📐 Custom Errors: THROW vs RAISERROR

```sql
-- THROW (SQL Server 2012+ — modern, preferred)
THROW 50001, 'Invalid customer ID provided.', 1;

-- Re-throw current error in CATCH block (no arguments)
THROW;

-- RAISERROR (legacy — still tested)
RAISERROR('Custom error: %s', 16, 1, 'Invalid input');
```

### 📊 Error Functions Reference

| Function | Returns |
|----------|---------|
| `ERROR_NUMBER()` | Error number (e.g., 2627 = duplicate key) |
| `ERROR_SEVERITY()` | Severity level (1–25) |
| `ERROR_STATE()` | State number (1–127) |
| `ERROR_PROCEDURE()` | Stored procedure name where error occurred |
| `ERROR_LINE()` | Line number where error occurred |
| `ERROR_MESSAGE()` | Full error message text |
| `XACT_STATE()` | Transaction state: 1=active, -1=uncommittable, 0=none |

### 🎯 XACT_STATE Pattern (Robust Error Handling)

```sql
BEGIN TRY
    BEGIN TRANSACTION;
    -- ... work ...
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF XACT_STATE() = -1
        ROLLBACK TRANSACTION;     -- Uncommittable state
    ELSE IF XACT_STATE() = 1
        ROLLBACK TRANSACTION;     -- Active but errored

    THROW;  -- Re-raise
END CATCH;
```

---

## ❓ Practice Questions

**Q1:** You need to traverse an employee hierarchy with variable depth (10+ levels). Which T-SQL construct is BEST?
- A) Nested subqueries
- B) **Recursive CTE** ✅
- C) Window functions
- D) Graph MATCH

> **Explanation:** Recursive CTEs are specifically designed for hierarchical/tree traversal with variable depth.

---

**Q2:** A report needs: each sale's amount, running total per region, and the region's overall total — in one query without grouping rows. Which feature is BEST?
- A) GROUP BY with ROLLUP
- B) Subqueries
- C) **Window functions (SUM() OVER)** ✅
- D) CTEs

---

**Q3:** Column `ProductData` contains `{"specs":{"weight":1.5},"tags":["sale","new"]}`. Which function extracts the `tags` array as a JSON string?
- A) `JSON_VALUE`
- B) `OPENJSON`
- C) **`JSON_QUERY`** ✅
- D) `JSON_ARRAYAGG`

> **Explanation:** `JSON_VALUE` returns scalars only; `JSON_QUERY` returns objects/arrays.

---

**Q4:** You want to find customers whose names are ≥ 85% similar to "John Smith". Which function returns a 0–100 similarity score?
- A) `EDIT_DISTANCE`
- B) **`EDIT_DISTANCE_SIMILARITY`** ✅
- C) `REGEXP_LIKE`
- D) `JARO_WINKLER_DISTANCE`

---

**Q5:** In a CATCH block, you want to re-raise the caught error to the caller. What is the correct syntax?
- A) `RAISERROR(ERROR_MESSAGE(), 16, 1)`
- B) `RETURN ERROR_NUMBER()`
- C) **`THROW;`** ✅ (no arguments)
- D) `RAISE ERROR_NUMBER()`

---

**Q6:** Which REGEXP function returns ALL non-overlapping matches as a multi-row result set?
- A) `REGEXP_INSTR`
- B) `REGEXP_SUBSTR`
- C) `REGEXP_COUNT`
- D) **`REGEXP_MATCHES`** ✅

---

**Q7:** You need to find 2-hop connections in a social graph (friends of friends). Which SQL feature is BEST suited?
- A) Recursive CTE with self-join
- B) Window functions
- C) **SQL Graph tables with MATCH operator** ✅
- D) CROSS APPLY

---

## 📚 Resources & References

| Resource | Link |
|----------|------|
| CTEs (WITH clause) | [learn.microsoft.com/sql/t-sql/queries/with-common-table-expression-transact-sql](https://learn.microsoft.com/sql/t-sql/queries/with-common-table-expression-transact-sql) |
| Window functions | [learn.microsoft.com/sql/t-sql/queries/select-over-clause-transact-sql](https://learn.microsoft.com/sql/t-sql/queries/select-over-clause-transact-sql) |
| JSON functions | [learn.microsoft.com/sql/relational-databases/json/json-data-sql-server](https://learn.microsoft.com/sql/relational-databases/json/json-data-sql-server) |
| OPENJSON | [learn.microsoft.com/sql/t-sql/functions/openjson-transact-sql](https://learn.microsoft.com/sql/t-sql/functions/openjson-transact-sql) |
| REGEXP functions (Azure SQL) | [learn.microsoft.com/azure/azure-sql/database/regular-expressions-support](https://learn.microsoft.com/azure/azure-sql/database/regular-expressions-support) |
| Fuzzy matching (Azure SQL) | [learn.microsoft.com/azure/azure-sql/database/fuzzy-string-matching](https://learn.microsoft.com/azure/azure-sql/database/fuzzy-string-matching) |
| SQL Graph | [learn.microsoft.com/sql/relational-databases/graphs/sql-graph-overview](https://learn.microsoft.com/sql/relational-databases/graphs/sql-graph-overview) |
| TRY...CATCH | [learn.microsoft.com/sql/t-sql/language-elements/try-catch-transact-sql](https://learn.microsoft.com/sql/t-sql/language-elements/try-catch-transact-sql) |
| THROW | [learn.microsoft.com/sql/t-sql/language-elements/throw-transact-sql](https://learn.microsoft.com/sql/t-sql/language-elements/throw-transact-sql) |

---

> 📌 **Study Tip:** Practice JSON functions (`FOR JSON`, `OPENJSON`) together — they're high-frequency in DP-800. Also test `REGEXP_*` and fuzzy matching in Azure SQL or Fabric SQL (not available in SQL Server on-prem).

---

*Last updated: September 2026 | DP-800 Skills measured as of March 12, 2026*
