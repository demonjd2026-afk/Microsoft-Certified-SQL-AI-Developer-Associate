# DP-800 Exam Notes — Design and Implement Database Objects

> **Module:** Design and implement database objects with SQL
> **Level:** Intermediate | **Roles:** Data Engineer, DBA, Developer, Solution Architect
> **Platforms Covered:** SQL Server, Azure SQL Database, Azure SQL Managed Instance, Microsoft Fabric

---

## Unit 1 — Platform Choices

### IaaS vs PaaS Responsibility Split
- **PaaS (Azure SQL DB, Managed Instance, Fabric SQL):** Azure manages physical servers, networking, OS patches, and engine updates. You manage tables, indexes, constraints, and data.
- **IaaS (SQL Server on Azure VMs):** You manage everything including the OS and SQL Server instance.

---

### Azure SQL Database (PaaS)
- Fully managed, enterprise-grade performance with no infrastructure management.
- **Hyperscale tier:** No predefined max storage size; scales out with more replicas for read-heavy workloads; billed only for capacity used.
- **Serverless compute tier:** Auto-scales compute; pauses when idle (pay only for storage during pause); resumes automatically on connection.
  - Design apps with **connection retry logic** for resume delays.
  - Avoid long-running transactions that prevent autopause.
- **Intelligent Query Processing** and **Automatic Tuning** analyze workload patterns to recommend/create indexes automatically.
- **Automatic plan correction** detects and fixes query regressions.
- Built-in HA with **99.99% uptime SLA**.

---

### Azure SQL Managed Instance (PaaS)
- Near **100% compatibility** with SQL Server Enterprise Edition (latest version, auto-patched).
- **Instance-level features:** SQL Server Agent, Service Broker, linked servers, cross-database queries (3-part naming), database mail.
- **Native VNet integration** for security isolation.
- **Managed Instance Link:** Uses distributed availability groups to sync data from SQL Server to Azure in near real-time (hybrid scenarios, DR, minimal-downtime migrations).
- **In-Memory OLTP** available in the **Business Critical tier** (memory-optimized tables + natively compiled stored procedures).
- Deployment options: Single managed instance or managed instance pool.

---

### SQL Server on Azure Virtual Machines (IaaS)
- Full control over SQL Server instance, engine config, and OS (Windows or Linux).
- **SQL IaaS Agent Extension** provides: automated backups, auto patching, Azure Key Vault integration, tempdb config via Azure portal.
- **SQL Best Practices Assessment** validates config against recommended settings.
- Supports Always On availability groups and failover cluster instances with full control over replica placement.

---

### SQL Database in Microsoft Fabric
- Built on Azure SQL Database technology; developer-friendly transactional + analytics platform.
- **Automatic mirroring:** Replicates changes to OneLake as Delta Parquet files automatically (no ETL, no triggers needed).
- **SQL analytics endpoint:** Read-only analytical view; analytical queries run on Delta Parquet copies so they never slow OLTP.
- **Cross-database queries:** Join SQL DB with Fabric warehouses, lakehouses, and other SQL DBs using 3-part naming syntax.
- **Automatic index creation** monitors query patterns and creates indexes without manual intervention.
- Supports AI development with **semantic search** and **RAG (Retrieval-Augmented Generation)**.
- Database portability via SqlPackage (.bacpac/.dacpac), Fabric source control (git), GraphQL APIs.

---

## Unit 2 — Build Effective Tables

### Why Data Type Choice Matters
- Wrong types lead to wasted storage, poor performance, data loss, or application errors.
- Changing column data types in production often requires a **full table rebuild** (hours of downtime for large tables).
- Choose data types carefully at initial schema design — it is the easiest time to get it right.

---

### Common Data Types Reference

| Category | Types | Storage | Key Guidelines |
|---|---|---|---|
| Numeric | `INT`, `BIGINT`, `DECIMAL`, `FLOAT` | 4 bytes, 8 bytes, varies | Use `DECIMAL` for financial data (not `FLOAT` — rounding errors) |
| String | `VARCHAR`, `CHAR`, `NVARCHAR` | 1 byte/char, fixed, 2 bytes/char | `VARCHAR` = variable-length; `CHAR` = fixed-length; `NVARCHAR` = Unicode |
| Date/Time | `DATE`, `DATETIME2`, `DATETIMEOFFSET` | 3 bytes, 6-8 bytes, 10 bytes | Prefer `DATETIME2` over legacy `DATETIME` (better precision) |
| Binary | `VARBINARY`, `IMAGE` | varies | For images, documents |
| Special | `UNIQUEIDENTIFIER`, `XML`, `JSON` | 16 bytes, varies | GUID primary key is 3× larger than INT; avoid unless needed |

**Key nuances:**
- `FLOAT` for financial data introduces rounding errors — always use `DECIMAL`.
- `UNIQUEIDENTIFIER` primary key triples index size and slows every JOIN.
- `NVARCHAR` uses 2 bytes/char vs `VARCHAR` at 1 byte/char. Use `VARCHAR` for ASCII-only data to halve string storage.

---

### Estimating Table Size
- Estimated row size = sum of fixed-length columns + average size of variable-length columns + ~7 bytes row overhead.
- Example: `Employee` table with INT(4) + NVARCHAR(50)(~40) + NVARCHAR(50)(~40) + DATE(3) + DECIMAL(10,2)(5) = ~99 bytes/row → 1M rows ≈ 94 MB.
- 100M rows with extra 50 bytes/row wastes 5 GB/day = 1.8 TB/year unnecessarily.

---

### Table Design Best Practices
- Use **appropriate data types** — smaller types reduce storage and improve performance.
- Use `INT` for PK (4 bytes) vs `BIGINT` (8 bytes) or `UNIQUEIDENTIFIER` (16 bytes) unless scale demands otherwise.
- Use `DECIMAL(10,2)` for prices/financial columns, never `FLOAT`.
- Use `DATETIME2` over legacy `DATETIME`.
- Size string columns (`NVARCHAR(100)`, `NVARCHAR(50)`) based on expected actual data length.
- Use `NOT NULL` on critical columns to enforce data quality.
- Use `DEFAULT` values to reduce application code complexity (e.g., `DEFAULT GETUTCDATE()` for timestamps).
- Use `IDENTITY` for PK — sequential keys cluster efficiently.
- Index columns used in `WHERE`, `JOIN`, and `ORDER BY`.
- Use columnstore indexes for large analytical tables.
- Enable row/page compression for large tables.
- Normalize appropriately — balance normalization with query performance.

---

## Unit 3 — Optimize with Indexes

### Fundamentals
- Indexes convert potentially millions of row comparisons into efficient lookups.
- Trade-off: indexes consume storage and **slow INSERT, UPDATE, DELETE** (index structures must be maintained alongside data).
- Over-indexing, under-indexing, and poorly designed indexes are top causes of database performance problems.

---

### Rowstore Indexes

#### Clustered Index
- Sorts and stores **data rows physically** based on key values.
- Only **one clustered index per table** (data can only be in one physical order).
- **Best for:** Range queries, stable/narrow keys, natural sort order (identity columns, date fields).

#### Nonclustered Index
- Separate structure from data rows; contains key values + pointer to data rows.
- **Multiple** nonclustered indexes allowed per table.
- **Best for:** Fast lookups for specific predicates/joins/sorts not aligned with the clustered key; cover queries with `INCLUDE` columns to avoid key lookups.

```sql
-- Clustered index
CREATE CLUSTERED INDEX IX_Product_ProductID ON Product(ProductID);

-- Nonclustered index with included columns
CREATE NONCLUSTERED INDEX IX_Product_Category
ON Product(Category)
INCLUDE (ProductName, Price);
```

---

### Columnstore Indexes

#### Architecture
- Stores data **column by column** (not row by row) — reads only columns needed by a query.
- Data organized into **rowgroups** (up to 1,048,576 rows each).
- Each column in a rowgroup stored as a **column segment**, compressed independently.
- **Deltastore:** Small inserts go to a temporary rowstore (B+ tree). Once ≥102,400 rows accumulate, the **tuple-mover** background process compresses them into the columnstore.
- Bulk loads of ≥102,400 rows bypass the deltastore and compress directly.

#### When to Use Columnstore

| Scenario | Recommendation | Reason |
|---|---|---|
| Data warehouse fact tables | ✅ Use | Millions+ rows, analytics benefit from columnar + compression |
| Reporting/read-heavy databases | ✅ Use | Aggregate queries faster with column-oriented access |
| Historical/archival data | ✅ Use | Rarely updated, high compression ratios |
| Small tables (<1M rows) | ❌ Avoid | Overhead outweighs benefits |
| High-frequency updates/deletes | ❌ Avoid | Modifications mark rows as deleted → fragmentation |
| Single-row lookups | ❌ Avoid | Rowstore faster for individual record retrieval |

#### Clustered Columnstore Index (CCI)
- Becomes the **primary storage** for the entire table — replaces any clustered rowstore index.
- All table data stored exclusively in columnar format.
- Use when **analytics is the primary workload** and row-level transactional access isn't needed.

```sql
CREATE CLUSTERED COLUMNSTORE INDEX CCI_SalesHistory ON SalesHistory;
ALTER INDEX CCI_SalesHistory ON SalesHistory REBUILD;
```

#### Nonclustered Columnstore Index (NCCI)
- Creates a **separate columnar copy** alongside the existing rowstore table.
- Table maintains clustered rowstore for OLTP; NCCI provides columnar access for analytics.
- Query optimizer automatically chooses rowstore vs columnstore based on query pattern.

```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_Product_Analytics
ON Product(Price, StockQuantity, Category, ProductName);
```

#### Monitoring Columnstore Health
- Use DMV: `sys.dm_db_column_store_row_group_physical_stats`
- States: **Open** (accepting inserts in deltastore) → **Closed** (waiting for tuple-mover) → **Compressed** (columnar format).
- High deleted row counts or many small rowgroups = fragmentation → fix with `ALTER INDEX REORGANIZE`.

```sql
SELECT object_name(object_id) AS TableName, state_desc, total_rows, deleted_rows,
       size_in_bytes / 1024 / 1024 AS SizeMB
FROM sys.dm_db_column_store_row_group_physical_stats
WHERE object_id = OBJECT_ID('SalesHistory');
```

---

## Unit 4 — Specialized Table Types

### In-Memory Optimized Tables
- Store data **entirely in RAM** with lock-free, optimistic concurrency — eliminates disk I/O latency.
- **Use cases:** Session state (millions of concurrent sessions), real-time analytics, high-frequency OLTP (10,000+ tx/sec), caching reference data, ETL staging tables.
- **Limitations:**
  - Data size limited by available RAM.
  - Do NOT support `VARCHAR(MAX)`, `NVARCHAR(MAX)`, `VARBINARY(MAX)`.
  - Transaction logs still written to disk (committed transactions survive restart via log recovery).
- Requires `MEMORY_OPTIMIZED = ON` option.
- `DURABILITY = SCHEMA_AND_DATA` (default) persists both schema and data; `SCHEMA_ONLY` loses data on restart.

```sql
CREATE TABLE dbo.OrderCache (
    OrderID INT PRIMARY KEY NONCLUSTERED,
    CustomerID INT,
    OrderDate DATETIME2,
    Amount DECIMAL(10,2),
    INDEX IX_CustomerID NONCLUSTERED (CustomerID)
) WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_AND_DATA);
```

---

### Temporal Tables (System-Versioned)
- Automatically track the **complete history** of data changes.
- On UPDATE/DELETE, SQL Server automatically stores the previous version in a linked **history table** with timestamps.
- Requires two `DATETIME2` columns (`SysStartTime`, `SysEndTime`) and a `PERIOD FOR SYSTEM_TIME` clause.
- Data modifications use normal `INSERT`/`UPDATE`/`DELETE` — versioning is transparent.
- **Use cases:** Compliance/auditing, dispute troubleshooting, trend analysis, data recovery (revert accidental updates without backup restore), SCD Type 2 dimensions.
- **Benefits:** Zero application code changes, point-in-time queries with simple syntax, automatic history cleanup.
- **Trade-off:** Roughly **doubles storage requirements**.

```sql
CREATE TABLE Employee (
    EmployeeID INT PRIMARY KEY,
    EmployeeName NVARCHAR(100),
    Department NVARCHAR(50),
    SysStartTime DATETIME2 GENERATED ALWAYS AS ROW START,
    SysEndTime DATETIME2 GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (SysStartTime, SysEndTime)
) WITH (SYSTEM_VERSIONING = ON);

-- Query data as it existed at a point in time
SELECT * FROM Employee
FOR SYSTEM_TIME AS OF '2026-01-01'
WHERE EmployeeID = 1;
```

---

### External Tables
- Enable **data virtualization** — query data where it lives without moving it (no ETL required).
- **Use cases:** Query Parquet/CSV in Azure Data Lake, data exploration before importing, joining DB tables with external files, accessing archived blob storage, IoT JSON files.
- **Limitations:**
  - Often slower than native tables (network latency, file parsing).
  - Mostly **read-only** (can't UPDATE/DELETE in most scenarios).
  - Limited indexing and optimization.
  - No data movement or storage duplication.

```sql
CREATE EXTERNAL TABLE dbo.ExternalSalesData (
    OrderID INT, CustomerID INT, OrderAmount DECIMAL(10,2), OrderDate DATE
) WITH (
    LOCATION = '/raw/sales/',
    DATA_SOURCE = DataLakeSource,
    FILE_FORMAT = ParquetFormat
);
```

---

### Ledger Tables
- Use **cryptographic verification** (inspired by blockchain) to create tamper-evident records.
- Provides cryptographic proof of data integrity — proves data hasn't been modified by admins or backdated.
- **Use cases:** Financial transactions (banking, payments), supply chain provenance, legal records, healthcare prescriptions, government voting/land registries.

**Two types:**

| Type | Operations Allowed | Use When |
|---|---|---|
| Updatable Ledger | INSERT, UPDATE, DELETE | Need full change history with tamper-proof verification |
| Append-Only Ledger | INSERT only | Need absolutely immutable records |

- Can combine **Ledger + Temporal** on the same table for cryptographic verification + point-in-time query.
- SQL Server adds hidden columns and creates supporting objects to track the cryptographic chain.
- Every row modification generates a cryptographic hash linked to previous operations.
- Verify integrity with `sys.database_ledger_transactions` and `sp_verify_database_ledger`.

```sql
-- Updatable ledger
CREATE TABLE dbo.FinancialTransaction (
    TransactionID INT PRIMARY KEY IDENTITY,
    AccountNumber NVARCHAR(20),
    Amount DECIMAL(15,2),
    TransactionType NVARCHAR(20)
) WITH (LEDGER = ON);

-- Append-only ledger
CREATE TABLE dbo.AuditLog (
    LogID INT PRIMARY KEY IDENTITY,
    EventDescription NVARCHAR(500),
    EventTimestamp DATETIME2
) WITH (LEDGER = ON, APPEND_ONLY = ON);
```

---

### Graph Tables (SQL Graph)
- Natively model **nodes** (entities) and **edges** (relationships) for highly connected data.
- Simplifies queries like "friends of friends" or multi-hop relationship traversals.
- **Node tables:** Store entities; include hidden `$node_id` column.
- **Edge tables:** Store relationships; include hidden `$edge_id`, `$from_id`, `$to_id` columns.
- Use `MATCH` syntax for relationship traversal.
- **Avoid graph tables for:** Simple parent-child (foreign keys work fine), primarily transactional data without complex relationships, highly structured stable schemas.

```sql
CREATE TABLE Person AS NODE;
CREATE TABLE Manages AS EDGE;

-- Query relationships using MATCH
SELECT Person1.name, Person2.name
FROM Person AS Person1, Manages, Person AS Person2
WHERE MATCH (Person1-(Manages)->Person2)
AND Person1.id = 1;
```

**Specialized Table Trade-offs Summary:**

| Table Type | Key Trade-off |
|---|---|
| In-memory | Data size limited by RAM |
| Temporal | Doubles storage |
| External | Network latency; mostly read-only |
| Ledger | Cannot delete records (append-only variant) |
| Graph | Requires learning MATCH syntax |

> Choose the right table type during design — these decisions are hard to change after deployment.

---

## Unit 5 — Enforce Data Integrity with Constraints

### Why Constraints Matter
- Application code can be bypassed via bulk imports, direct queries, ad-hoc scripts, or new applications.
- **Database constraints enforce rules at the engine level** — always apply regardless of access method.
- Every `INSERT`, `UPDATE`, `DELETE` must satisfy all constraints before the engine commits the change.

---

### Constraint Types

#### Primary Key
- Guarantees uniqueness + enforces entity integrity.
- Automatically creates a **unique clustered index** on PK columns.
- Only **one PK per table**.
- All PK columns must be `NOT NULL`.

```sql
CREATE TABLE Customer (
    CustomerID INT PRIMARY KEY IDENTITY(1,1),
    EmailAddress NVARCHAR(100) NOT NULL
);
```

#### Foreign Key
- Enforces **referential integrity** between tables.
- Prevents changes to the referenced table if it would break existing links.
- **Cascading actions:** `CASCADE`, `SET NULL`, `SET DEFAULT` — define behavior when referenced key is deleted/updated.
- Does not automatically create an index, but indexing FK columns is recommended (frequently used in JOIN criteria).

```sql
CREATE TABLE [Order] (
    OrderID INT PRIMARY KEY IDENTITY,
    CustomerID INT NOT NULL,
    OrderDate DATETIME2,
    FOREIGN KEY (CustomerID) REFERENCES Customer(CustomerID)
);
```

#### Unique Constraint
- Prevents duplicate values in non-PK columns.
- Unlike PK, **allows NULL** — but only one NULL per column.
- Automatically creates a **unique nonclustered index**.

```sql
CREATE TABLE Product (
    ProductID INT PRIMARY KEY,
    SKU NVARCHAR(50) UNIQUE,
    ProductName NVARCHAR(100)
);
```

#### Check Constraint
- Enforces domain integrity — limits values accepted by columns.
- Uses any logical expression returning TRUE or FALSE.
- Multiple CHECK constraints can be applied to a single column.
- **Important:** `NULL` values evaluate to `UNKNOWN` — a CHECK constraint does NOT prevent NULL (e.g., `CHECK (Salary >= 20000)` still allows NULL for Salary).

```sql
CREATE TABLE Employee (
    EmployeeID INT PRIMARY KEY,
    HireDate DATE,
    Salary DECIMAL(10,2),
    CHECK (Salary >= 20000),
    CHECK (HireDate <= GETDATE())
);
```

#### Default Constraint
- Provides default values when no value is specified during INSERT.
- Best practice: use **explicit constraint names** (not system-generated) for portability across environments.

```sql
CREATE TABLE Activity (
    ActivityID INT PRIMARY KEY IDENTITY,
    Description NVARCHAR(200),
    CreatedDate DATETIME2 CONSTRAINT DF_Activity_CreatedDate DEFAULT GETUTCDATE(),
    IsActive BIT CONSTRAINT DF_Activity_IsActive DEFAULT 1
);
```

---

### Sequence Objects vs Identity Columns

#### Identity Columns
- Auto-incrementing numbers **tied to a specific table**.
- Simple, automatic, single-table only.

#### Sequence Objects
- **User-defined, schema-bound** numeric generator — not tied to any table.
- Application retrieves next value using `NEXT VALUE FOR`.
- Retrieved **outside transaction scope** — consumed whether transaction commits or rolls back.
- **Uniqueness is NOT automatically enforced** — create a UNIQUE constraint if needed.

**When to use Sequences instead of Identity:**

| Scenario | Use Sequence |
|---|---|
| Shared number series across multiple tables | ✅ |
| Need the next value BEFORE the INSERT | ✅ |
| Must cycle/restart at a specified number | ✅ |
| Reserve multiple sequential numbers at once | ✅ |
| Need to change increment value after creation | ✅ |
| Simple autoincrement for single table | ❌ Use Identity |

**Feature Comparison:**

| Feature | Sequence | Identity |
|---|---|---|
| Tied to table | No | Yes |
| Shared across tables | Yes | No |
| Get next value before INSERT | Yes | No |
| Custom min/max values | Yes | Limited |
| Retrieve multiple numbers at once | Yes | No |
| Cycle/restart at specified number | Yes | No |
| Change increment after creation | Yes | No |

```sql
CREATE SEQUENCE OrderNumber
    START WITH 1000
    INCREMENT BY 1
    MINVALUE 1000
    MAXVALUE 999999
    NO CYCLE;

-- Use inline in INSERT
INSERT INTO [Order] (OrderID, CustomerID, OrderNumber, OrderDate)
VALUES (1, 100, NEXT VALUE FOR OrderNumber, GETDATE());

-- Get value before INSERT
DECLARE @NextOrderNum INT = NEXT VALUE FOR OrderNumber;

-- Reserve a batch of numbers
EXEC sp_sequence_get_range
    @sequence_name = N'OrderNumber',
    @range_size = 100,
    @range_first_value = @FirstSeq OUTPUT,
    @range_last_value = @LastSeq OUTPUT;

-- Reset
ALTER SEQUENCE OrderNumber RESTART WITH 1000;
```

---

## Unit 6 — Manage JSON Columns and Indexes

### When to Use JSON Columns
- Best for **semi-structured or variable data** that differs per record (not fixed schemas).
- Use cases: user preferences, API responses, audit logs, multi-tenant custom fields, flexible metadata/tags.
- Avoid for data where every row has the same fields — use regular typed columns instead.

---

### JSON Data Type (SQL Server 2025+)
- Native `JSON` data type stores documents in **binary format** (optimized for querying/manipulation).
- Benefits over `NVARCHAR(MAX)`:
  - More efficient reads (already parsed).
  - More efficient writes (update individual values without rewriting entire document).
  - Better storage compression.
- For earlier SQL Server versions: store JSON in `NVARCHAR(MAX)`.

---

### Key JSON Functions

| Function | Purpose |
|---|---|
| `JSON_VALUE(col, '$.path')` | Extract a single scalar value |
| `JSON_QUERY(col, '$.path')` | Return an object or array |
| `JSON_MODIFY(col, '$.path', value)` | Update a value (works with both JSON and NVARCHAR(MAX)) |
| `JSON_PATH_EXISTS(col, '$.path')` | Check if a path exists (useful in CHECK constraints) |
| `.modify('$.path', value)` | Update single property without rewriting document (SQL 2025+ preview) |

---

### Indexing JSON Properties
- `JSON` type **cannot be used as an index key column directly**.
- Solution: Create a **computed column** that extracts the JSON property, then index that computed column.

```sql
-- Create table with native JSON type
CREATE TABLE ConfigurationData (
    ConfigID INT PRIMARY KEY,
    ConfigSettings JSON NOT NULL
);

-- Insert JSON
INSERT INTO ConfigurationData VALUES (1, '{"theme":"dark","language":"en","notifications":true}');

-- Query JSON properties
SELECT ConfigID,
       JSON_VALUE(ConfigSettings, '$.theme') AS Theme,
       JSON_VALUE(ConfigSettings, '$.language') AS Language
FROM ConfigurationData;

-- Index a frequently queried JSON property via computed column
ALTER TABLE ConfigurationData ADD ThemeValue AS JSON_VALUE(ConfigSettings, '$.theme');
CREATE INDEX IX_Theme ON ConfigurationData(ThemeValue);

-- Validate required properties with CHECK constraint
CREATE TABLE ProductMetadata (
    ProductID INT PRIMARY KEY,
    AdditionalAttributes JSON NOT NULL
        CHECK (JSON_PATH_EXISTS(AdditionalAttributes, '$.weight') = 1),
    FOREIGN KEY (ProductID) REFERENCES Product(ProductID)
);
```

---

### JSON Design Principles
- Use JSON for **semi-structured data only** — not consistent fixed schemas.
- Index frequently queried JSON paths via computed columns.
- Validate required properties with `CHECK` + `JSON_PATH_EXISTS`.
- Keep predictable data in regular typed columns; use JSON only for variable parts.
- JSON should **complement**, not replace, relational design.

---

## Unit 7 — Partition Tables for Scale

> ⚠️ Partitioning decisions are nearly permanent. Re-partitioning a multi-TB table requires a full rebuild with hours of downtime.

### Core Concepts

| Component | Purpose |
|---|---|
| **Partition Function** | Defines how data is divided (boundary values) |
| **Partition Scheme** | Maps partitions to filegroups |
| **Partition Column (key)** | Determines which partition each row belongs to |

---

### Benefits of Partitioning

**Query Performance:**
- **Partition elimination:** Queries filtering by partition key access only relevant partitions.
- **Parallel processing:** Multiple partitions processed simultaneously across CPU cores.
- Faster statistics (calculated per partition).
- Index seeks on smaller partitions = shallower B-trees.

**Operational:**
- **Granular maintenance:** Rebuild indexes on current partition while older partitions stay online.
- **Fast archival:** Switch old partitions to archive tables in seconds (metadata operation).
- **Tiered storage:** Move older partitions to cheaper, slower storage.
- Improved availability (maintain partitions independently).

---

### When to Use (or Avoid) Partitioning

| Scenario | Partition? | Why |
|---|---|---|
| 80%+ queries filter on a specific column (date, region) | ✅ Yes | Partition elimination |
| Regular archival of old data (monthly/quarterly) | ✅ Yes | Partition switching in seconds |
| Need to rebuild indexes on recent data only | ✅ Yes | Rebuild current partition independently |
| Large multi-TB tables with tiered storage | ✅ Yes | Move old partitions to cheaper storage |
| Most queries scan full table or filter on various columns | ❌ No | All partitions scanned — worse than unpartitioned |
| Single-row lookups or small range scans | ❌ No | Partitioning adds overhead without benefit |
| No clear column aligns with query patterns | ❌ No | Can't choose an effective partition key |

---

### Choosing a Partition Key
The partition key is the most important decision:
- Must appear in `WHERE` clause of **80%+ of queries**.
- Creates **reasonably balanced** partitions (no single partition with 90% of data).
- Aligns with **maintenance patterns** (date columns for time-based archival).
- Value should be **stable** — avoid columns that change after INSERT.

---

### Range Partitioning (Most Common)
- Divides data based on value ranges (most commonly dates).
- `RANGE RIGHT` for datetime columns: boundary value goes to the right partition (same-day values together).
- `RANGE LEFT` for string/categorical: boundary value goes to the left partition.

**Common Patterns:**

| Pattern | When to Use |
|---|---|
| Daily | High-volume, short retention |
| Weekly | Medium volume, 6-12 month retention |
| Monthly | Most common — balances partition count and size |
| Quarterly | Lower volume, multi-year retention |
| Yearly | Archive scenarios, long-term historical |

```sql
-- Partition function (RANGE RIGHT for dates)
CREATE PARTITION FUNCTION PF_OrderDate (DATETIME2)
    AS RANGE RIGHT FOR VALUES
    ('2024-01-01', '2024-04-01', '2024-07-01', '2024-10-01');
-- Creates 5 partitions: before 2024, Q1, Q2, Q3, Q4

-- Partition scheme
CREATE PARTITION SCHEME PS_OrderDate
    AS PARTITION PF_OrderDate ALL TO ([PRIMARY]);

-- Partitioned table (partition key must be in primary key)
CREATE TABLE Orders (
    OrderID INT NOT NULL,
    OrderDate DATETIME2 NOT NULL,
    CustomerID INT,
    Amount DECIMAL(10,2),
    CONSTRAINT PK_Orders PRIMARY KEY (OrderID, OrderDate)
) ON PS_OrderDate(OrderDate);
```

---

### Index Partitioning: Aligned vs Nonaligned

| Type | Description | Supports Partition Switching |
|---|---|---|
| **Aligned** | Same partition scheme as the table | ✅ Yes |
| **Nonaligned** | Different partitioning or no partitioning | ❌ No |

- Nonclustered indexes on partitioned tables **inherit the table's partition scheme by default**.
- Nonaligned indexes NOT supported on tables with more than 1,000 partitions.
- Use aligned indexes for partition switching, independent maintenance, and partition elimination.

```sql
-- Aligned nonclustered index
CREATE NONCLUSTERED INDEX IX_Orders_Customer
    ON Orders(CustomerID)
    ON PS_OrderDate(OrderDate);

-- Aligned columnstore index
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_SalesData_CS
    ON SalesData(Revenue, Region)
    ON PS_SalesDate(SaleDate);
```

---

### Managing Partitions Over Time

```sql
-- View partition info
SELECT
    $PARTITION.PF_OrderDate(OrderDate) AS PartitionNumber,
    MIN(OrderDate) AS MinDate,
    MAX(OrderDate) AS MaxDate,
    COUNT(*) AS RowCount
FROM Orders
GROUP BY $PARTITION.PF_OrderDate(OrderDate)
ORDER BY PartitionNumber;

-- Add new partition boundary (SPLIT)
ALTER PARTITION FUNCTION PF_OrderDate()
    SPLIT RANGE ('2024-11-01');

-- Archive/remove old partition (MERGE)
ALTER PARTITION FUNCTION PF_OrderDate()
    MERGE RANGE ('2023-12-31');
```

---

### Partitioning Best Practices
- **Align indexes** with table partitions (same partition scheme) to enable partition switching and maintenance.
- **Monitor data distribution** regularly to identify imbalanced partitions and verify partition elimination.
- **Automate partition management:** Schedule jobs to add new partitions before boundaries are reached.
- **Avoid over-partitioning:** Target millions of rows per partition (not thousands) — too many partitions create overhead.
- **Include partition key in primary key** — required for clustered index alignment on the partition scheme.

---

## Quick Reference Summary

| Object | Key Syntax | Notes |
|---|---|---|
| In-memory table | `WITH (MEMORY_OPTIMIZED = ON)` | No MAX types; data limited by RAM |
| Temporal table | `WITH (SYSTEM_VERSIONING = ON)` | Requires 2 DATETIME2 + PERIOD clause |
| External table | `CREATE EXTERNAL TABLE ... WITH (LOCATION, DATA_SOURCE, FILE_FORMAT)` | Read-only, no data movement |
| Ledger table | `WITH (LEDGER = ON)` | Append-only: add `APPEND_ONLY = ON` |
| Graph node | `CREATE TABLE X AS NODE` | Hidden `$node_id` |
| Graph edge | `CREATE TABLE X AS EDGE` | Hidden `$edge_id`, `$from_id`, `$to_id` |
| Sequence | `CREATE SEQUENCE ... NEXT VALUE FOR` | Not table-bound; uniqueness not auto-enforced |
| CCI | `CREATE CLUSTERED COLUMNSTORE INDEX` | Replaces clustered rowstore |
| NCCI | `CREATE NONCLUSTERED COLUMNSTORE INDEX` | Alongside existing rowstore |
| Partition function | `CREATE PARTITION FUNCTION ... AS RANGE RIGHT FOR VALUES (...)` | Defines boundaries |
| Partition scheme | `CREATE PARTITION SCHEME ... AS PARTITION ... ALL TO (...)` | Maps to filegroups |
| JSON index | Computed column + regular index | JSON type can't be indexed directly |

---

*End of DP-800 Module 1 Notes — Design and Implement Database Objects*