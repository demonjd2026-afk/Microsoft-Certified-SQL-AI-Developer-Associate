# DP-800 Exam Notes — Implement Programmability Objects with SQL

> **Module:** Implement programmability objects with SQL
> **Level:** Intermediate | **Roles:** Data Engineer, DBA, Developer, Solution Architect
> **Platforms Covered:** SQL Server, Azure SQL Database, Azure SQL Managed Instance, Microsoft Fabric

---

## Overview of Programmability Objects

SQL Server provides five main programmability objects to encapsulate and reuse logic in your database:

| Object | Purpose |
|---|---|
| **Views** | Virtual tables that simplify complex queries and provide security boundaries |
| **Stored Procedures** | Compiled T-SQL blocks that encapsulate business logic and data modifications |
| **Scalar Functions** | Return a single calculated value; embeddable in SQL expressions |
| **Table-Valued Functions (TVF)** | Return a result set (table); usable in queries like a table |
| **Triggers** | Auto-execute in response to data (DML) or schema (DDL) events |

---

## Unit 1 — Create Views

### What Is a View?
- A **virtual table** based on a `SELECT` statement.
- Does **not store data** — retrieves data from underlying base tables each time queried.
- Hides the complexity of JOINs, calculations, and filters from application code.
- Provides a **security layer** — grant column/row-level access through a view while restricting access to underlying tables.

---

### Creating a Basic View

```sql
CREATE VIEW Sales.CustomerOrders
AS
SELECT
    c.CustomerID,
    c.CustomerName,
    c.Email,
    o.OrderID,
    o.OrderDate,
    o.TotalAmount
FROM Sales.Customers c
INNER JOIN Sales.Orders o ON c.CustomerID = o.CustomerID;
```

Query it like any table:

```sql
SELECT * FROM Sales.CustomerOrders
WHERE CustomerName = 'Contoso Ltd';
```

---

### Views with Calculated Columns

```sql
CREATE VIEW Sales.OrderSummary
AS
SELECT
    OrderID,
    CustomerID,
    OrderDate,
    TotalAmount,
    CASE
        WHEN TotalAmount < 100  THEN 'Small'
        WHEN TotalAmount < 1000 THEN 'Medium'
        ELSE 'Large'
    END AS OrderSize
FROM Sales.Orders;
```

Centralises the CASE logic — every query against the view gets the same calculation.

---

### Key Design Considerations

#### Specify Columns Explicitly — Never `SELECT *`
- Using `SELECT *` causes unexpected results when underlying tables change.
- Explicit column lists keep view definitions consistent.

```sql
CREATE VIEW Sales.ActiveCustomers
AS
SELECT CustomerID, CustomerName, Email, Phone
FROM Sales.Customers
WHERE IsActive = 1;
```

#### WITH CHECK OPTION
- Ensures `INSERT` and `UPDATE` through the view only affect rows **visible** in the view.
- If a new row would not satisfy the view's `WHERE` condition, the operation is rejected.

```sql
CREATE VIEW Sales.HighValueOrders
AS
SELECT OrderID, CustomerID, OrderDate, TotalAmount
FROM Sales.Orders
WHERE TotalAmount > 1000
WITH CHECK OPTION;
-- Inserting an order with TotalAmount = 500 through this view is REJECTED
```

---

### When to Use Views

| Use Views When... | Avoid / Use Alternative When... |
|---|---|
| Simplifying complex joins used by multiple apps | Performance-critical queries — consider **indexed views** |
| Restricting column/row access for security | Logic requires input parameters — use **functions** |
| Presenting data in different formats/aggregations | Data modification logic — use **stored procedures** |
| Creating a stable interface to changing tables | |

> Views excel at **encapsulating reusable query logic without parameters** and **presenting a simplified logical view** of data.

---

### Indexed Views (Materialized Views)
- For performance-critical queries that always return the same results, indexed views store the result set **physically**.
- Useful when the same complex aggregation is needed frequently.

---

## Unit 2 — Create Stored Procedures

### What Is a Stored Procedure?
- A **compiled collection of T-SQL statements** stored and executed as a single unit.
- **Precompiled and optimised** — execution plan is cached after the first run, reducing overhead for repeated operations.
- Used to encapsulate complex business logic, enforce validation rules, and control how applications interact with the database.
- Reduces network traffic (code runs on the server, not the client).

---

### Creating a Basic Stored Procedure

```sql
CREATE PROCEDURE dbo.GetCustomerOrders
AS
BEGIN
    SET NOCOUNT ON;

    SELECT OrderID, CustomerID, OrderDate, TotalAmount
    FROM dbo.Orders
    ORDER BY OrderDate DESC;
END
```

- `SET NOCOUNT ON` — prevents the "rows affected" message from being sent to the client; reduces network traffic, especially with multiple statements.
- Use `BEGIN...END` to clearly define the procedure body.

---

### Parameters

#### Input Parameters (with defaults)

```sql
CREATE PROCEDURE dbo.GetCustomerOrdersByDate
    @CustomerID INT,
    @StartDate  DATETIME = NULL,   -- optional with default
    @EndDate    DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;

    SELECT OrderID, OrderDate, TotalAmount
    FROM dbo.Orders
    WHERE CustomerID = @CustomerID
      AND (@StartDate IS NULL OR OrderDate >= @StartDate)
      AND (@EndDate   IS NULL OR OrderDate <= @EndDate)
    ORDER BY OrderDate DESC;
END
```

#### Output Parameters

```sql
CREATE PROCEDURE dbo.CalculateOrderTotal
    @OrderID     INT,
    @TotalAmount DECIMAL(10,2) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    SELECT @TotalAmount = SUM(Quantity * UnitPrice)
    FROM dbo.OrderDetails
    WHERE OrderID = @OrderID;

    RETURN 0;
END
```

- Caller must declare a variable and use the `OUTPUT` keyword in the `EXECUTE` statement.

---

### Error Handling with TRY...CATCH

```sql
CREATE PROCEDURE dbo.InsertCustomerOrder
    @CustomerID  INT,
    @OrderDate   DATETIME,
    @TotalAmount DECIMAL(10,2)
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        IF NOT EXISTS (SELECT 1 FROM dbo.Customers WHERE CustomerID = @CustomerID)
        BEGIN
            RAISERROR('Customer does not exist.', 16, 1);
        END

        INSERT INTO dbo.Orders (CustomerID, OrderDate, TotalAmount)
        VALUES (@CustomerID, @OrderDate, @TotalAmount);

        COMMIT TRANSACTION;
        RETURN 0;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;

        DECLARE @ErrorMessage  NVARCHAR(4000) = ERROR_MESSAGE();
        DECLARE @ErrorSeverity INT            = ERROR_SEVERITY();
        DECLARE @ErrorState    INT            = ERROR_STATE();

        RAISERROR(@ErrorMessage, @ErrorSeverity, @ErrorState);
        RETURN -1;
    END CATCH
END
```

**Key error functions:**
- `ERROR_MESSAGE()` — description of the error
- `ERROR_SEVERITY()` — severity level
- `ERROR_STATE()` — state number
- Always check `@@TRANCOUNT > 0` before rolling back (prevents errors if transaction never started or already completed).

---

### Best Practices for Stored Procedures

| Practice | Detail |
|---|---|
| **Use schema-qualified names** | `dbo.Orders` not just `Orders` — eliminates ambiguity, improves performance |
| **Validate parameters early** | Fail fast on invalid inputs; return before processing bad data |
| **Avoid `SELECT *`** | Explicitly list columns — prevents issues when tables change |
| **Use meaningful names with verbs** | `GetActiveCustomersByRegion`, `UpdateCustomerAddress`, `DeleteExpiredOrders` |
| **Never use `sp_` prefix** | SQL Server searches `master` DB first for `sp_` named procedures — unnecessary overhead |
| **Use `SET NOCOUNT ON`** | Reduces network traffic by suppressing row count messages |
| **Always handle transactions** | Use `TRY...CATCH` + `@@TRANCOUNT` check before rollback |

---

## Unit 3 — Create Scalar Functions

### What Is a Scalar Function?
- Accepts zero or more parameters and **returns exactly one value** of a specified data type.
- Unlike stored procedures, can be **embedded directly** in SQL expressions — `SELECT`, `WHERE`, `JOIN`, computed columns.
- Return data type is defined explicitly and validated at creation time.
- Promotes code reuse and consistency across the database.

---

### Syntax

```sql
CREATE FUNCTION schema_name.function_name
(
    @parameter1 datatype,
    @parameter2 datatype
)
RETURNS return_datatype
AS
BEGIN
    -- Logic
    RETURN @result
END
```

---

### Examples

#### Sales Tax Calculation

```sql
CREATE FUNCTION dbo.CalculateSalesTax
(
    @Amount  DECIMAL(10,2),
    @TaxRate DECIMAL(5,4)
)
RETURNS DECIMAL(10,2)
AS
BEGIN
    DECLARE @TaxAmount DECIMAL(10,2)
    SET @TaxAmount = ROUND(@Amount * @TaxRate, 2)
    RETURN @TaxAmount
END
```

#### Employee Tenure

```sql
CREATE FUNCTION dbo.GetEmployeeTenure
(
    @HireDate DATE
)
RETURNS INT
AS
BEGIN
    DECLARE @Tenure INT
    SET @Tenure = DATEDIFF(YEAR, @HireDate, GETDATE())
    RETURN @Tenure
END
```

Usage in a query:

```sql
SELECT EmployeeName, HireDate,
       dbo.GetEmployeeTenure(HireDate) AS YearsOfService
FROM Employees
WHERE dbo.GetEmployeeTenure(HireDate) >= 5
```

> **Note:** This function uses `GETDATE()` which makes it **non-deterministic**. Non-deterministic functions **cannot** be used in indexed views or indexes on computed columns. Pass the current date as a parameter instead if determinism is required.

---

### Deterministic vs Non-Deterministic Functions

| Type | Definition | Can Index? |
|---|---|---|
| **Deterministic** | Always returns same result for same inputs | ✅ Yes |
| **Non-Deterministic** | May return different results (e.g., uses `GETDATE()`, reads tables) | ❌ No |

---

### Best Practices for Scalar Functions

- **Keep functions deterministic** where possible — enables indexing and query optimisations.
- **Avoid side effects** — functions should not modify database state or depend on external resources. SQL Server may execute functions multiple times or in unexpected order.
- **Be mindful of performance** — scalar functions in `WHERE` clauses or `SELECT` lists on large tables execute **per row**, which can severely degrade performance. Consider **inline table-valued functions** as a faster alternative.

---

### Managing Scalar Functions

```sql
-- Modify without dropping (preserves permissions and dependencies)
ALTER FUNCTION dbo.CalculateSalesTax ( ... ) ...

-- Drop safely
DROP FUNCTION IF EXISTS dbo.CalculateSalesTax

-- Check dependencies before dropping
SELECT * FROM sys.sql_expression_dependencies
WHERE referenced_entity_name = 'CalculateSalesTax'
```

---

## Unit 4 — Create Table-Valued Functions (TVF)

### What Is a Table-Valued Function?
- Returns a **result set (table)** instead of a single value.
- Can be used directly in queries — in `FROM`, `JOIN`, `CROSS APPLY` clauses — **unlike stored procedures**.
- Accepts parameters, making them like **parameterised views**.
- Two types: **Inline TVF** and **Multi-Statement TVF**.

---

### Type 1 — Inline Table-Valued Function (iTVF)

- Contains a **single SELECT statement**; no `BEGIN...END` block.
- SQL Server infers the return table structure from the SELECT.
- Query optimiser treats iTVFs like **views with parameters** — often produces better execution plans.
- **Preferred for performance** when a single SELECT suffices.

```sql
CREATE FUNCTION dbo.GetCustomerOrders
(
    @CustomerID INT
)
RETURNS TABLE
AS
RETURN
(
    SELECT OrderID, OrderDate, TotalAmount, Status
    FROM Sales.Orders
    WHERE CustomerID = @CustomerID
);
```

Usage:

```sql
SELECT OrderID, OrderDate, TotalAmount
FROM dbo.GetCustomerOrders(1001)
WHERE OrderDate >= '2024-01-01';
```

---

### Type 2 — Multi-Statement Table-Valued Function (MSTVF)

- Uses `BEGIN...END` block; **explicitly declares the returned table structure**.
- Allows multiple statements, complex calculations, conditional logic, and iterative building of results.
- **Performance trade-off:** Optimiser treats MSTVFs as "black boxes" — cannot see inside them, leading to inaccurate row estimates and potentially suboptimal plans.
- Use only when procedural logic or multiple steps are genuinely needed.

```sql
CREATE FUNCTION dbo.GetProductSalesSummary
(
    @StartDate DATE,
    @EndDate   DATE
)
RETURNS @SalesSummary TABLE
(
    ProductID    INT,
    ProductName  NVARCHAR(100),
    TotalQty     INT,
    TotalRevenue DECIMAL(18,2),
    AvgPrice     DECIMAL(18,2)
)
AS
BEGIN
    INSERT INTO @SalesSummary
    SELECT
        p.ProductID,
        p.ProductName,
        SUM(od.Quantity),
        SUM(od.Quantity * od.UnitPrice),
        AVG(od.UnitPrice)
    FROM Production.Products p
    INNER JOIN Sales.OrderDetails od ON p.ProductID = od.ProductID
    INNER JOIN Sales.Orders o        ON od.OrderID  = o.OrderID
    WHERE o.OrderDate BETWEEN @StartDate AND @EndDate
    GROUP BY p.ProductID, p.ProductName;

    RETURN;
END;
```

---

### Using TVFs in Queries

#### CROSS APPLY — Row-by-Row with Column as Parameter

```sql
SELECT c.CustomerName, o.OrderID, o.TotalAmount
FROM Customers c
CROSS APPLY dbo.GetCustomerOrders(c.CustomerID) o
WHERE o.Status = 'Completed';
```

- `CROSS APPLY` calls the function **for each row** from the left table, passing a column value as the parameter.
- Particularly powerful for correlated data retrieval with better readability than correlated subqueries.

#### CROSS APPLY with Constant Parameters

```sql
SELECT c.CustomerName, s.ProductName, s.TotalRevenue
FROM Customers c
CROSS APPLY dbo.GetProductSalesSummary('2024-01-01', '2024-12-31') s
WHERE s.TotalRevenue > 10000;
```

---

### Inline TVF vs Multi-Statement TVF Comparison

| Feature | Inline TVF | Multi-Statement TVF |
|---|---|---|
| Syntax | Single `RETURN (SELECT ...)` | `BEGIN...END` with table variable |
| Table structure | Inferred from SELECT | Explicitly declared |
| Optimiser treatment | Like a parameterised view — good plans | Black box — inaccurate estimates |
| Performance | Better | Worse (use only when needed) |
| Supports multiple statements | ❌ No | ✅ Yes |
| Supports conditional/iterative logic | ❌ No | ✅ Yes |

---

## Unit 5 — Create Triggers

### What Is a Trigger?
- A **special stored procedure** that automatically executes when a specific database event occurs.
- Unlike stored procedures, triggers are **never called explicitly** — they fire in response to events.
- Used to maintain data integrity, enforce business rules, and automate operations.
- Two main categories: **DML Triggers** and **DDL Triggers**.

---

### DML Triggers (Data Manipulation Language)

Respond to `INSERT`, `UPDATE`, `DELETE` on tables or views.

#### Two Types:

| Type | When It Fires | Use Case |
|---|---|---|
| **AFTER** | After the data modification completes | Validate changes, update related tables, log modifications |
| **INSTEAD OF** | Replaces the original modification | Modify otherwise non-updatable views; implement complex pre-modification logic |

---

#### AFTER Trigger Example — Update Inventory

```sql
CREATE TRIGGER tr_UpdateInventory
ON Sales.OrderDetails
AFTER INSERT
AS
BEGIN
    UPDATE Inventory.Products
    SET QuantityInStock = QuantityInStock - i.Quantity
    FROM Inventory.Products p
    INNER JOIN inserted i ON p.ProductID = i.ProductID;
END;
```

---

#### INSTEAD OF Trigger Example — Updatable View

```sql
CREATE TRIGGER tr_UpdateOrderView
ON Sales.OrderSummaryView
INSTEAD OF UPDATE
AS
BEGIN
    UPDATE Sales.Orders
    SET OrderStatus  = i.OrderStatus,
        ModifiedDate = GETDATE()
    FROM Sales.Orders o
    INNER JOIN inserted i ON o.OrderID = i.OrderID;
END;
```

---

### The `inserted` and `deleted` Pseudo-Tables

- Available inside DML triggers — **temporary tables** holding copies of affected rows.

| Operation | `inserted` | `deleted` |
|---|---|---|
| INSERT | ✅ Contains new rows | ❌ Empty |
| DELETE | ❌ Empty | ✅ Contains removed rows |
| UPDATE | ✅ Contains new values | ✅ Contains old values |

---

### Multi-Event Triggers

A single trigger can respond to multiple DML events:

```sql
CREATE TRIGGER tr_AuditEmployeeChanges
ON HR.Employees
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    DECLARE @Operation NVARCHAR(10);

    IF EXISTS (SELECT * FROM inserted) AND NOT EXISTS (SELECT * FROM deleted)
        SET @Operation = 'INSERT';
    ELSE IF EXISTS (SELECT * FROM inserted) AND EXISTS (SELECT * FROM deleted)
        SET @Operation = 'UPDATE';
    ELSE
        SET @Operation = 'DELETE';

    INSERT INTO Audit.EmployeeLog (EmployeeID, Operation, ChangeDate)
    SELECT COALESCE(i.EmployeeID, d.EmployeeID), @Operation, GETDATE()
    FROM inserted i
    FULL OUTER JOIN deleted d ON i.EmployeeID = d.EmployeeID;
END;
```

---

### Column-Specific Trigger Logic

Use the `UPDATE()` function to check whether a specific column was modified:

```sql
CREATE TRIGGER tr_LogPriceChanges
ON Products.Catalog
AFTER UPDATE
AS
BEGIN
    IF UPDATE(Price)
    BEGIN
        INSERT INTO Audit.PriceHistory (ProductID, OldPrice, NewPrice, ChangeDate)
        SELECT d.ProductID, d.Price, i.Price, GETDATE()
        FROM deleted d
        INNER JOIN inserted i ON d.ProductID = i.ProductID
        WHERE d.Price <> i.Price;
    END;
END;
```

---

### DDL Triggers (Data Definition Language)

- Respond to schema changes: `CREATE`, `ALTER`, `DROP` statements.
- Used to prevent or audit structural changes to the database.

---

### Error Handling in Triggers

- If a trigger encounters an error and it is not handled, SQL Server **rolls back both the trigger and the original statement**.
- Use `THROW` to raise errors and cancel the triggering operation:

```sql
CREATE TRIGGER tr_ValidateOrderDate
ON Sales.Orders
AFTER INSERT, UPDATE
AS
BEGIN
    IF EXISTS (SELECT * FROM inserted WHERE OrderDate > GETDATE())
    BEGIN
        THROW 50001, 'Order date cannot be in the future', 1;
    END;
END;
```

---

### Trigger Best Practices

- **Keep trigger logic minimal** — execute only essential operations. Complex/long-running operations should be queued for async processing.
- **Avoid recursive triggers** — a trigger modification that re-fires the same trigger causes infinite loops. Set `RECURSIVE_TRIGGERS` database option appropriately.
- **Handle errors properly** — unhandled errors roll back the original statement too.
- **Document triggers thoroughly** — they execute invisibly during normal operations; other developers must understand why they exist.
- **Use separate triggers per event** when logic differs — simpler, easier to maintain.
- **Avoid heavy computation in triggers** — they execute on every qualifying DML operation.

---

## Unit 6 — Choosing the Right Programmability Object

### Capability Comparison Table

| Capability | Views | Stored Procedures | Functions | Triggers |
|---|---|---|---|---|
| Accept parameters | ❌ No | ✅ Yes | ✅ Yes | ❌ No |
| Modify data | Limited* | ✅ Yes | ❌ No | ✅ Yes |
| Return result sets | ✅ Yes | ✅ Yes | ✅ Yes (TVFs) | ❌ No |
| Use in `SELECT`/`JOIN` | ✅ Yes | ❌ No | ✅ Yes | ❌ No |
| Transaction control | ❌ No | ✅ Yes | ❌ No | ✅ Yes |
| Automatic execution | ❌ No | ❌ No | ❌ No | ✅ Yes |
| Execution plan caching | ❌ No | ✅ Yes | Varies** | ✅ Yes |

> *Views can modify data only when changes affect a **single base table**.
> **Inline TVFs benefit from plan caching (expanded into query). Multi-statement TVFs and scalar functions are treated as "black boxes" — optimiser can't see inside them, causing inaccurate row estimates and suboptimal plans.

---

### Decision Framework

#### Choose a View when you need to:
- Simplify access to complex joins or commonly filtered data
- Provide a security layer (control column and row visibility)
- Create a stable interface to underlying tables that might change
- Present data without accepting parameters or modifying values

#### Choose a Stored Procedure when you need to:
- Execute complex business logic with multiple statements
- Modify data across multiple tables in a single transaction
- Accept input/output parameters
- Implement error handling and transaction control

#### Choose a Function when you need to:
- Perform reusable calculations that return values for use in queries
- Return parameterised result sets (TVFs)
- Embed logic directly in `SELECT`, `WHERE`, or `JOIN` clauses
- Ensure deterministic results for indexing

#### Choose a Trigger when you need to:
- Automatically respond to data modification events
- Enforce complex business rules beyond what constraints can handle
- Maintain audit logs transparently
- Synchronise related data across tables automatically

---

### Scenario-Based Decision Guide

| Scenario | Best Object | Reason |
|---|---|---|
| Simplify a 5-table join used by multiple reports | View | Encapsulates complexity; no parameters needed |
| Process an order: validate stock, insert order, update inventory | Stored Procedure | Multiple modifications in a single transaction |
| Calculate shipping cost based on weight and destination | Scalar Function | Reusable calculation embeddable in queries |
| Return all orders for a customer within a date range | Table-Valued Function | Parameterised result set usable in JOIN |
| Log all changes to the `Salary` column automatically | Trigger | Automatic, transparent audit trail |
| Provide read-only access to employee data without SSN column | View | Security layer hiding sensitive columns |

---

### Common Mistakes to Avoid

| Mistake | Impact | Fix |
|---|---|---|
| **Scalar functions in `WHERE` clauses on large tables** | Function executes per row — severe performance degradation | Use inline TVFs or rewrite logic inline |
| **Using triggers for logic that stored procedures handle better** | Triggers execute implicitly, hard to debug | Use triggers only when automatic execution is truly needed |
| **Deeply nested views (views within views)** | Hard to optimise and maintain | Keep view definitions focused and shallow |
| **Using stored procedures when a function integrates better** | Requires temp tables + EXEC for results in SELECT | Use a function if the result needs to appear in a SELECT statement |
| **Non-deterministic scalar functions on indexed computed columns** | Cannot be used for indexing | Make functions deterministic or pass date as parameter |

---

## Quick Reference Summary

| Object | Key Syntax | Key Characteristics |
|---|---|---|
| View | `CREATE VIEW schema.name AS SELECT ...` | No parameters; virtual table; optional `WITH CHECK OPTION` |
| Stored Procedure | `CREATE PROCEDURE dbo.name @param type AS BEGIN...END` | Precompiled; plan cached; supports transactions, error handling |
| Scalar Function | `CREATE FUNCTION dbo.name(@p type) RETURNS type AS BEGIN RETURN val END` | Returns 1 value; embeddable in SQL expressions; determinism matters |
| Inline TVF | `CREATE FUNCTION dbo.name(@p type) RETURNS TABLE AS RETURN (SELECT ...)` | Single SELECT; better optimiser support; parameterised view |
| Multi-Statement TVF | `CREATE FUNCTION dbo.name(@p type) RETURNS @t TABLE(...) AS BEGIN...RETURN END` | Explicit table structure; flexible; black box to optimiser |
| AFTER Trigger | `CREATE TRIGGER name ON table AFTER INSERT,UPDATE,DELETE AS BEGIN...END` | Fires after modification; `inserted`/`deleted` pseudo-tables available |
| INSTEAD OF Trigger | `CREATE TRIGGER name ON view INSTEAD OF UPDATE AS BEGIN...END` | Replaces the original operation; used on views |
| DDL Trigger | `CREATE TRIGGER name ON DATABASE FOR CREATE_TABLE AS BEGIN...END` | Responds to schema changes (CREATE, ALTER, DROP) |

---

## Key Facts to Remember for the Exam

1. Views do **not** store data — they always retrieve from base tables.
2. `WITH CHECK OPTION` rejects DML through a view if the resulting row would not be visible in the view.
3. Stored procedures use `SET NOCOUNT ON` to suppress row count messages and reduce network traffic.
4. Never prefix stored procedures with `sp_` — SQL Server searches `master` first.
5. Scalar functions using `GETDATE()` are **non-deterministic** — cannot be used in indexed views or indexed computed columns.
6. Inline TVFs = better performance (treated as parameterised views by optimiser). Multi-statement TVFs = black box.
7. `CROSS APPLY` calls a TVF for each row from the left table, passing column values as parameters.
8. In DML triggers, `inserted` holds new/updated rows; `deleted` holds old/removed rows. In UPDATE triggers, **both** are populated.
9. `UPDATE(columnname)` inside a trigger checks if a specific column was included in the UPDATE statement.
10. AFTER trigger fires **after** the modification. INSTEAD OF trigger **replaces** the modification.
11. Unhandled errors in triggers roll back **both the trigger and the original DML statement**.
12. Stored procedures and AFTER triggers benefit from execution plan caching. Views and multi-statement TVFs do not.

---

*End of DP-800 Module 2 Notes — Implement Programmability Objects with SQL*