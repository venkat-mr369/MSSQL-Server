# T-SQL & Performance Tuning — In-Depth Interview Questions
## SQL Server All Versions: 2008 → 2012 → 2014 → 2016 → 2017 → 2019 → 2022 → 2025

> **150+ Questions with Detailed Answers**
> For Sr DBA / Database Developer / Performance Engineer Interviews

---
---

# SECTION A: T-SQL FUNDAMENTALS & ADVANCED

---

## A1. Core T-SQL Concepts

### Q1. What is the difference between WHERE and HAVING clause?

**Answer:** `WHERE` filters rows BEFORE grouping (works on individual rows). `HAVING` filters groups AFTER the `GROUP BY` is applied (works on aggregated results).

```sql
-- WHERE: Filters individual rows before aggregation
SELECT department, COUNT(*) AS emp_count
FROM employees
WHERE salary > 50000          -- filters rows first
GROUP BY department;

-- HAVING: Filters groups after aggregation
SELECT department, COUNT(*) AS emp_count
FROM employees
GROUP BY department
HAVING COUNT(*) > 10;         -- filters grouped results

-- Combined: WHERE first, then GROUP BY, then HAVING
SELECT department, AVG(salary) AS avg_salary
FROM employees
WHERE hire_date > '2020-01-01'   -- Step 1: filter rows
GROUP BY department               -- Step 2: group
HAVING AVG(salary) > 60000;      -- Step 3: filter groups
```

**Key Point:** `WHERE` cannot use aggregate functions (`SUM`, `COUNT`, `AVG`). `HAVING` can.

---

### Q2. Explain the SQL Server logical query processing order.

**Answer:** SQL Server processes a query in a specific logical order, which is different from how we write it:

```
  WRITING ORDER:          LOGICAL PROCESSING ORDER:
  1. SELECT               1. FROM & JOINs
  2. FROM                 2. WHERE
  3. WHERE                3. GROUP BY
  4. GROUP BY             4. HAVING
  5. HAVING               5. SELECT (& DISTINCT)
  6. ORDER BY             6. ORDER BY
                          7. TOP / OFFSET-FETCH
```

**Why this matters in interviews:** This explains why you can't use a column alias defined in `SELECT` inside `WHERE` — `WHERE` is processed before `SELECT`.

```sql
-- This FAILS:
SELECT salary * 12 AS annual_salary FROM employees WHERE annual_salary > 100000;
-- Error: "Invalid column name 'annual_salary'" — WHERE runs before SELECT

-- Correct approach: use a subquery or CTE
SELECT * FROM (
    SELECT salary * 12 AS annual_salary FROM employees
) sub WHERE annual_salary > 100000;
```

---

### Q3. What is the difference between DELETE, TRUNCATE, and DROP?

**Answer:**

| Feature | DELETE | TRUNCATE | DROP |
|---------|--------|----------|------|
| **What it removes** | Specific rows (with WHERE) | All rows | Entire table (structure + data) |
| **Logging** | Fully logged (row by row) | Minimally logged (page deallocation) | Fully logged |
| **WHERE clause** | Yes | No | No |
| **Transaction rollback** | Yes | Yes (contrary to popular myth!) | Yes |
| **Triggers fired** | Yes (DELETE trigger) | No | No |
| **Identity reset** | No (continues from last value) | Yes (resets to seed) | N/A |
| **Space reclaimed** | No (unless DBCC SHRINKFILE) | Yes (immediately) | Yes |
| **Permissions needed** | DELETE on table | ALTER on table | ALTER on schema |
| **Can use with FK** | Yes | No (if referenced by FK) | No (if referenced by FK) |
| **Speed on large table** | Slow (row-by-row log) | Very fast (page-level) | Fast |

**Common interview trap:** "TRUNCATE cannot be rolled back" — **FALSE!** TRUNCATE CAN be rolled back if inside a transaction:

```sql
BEGIN TRANSACTION;
TRUNCATE TABLE employees;    -- all rows gone
ROLLBACK;                     -- all rows restored!
-- TRUNCATE is minimally logged, but IS transactional
```

---

### Q4. Explain the different types of JOINs with examples.

**Answer:**

```sql
-- INNER JOIN: Only matching rows from both tables
SELECT e.name, d.dept_name
FROM employees e INNER JOIN departments d ON e.dept_id = d.dept_id;
-- Returns: only employees who have a matching department

-- LEFT (OUTER) JOIN: All rows from left + matching from right
SELECT e.name, d.dept_name
FROM employees e LEFT JOIN departments d ON e.dept_id = d.dept_id;
-- Returns: ALL employees, dept_name = NULL if no match

-- RIGHT (OUTER) JOIN: All rows from right + matching from left
SELECT e.name, d.dept_name
FROM employees e RIGHT JOIN departments d ON e.dept_id = d.dept_id;
-- Returns: ALL departments, name = NULL if no employee

-- FULL OUTER JOIN: All rows from both tables
SELECT e.name, d.dept_name
FROM employees e FULL OUTER JOIN departments d ON e.dept_id = d.dept_id;
-- Returns: everything from both, NULL where no match on either side

-- CROSS JOIN: Cartesian product (every row × every row)
SELECT e.name, d.dept_name
FROM employees e CROSS JOIN departments d;
-- Returns: employees × departments rows (if 100 emps × 10 depts = 1000 rows)

-- SELF JOIN: Table joined with itself
SELECT e.name AS employee, m.name AS manager
FROM employees e LEFT JOIN employees m ON e.manager_id = m.emp_id;
```

---

### Q5. What are Common Table Expressions (CTEs)? Explain recursive CTEs.

**Answer:** A CTE is a temporary named result set defined within a `WITH` clause. It exists only for the duration of the query.

```sql
-- Simple CTE
WITH high_earners AS (
    SELECT name, salary, department
    FROM employees
    WHERE salary > 100000
)
SELECT department, COUNT(*) AS count
FROM high_earners
GROUP BY department;

-- Multiple CTEs
WITH
  dept_totals AS (
      SELECT dept_id, SUM(salary) AS total_salary FROM employees GROUP BY dept_id
  ),
  dept_avg AS (
      SELECT AVG(total_salary) AS company_avg FROM dept_totals
  )
SELECT d.dept_id, d.total_salary, a.company_avg,
       d.total_salary - a.company_avg AS diff
FROM dept_totals d CROSS JOIN dept_avg a;
```

**Recursive CTE** — for hierarchical data (org charts, bill of materials, tree structures):

```sql
-- Employee hierarchy: Who reports to whom?
WITH org_chart AS (
    -- Anchor: Start with the CEO (no manager)
    SELECT emp_id, name, manager_id, 0 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: Find employees who report to the previous level
    SELECT e.emp_id, e.name, e.manager_id, oc.level + 1
    FROM employees e
    INNER JOIN org_chart oc ON e.manager_id = oc.emp_id
)
SELECT level, REPLICATE('  ', level) + name AS org_tree
FROM org_chart
ORDER BY level, name
OPTION (MAXRECURSION 100);  -- default is 100, 0 = unlimited
```

---

### Q6. Explain Window Functions with examples (ROW_NUMBER, RANK, DENSE_RANK, NTILE, LAG, LEAD).

**Answer:** Window functions perform calculations across a set of rows related to the current row without collapsing the result set (unlike GROUP BY).

```sql
-- Sample data scenario: Employee salaries by department

-- ROW_NUMBER(): Unique sequential number (no ties)
SELECT name, department, salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num
FROM employees;
-- Result: 1, 2, 3, 4... (even if salaries are equal, numbers are unique)

-- RANK(): Ranking with gaps for ties
SELECT name, department, salary,
    RANK() OVER (ORDER BY salary DESC) AS rank_num
FROM employees;
-- If two people have same salary: 1, 2, 2, 4 (skips 3)

-- DENSE_RANK(): Ranking without gaps for ties
SELECT name, department, salary,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank_num
FROM employees;
-- If two people have same salary: 1, 2, 2, 3 (no gap)

-- Comparison table:
-- Salary: 100K, 90K, 90K, 80K
-- ROW_NUMBER: 1, 2, 3, 4
-- RANK:       1, 2, 2, 4
-- DENSE_RANK: 1, 2, 2, 3

-- NTILE(n): Divides rows into n groups
SELECT name, salary,
    NTILE(4) OVER (ORDER BY salary DESC) AS quartile
FROM employees;
-- Divides all employees into 4 equal groups (quartiles)

-- LAG(): Access previous row's value (introduced in SQL 2012)
-- LEAD(): Access next row's value
SELECT name, salary,
    LAG(salary, 1, 0) OVER (ORDER BY hire_date) AS prev_salary,
    LEAD(salary, 1, 0) OVER (ORDER BY hire_date) AS next_salary,
    salary - LAG(salary, 1, 0) OVER (ORDER BY hire_date) AS salary_diff
FROM employees;

-- Running total (cumulative sum)
SELECT order_date, amount,
    SUM(amount) OVER (ORDER BY order_date ROWS UNBOUNDED PRECEDING) AS running_total
FROM orders;

-- Moving average (last 3 orders)
SELECT order_date, amount,
    AVG(amount) OVER (ORDER BY order_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg_3
FROM orders;

-- FIRST_VALUE / LAST_VALUE (SQL 2012+)
SELECT name, department, salary,
    FIRST_VALUE(name) OVER (PARTITION BY department ORDER BY salary DESC) AS top_earner
FROM employees;
```

---

### Q7. What is the difference between UNION and UNION ALL?

**Answer:**
- **UNION:** Combines results and REMOVES duplicate rows (performs DISTINCT sort — slower)
- **UNION ALL:** Combines results and KEEPS ALL rows including duplicates (faster, no sort)

**Performance tip:** Always use `UNION ALL` unless you specifically need to eliminate duplicates. `UNION` forces a sort/hash operation which is expensive on large datasets.

```sql
-- UNION: removes duplicates (slower)
SELECT city FROM customers
UNION
SELECT city FROM suppliers;

-- UNION ALL: keeps duplicates (faster)
SELECT city FROM customers
UNION ALL
SELECT city FROM suppliers;
```

---

### Q8. Explain APPLY operator (CROSS APPLY vs OUTER APPLY).

**Answer:** `APPLY` is like a JOIN but evaluates a table-valued expression for EACH row of the outer table. Introduced in SQL 2005.

```sql
-- CROSS APPLY: Like INNER JOIN for table-valued functions
-- Returns only rows where the function returns results
SELECT d.dept_name, e.name, e.salary
FROM departments d
CROSS APPLY (
    SELECT TOP 3 name, salary
    FROM employees
    WHERE dept_id = d.dept_id
    ORDER BY salary DESC
) e;
-- Returns top 3 earners per department (only departments that have employees)

-- OUTER APPLY: Like LEFT JOIN for table-valued functions
-- Returns all rows from left, NULL where function returns nothing
SELECT d.dept_name, e.name, e.salary
FROM departments d
OUTER APPLY (
    SELECT TOP 3 name, salary
    FROM employees
    WHERE dept_id = d.dept_id
    ORDER BY salary DESC
) e;
-- Returns ALL departments, even if they have no employees (with NULLs)
```

**When to use APPLY:**
- Call a table-valued function for each row
- Get TOP N per group (more efficient than ROW_NUMBER in some cases)
- When one side of the join depends on values from the other side

---

### Q9. What is PIVOT and UNPIVOT? Give practical examples.

**Answer:**

```sql
-- PIVOT: Converts rows into columns
-- Scenario: Monthly sales by product
SELECT * FROM (
    SELECT product, month_name, revenue
    FROM monthly_sales
) src
PIVOT (
    SUM(revenue)
    FOR month_name IN ([Jan], [Feb], [Mar], [Apr], [May], [Jun])
) pvt;
-- Result: One row per product, columns for each month

-- UNPIVOT: Converts columns into rows (reverse of PIVOT)
SELECT product, month_name, revenue
FROM sales_wide
UNPIVOT (
    revenue FOR month_name IN ([Jan], [Feb], [Mar], [Apr], [May], [Jun])
) unpvt;
```

---

### Q10. Explain MERGE statement (UPSERT).

**Answer:** `MERGE` performs INSERT, UPDATE, and DELETE in a single atomic statement based on matching conditions. Introduced in SQL 2008.

```sql
MERGE INTO target_table AS tgt
USING source_table AS src
ON tgt.id = src.id

WHEN MATCHED AND src.is_deleted = 1 THEN
    DELETE

WHEN MATCHED THEN
    UPDATE SET tgt.name = src.name, tgt.salary = src.salary, tgt.modified_date = GETDATE()

WHEN NOT MATCHED BY TARGET THEN
    INSERT (id, name, salary, created_date)
    VALUES (src.id, src.name, src.salary, GETDATE())

WHEN NOT MATCHED BY SOURCE THEN
    DELETE

OUTPUT $action, INSERTED.*, DELETED.*;  -- track what happened
```

**Interview tip:** MERGE has known concurrency bugs in SQL Server (race conditions with parallel execution). Many senior DBAs prefer separate INSERT/UPDATE/DELETE with proper locking hints over MERGE in high-concurrency OLTP systems.

---

## A2. T-SQL Version-Specific Features

### Q11. What T-SQL features were introduced in each SQL Server version?

**Answer — Complete Version Timeline:**

```
SQL SERVER 2008:
├── MERGE statement (UPSERT)
├── Table-Valued Parameters (TVPs)
├── GROUPING SETS, CUBE, ROLLUP
├── Sparse Columns
├── Filtered Indexes
├── INSERT...EXEC with OUTPUT
├── DATETIME2, DATE, TIME, DATETIMEOFFSET data types
├── Compound operators (+=, -=, *=, /=)
└── HIERARCHYID data type

SQL SERVER 2008 R2:
├── Multi-Server Management
├── PowerPivot for Excel
└── StreamInsight (complex event processing)

SQL SERVER 2012:
├── Window Function enhancements (LAG, LEAD, FIRST_VALUE, LAST_VALUE)
├── ROWS/RANGE frame specification for window functions
├── SEQUENCE objects
├── OFFSET-FETCH (pagination)
├── IIF() and CHOOSE() functions
├── TRY_CONVERT(), TRY_PARSE(), FORMAT()
├── THROW statement (replacing RAISERROR)
├── WITH RESULT SETS (EXEC)
├── Contained Databases
├── FileTables
└── Always On Availability Groups

SQL SERVER 2014:
├── In-Memory OLTP (Hekaton) — natively compiled tables & procedures
├── Clustered Columnstore Indexes (updatable)
├── Buffer Pool Extension (SSD)
├── Delayed Durability (lazy commit)
├── Inline Index specification in CREATE TABLE
├── SELECT INTO parallelism
├── Cardinality Estimator v2 (CE70 → CE120)
└── Managed Lock Priority (ALTER INDEX ... WAIT_AT_LOW_PRIORITY)

SQL SERVER 2016:
├── Always Encrypted
├── Dynamic Data Masking
├── Row-Level Security (RLS)
├── Temporal Tables (system-versioned)
├── JSON support (FOR JSON, OPENJSON, JSON_VALUE, JSON_QUERY)
├── STRING_SPLIT() function
├── DROP IF EXISTS (DROP TABLE IF EXISTS)
├── CREATE OR ALTER (procedures, functions, triggers, views)
├── Query Store
├── Live Query Statistics
├── Stretch Database
├── R Services (in-database analytics)
├── Polybase
└── Non-Clustered Columnstore on in-memory tables

SQL SERVER 2017:
├── Graph database support (NODE, EDGE tables)
├── Adaptive Query Processing (Batch Mode Adaptive Join, 
│   Interleaved Execution, Memory Grant Feedback)
├── Automatic Tuning (force/unforce plans)
├── STRING_AGG() function
├── TRIM(), CONCAT_WS(), TRANSLATE() functions
├── SELECT INTO with filegroup
├── SQL Server on Linux
├── Python integration (ML Services)
├── Resumable Online Index Rebuild
└── Clusterless Availability Groups

SQL SERVER 2019:
├── Intelligent Query Processing (IQP):
│   ├── Batch mode on rowstore
│   ├── Scalar UDF inlining
│   ├── Table variable deferred compilation
│   ├── Memory grant feedback (row mode + persistence)
│   └── Approximate query processing (APPROX_COUNT_DISTINCT)
├── Accelerated Database Recovery (ADR)
├── Verbose Truncation Warnings
├── UTF-8 support (CHAR/VARCHAR)
├── OPTIMIZE_FOR_SEQUENTIAL_KEY (last-page insert contention)
├── Hybrid Buffer Pool (persistent memory / PMEM)
├── Always Encrypted with Secure Enclaves
├── Big Data Clusters
├── Java extensibility
├── Tempdb metadata optimization
└── sys.dm_exec_describe_first_result_set_for_object

SQL SERVER 2022:
├── Ledger Tables (blockchain-verified data)
├── Parameter Sensitive Plan Optimization (PSP)
├── Cardinality Estimation feedback
├── DOP (Degree of Parallelism) feedback
├── Query Store hints
├── Optimized plan forcing
├── WINDOW clause (named window definitions)
├── IS [NOT] DISTINCT FROM
├── GENERATE_SERIES()
├── DATE_BUCKET()
├── GREATEST() / LEAST()
├── STRING_SPLIT() with ordinal
├── TRIM enhancements (LEADING, TRAILING, BOTH)
├── JSON improvements (JSON_OBJECT, JSON_ARRAY, JSON_PATH_EXISTS)
├── Bit manipulation functions (LEFT_SHIFT, RIGHT_SHIFT, BIT_COUNT)
├── Contained Availability Groups
├── Intel QAT compression acceleration
├── Buffer pool parallel scan
├── Tempdb improvements (system page latch concurrency)
├── Resumable ADD TABLE CONSTRAINT
├── BACKUP ... WITH ZSTD compression (new algorithm)
└── Azure Synapse Link for SQL Server

SQL SERVER 2025 (PREVIEW):
├── VECTOR data type (AI/ML embeddings)
├── Native vector search (DiskANN)
├── Native REST API (no middleware needed)
├── Mirrored databases to Fabric/OneLake
├── Standard Edition: up to 32 cores (was 24)
├── SQL 2025 CTP improvements
└── Enhanced AI integration
```

---

### Q12. Explain Temporal Tables with examples (SQL 2016+).

**Answer:** Temporal Tables (system-versioned) automatically maintain a complete history of data changes. SQL Server tracks every INSERT, UPDATE, and DELETE with valid_from and valid_to timestamps.

```sql
-- Create a temporal table
CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    name NVARCHAR(100),
    salary DECIMAL(10,2),
    department NVARCHAR(50),
    valid_from DATETIME2 GENERATED ALWAYS AS ROW START,
    valid_to DATETIME2 GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (valid_from, valid_to)
) WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.employees_history));

-- Normal DML — history is tracked automatically!
UPDATE employees SET salary = 120000 WHERE emp_id = 1;
-- Old row moves to employees_history automatically

-- Query data AS OF a specific point in time
SELECT * FROM employees FOR SYSTEM_TIME AS OF '2024-06-15 10:30:00';

-- Query all changes between two dates
SELECT * FROM employees FOR SYSTEM_TIME BETWEEN '2024-01-01' AND '2024-12-31';

-- Query all versions of a row (ALL keyword)
SELECT * FROM employees FOR SYSTEM_TIME ALL WHERE emp_id = 1 ORDER BY valid_from;
```

**Use Cases:** Audit trails, regulatory compliance, point-in-time reporting, slowly changing dimensions.

---

### Q13. Explain JSON support in SQL Server (2016+).

```sql
-- Parse JSON in SQL Server
DECLARE @json NVARCHAR(MAX) = N'[
    {"name":"Venkat", "salary":95000, "skills":["SQL","Python"]},
    {"name":"Kishore", "salary":88000, "skills":["MySQL","AWS"]}
]';

-- OPENJSON: Parse JSON into rows
SELECT * FROM OPENJSON(@json)
WITH (
    name NVARCHAR(100) '$.name',
    salary INT '$.salary',
    first_skill NVARCHAR(50) '$.skills[0]'
);

-- JSON_VALUE: Extract scalar value
SELECT JSON_VALUE(@json, '$[0].name');  -- Returns: Venkat

-- JSON_QUERY: Extract object or array
SELECT JSON_QUERY(@json, '$[0].skills');  -- Returns: ["SQL","Python"]

-- FOR JSON: Convert table to JSON
SELECT name, salary FROM employees FOR JSON PATH;
-- Output: [{"name":"Venkat","salary":95000},...]

-- FOR JSON AUTO (auto-nesting based on joins)
SELECT d.dept_name, e.name, e.salary
FROM departments d JOIN employees e ON d.dept_id = e.dept_id
FOR JSON AUTO;

-- SQL 2022: JSON_OBJECT and JSON_ARRAY
SELECT JSON_OBJECT('name': name, 'salary': salary) FROM employees;
SELECT JSON_ARRAY(1, 2, 'hello', NULL);

-- ISJSON(): Validate JSON
SELECT ISJSON(@json);  -- Returns 1 if valid
```

---

### Q14. What is STRING_AGG and how does it differ from FOR XML PATH?

**Answer:** `STRING_AGG()` (SQL 2017+) concatenates string values with a separator — replacing the old `FOR XML PATH` trick.

```sql
-- OLD way (SQL 2005-2016): FOR XML PATH (ugly but works)
SELECT department,
    STUFF((SELECT ', ' + name FROM employees e2 
           WHERE e2.department = e1.department 
           FOR XML PATH(''), TYPE).value('.','NVARCHAR(MAX)'), 1, 2, '') AS employees_list
FROM employees e1
GROUP BY department;

-- NEW way (SQL 2017+): STRING_AGG (clean and fast!)
SELECT department,
    STRING_AGG(name, ', ') WITHIN GROUP (ORDER BY name) AS employees_list
FROM employees
GROUP BY department;
```

---

### Q15. Explain the WINDOW clause introduced in SQL Server 2022.

**Answer:** SQL 2022 introduced named window definitions to avoid repeating the same `OVER()` clause:

```sql
-- Before SQL 2022: Repetitive OVER clauses
SELECT name, salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC),
    RANK()       OVER (PARTITION BY department ORDER BY salary DESC),
    SUM(salary)  OVER (PARTITION BY department ORDER BY salary DESC),
    AVG(salary)  OVER (PARTITION BY department ORDER BY salary DESC)
FROM employees;

-- SQL 2022: Named WINDOW clause (DRY principle!)
SELECT name, salary,
    ROW_NUMBER() OVER w,
    RANK()       OVER w,
    SUM(salary)  OVER w,
    AVG(salary)  OVER w
FROM employees
WINDOW w AS (PARTITION BY department ORDER BY salary DESC);
```

---

### Q16. What are new functions in SQL Server 2022?

```sql
-- GREATEST / LEAST: Return max/min from a list of values
SELECT GREATEST(10, 20, 5, 30);    -- Returns: 30
SELECT LEAST(10, 20, 5, 30);       -- Returns: 5
SELECT GREATEST(salary, bonus, commission) FROM employees;

-- GENERATE_SERIES: Generate a sequence of numbers
SELECT value FROM GENERATE_SERIES(1, 10);          -- 1 to 10
SELECT value FROM GENERATE_SERIES(1, 100, 5);      -- 1, 6, 11, ..., 96

-- DATE_BUCKET: Group dates into fixed-width buckets
SELECT DATE_BUCKET(WEEK, 1, order_date) AS week_start, COUNT(*) AS orders
FROM orders GROUP BY DATE_BUCKET(WEEK, 1, order_date);

-- IS [NOT] DISTINCT FROM: NULL-safe comparison
SELECT * FROM employees WHERE salary IS NOT DISTINCT FROM bonus;
-- Unlike salary = bonus, this returns TRUE when BOTH are NULL

-- STRING_SPLIT with ordinal (SQL 2022)
SELECT value, ordinal FROM STRING_SPLIT('SQL,MySQL,PostgreSQL', ',', 1);
-- Returns: (SQL, 1), (MySQL, 2), (PostgreSQL, 3)

-- TRIM enhancements
SELECT TRIM(LEADING '0' FROM '000123');   -- Returns: '123'
SELECT TRIM(TRAILING '.' FROM 'hello...'); -- Returns: 'hello'

-- BIT_COUNT: Count set bits
SELECT BIT_COUNT(15);  -- Returns: 4 (binary 1111)
```

---

## A3. Error Handling & Transactions

### Q17. Explain TRY...CATCH error handling in T-SQL.

```sql
BEGIN TRY
    BEGIN TRANSACTION;
    
    INSERT INTO orders (customer_id, amount) VALUES (1, 500.00);
    UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 100;
    
    -- If product doesn't exist, this will fail:
    IF @@ROWCOUNT = 0
        THROW 50001, 'Product not found in inventory', 1;
    
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;
    
    -- Capture error details
    SELECT
        ERROR_NUMBER()    AS ErrorNumber,
        ERROR_SEVERITY()  AS ErrorSeverity,
        ERROR_STATE()     AS ErrorState,
        ERROR_PROCEDURE() AS ErrorProcedure,
        ERROR_LINE()      AS ErrorLine,
        ERROR_MESSAGE()   AS ErrorMessage;
    
    -- Re-throw the original error (SQL 2012+)
    THROW;
END CATCH;
```

### Q18. What is the difference between RAISERROR and THROW?

| Feature | RAISERROR | THROW (SQL 2012+) |
|---------|-----------|-------------------|
| Re-throw original error | No (must rebuild) | Yes (just `THROW;`) |
| Requires message ID | Can use ad-hoc msg | Error number 50000+ |
| Format strings | Yes (%s, %d, etc.) | No |
| Severity parameter | Required | Always 16 |
| Requires semicolon | No | Yes (before THROW) |

**Best Practice:** Use `THROW` for SQL 2012+ code. Use `RAISERROR` only when you need format strings or severity control.

---

### Q19. Explain Transaction Isolation Levels.

```
┌────────────────────────┬──────────┬──────────────────┬──────────────────┬──────────┐
│ Isolation Level        │ Dirty    │ Non-Repeatable   │ Phantom          │ Locking  │
│                        │ Reads    │ Reads            │ Reads            │ Impact   │
├────────────────────────┼──────────┼──────────────────┼──────────────────┼──────────┤
│ READ UNCOMMITTED       │ Yes      │ Yes              │ Yes              │ None     │
│ READ COMMITTED (def)   │ No       │ Yes              │ Yes              │ Low      │
│ REPEATABLE READ        │ No       │ No               │ Yes              │ Medium   │
│ SERIALIZABLE           │ No       │ No               │ No               │ High     │
│ SNAPSHOT               │ No       │ No               │ No               │ None*    │
│ READ COMMITTED SNAPSHOT│ No       │ Yes              │ Yes              │ None*    │
└────────────────────────┴──────────┴──────────────────┴──────────────────┴──────────┘
* Uses row versioning in tempdb instead of locks
```

**Dirty Read:** Reading uncommitted data from another transaction.
**Non-Repeatable Read:** Same query returns different values within the same transaction.
**Phantom Read:** Same query returns different rows (new rows inserted by another transaction).

```sql
-- Set isolation level
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Enable SNAPSHOT isolation (database level)
ALTER DATABASE mydb SET ALLOW_SNAPSHOT_ISOLATION ON;

-- Enable READ COMMITTED SNAPSHOT (recommended for OLTP)
ALTER DATABASE mydb SET READ_COMMITTED_SNAPSHOT ON;
-- This makes READ COMMITTED use row versioning instead of locks
-- Readers don't block writers, writers don't block readers
```

---

## A4. Stored Procedures, Functions & Triggers

### Q20. What is the difference between Stored Procedures and Functions?

| Feature | Stored Procedure | Function |
|---------|-----------------|----------|
| Return type | 0 or more result sets + output params | Single value (scalar) or table |
| Used in SELECT | No | Yes (SELECT dbo.fn_GetName(1)) |
| Used in WHERE | No | Yes |
| DML allowed | Yes (INSERT, UPDATE, DELETE) | No (read-only) |
| TRY...CATCH | Yes | No |
| Transactions | Yes | No |
| Non-deterministic funcs | Yes (GETDATE, NEWID, etc.) | Limited |
| Temp tables | Yes | No (table variables only) |
| Performance | Can be cached | Can cause row-by-row execution |

**Critical Performance Point:** Scalar UDFs in SQL Server (before 2019) execute row-by-row, causing massive performance problems. SQL 2019 introduced **Scalar UDF Inlining** which fixes this.

---

### Q21. What are the types of triggers? When to use each?

```sql
-- DML Trigger (AFTER): Fires after INSERT/UPDATE/DELETE
CREATE TRIGGER tr_audit_employees
ON employees
AFTER UPDATE
AS
BEGIN
    INSERT INTO audit_log (table_name, action, old_value, new_value, changed_date)
    SELECT 'employees', 'UPDATE', 
           d.salary, i.salary, GETDATE()
    FROM INSERTED i JOIN DELETED d ON i.emp_id = d.emp_id;
END;

-- DML Trigger (INSTEAD OF): Replaces the original action
-- Commonly used on views to make them updatable
CREATE TRIGGER tr_instead_of_delete
ON employees
INSTEAD OF DELETE
AS
BEGIN
    -- Soft delete instead of hard delete
    UPDATE employees SET is_active = 0, deleted_date = GETDATE()
    WHERE emp_id IN (SELECT emp_id FROM DELETED);
END;

-- DDL Trigger: Fires on schema changes (CREATE, ALTER, DROP)
CREATE TRIGGER tr_prevent_table_drop
ON DATABASE
FOR DROP_TABLE
AS
BEGIN
    PRINT 'Table drops are not allowed! Contact DBA team.';
    ROLLBACK;
END;

-- LOGON Trigger (SQL 2005+): Fires when a session is established
CREATE TRIGGER tr_limit_connections
ON ALL SERVER
FOR LOGON
AS
BEGIN
    IF (SELECT COUNT(*) FROM sys.dm_exec_sessions 
        WHERE login_name = ORIGINAL_LOGIN()) > 10
    BEGIN
        PRINT 'Max 10 connections per user exceeded!';
        ROLLBACK;
    END;
END;
```

---
---

# SECTION B: PERFORMANCE TUNING — DEEP DIVE

---

## B1. Indexing Strategy

### Q22. Explain the types of indexes in SQL Server.

**Answer:**

```
INDEX TYPES IN SQL SERVER:

1. CLUSTERED INDEX
   ├── Defines physical order of data in the table
   ├── Only ONE per table (it IS the table)
   ├── Leaf level = actual data rows
   ├── Best for: range queries, ORDER BY, primary keys
   └── If no clustered index → table is a HEAP

2. NON-CLUSTERED INDEX
   ├── Separate structure pointing to clustered key (or RID for heaps)
   ├── Up to 999 per table
   ├── Leaf level = index key + pointer to data
   ├── Best for: lookup queries, WHERE clause columns, JOINs
   └── May cause KEY LOOKUP if query needs columns not in the index

3. COLUMNSTORE INDEX (SQL 2012+)
   ├── Stores data by column instead of by row
   ├── Massive compression (10x reduction)
   ├── Batch mode execution (processes 900 rows at a time)
   ├── Best for: analytics, data warehouse, aggregations
   ├── Clustered Columnstore (SQL 2014+): replaces entire table
   └── Non-Clustered Columnstore: secondary index alongside rowstore

4. FILTERED INDEX (SQL 2008+)
   ├── Index with a WHERE clause
   ├── Smaller size, better performance, less maintenance
   ├── Example: INDEX ON orders(customer_id) WHERE status = 'Active'
   └── Best for: queries that always filter on a specific condition

5. COVERING INDEX / INCLUDED COLUMNS (SQL 2005+)
   ├── Non-clustered index with additional columns in leaf
   ├── Avoids KEY LOOKUP (no need to go back to table)
   └── CREATE INDEX ix_name ON table(col1) INCLUDE(col2, col3)

6. FULL-TEXT INDEX
   ├── For searching text content (CONTAINS, FREETEXT)
   └── Uses Full-Text Search engine

7. XML INDEX / SPATIAL INDEX / HASH INDEX (In-Memory OLTP)
```

---

### Q23. What is a Key Lookup and how do you eliminate it?

**Answer:** A Key Lookup (or Bookmark Lookup) occurs when SQL Server finds a row using a non-clustered index but then must go back to the clustered index (or heap) to get additional columns requested in the query.

```sql
-- Table: orders (clustered index on order_id)
-- Non-clustered index: ix_customer on (customer_id)

-- This query causes a KEY LOOKUP:
SELECT order_id, customer_id, order_date, total_amount
FROM orders
WHERE customer_id = 12345;
-- SQL uses ix_customer to find customer_id = 12345
-- But order_date and total_amount are NOT in the index
-- So it does a KEY LOOKUP to the clustered index for each row

-- FIX: Create a COVERING INDEX with INCLUDE columns
CREATE NONCLUSTERED INDEX ix_customer_covering
ON orders (customer_id)
INCLUDE (order_date, total_amount);
-- Now the index has everything the query needs — no lookup!
```

**Performance Impact:** A Key Lookup turns a single index seek into seek + lookup for EVERY row. On 10,000 matching rows, that's 10,000 additional I/O operations.

---

### Q24. What is a Filtered Index and when to use it?

```sql
-- Regular index: covers ALL rows (including NULLs, inactive, etc.)
CREATE INDEX ix_all_orders ON orders(status, order_date);
-- Size: indexes all 10 million rows

-- Filtered index: covers ONLY rows matching the WHERE clause
CREATE INDEX ix_active_orders ON orders(status, order_date)
WHERE status = 'Active';
-- Size: indexes only the 50,000 active orders (much smaller!)

-- Use cases:
-- 1. Sparse columns (mostly NULL)
CREATE INDEX ix_discount ON orders(discount_code) WHERE discount_code IS NOT NULL;

-- 2. Soft deletes
CREATE INDEX ix_active_employees ON employees(department, name) WHERE is_deleted = 0;

-- 3. Status-based queries
CREATE INDEX ix_pending ON orders(created_date) WHERE status = 'Pending';
```

---

### Q25. Explain Columnstore Indexes and Batch Mode execution.

**Answer:** Columnstore indexes store data by column instead of by row, enabling massive compression and batch mode execution for analytical queries.

```
  ROW STORE (Traditional):                  COLUMN STORE:
  ┌─────┬──────┬────────┬────────┐        ┌─────┐  ┌──────┐  ┌────────┐  ┌────────┐
  │ ID  │ Name │ Salary │ Dept   │        │ ID  │  │ Name │  │ Salary │  │ Dept   │
  ├─────┼──────┼────────┼────────┤        ├─────┤  ├──────┤  ├────────┤  ├────────┤
  │  1  │ Ven  │ 90000  │ IT     │        │  1  │  │ Ven  │  │ 90000  │  │ IT     │
  │  2  │ Kis  │ 85000  │ IT     │        │  2  │  │ Kis  │  │ 85000  │  │ IT     │
  │  3  │ Mee  │ 78000  │ HR     │        │  3  │  │ Mee  │  │ 78000  │  │ HR     │
  └─────┴──────┴────────┴────────┘        └─────┘  └──────┘  └────────┘  └────────┘
  Reads ALL columns for each row            Reads ONLY the columns needed
  Good for: OLTP (point lookups)            Good for: OLAP (aggregations)
```

```sql
-- Create Clustered Columnstore Index (replaces table storage)
CREATE CLUSTERED COLUMNSTORE INDEX cci_sales ON fact_sales;

-- Create Non-Clustered Columnstore Index (alongside rowstore)
CREATE NONCLUSTERED COLUMNSTORE INDEX ncci_orders
ON orders (order_date, customer_id, total_amount);

-- SQL 2019: Batch Mode on Rowstore
-- Even without columnstore, SQL 2019+ can use batch mode execution
-- on rowstore tables for eligible queries (aggregations, window functions)
```

**Batch Mode vs Row Mode:**
- Row Mode: processes 1 row at a time
- Batch Mode: processes ~900 rows at a time (much faster for aggregations)
- SQL 2019+ enables batch mode even on rowstore tables!

---

## B2. Query Execution Plans

### Q26. How do you read an execution plan? What are the key things to look for?

**Answer:**

```
  KEY THINGS TO CHECK IN EXECUTION PLANS:

  1. ESTIMATED vs ACTUAL ROW COUNTS
     └── If estimated = 100 but actual = 1,000,000 → BAD cardinality estimate
         This causes wrong plan choices (wrong join type, wrong memory grant)

  2. COSTLY OPERATORS (% of total cost)
     ├── Table Scan → Missing index or no usable index
     ├── Clustered Index Scan → May need a non-clustered index
     ├── Key Lookup → Need INCLUDE columns in index
     ├── Sort → Missing index with proper ORDER
     ├── Hash Match → Large tables with no index for join
     └── Parallelism → May be unnecessary on small queries

  3. WARNING SYMBOLS (yellow triangle ⚠)
     ├── Missing Index suggestion
     ├── Implicit Conversion (kills SARGability!)
     ├── Residual Predicate (filter after seek, not ideal)
     ├── Spill to Tempdb (memory grant too low)
     └── No Join Predicate (accidental cross join)

  4. WAIT STATS in Actual Plan (SQL 2016+)
     └── Shows what the query waited on (IO, CPU, memory, locks)

  5. MEMORY GRANT
     ├── Requested vs Used → if used << requested, memory is wasted
     └── If grant too low → spills to tempdb (very slow)
```

```sql
-- View estimated plan (doesn't execute)
SET SHOWPLAN_XML ON;
GO
SELECT * FROM orders WHERE customer_id = 123;
GO
SET SHOWPLAN_XML OFF;

-- View actual plan (executes the query)
SET STATISTICS XML ON;
SELECT * FROM orders WHERE customer_id = 123;
SET STATISTICS XML OFF;

-- Quick I/O and time stats
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
SELECT * FROM orders WHERE customer_id = 123;
-- Look at: logical reads, physical reads, scan count
```

---

### Q27. What is SARGability and why is it critical?

**Answer:** SARG = Search ARGument. A SARGable predicate allows SQL Server to use an index SEEK instead of a SCAN.

```sql
-- NON-SARGable (forces INDEX SCAN — slow!):
WHERE YEAR(order_date) = 2024              -- function on column
WHERE ISNULL(status, 'Unknown') = 'Active' -- function on column
WHERE salary + 1000 > 50000                -- expression on column
WHERE name LIKE '%smith'                   -- leading wildcard
WHERE CAST(emp_id AS VARCHAR) = '123'      -- implicit conversion
WHERE LEFT(phone, 3) = '091'              -- function on column

-- SARGable (allows INDEX SEEK — fast!):
WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01'
WHERE status = 'Active'
WHERE salary > 49000
WHERE name LIKE 'smith%'                   -- trailing wildcard OK
WHERE emp_id = 123
WHERE phone LIKE '091%'
```

**Rule of thumb:** Never put a function or calculation on the LEFT side of a WHERE clause comparison. Keep the column clean.

---

### Q28. Explain Parameter Sniffing and how to handle it.

**Answer:** Parameter Sniffing means SQL Server compiles a stored procedure's execution plan based on the FIRST parameter values it receives. If those values are atypical, the plan may be terrible for subsequent executions.

```sql
-- Problem scenario:
CREATE PROCEDURE sp_GetOrders @status VARCHAR(20)
AS
SELECT * FROM orders WHERE status = @status;

-- First call: EXEC sp_GetOrders 'Cancelled'  (100 rows → Index Seek plan)
-- Second call: EXEC sp_GetOrders 'Active'    (5 million rows → needs Table Scan!)
-- But it REUSES the Index Seek plan from first call → terrible performance!
```

**Solutions:**

```sql
-- Solution 1: OPTIMIZE FOR hint
CREATE PROCEDURE sp_GetOrders @status VARCHAR(20)
AS
SELECT * FROM orders WHERE status = @status
OPTION (OPTIMIZE FOR (@status = 'Active'));  -- plan for the common case

-- Solution 2: OPTIMIZE FOR UNKNOWN
OPTION (OPTIMIZE FOR UNKNOWN);  -- uses average statistics

-- Solution 3: RECOMPILE hint (fresh plan every time — use for infrequent queries)
OPTION (RECOMPILE);

-- Solution 4: Local variable (breaks parameter sniffing)
CREATE PROCEDURE sp_GetOrders @status VARCHAR(20)
AS
DECLARE @local_status VARCHAR(20) = @status;
SELECT * FROM orders WHERE status = @local_status;

-- Solution 5: Query Store plan forcing (SQL 2016+)
EXEC sp_query_store_force_plan @query_id = 42, @plan_id = 3;

-- Solution 6: Parameter Sensitive Plan Optimization (SQL 2022)
-- SQL Server automatically creates multiple plans for different parameter ranges!
ALTER DATABASE SCOPED CONFIGURATION SET PARAMETER_SENSITIVE_PLAN_OPTIMIZATION = ON;
```

---

## B3. DMVs for Performance Tuning

### Q29. What are the most important DMVs for a DBA?

```sql
-- TOP QUERIES BY CPU
SELECT TOP 20
    qs.total_worker_time/1000 AS total_cpu_ms,
    qs.execution_count,
    qs.total_worker_time/qs.execution_count/1000 AS avg_cpu_ms,
    SUBSTRING(qt.text, qs.statement_start_offset/2 + 1,
        (CASE qs.statement_end_offset WHEN -1 THEN LEN(qt.text)
         ELSE qs.statement_end_offset/2 END - qs.statement_start_offset/2) + 1) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
ORDER BY qs.total_worker_time DESC;

-- TOP QUERIES BY IO
SELECT TOP 20
    qs.total_logical_reads + qs.total_logical_writes AS total_io,
    qs.execution_count,
    qs.total_logical_reads/qs.execution_count AS avg_reads,
    SUBSTRING(qt.text, qs.statement_start_offset/2 + 1, 100) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
ORDER BY total_io DESC;

-- MISSING INDEXES (top suggestions)
SELECT TOP 20
    ROUND(gs.avg_total_user_cost * gs.avg_user_impact * (gs.user_seeks + gs.user_scans), 0) AS improvement_measure,
    db_name(d.database_id) AS db_name,
    d.statement AS table_name,
    d.equality_columns, d.inequality_columns, d.included_columns,
    gs.user_seeks, gs.user_scans
FROM sys.dm_db_missing_index_groups g
JOIN sys.dm_db_missing_index_group_stats gs ON g.index_group_handle = gs.group_handle
JOIN sys.dm_db_missing_index_details d ON g.index_handle = d.index_handle
ORDER BY improvement_measure DESC;

-- INDEX USAGE STATS (find unused indexes to drop)
SELECT OBJECT_NAME(i.object_id) AS table_name,
    i.name AS index_name, i.type_desc,
    us.user_seeks, us.user_scans, us.user_lookups, us.user_updates,
    us.last_user_seek, us.last_user_scan
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats us ON i.object_id = us.object_id AND i.index_id = us.index_id
WHERE OBJECTPROPERTY(i.object_id, 'IsMsShipped') = 0
    AND us.user_seeks = 0 AND us.user_scans = 0 AND us.user_lookups = 0
ORDER BY us.user_updates DESC;

-- WAIT STATISTICS (what is SQL Server waiting on?)
SELECT TOP 20
    wait_type,
    wait_time_ms / 1000.0 AS wait_time_sec,
    signal_wait_time_ms / 1000.0 AS signal_wait_sec,
    waiting_tasks_count,
    wait_time_ms * 100.0 / SUM(wait_time_ms) OVER() AS pct
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN ('SLEEP_TASK','BROKER_TO_FLUSH','SQLTRACE_BUFFER_FLUSH',
    'CLR_AUTO_EVENT','CLR_MANUAL_EVENT','LAZYWRITER_SLEEP','CHECKPOINT_QUEUE',
    'WAITFOR','XE_TIMER_EVENT','BROKER_EVENTHANDLER','FT_IFTS_SCHEDULER_IDLE_WAIT',
    'XE_DISPATCHER_WAIT','HADR_FILESTREAM_IOMGR_IOCOMPLETION','DIRTY_PAGE_POLL')
ORDER BY wait_time_ms DESC;

-- BLOCKING QUERIES (who is blocking whom?)
SELECT
    r.session_id AS blocked_session,
    r.blocking_session_id AS blocking_session,
    r.wait_type, r.wait_time,
    SUBSTRING(qt.text, r.statement_start_offset/2+1, 100) AS blocked_query,
    SUBSTRING(qt2.text, 1, 100) AS blocking_query
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) qt
OUTER APPLY sys.dm_exec_sql_text(
    (SELECT most_recent_sql_handle FROM sys.dm_exec_connections WHERE session_id = r.blocking_session_id)
) qt2
WHERE r.blocking_session_id > 0;

-- TEMPDB USAGE (who is consuming tempdb?)
SELECT
    t.session_id,
    t.internal_objects_alloc_page_count * 8 / 1024 AS internal_alloc_mb,
    t.user_objects_alloc_page_count * 8 / 1024 AS user_alloc_mb,
    s.login_name, s.program_name
FROM sys.dm_db_task_space_usage t
JOIN sys.dm_exec_sessions s ON t.session_id = s.session_id
WHERE t.internal_objects_alloc_page_count > 0
ORDER BY internal_alloc_mb DESC;
```

---

## B4. Query Store (SQL 2016+)

### Q30. Explain Query Store and how it helps with performance tuning.

**Answer:** Query Store captures query execution plans and runtime statistics INSIDE the database. It's like a "flight recorder" for queries.

```sql
-- Enable Query Store
ALTER DATABASE mydb SET QUERY_STORE = ON (
    OPERATION_MODE = READ_WRITE,
    DATA_FLUSH_INTERVAL_SECONDS = 900,
    INTERVAL_LENGTH_MINUTES = 60,
    MAX_STORAGE_SIZE_MB = 1024,
    QUERY_CAPTURE_MODE = AUTO,          -- AUTO captures significant queries only
    SIZE_BASED_CLEANUP_MODE = AUTO,
    WAIT_STATS_CAPTURE_MODE = ON        -- SQL 2017+
);

-- Find regressed queries (plan changed and got slower)
SELECT TOP 20
    q.query_id,
    qt.query_sql_text,
    rs.avg_duration / 1000.0 AS avg_duration_ms,
    rs.avg_cpu_time / 1000.0 AS avg_cpu_ms,
    rs.avg_logical_io_reads,
    rs.count_executions,
    p.plan_id
FROM sys.query_store_query q
JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
ORDER BY rs.avg_duration DESC;

-- Force a specific plan (pin the good plan)
EXEC sp_query_store_force_plan @query_id = 42, @plan_id = 3;

-- Unforce a plan
EXEC sp_query_store_unforce_plan @query_id = 42, @plan_id = 3;

-- Query Store Hints (SQL 2022)
EXEC sp_query_store_set_hints @query_id = 42, @query_hints = N'OPTION(RECOMPILE)';
EXEC sp_query_store_set_hints @query_id = 55, @query_hints = N'OPTION(MAXDOP 4)';
```

---

## B5. Intelligent Query Processing (SQL 2017-2022)

### Q31. Explain the Intelligent Query Processing features across versions.

```
  INTELLIGENT QUERY PROCESSING EVOLUTION:

  SQL 2017 — Adaptive Query Processing:
  ├── Batch Mode Adaptive Join
  │   └── At runtime, switches between Hash Join and Nested Loop
  │       based on actual row counts (not estimated)
  ├── Interleaved Execution for MSTVFs
  │   └── Pauses optimization to get actual row counts from
  │       Multi-Statement Table-Valued Functions
  └── Batch Mode Memory Grant Feedback
      └── Adjusts memory grants based on actual usage from
          previous executions (batch mode only)

  SQL 2019 — Intelligent Query Processing:
  ├── Table Variable Deferred Compilation
  │   └── Table variables now get accurate row counts
  │       (before 2019: always estimated as 1 row!)
  ├── Scalar UDF Inlining
  │   └── Inline scalar functions into the query plan
  │       (eliminates row-by-row execution!)
  ├── Batch Mode on Rowstore
  │   └── Batch mode execution even without columnstore index
  ├── Row Mode Memory Grant Feedback
  │   └── Memory grant feedback for row mode queries too
  ├── APPROX_COUNT_DISTINCT
  │   └── ~2% error but 4-10x faster than COUNT(DISTINCT)
  └── Optimized Plan Forcing
      └── Reduces overhead of forcing plans via Query Store

  SQL 2022 — Next Generation IQP:
  ├── Parameter Sensitive Plan Optimization (PSP)
  │   └── Multiple plans for different parameter ranges
  ├── Cardinality Estimation Feedback
  │   └── Auto-corrects bad cardinality estimates
  ├── DOP (Degree of Parallelism) Feedback
  │   └── Auto-adjusts parallelism per query
  ├── Memory Grant Feedback Percentile
  │   └── Uses percentile-based grants (not just last execution)
  ├── Query Store Hints
  │   └── Add hints without changing code
  └── Optimized Plan Forcing (enhanced)
```

---

## B6. Statistics & Cardinality Estimation

### Q32. Explain Statistics in SQL Server and how they affect performance.

**Answer:** Statistics are histograms that tell the query optimizer how data is distributed in columns. Bad statistics → bad row estimates → bad execution plans → poor performance.

```sql
-- View statistics for a table
DBCC SHOW_STATISTICS('orders', 'ix_customer_id');
-- Shows: Header (last updated, rows sampled), Density Vector, Histogram

-- Update statistics manually
UPDATE STATISTICS orders;                    -- all stats on table
UPDATE STATISTICS orders ix_customer_id;     -- specific index
UPDATE STATISTICS orders WITH FULLSCAN;      -- scan all rows (most accurate)
UPDATE STATISTICS orders WITH SAMPLE 50 PERCENT;  -- sample 50%

-- Auto-update statistics threshold:
-- Before SQL 2016: 20% + 500 rows must change before auto-update
-- SQL 2016+ (Trace Flag 2371 or database setting):
--   Threshold decreases as table grows (sqrt formula)
--   100K rows → ~0.3% change triggers update

-- Enable auto-update with lower threshold (SQL 2016+)
ALTER DATABASE SCOPED CONFIGURATION SET AUTO_CREATE_STATISTICS = ON;

-- Check when statistics were last updated
SELECT OBJECT_NAME(object_id) AS table_name,
    name AS stats_name,
    STATS_DATE(object_id, stats_id) AS last_updated,
    auto_created, user_created, no_recompute
FROM sys.stats WHERE object_id = OBJECT_ID('orders');

-- Async statistics update (reduces query delay)
ALTER DATABASE mydb SET AUTO_UPDATE_STATISTICS_ASYNC ON;
```

---

## B7. Tempdb Optimization

### Q33. How do you optimize Tempdb?

```sql
-- Best Practices:
-- 1. Multiple data files (1 per CPU core, up to 8)
--    SQL 2016+ installer offers this automatically
ALTER DATABASE tempdb ADD FILE (
    NAME = tempdev2, FILENAME = 'T:\tempdb\tempdev2.ndf', SIZE = 8GB, FILEGROWTH = 1GB);

-- 2. Pre-size tempdb files equally (avoid auto-growth)
ALTER DATABASE tempdb MODIFY FILE (NAME = tempdev, SIZE = 8GB, FILEGROWTH = 1GB);

-- 3. Place on fastest storage (SSD/NVMe)
-- 4. Trace Flag 1118 (SQL 2014 and earlier): uniform extent allocation
-- 5. SQL 2016+: Uniform extent allocation is DEFAULT (no TF needed)

-- 6. SQL 2019: Tempdb metadata optimization
ALTER SERVER CONFIGURATION SET MEMORY_OPTIMIZED TEMPDB_METADATA = ON;
-- Moves tempdb system tables to in-memory (eliminates page latch contention)

-- 7. SQL 2022: System page latch concurrency enhancements
-- Automatic optimization of GAM, SGAM, PFS pages

-- Monitor tempdb usage
SELECT SUM(unallocated_extent_page_count) * 8 / 1024 AS free_mb,
    SUM(internal_object_reserved_page_count) * 8 / 1024 AS internal_mb,
    SUM(user_object_reserved_page_count) * 8 / 1024 AS user_mb,
    SUM(version_store_reserved_page_count) * 8 / 1024 AS version_store_mb
FROM sys.dm_db_file_space_usage;
```

---

## B8. Wait Statistics & Troubleshooting

### Q34. What are the most common wait types and what do they mean?

| Wait Type | Category | Meaning | Solution |
|-----------|----------|---------|----------|
| **CXPACKET / CXCONSUMER** | Parallelism | Query waiting for parallel threads | Check MAXDOP, CTFP settings |
| **LCK_M_X / LCK_M_S** | Locking | Blocked by another session | Find blocking chain, optimize queries |
| **PAGEIOLATCH_SH** | Disk I/O | Waiting to read page from disk | Add memory, optimize queries, faster storage |
| **WRITELOG** | Transaction Log | Waiting for log write | Faster log disk, reduce log writes |
| **SOS_SCHEDULER_YIELD** | CPU | CPU pressure | Add CPUs, optimize queries |
| **ASYNC_NETWORK_IO** | Network | Client not consuming results fast | Check application, network issues |
| **RESOURCE_SEMAPHORE** | Memory | Waiting for query memory grant | Add RAM, fix memory-consuming queries |
| **LATCH_EX / LATCH_SH** | Internal | Internal resource contention | Tempdb optimization, reduce contention |
| **PAGELATCH_EX** | Buffer | Last-page insert contention | OPTIMIZE_FOR_SEQUENTIAL_KEY (SQL 2019+) |
| **HADR_SYNC_COMMIT** | AG | Waiting for AG synchronous commit | Network latency, async mode |

---

## B9. In-Memory OLTP (Hekaton)

### Q35. Explain In-Memory OLTP (SQL 2014+).

```sql
-- Create memory-optimized filegroup
ALTER DATABASE mydb ADD FILEGROUP mem_fg CONTAINS MEMORY_OPTIMIZED_DATA;
ALTER DATABASE mydb ADD FILE (
    NAME = mem_file, FILENAME = 'C:\Data\mem_oltp') TO FILEGROUP mem_fg;

-- Create memory-optimized table
CREATE TABLE orders_hot (
    order_id INT NOT NULL PRIMARY KEY NONCLUSTERED HASH WITH (BUCKET_COUNT = 1000000),
    customer_id INT NOT NULL INDEX ix_cust NONCLUSTERED HASH WITH (BUCKET_COUNT = 500000),
    order_date DATETIME2 NOT NULL,
    amount DECIMAL(10,2) NOT NULL
) WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_AND_DATA);

-- Natively compiled stored procedure (runs as compiled C code!)
CREATE PROCEDURE sp_InsertOrder
    @order_id INT, @customer_id INT, @amount DECIMAL(10,2)
WITH NATIVE_COMPILATION, SCHEMABINDING
AS
BEGIN ATOMIC WITH (TRANSACTION ISOLATION LEVEL = SNAPSHOT, LANGUAGE = N'English')
    INSERT INTO dbo.orders_hot (order_id, customer_id, order_date, amount)
    VALUES (@order_id, @customer_id, GETUTCDATE(), @amount);
END;
```

---

## B10. Accelerated Database Recovery (SQL 2019+)

### Q36. What is Accelerated Database Recovery (ADR)?

**Answer:** ADR redesigns the SQL Server recovery process to make it near-instantaneous, regardless of transaction size.

```sql
-- Enable ADR
ALTER DATABASE mydb SET ACCELERATED_DATABASE_RECOVERY = ON;

-- Benefits:
-- 1. Instant rollback: Rolling back a 10-hour transaction takes seconds (not hours)
-- 2. Fast recovery: Database comes online in seconds after crash (not minutes/hours)
-- 3. Aggressive log truncation: Transaction log doesn't grow as much

-- How it works:
-- Uses a Persistent Version Store (PVS) in the database itself
-- Instead of undoing changes from the log, it uses versions
```

---

## B11. MAXDOP & Cost Threshold for Parallelism

### Q37. What are the recommended MAXDOP and CTFP settings?

```sql
-- MAXDOP (Max Degree of Parallelism)
-- Controls how many CPU cores a single query can use in parallel

-- Microsoft recommendations:
-- Servers with ≤ 8 CPUs:  MAXDOP = number of CPUs
-- Servers with > 8 CPUs:  MAXDOP = 8
-- NUMA nodes:             MAXDOP = cores per NUMA node (max 8)

-- Check current setting
SELECT name, value FROM sys.configurations WHERE name = 'max degree of parallelism';

-- Set MAXDOP at server level
EXEC sp_configure 'max degree of parallelism', 4;
RECONFIGURE;

-- Set per database (SQL 2016+)
ALTER DATABASE SCOPED CONFIGURATION SET MAXDOP = 4;

-- Set per query
SELECT * FROM orders OPTION (MAXDOP 2);

-- Cost Threshold for Parallelism (CTFP)
-- Queries below this cost run serial; above run parallel
-- Default: 5 (too low! — causes excessive parallelism)
-- Recommended: 25-50 for OLTP, 10-25 for mixed workloads

EXEC sp_configure 'cost threshold for parallelism', 50;
RECONFIGURE;

-- SQL 2022: DOP Feedback automatically adjusts parallelism per query
ALTER DATABASE SCOPED CONFIGURATION SET DOP_FEEDBACK = ON;
```

---

## B12. Performance Tuning Checklist

### Q38. What is your approach to tuning a slow SQL Server?

**Answer — Systematic Performance Tuning Methodology:**

```
  STEP 1: IDENTIFY THE BOTTLENECK
  ├── Check Wait Statistics (sys.dm_os_wait_stats)
  │   ├── CPU waits → SOS_SCHEDULER_YIELD
  │   ├── I/O waits → PAGEIOLATCH_SH/EX
  │   ├── Memory waits → RESOURCE_SEMAPHORE
  │   ├── Lock waits → LCK_M_X, LCK_M_S
  │   └── Parallelism waits → CXPACKET
  │
  STEP 2: FIND THE BAD QUERIES
  ├── Query Store → Regressed Queries report
  ├── sys.dm_exec_query_stats → Top CPU/IO queries
  ├── sys.dm_exec_requests → Currently running long queries
  └── Extended Events → Capture queries > X seconds
  │
  STEP 3: ANALYZE EXECUTION PLANS
  ├── Check actual vs estimated row counts
  ├── Look for Table Scans, Key Lookups, Sorts
  ├── Check for implicit conversions
  ├── Check for parameter sniffing issues
  └── Check memory grant warnings (spills)
  │
  STEP 4: FIX THE ISSUES
  ├── Create/modify indexes (based on missing index DMV + plan analysis)
  ├── Update statistics (FULLSCAN on problem tables)
  ├── Rewrite queries (eliminate non-SARGable predicates)
  ├── Fix parameter sniffing (OPTIMIZE FOR, RECOMPILE, Query Store)
  ├── Add INCLUDE columns to eliminate Key Lookups
  ├── Set proper MAXDOP and CTFP
  └── Consider In-Memory OLTP for hot tables
  │
  STEP 5: INFRASTRUCTURE CHECKS
  ├── Memory: Is Buffer Pool Hit Ratio > 99%?
  ├── Disk: Is average disk latency < 10ms?
  ├── CPU: Is CPU utilization < 70% sustained?
  ├── Tempdb: Multiple files? Pre-sized?
  └── Transaction Log: On fast separate disk?
  │
  STEP 6: MONITOR AND ITERATE
  ├── Set up baseline metrics
  ├── Configure CloudWatch / Idera / SentryOne
  ├── Schedule regular index maintenance
  └── Review Query Store weekly
```

---

*T-SQL & Performance Tuning — In-Depth Interview Questions*
*SQL Server 2008 → 2012 → 2014 → 2016 → 2017 → 2019 → 2022 → 2025*
*150+ Questions with Detailed Answers, Code Examples & Diagrams*
