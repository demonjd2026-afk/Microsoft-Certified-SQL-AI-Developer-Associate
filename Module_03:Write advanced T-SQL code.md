# DP-800 Exam Notes — Write Advanced T-SQL Code

> **Module:** Write advanced T-SQL code
> **Level:** Intermediate | **Roles:** Data Engineer, DBA, Developer, Solution Architect
> **Platforms Covered:** SQL Server 2019+, Azure SQL Database, Azure SQL Managed Instance, Microsoft Fabric

---

## Module Overview

This module covers 8 advanced T-SQL techniques:

| Unit | Topic |
|---|---|
| 1 | Common Table Expressions (CTEs) — non-recursive and recursive |
| 2 | Window Functions — ranking, aggregate, analytical |
| 3 | JSON Functions — parse, construct, validate, optimise |
| 4 | Regular Expressions — pattern matching and text manipulation |
| 5 | Fuzzy String Matching — approximate match functions |
| 6 | Graph Queries — MATCH operator and SHORTEST_PATH |
| 7 | Correlated Subqueries — row-by-row comparisons |
| 8 | Error Handling — TRY...CATCH, THROW, transactions |

---

## Unit 1 — Common Table Expressions (CTEs)

### What Is a CTE?
- A **temporary named result set** defined using the `WITH` clause.
- Exists **only during the execution of a single query** — does not persist; requires no explicit cleanup.
- Can be referenced in `SELECT`, `INSERT`, `UPDATE`, or `DELETE` statements.

### Advantages Over Subqueries / Derived Tables
- **Improved readability** — complex logic broken into named logical sections.
- **Self-referencing** — recursive CTEs can reference themselves for hierarchical data.
- **Multiple references** — reference the same CTE multiple times without redefining it.
- **Modular design** — build queries incrementally using chained CTEs.

---

### Basic CTE Syntax

```sql
WITH CTE_Name (Column1, Column2)
AS
(
    SELECT Column1, Column2
    FROM SomeTable
    WHERE SomeCondition = 'Value'
)
SELECT * FROM CTE_Name;
```

---

### Non-Recursive CTEs

Used to simplify complex joins, multi-step calculations, or improve code organisation.

```sql
WITH SalesSummary AS
(
    SELECT ProductID,
           SUM(OrderQty)         AS TotalQuantity,
           SUM(LineTotal)        AS TotalRevenue,
           COUNT(DISTINCT SalesOrderID) AS OrderCount
    FROM SalesLT.SalesOrderDetail
    GROUP BY ProductID
)
SELECT p.Name, ss.TotalQuantity, ss.TotalRevenue,
       ss.TotalRevenue / NULLIF(ss.TotalQuantity, 0) AS AverageUnitPrice
FROM SalesLT.Product AS p
INNER JOIN SalesSummary AS ss ON p.ProductID = ss.ProductID
ORDER BY ss.TotalRevenue DESC;
```

> Use `NULLIF(ss.TotalQuantity, 0)` to prevent division-by-zero errors.

#### Multiple CTEs in One `WITH` Clause

Later CTEs can reference earlier ones — enables progressive data transformation:

```sql
WITH CategorySales AS ( ... ),
RankedCategories AS
(
    SELECT ProductCategoryID, CategoryRevenue,
           RANK() OVER (ORDER BY CategoryRevenue DESC) AS RevenueRank
    FROM CategorySales
)
SELECT ... FROM RankedCategories WHERE RevenueRank <= 5;
```

> **Tip:** Start with the most granular data transformations and aggregate progressively — easier to test and debug each CTE independently.

---

### Recursive CTEs

Reference themselves to process **hierarchical or graph-like data** (org charts, bill-of-materials, date sequences).

**Two mandatory parts:**
1. **Anchor member** — initial non-recursive result set (starting point).
2. **Recursive member** — references the CTE itself, connected via `UNION ALL`.

```sql
WITH RecursiveCTE AS
(
    -- Anchor member
    SELECT columns FROM table WHERE starting_condition

    UNION ALL

    -- Recursive member
    SELECT columns FROM table
    INNER JOIN RecursiveCTE ON join_condition
)
SELECT * FROM RecursiveCTE;
```

#### Org Hierarchy Example

```sql
WITH EmployeeHierarchy AS
(
    -- Anchor: top-level managers (no manager)
    SELECT EmployeeID, FirstName, LastName, ManagerID,
           0 AS Level,
           CAST(FirstName + ' ' + LastName AS NVARCHAR(500)) AS HierarchyPath
    FROM HumanResources.Employee
    WHERE ManagerID IS NULL

    UNION ALL

    -- Recursive: employees reporting to previously found employees
    SELECT e.EmployeeID, e.FirstName, e.LastName, e.ManagerID,
           eh.Level + 1,
           CAST(eh.HierarchyPath + ' > ' + e.FirstName + ' ' + e.LastName AS NVARCHAR(500))
    FROM HumanResources.Employee AS e
    INNER JOIN EmployeeHierarchy AS eh ON e.ManagerID = eh.EmployeeID
)
SELECT EmployeeID, FirstName, LastName, Level, HierarchyPath
FROM EmployeeHierarchy
ORDER BY HierarchyPath;
```

> ⚠️ **Important:** Recursive CTEs can cause **infinite loops** if the termination condition is never met.
> - Default recursion limit: **100 levels**.
> - Change with: `OPTION (MAXRECURSION n)` — use `0` for unlimited.

#### Generating Date/Number Sequences

Recursive CTEs can generate sequences without a physical numbers table:

```sql
WITH DateSequence AS
(
    SELECT CAST(DATEADD(MONTH, DATEDIFF(MONTH, 0, GETDATE()), 0) AS DATE) AS DateValue
    UNION ALL
    SELECT DATEADD(DAY, 1, DateValue)
    FROM DateSequence
    WHERE DateValue < EOMONTH(GETDATE())
)
SELECT DateValue, DATENAME(WEEKDAY, DateValue) AS DayName
FROM DateSequence
OPTION (MAXRECURSION 31);
```

---

### CTEs with Data Modification Statements

CTEs can be used with `INSERT`, `UPDATE`, and `DELETE` — not just `SELECT`:

```sql
WITH DiscontinuedProducts AS
(
    SELECT ProductID FROM SalesLT.Product
    WHERE SellEndDate < DATEADD(YEAR, -2, GETDATE())
      AND ProductID NOT IN (
          SELECT DISTINCT ProductID FROM SalesLT.SalesOrderDetail
          WHERE ModifiedDate > DATEADD(YEAR, -1, GETDATE())
      )
)
UPDATE SalesLT.Product
SET DiscontinuedDate = GETDATE()
WHERE ProductID IN (SELECT ProductID FROM DiscontinuedProducts);
```

---

## Unit 2 — Window Functions

### What Are Window Functions?
- Perform calculations across a **"window" of rows** related to the current row.
- Unlike `GROUP BY` aggregate functions, window functions **do NOT collapse rows** — all original rows are preserved in the result.
- Defined using the `OVER` clause.

### OVER Clause Syntax

```
function_name(arguments) OVER (
    [PARTITION BY partition_expression]
    [ORDER BY order_expression [ASC | DESC]]
    [ROWS | RANGE frame_specification]
)
```

| Component | Purpose |
|---|---|
| `PARTITION BY` | Divides rows into independent groups for the calculation |
| `ORDER BY` | Logical order of rows within each partition |
| `ROWS / RANGE` | Frame boundaries relative to the current row |

> **Default frame** when `ORDER BY` is present: `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` (cumulative calculation).

---

### Ranking Functions

| Function | Ties Handling | Gaps in Sequence |
|---|---|---|
| `ROW_NUMBER()` | Unique number even for ties | N/A — always sequential |
| `RANK()` | Same rank for ties | ✅ Yes — skips numbers after ties |
| `DENSE_RANK()` | Same rank for ties | ❌ No — consecutive numbers |
| `NTILE(n)` | Distributes rows into n groups | N/A |

```sql
-- ROW_NUMBER: unique sequential number per row
SELECT ProductID, Name, ListPrice,
       ROW_NUMBER() OVER (ORDER BY ListPrice DESC) AS PriceRank
FROM SalesLT.Product WHERE ListPrice > 0;

-- RANK: ties get same rank, next rank skips (1,1,1,4...)
SELECT ProductID, Name, ListPrice,
       RANK() OVER (ORDER BY ListPrice DESC) AS PriceRank
FROM SalesLT.Product WHERE ListPrice > 0;

-- DENSE_RANK: ties get same rank, no skipping (1,1,1,2...)
SELECT ProductID, Name, ListPrice,
       DENSE_RANK() OVER (ORDER BY ListPrice DESC) AS PriceRank
FROM SalesLT.Product WHERE ListPrice > 0;

-- NTILE(4): divide into 4 quartile groups
SELECT ProductID, Name, ListPrice,
       NTILE(4) OVER (ORDER BY ListPrice DESC) AS PriceQuartile
FROM SalesLT.Product WHERE ListPrice > 0;
```

#### Ranking Within Partitions (Per-Group)

```sql
SELECT pc.Name AS Category, p.Name AS Product, p.ListPrice,
       ROW_NUMBER() OVER (
           PARTITION BY p.ProductCategoryID
           ORDER BY p.ListPrice DESC
       ) AS CategoryPriceRank
FROM SalesLT.Product AS p
INNER JOIN SalesLT.ProductCategory AS pc ON p.ProductCategoryID = pc.ProductCategoryID;
-- CategoryPriceRank = 1 identifies the most expensive product per category
```

> **Tip:** Use `ROW_NUMBER()` when you need exactly one row per rank (top-N per group). Use `RANK()` or `DENSE_RANK()` when preserving tie information matters.

---

### Aggregate Window Functions

Standard aggregates (`SUM`, `AVG`, `COUNT`, `MIN`, `MAX`) used with `OVER` to compute values **while retaining individual rows**:

```sql
SELECT SalesOrderID, OrderDate, TotalDue,
       SUM(TotalDue)   OVER (ORDER BY OrderDate, SalesOrderID) AS RunningTotal,
       AVG(TotalDue)   OVER (ORDER BY OrderDate, SalesOrderID) AS RunningAverage,
       COUNT(*)        OVER (ORDER BY OrderDate, SalesOrderID) AS OrderNumber
FROM SalesLT.SalesOrderHeader
ORDER BY OrderDate, SalesOrderID;
```

> Without `ORDER BY` in `OVER`, the aggregate calculates across the **entire partition**. Adding `ORDER BY` creates a **running/cumulative** calculation.

---

### Window Frames: ROWS vs RANGE

Frame boundaries:
- `UNBOUNDED PRECEDING` — from partition start
- `n PRECEDING` — n rows before current row
- `CURRENT ROW` — current row
- `n FOLLOWING` — n rows after current row
- `UNBOUNDED FOLLOWING` — to partition end

**`ROWS`** — counts **physical rows**. **`RANGE`** — groups rows with equal values.

```sql
-- 3-order moving average (current row + 2 preceding rows)
SELECT SalesOrderID, OrderDate, TotalDue,
       AVG(TotalDue) OVER (
           ORDER BY OrderDate
           ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
       ) AS MovingAvg3Orders
FROM SalesLT.SalesOrderHeader;
```

---

### Analytical Functions

Access values from **other rows** without self-joins or subqueries.

#### LAG() and LEAD()

```sql
SELECT SalesOrderID, OrderDate, TotalDue,
       LAG(TotalDue, 1, 0)  OVER (ORDER BY OrderDate) AS PreviousOrderTotal,
       LEAD(TotalDue, 1, 0) OVER (ORDER BY OrderDate) AS NextOrderTotal,
       TotalDue - LAG(TotalDue, 1, 0) OVER (ORDER BY OrderDate) AS ChangeFromPrevious
FROM SalesLT.SalesOrderHeader;
```

- `LAG(column, offset, default)` — value from **previous** row (offset rows back).
- `LEAD(column, offset, default)` — value from **next** row (offset rows ahead).
- Third parameter = default when no row exists (e.g., first row with `LAG`).

#### FIRST_VALUE() and LAST_VALUE()

```sql
SELECT ProductID, Name, ListPrice, ProductCategoryID,
       FIRST_VALUE(Name) OVER (
           PARTITION BY ProductCategoryID ORDER BY ListPrice DESC
       ) AS MostExpensiveInCategory,
       LAST_VALUE(Name) OVER (
           PARTITION BY ProductCategoryID ORDER BY ListPrice DESC
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS LeastExpensiveInCategory
FROM SalesLT.Product WHERE ListPrice > 0;
```

> ⚠️ `LAST_VALUE()` **requires an explicit frame** `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`. Without it, the default frame only includes rows up to the current row, making `LAST_VALUE()` return the **current row's value**.

#### PERCENT_RANK() and CUME_DIST()

```sql
SELECT Name, ListPrice,
       PERCENT_RANK() OVER (ORDER BY ListPrice) AS PercentRank,
       CUME_DIST()    OVER (ORDER BY ListPrice) AS CumulativeDistribution
FROM SalesLT.Product WHERE ListPrice > 0;
```

- `PERCENT_RANK()` — value between 0 and 1 (0 = lowest, 1 = highest). What % of rows have lower values.
- `CUME_DIST()` — cumulative distribution: what % of rows have values ≤ current row.

---

## Unit 3 — JSON Functions

### Extracting Values

| Function | Returns | Use When |
|---|---|---|
| `JSON_VALUE(json, '$.path')` | Scalar value as `NVARCHAR(4000)` | Need a single scalar value |
| `JSON_QUERY(json, '$.path')` | JSON object or array (preserves structure) | Need nested objects/arrays |

```sql
-- Extract scalar value
SELECT JSON_VALUE(@json, '$.customer.id')    AS CustomerID,
       JSON_VALUE(@json, '$.customer.name')  AS CustomerName,
       JSON_VALUE(@json, '$.orderTotal')     AS OrderTotal;

-- Extract object/array (preserves JSON structure)
SELECT JSON_QUERY(@json, '$.customer') AS CustomerObject,
       JSON_QUERY(@json, '$.items')    AS ItemsArray;

-- Access array element by 0-based index
SELECT JSON_VALUE(@json, '$.items[0].product') AS FirstProduct;
```

---

### Parsing JSON Arrays with OPENJSON

`OPENJSON` is a **table-valued function** that transforms JSON into a relational rowset.

#### Default Schema (no WITH clause)

```sql
SELECT * FROM OPENJSON(@json);
-- Returns: key (index/property name), value (JSON content), type (0=null,1=string,2=number,3=boolean,4=array,5=object)
```

#### Explicit Schema (WITH clause)

```sql
SELECT ProductID, ProductName, Price
FROM OPENJSON(@json)
WITH (
    ProductID   INT             '$.id',
    ProductName NVARCHAR(100)   '$.name',
    Price       DECIMAL(10,2)   '$.price'
);
```

#### OPENJSON with CROSS APPLY

```sql
SELECT o.OrderID, o.CustomerID, items.ProductName, items.Quantity, items.UnitPrice
FROM Orders AS o
CROSS APPLY OPENJSON(o.OrderDetails)
WITH (
    ProductName NVARCHAR(100) '$.product',
    Quantity    INT            '$.qty',
    UnitPrice   DECIMAL(10,2)  '$.price'
) AS items;
```

> Use `OUTER APPLY` instead of `CROSS APPLY` if you need to include rows where the JSON column is NULL or empty.

---

### Constructing JSON (SQL Server 2022+ / Azure SQL / Fabric)

#### JSON_OBJECT() — Build a JSON object

```sql
SELECT JSON_OBJECT(
    'id':    ProductID,
    'name':  Name,
    'price': ListPrice
) AS ProductJson
FROM SalesLT.Product WHERE ProductID = 680;
```

#### JSON_ARRAY() — Build a JSON array

```sql
SELECT JSON_ARRAY('SQL Server', 'Azure SQL Database', 'SQL Database in Fabric') AS Platforms;
```

#### Nested JSON — Combine JSON_OBJECT calls

```sql
SELECT JSON_OBJECT(
    'orderId':   soh.SalesOrderID,
    'customer':  JSON_OBJECT('id': c.CustomerID, 'name': c.CompanyName),
    'totals':    JSON_OBJECT('subtotal': soh.SubTotal, 'tax': soh.TaxAmt, 'total': soh.TotalDue)
) AS OrderJson
FROM SalesLT.SalesOrderHeader AS soh
INNER JOIN SalesLT.Customer AS c ON soh.CustomerID = c.CustomerID;
```

#### JSON_ARRAYAGG() — Aggregate rows into a JSON array

```sql
SELECT c.CustomerID, c.CompanyName,
       JSON_ARRAYAGG(soh.SalesOrderID) AS OrderIds
FROM SalesLT.Customer AS c
INNER JOIN SalesLT.SalesOrderHeader AS soh ON c.CustomerID = soh.CustomerID
GROUP BY c.CustomerID, c.CompanyName;
-- Result: {"CustomerID": 29825, "OrderIds": [71774,71776,71780]}
```

> ⚠️ `JSON_OBJECT`, `JSON_ARRAY`, `JSON_ARRAYAGG` require **SQL Server 2022+**, Azure SQL Database, or Fabric. Use `FOR JSON PATH` for earlier versions.

---

### Validating JSON

#### Path Modes: lax vs strict

```sql
-- lax (default): returns NULL for missing paths
SELECT JSON_VALUE(@json, 'lax $.description') AS LaxResult;

-- strict: raises an error for missing paths
SELECT JSON_VALUE(@json, 'strict $.description') AS StrictResult;
```

#### Validation Functions

| Function | Returns | Use For |
|---|---|---|
| `ISJSON(string)` | 1 (valid), 0 (invalid), NULL (null input) | Check if a string is valid JSON |
| `JSON_PATH_EXISTS(json, '$.path')` | 1 (exists), 0 (not exists) | Check if a path exists before extracting |
| `JSON_CONTAINS(json, value, '$.path')` | 1 (contains), 0 (not contains) | Check if JSON contains a specific value |

```sql
-- ISJSON validation
SELECT ISJSON('{"name": "test"}') AS ValidJson,   -- 1
       ISJSON('not valid json')   AS InvalidJson;  -- 0

-- JSON_PATH_EXISTS
SELECT JSON_PATH_EXISTS(@json, '$.customer.name')  AS HasName,   -- 1
       JSON_PATH_EXISTS(@json, '$.customer.email') AS HasEmail;  -- 0

-- JSON_CONTAINS
SELECT JSON_CONTAINS(@json, '"sql"',    '$.tags') AS HasSqlTag,    -- 1
       JSON_CONTAINS(@json, '"python"', '$.tags') AS HasPythonTag; -- 0
```

---

### Optimising JSON with Computed Columns + Indexes

JSON parsing on every query = full table scan overhead. Solution: **computed columns** + **indexes**.

```sql
-- Add computed column extracting JSON property
ALTER TABLE Products
ADD ProductCategory AS JSON_VALUE(ProductData, '$.category');

-- Persisted: parses JSON only on INSERT/UPDATE, not SELECT
ALTER TABLE Products
ADD ProductCategory AS JSON_VALUE(ProductData, '$.category') PERSISTED;

-- Index the computed column for fast filtering
CREATE INDEX IX_Products_Category ON Products(ProductCategory);

-- Composite index for multi-property queries
ALTER TABLE Products
ADD ProductBrand AS JSON_VALUE(ProductData, '$.brand') PERSISTED,
    ProductPrice AS CAST(JSON_VALUE(ProductData, '$.price') AS DECIMAL(10,2)) PERSISTED;

CREATE INDEX IX_Products_Category_Brand ON Products(ProductCategory, ProductBrand);
```

> **Tip:** Create computed columns + indexes for JSON properties used in `WHERE`, `JOIN`, or `ORDER BY` clauses.

---

### FOR JSON PATH — Convert Relational Data to JSON

```sql
SELECT p.ProductID, p.Name, p.ListPrice, pc.Name AS CategoryName
FROM SalesLT.Product AS p
INNER JOIN SalesLT.ProductCategory AS pc ON p.ProductCategoryID = pc.ProductCategoryID
WHERE p.ListPrice > 1000
FOR JSON PATH, ROOT('products');
-- Wraps result in a root element named "products"
```

Use dot notation in column aliases to create nested objects:

```sql
SELECT p.ProductID AS 'product.id',
       p.Name      AS 'product.name',
       pc.Name     AS 'product.category'
FROM SalesLT.Product AS p
INNER JOIN SalesLT.ProductCategory AS pc ON p.ProductCategoryID = pc.ProductCategoryID
FOR JSON PATH;
-- Creates: [{"product": {"id": 680, "name": "...", "category": "..."}}]
```

---

## Unit 4 — Regular Expressions

> ⚠️ **Platform availability:** Regular expression functions available in **SQL Server 2025+** and **SQL databases in Microsoft Fabric**. For earlier versions, use CLR functions or application-layer processing.

### Common Regex Pattern Reference

| Pattern | Description | Example Match |
|---|---|---|
| `.` | Any single character | `a.c` → "abc", "a1c" |
| `*` | Zero or more of preceding | `ab*c` → "ac", "abc", "abbc" |
| `+` | One or more of preceding | `ab+c` → "abc", "abbc" (not "ac") |
| `?` | Zero or one of preceding | `colou?r` → "color", "colour" |
| `^` | Start of string | `^Hello` → strings starting with "Hello" |
| `$` | End of string | `world$` → strings ending with "world" |
| `[abc]` | Character class | `[aeiou]` → any vowel |
| `[^abc]` | Negated class | `[^0-9]` → non-digits |
| `\d` | Digit (0–9) | `\d{3}` → three digits |
| `\w` | Word character | `\w+` → word characters |
| `{n}` | Exactly n occurrences | `\d{4}` → exactly four digits |
| `{n,m}` | Between n and m occurrences | `\d{2,4}` → 2 to 4 digits |

> SQL Server regex uses **ECMAScript standard** syntax (same as JavaScript).

---

### Regex Functions

#### REGEXP_LIKE — Pattern matching / filtering

```sql
-- Filter by email domain
SELECT CustomerID, FirstName, LastName, EmailAddress
FROM SalesLT.Customer
WHERE REGEXP_LIKE(EmailAddress, '@(contoso|adventure-works|fabrikam)\.com$') = 1;

-- Case-insensitive matching with 'i' flag
SELECT Name FROM SalesLT.Product
WHERE REGEXP_LIKE(Name, 'frame', 'i') = 1;
```

#### REGEXP_REPLACE — Find and replace with patterns

```sql
-- Standardise phone numbers: remove non-digits, reformat
SELECT Phone AS OriginalPhone,
       REGEXP_REPLACE(
           REGEXP_REPLACE(Phone, '[^\d]', ''),
           '^(\d{3})(\d{3})(\d{4})$',
           '($1) $2-$3'
       ) AS StandardizedPhone
FROM SalesLT.Customer WHERE Phone IS NOT NULL;

-- Remove HTML tags
SELECT REGEXP_REPLACE(HtmlContent, '<[^>]+>', '') AS PlainText FROM WebPages;

-- Remove extra whitespace
SELECT REGEXP_REPLACE(Description, '\s+', ' ') AS CleanedDescription FROM Products;
```

#### REGEXP_SUBSTR — Extract matched substring

```sql
-- Extract domain from email
SELECT EmailAddress,
       REGEXP_SUBSTR(EmailAddress, '@(.+)$', 1, 1, '', 1) AS Domain
FROM SalesLT.Customer;
-- Signature: REGEXP_SUBSTR(source, pattern, start_position, occurrence, flags, capture_group)
```

#### REGEXP_INSTR — Find position of match

```sql
-- Returns 0 if no match found
SELECT ProductNumber,
       REGEXP_INSTR(ProductNumber, '\d') AS FirstDigitPosition
FROM SalesLT.Product;
```

#### REGEXP_COUNT — Count occurrences

```sql
-- Count words
SELECT Name, REGEXP_COUNT(Name, '\w+') AS WordCount FROM SalesLT.Product;

-- Count vowels (case-insensitive)
SELECT Name, REGEXP_COUNT(Name, '[aeiou]', 1, 'i') AS VowelCount FROM SalesLT.Product;
```

#### REGEXP_SPLIT_TO_TABLE — Split string into rows (TVF)

```sql
-- Split comma-separated values
SELECT value AS Tag FROM REGEXP_SPLIT_TO_TABLE(@tags, ',');

-- Split on multiple delimiters
SELECT value AS Fruit FROM REGEXP_SPLIT_TO_TABLE(@data, '[,;|]');

-- Combine with CROSS APPLY
SELECT p.ProductID, p.Name, t.value AS Tag
FROM Products AS p
CROSS APPLY REGEXP_SPLIT_TO_TABLE(p.Tags, ',\s*') AS t;
```

#### REGEXP_MATCHES — Return all matches as rows (TVF)

```sql
DECLARE @text NVARCHAR(200) = 'Order 12345 contains 3 items totaling $99.99';
SELECT match_value, match_index
FROM REGEXP_MATCHES(@text, '\d+\.?\d*');
-- Returns: 12345, 3, 99.99
```

---

## Unit 5 — Fuzzy String Matching

> ⚠️ **Platform availability:** `EDIT_DISTANCE`, `EDIT_DISTANCE_SIMILARITY`, `JARO_WINKLER_DISTANCE` available in **SQL Server 2025+**, Azure SQL Database, and Fabric.

### Two Approaches

| Approach | Description |
|---|---|
| **Edit distance (Levenshtein)** | Minimum single-character operations (insert, delete, substitute) to transform one string to another. Lower = more similar. |
| **Similarity score** | Percentage/ratio expressing similarity. Higher = more similar. |

---

### EDIT_DISTANCE — Levenshtein Distance

Returns the **minimum number of edits** needed to transform one string into another.

```sql
SELECT EDIT_DISTANCE('color',      'colour')    AS ColorVariant,  -- 1
       EDIT_DISTANCE('database',   'databaes')  AS Typo,          -- 2
       EDIT_DISTANCE('SQL Server', 'SQL Server') AS Exact,        -- 0
       EDIT_DISTANCE('hello',      'world')     AS Different;     -- 4
```

```sql
-- Find customers with similar names (edit distance ≤ 3)
DECLARE @searchName NVARCHAR(100) = 'Jon Smith';
SELECT CustomerID, FirstName + ' ' + LastName AS FullName,
       EDIT_DISTANCE(@searchName, FirstName + ' ' + LastName) AS EditDistance
FROM SalesLT.Customer
WHERE EDIT_DISTANCE(@searchName, FirstName + ' ' + LastName) <= 3
ORDER BY EditDistance;
```

> **Tip:** For short strings (5–10 chars), edit distance 1–2 indicates similarity. For longer strings, allow 3–5.

---

### EDIT_DISTANCE_SIMILARITY — Normalised Score (0–100)

Returns a percentage — **100 = identical**, easier to interpret across different string lengths.

```sql
SELECT EDIT_DISTANCE_SIMILARITY('color',   'colour')    AS ColorSimilarity,  -- ~85
       EDIT_DISTANCE_SIMILARITY('hello',   'hello')     AS Exact,            -- 100
       EDIT_DISTANCE_SIMILARITY('SQL',     'SQL Server') AS PartialMatch;    -- ~30
```

```sql
-- Find products at least 70% similar to a search term
DECLARE @searchTerm NVARCHAR(100) = 'Mountain Bike Frame';
SELECT ProductID, Name,
       EDIT_DISTANCE_SIMILARITY(@searchTerm, Name) AS SimilarityScore
FROM SalesLT.Product
WHERE EDIT_DISTANCE_SIMILARITY(@searchTerm, Name) >= 70
ORDER BY SimilarityScore DESC;
```

---

### JARO_WINKLER_DISTANCE — Phonetic Similarity (0–1)

- Designed specifically for **names and short strings**.
- Gives **higher scores to strings matching from the beginning** — prefixes weighted more.
- Score range: 0 to 1 (1 = identical). Score > **0.9** typically indicates a strong name match.

```sql
SELECT JARO_WINKLER_DISTANCE('MARTHA',  'MARHTA')   AS NameTypo,      -- ~0.96
       JARO_WINKLER_DISTANCE('JONES',   'JOHNSON')  AS SimilarNames,  -- ~0.83
       JARO_WINKLER_DISTANCE('JOHN',    'JON')      AS NameVariant,   -- ~0.93
       JARO_WINKLER_DISTANCE('SMITH',   'SMYTH')    AS SpellingVar;   -- ~0.96
```

> **Note:** Jaro-Winkler is best for **short strings like names**. For longer strings (addresses, descriptions), use `EDIT_DISTANCE_SIMILARITY`.

---

### Performance: Pre-filter Before Fuzzy Matching

Fuzzy functions compute character-by-character — expensive on large tables.

```sql
-- ❌ Bad: fuzzy match against entire table
SELECT * FROM LargeCustomerTable
WHERE EDIT_DISTANCE_SIMILARITY('John Smith', FullName) > 70;

-- ✅ Better: pre-filter with LIKE first, then fuzzy match
SELECT * FROM LargeCustomerTable
WHERE FullName LIKE 'J%'
  AND EDIT_DISTANCE_SIMILARITY('John Smith', FullName) > 70;

-- ✅ Best: multiple pre-filters to minimise candidate set
SELECT * FROM LargeCustomerTable
WHERE FirstName LIKE 'Jo%' AND LastName LIKE 'Sm%'
  AND JARO_WINKLER_DISTANCE('John', FirstName) > 0.85
  AND JARO_WINKLER_DISTANCE('Smith', LastName) > 0.85;
```

---

## Unit 6 — Graph Queries

> **Platform availability:** `MATCH` operator — SQL Server 2017+, Azure SQL Database. `SHORTEST_PATH` — SQL Server 2019+.

### Graph Concepts

- **Node tables** — store entities (people, products, locations). Created with `AS NODE`. Auto-includes `$node_id`.
- **Edge tables** — store relationships between nodes. Created with `AS EDGE`. Auto-includes `$edge_id`, `$from_id`, `$to_id`.
- The **`MATCH`** clause provides pattern-based relationship traversal using ASCII arrow syntax.

---

### Creating Node Tables

```sql
CREATE TABLE dbo.Person (
    PersonID   INT PRIMARY KEY,
    Name       NVARCHAR(100) NOT NULL,
    Department NVARCHAR(50)
) AS NODE;

CREATE TABLE dbo.Product (
    ProductID INT PRIMARY KEY,
    Name      NVARCHAR(200) NOT NULL,
    Category  NVARCHAR(100)
) AS NODE;
```

- SQL Server **auto-generates** `$node_id` — do not include in `INSERT` statements.

---

### Creating Edge Tables

```sql
CREATE TABLE dbo.ReportsTo (
    StartDate  DATE,
    ReportType NVARCHAR(50)
) AS EDGE;

CREATE TABLE dbo.Knows (
    ConnectionDate     DATE,
    ConnectionStrength INT  -- 1-10 scale
) AS EDGE;
```

Inserting edges using `$node_id` subqueries:

```sql
INSERT INTO dbo.ReportsTo ($from_id, $to_id, StartDate, ReportType)
SELECT
    (SELECT $node_id FROM dbo.Person WHERE Name = 'Alice Johnson'),
    (SELECT $node_id FROM dbo.Person WHERE Name = 'Bob Smith'),
    '2023-01-15', 'Direct';
```

---

### Querying with MATCH

Arrow direction matters:
- `(Node1)-(Edge)->(Node2)` — edge goes from Node1 **to** Node2
- `(Node1)<-(Edge)-(Node2)` — edge goes from Node2 **to** Node1

```sql
-- Who reports to whom?
SELECT Employee.Name AS Employee, Manager.Name AS Manager, r.StartDate
FROM dbo.Person AS Employee, dbo.ReportsTo AS r, dbo.Person AS Manager
WHERE MATCH(Employee-(r)->Manager);

-- Who knows Alice?
SELECT Connector.Name AS PersonWhoKnowsAlice, k.ConnectionStrength
FROM dbo.Person AS Connector, dbo.Knows AS k, dbo.Person AS Target
WHERE MATCH(Connector-(k)->Target)
  AND Target.Name = 'Alice Johnson';
```

---

### Multi-Hop Traversals

```sql
-- Friends of friends (2-hop)
SELECT DISTINCT Person1.Name AS Person, Person3.Name AS FriendOfFriend
FROM dbo.Person AS Person1, dbo.Knows AS k1, dbo.Person AS Person2,
     dbo.Knows AS k2, dbo.Person AS Person3
WHERE MATCH(Person1-(k1)->Person2-(k2)->Person3)
  AND Person1.Name = 'Alice Johnson'
  AND Person3.Name <> Person1.Name;
```

> ⚠️ Each **edge table alias can only appear once** in a single `MATCH` pattern. Use separate aliases for multiple traversals of the same edge type.

---

### SHORTEST_PATH — Variable-Length Traversals

```sql
SELECT StartPerson.Name,
       LAST_VALUE(ReachablePerson.Name)    WITHIN GROUP (GRAPH PATH) AS ReachablePerson,
       COUNT(ReachablePerson.Name)         WITHIN GROUP (GRAPH PATH) AS Distance
FROM dbo.Person AS StartPerson,
     dbo.Knows FOR PATH AS k,
     dbo.Person FOR PATH AS ReachablePerson
WHERE MATCH(SHORTEST_PATH(StartPerson(-(k)->ReachablePerson){1,3}))
  AND StartPerson.Name = 'Alice Johnson';
```

- `FOR PATH` keyword marks tables participating in variable-length matching.
- `{1,3}` = 1 to 3 hops. Use `+` for one or more hops.

---

### Graph vs Relational — When to Use Each

| Use Graph When... | Use Relational When... |
|---|---|
| Relationships are the primary query focus | Relationships are simple, fixed-depth (parent-child) |
| Variable/unknown traversal depth (friends of friends) | Primarily filtering/aggregating entity attributes |
| Data naturally forms a network | Data model is mostly tabular with few relationships |
| Many self-joins would be needed in relational SQL | Performance critical; indexes on FK are sufficient |
| Path-finding or connectivity analysis | Team more familiar with traditional SQL |

---

### Common Graph Query Troubleshooting

| Problem | Cause | Solution |
|---|---|---|
| No results | Arrow direction doesn't match edge insertion direction | Verify `$from_id` / `$to_id` order when inserting |
| Syntax error with repeated edge | Same edge alias used multiple times in one MATCH | Create separate aliases per traversal |
| SHORTEST_PATH fails | Tables not marked with `FOR PATH` | Add `FOR PATH` to participating tables |
| Edge references wrong nodes | Business key used instead of `$node_id` | Use subqueries to select `$node_id` when inserting edges |

---

## Unit 7 — Correlated Subqueries

### What Is a Correlated Subquery?
- A subquery that **references columns from the outer query**.
- Executes **once for each row** processed by the outer query (like a nested loop).
- Unlike non-correlated subqueries that execute once and return a fixed result.

```sql
-- Non-correlated (executes once): single AVG across all products
WHERE ListPrice > (SELECT AVG(ListPrice) FROM SalesLT.Product)

-- Correlated (executes per row): AVG per category for each product's category
WHERE p1.ListPrice > (
    SELECT AVG(p2.ListPrice) FROM SalesLT.Product AS p2
    WHERE p2.ProductCategoryID = p1.ProductCategoryID  -- references outer query
)
```

> The query optimiser often internally transforms correlated subqueries into equivalent joins, but understanding the logical behaviour helps write correct queries.

---

### Correlated Subqueries in WHERE — Row-Specific Filtering

```sql
-- Products priced above their own category's average
SELECT p.ProductID, p.Name, p.ListPrice, pc.Name AS Category
FROM SalesLT.Product AS p
INNER JOIN SalesLT.ProductCategory AS pc ON p.ProductCategoryID = pc.ProductCategoryID
WHERE p.ListPrice > (
    SELECT AVG(p2.ListPrice) FROM SalesLT.Product AS p2
    WHERE p2.ProductCategoryID = p.ProductCategoryID
)
ORDER BY pc.Name, p.ListPrice DESC;
```

---

### EXISTS and NOT EXISTS

- Tests whether **any matching rows exist** in a related table — returns true/false.
- Uses `SELECT 1` because actual values don't matter — only presence/absence.
- **Stops after finding the first match** — more efficient than `IN` for large tables.

```sql
-- Customers who have placed at least one order
SELECT CustomerID, FirstName, LastName FROM SalesLT.Customer AS c
WHERE EXISTS (SELECT 1 FROM SalesLT.SalesOrderHeader AS soh WHERE soh.CustomerID = c.CustomerID);

-- Customers who have NEVER placed an order
WHERE NOT EXISTS (SELECT 1 FROM SalesLT.SalesOrderHeader AS soh WHERE soh.CustomerID = c.CustomerID);

-- Categories where ALL products are priced above $100
SELECT pc.ProductCategoryID, pc.Name FROM SalesLT.ProductCategory AS pc
WHERE NOT EXISTS (
    SELECT 1 FROM SalesLT.Product AS p
    WHERE p.ProductCategoryID = pc.ProductCategoryID AND p.ListPrice <= 100
);
```

> **Tip:** `EXISTS` typically outperforms `IN` with subqueries on large tables — the optimiser can stop after the first match.

---

### Correlated Subqueries in SELECT — Per-Row Calculations

```sql
-- Each product with its category's average price
SELECT p.ProductID, p.Name, p.ListPrice,
    (SELECT AVG(p2.ListPrice) FROM SalesLT.Product AS p2
     WHERE p2.ProductCategoryID = p.ProductCategoryID) AS CategoryAvgPrice,
    p.ListPrice -
    (SELECT AVG(p2.ListPrice) FROM SalesLT.Product AS p2
     WHERE p2.ProductCategoryID = p.ProductCategoryID) AS DifferenceFromAvg
FROM SalesLT.Product AS p;
```

> ⚠️ Correlated subqueries in `SELECT` **must return exactly one value**. Wrap in an aggregate (`MAX`, `MIN`, `SUM`, `COUNT`) if multiple rows could be returned.

---

### Top-N Per Group

```sql
-- Top 3 most expensive products per category
SELECT pc.Name AS Category, p.Name AS Product, p.ListPrice
FROM SalesLT.Product AS p
INNER JOIN SalesLT.ProductCategory AS pc ON p.ProductCategoryID = pc.ProductCategoryID
WHERE p.ProductID IN (
    SELECT TOP 3 p2.ProductID FROM SalesLT.Product AS p2
    WHERE p2.ProductCategoryID = p.ProductCategoryID
    ORDER BY p2.ListPrice DESC
)
ORDER BY pc.Name, p.ListPrice DESC;

-- Alternative: count rows ranked higher (handles ties differently)
WHERE (
    SELECT COUNT(*) FROM SalesLT.Product AS p2
    WHERE p2.ProductCategoryID = p.ProductCategoryID AND p2.ListPrice > p.ListPrice
) < 3
```

---

### Consecutive Row Comparisons

```sql
-- Each order with the previous order total for same customer
SELECT soh.SalesOrderID, soh.OrderDate, soh.TotalDue,
    (SELECT TOP 1 soh2.TotalDue FROM SalesLT.SalesOrderHeader AS soh2
     WHERE soh2.CustomerID = soh.CustomerID AND soh2.OrderDate < soh.OrderDate
     ORDER BY soh2.OrderDate DESC) AS PreviousOrderTotal
FROM SalesLT.SalesOrderHeader AS soh;
```

> **Tip:** For consecutive row comparisons, `LAG()` / `LEAD()` window functions are typically **more efficient and readable** than correlated subqueries.

---

### When to Use Correlated Subqueries vs Alternatives

| Use This | When You Need... |
|---|---|
| **Correlated subquery** | Row-against-dynamically-calculated-value comparisons, `EXISTS`/`NOT EXISTS` checks, exactly one related value with complex logic |
| **Joins** | Columns from multiple tables; straightforward relationships without per-row calculations |
| **Window functions** | Running totals, rankings, previous/next row access (`LAG`/`LEAD`) — more efficient |
| **CTEs** | Reference same calculated result multiple times; break complex logic into readable steps |

---

### Performance Optimisation Tips

- **Index correlation columns** — columns in the subquery's `WHERE` clause referencing the outer query.
- **Composite indexes** — if subquery also filters/aggregates on other columns, include them.
- **Review execution plans** — use `SET STATISTICS IO ON`; optimiser may transform to join internally.
- **Test with realistic data volumes** — correlated subqueries that work on small datasets can be slow in production.
- **Evaluate alternatives** — `ROW_NUMBER()` window function often outperforms per-row `MAX()` correlated subquery.

---

## Unit 8 — Error Handling with TRY...CATCH

### Why Error Handling Matters
- Database operations can leave data inconsistent if errors occur mid-process.
- Proper handling ensures **all-or-nothing** transaction behaviour.
- Provides meaningful diagnostic information instead of cryptic error messages.
- Creates an **audit trail** for recurring issues.

---

### TRY...CATCH Structure

```sql
BEGIN TRY
    -- Code that might cause an error
    SELECT 1/0;  -- Division by zero
END TRY
BEGIN CATCH
    -- Runs only if an error occurred in TRY block
    PRINT 'An error occurred';
END CATCH;
```

> `TRY...CATCH` **cannot catch:** compilation errors (syntax, missing objects) or errors with severity ≥ 20 that close the connection.

---

### Error Information Functions

| Function | Returns |
|---|---|
| `ERROR_NUMBER()` | Error number |
| `ERROR_MESSAGE()` | Complete error message text |
| `ERROR_SEVERITY()` | Severity level (0–25) |
| `ERROR_STATE()` | Error state number |
| `ERROR_LINE()` | Line number where error occurred |
| `ERROR_PROCEDURE()` | Name of the stored procedure or trigger (NULL if ad hoc) |

```sql
BEGIN TRY
    INSERT INTO SalesLT.Customer (CustomerID, FirstName, LastName)
    VALUES (1, 'Test', 'Customer');  -- Duplicate key error
END TRY
BEGIN CATCH
    INSERT INTO ErrorLog (ErrorTime, ErrorNumber, ErrorSeverity, ErrorState,
                          ErrorProcedure, ErrorLine, ErrorMessage)
    VALUES (
        GETDATE(),
        ERROR_NUMBER(),
        ERROR_SEVERITY(),
        ERROR_STATE(),
        ISNULL(ERROR_PROCEDURE(), 'Ad hoc query'),
        ERROR_LINE(),
        ERROR_MESSAGE()
    );

    THROW;  -- Re-raise to caller
END CATCH;
```

> ⚠️ **Always log errors BEFORE re-raising.** After `THROW`, the `ERROR_*` functions return `NULL`.

---

### Transaction Management in TRY...CATCH

```sql
BEGIN TRY
    BEGIN TRANSACTION;

    UPDATE SalesLT.Product SET ListPrice = ListPrice * 1.05 WHERE ProductCategoryID = 5;
    UPDATE SalesLT.SalesOrderHeader SET TotalDue = TotalDue * 1.05 WHERE CustomerID = 12345;

    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;  -- Undo ALL changes from this transaction

    THROW;  -- Propagate error to caller
END CATCH;
```

**Why check `@@TRANCOUNT > 0` before ROLLBACK:**
- Error might occur before `BEGIN TRANSACTION`.
- Some errors auto-rollback the transaction before reaching `CATCH`.
- Rolling back without an active transaction causes another error.

> ⚠️ **Always check `@@TRANCOUNT > 0` before `ROLLBACK TRANSACTION`** in a CATCH block. Failing to do so causes: *"ROLLBACK TRANSACTION request has no corresponding BEGIN TRANSACTION."*

---

### THROW — Raise Custom Errors

```sql
-- Custom error: number must be ≥ 50000 for user-defined errors
IF @Quantity <= 0
    THROW 50001, 'Quantity must be greater than zero.', 1;

IF NOT EXISTS (SELECT 1 FROM Orders WHERE OrderID = @OrderID)
    THROW 50002, 'Order not found.', 1;

-- Re-raise current error (no parameters)
THROW;
```

- Custom error numbers **must be ≥ 50000**.
- State parameter: user-defined value 1–255, helps identify where error was raised.
- `THROW` without parameters **re-raises the current error** in a CATCH block.
- **Recommended over `RAISERROR` for new code** — simpler, always includes a stack trace.

---

### RAISERROR — Formatted Error Messages

```sql
RAISERROR(
    'Insufficient stock for "%s". Available: %d, Requested: %d',
    16,    -- Severity
    1,     -- State
    @ProductName, @CurrentStock, @RequestedQty
);
```

- Use `RAISERROR` when you need **printf-style formatted messages** or backward compatibility.
- Use `THROW` for all new code.

---

### Nested Error Handling

```sql
CREATE PROCEDURE OuterProcedure AS
BEGIN
    BEGIN TRY
        BEGIN TRANSACTION;
        UPDATE SomeTable SET Column1 = 'Value';
        EXEC InnerProcedure;         -- Error propagates here if it fails
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;  -- Outer owns the rollback
        THROW;
    END CATCH;
END;

CREATE PROCEDURE InnerProcedure AS
BEGIN
    BEGIN TRY
        UPDATE AnotherTable SET Column2 = 'Value';
    END TRY
    BEGIN CATCH
        THROW;  -- Don't rollback here — let outer procedure handle it
    END CATCH;
END;
```

**Pattern:** Inner procedures re-raise errors; the **outermost procedure owns transaction management** (commit/rollback).

---

### XACT_ABORT — Automatic Rollback on Error

```sql
SET XACT_ABORT ON;  -- Auto-rollback transaction on ANY error

BEGIN TRY
    BEGIN TRANSACTION;
    EXEC Procedure1;
    EXEC Procedure2;
    EXEC Procedure3;
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;  -- Handles edge cases
    EXEC LogError;
    THROW;
END CATCH;
```

- `XACT_ABORT ON` — SQL Server automatically rolls back the transaction when any error occurs.
- Combining `XACT_ABORT` with `TRY...CATCH` gives both **guaranteed rollback** and **custom error logging**.
- **Best practice for stored procedures** — especially those spanning multiple operations.

---

## Quick Reference Summary

| Feature | Key Syntax / Note |
|---|---|
| Non-recursive CTE | `WITH name AS (SELECT ...) SELECT ...` |
| Multiple CTEs | Separate with commas; later CTEs can reference earlier ones |
| Recursive CTE | Anchor `UNION ALL` Recursive member; default 100 levels max |
| MAXRECURSION | `OPTION (MAXRECURSION n)` — 0 = unlimited |
| ROW_NUMBER | Unique; no gaps; no duplicate ranks |
| RANK | Ties share rank; gaps after ties (1,1,1,4) |
| DENSE_RANK | Ties share rank; no gaps (1,1,1,2) |
| LAST_VALUE | Requires explicit `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` |
| LAG / LEAD | `LAG(col, offset, default)` / `LEAD(col, offset, default)` |
| JSON_VALUE | Scalar value; returns NVARCHAR(4000) |
| JSON_QUERY | Object/array; preserves JSON structure |
| OPENJSON WITH | Explicit schema with typed columns |
| JSON_OBJECT / JSON_ARRAY | SQL Server 2022+ / Azure SQL / Fabric |
| JSON_ARRAYAGG | Aggregate rows → JSON array (2022+) |
| FOR JSON PATH | Convert relational result to JSON |
| Regex functions | SQL Server 2025+ / Fabric only |
| EDIT_DISTANCE | Raw Levenshtein distance (lower = more similar) |
| EDIT_DISTANCE_SIMILARITY | Normalised 0–100 score (100 = identical) |
| JARO_WINKLER_DISTANCE | 0–1 score for names; >0.9 = strong match |
| Graph node | `CREATE TABLE X AS NODE` → auto `$node_id` |
| Graph edge | `CREATE TABLE X AS EDGE` → auto `$edge_id`, `$from_id`, `$to_id` |
| MATCH syntax | `WHERE MATCH(Node1-(Edge)->Node2)` |
| SHORTEST_PATH | Requires `FOR PATH`; SQL Server 2019+ |
| EXISTS | Stops at first match; more efficient than IN for large tables |
| Correlated subquery in SELECT | Must return exactly one value |
| TRY...CATCH | Cannot catch severity ≥ 20 connection-closing errors |
| @@TRANCOUNT | Check > 0 before ROLLBACK in CATCH |
| THROW | User errors must be ≥ 50000; re-raise with `THROW;` (no params) |
| XACT_ABORT ON | Auto-rollback entire transaction on any error |
| Log before THROW | ERROR_* functions return NULL after THROW |

---

## Key Facts to Remember for the Exam

1. CTEs exist only during query execution — not persistent tables. Cleanup is automatic.
2. Default recursion limit for recursive CTEs is **100**. Change with `OPTION (MAXRECURSION n)`; `0` = unlimited.
3. `ROW_NUMBER()` — always unique. `RANK()` — gaps after ties. `DENSE_RANK()` — no gaps.
4. `LAST_VALUE()` requires an explicit `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` frame.
5. `JSON_VALUE` returns scalar (NVARCHAR); `JSON_QUERY` returns object/array preserving JSON structure.
6. `OPENJSON` with `CROSS APPLY` excludes NULL JSON rows — use `OUTER APPLY` to include them.
7. `JSON_OBJECT`, `JSON_ARRAY`, `JSON_ARRAYAGG` require **SQL Server 2022+** or Azure SQL / Fabric.
8. JSON type columns cannot be indexed directly — create a **computed column** then index it.
9. Regular expression functions (`REGEXP_*`) are **SQL Server 2025+ / Fabric only**.
10. `EDIT_DISTANCE` — raw count (lower = better). `EDIT_DISTANCE_SIMILARITY` — 0-100% (higher = better).
11. `JARO_WINKLER_DISTANCE` — 0 to 1 score; best for **names**; score > 0.9 = strong match.
12. Always **pre-filter** with `LIKE` or exact matches before applying fuzzy functions on large tables.
13. Graph `MATCH` arrow direction must match how edges were inserted (`$from_id`/`$to_id`).
14. Same edge alias **cannot appear twice** in a single `MATCH` pattern — use separate aliases.
15. `SHORTEST_PATH` requires `FOR PATH` on participating tables; SQL Server 2019+.
16. `EXISTS` stops at first match — more efficient than `IN` for existence checks on large tables.
17. Correlated subquery in `SELECT` clause **must return exactly one value** — wrap in aggregate if needed.
18. Always check `@@TRANCOUNT > 0` before `ROLLBACK TRANSACTION` in a CATCH block.
19. **Log errors BEFORE calling `THROW`** — `ERROR_*` functions return NULL after THROW.
20. Custom error numbers for `THROW` must be **≥ 50000**.
21. `XACT_ABORT ON` auto-rolls back transactions on any error — best practice for stored procedures.
22. `TRY...CATCH` cannot catch compilation errors or severity ≥ 20 connection-closing errors.

---

*End of DP-800 Module 3 Notes — Write Advanced T-SQL Code*