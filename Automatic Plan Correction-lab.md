**step-by-step demo script** for **Automatic Plan Correction (APC)** in SQL Server.

This demo will show:

1. A query that gets a **good plan**.
2. We’ll force SQL to pick a **bad plan** (parameter sniffing).
3. SQL Server will detect the **regression** and **force the last good plan automatically**.

---

# 🛠 Step 1: Prepare Demo Database

```sql
USE master;
GO
IF DB_ID('APC_Demo') IS NOT NULL
    DROP DATABASE APC_Demo;
GO
CREATE DATABASE APC_Demo;
GO
USE APC_Demo;
GO
```

---

# 🛠 Step 2: Create Table & Load Data

We’ll create a skewed table where some values are rare, others common.

```sql
CREATE TABLE Orders
(
    OrderID INT IDENTITY PRIMARY KEY,
    CustomerID INT,
    OrderDate DATETIME DEFAULT GETDATE(),
    Amount DECIMAL(10,2)
);

-- Insert 1 million rows for CustomerID = 1 (common)
INSERT INTO Orders (CustomerID, Amount)
SELECT TOP (1000000) 1, RAND()*100
FROM sys.all_objects a CROSS JOIN sys.all_objects b;

-- Insert 100 rows for CustomerID = 99999 (rare)
INSERT INTO Orders (CustomerID, Amount)
SELECT TOP (100) 99999, RAND()*100
FROM sys.all_objects;
GO

-- Create index on CustomerID
CREATE INDEX IX_Orders_CustomerID ON Orders(CustomerID);
GO
```

---

# 🛠 Step 3: Enable Query Store & Automatic Plan Correction

```sql
ALTER DATABASE APC_Demo SET QUERY_STORE = ON;
ALTER DATABASE APC_Demo 
SET AUTOMATIC_TUNING ( FORCE_LAST_GOOD_PLAN = ON );
```

---

# 🛠 Step 4: Run Query with Parameter Sniffing

```sql
-- This query parameterized will sniff first execution
DECLARE @CustID INT = 1;  -- common
SELECT * FROM Orders WHERE CustomerID = @CustID;
GO

DECLARE @CustID INT = 99999;  -- rare
SELECT * FROM Orders WHERE CustomerID = @CustID;
GO
```

👉 What happens:

* First execution with `@CustID = 1` → SQL chooses **Index Seek** (good plan).
* Second execution with `@CustID = 99999` → SQL may reuse the seek plan, but full scan might actually be faster → this mismatch causes regression.

---

# 🛠 Step 5: Force Plan Regression

Run repeatedly with bad parameters so SQL considers new plan “better”:

```sql
DECLARE @i INT = 0;
WHILE @i < 50
BEGIN
   DECLARE @CustID INT = 99999;
   SELECT * FROM Orders WHERE CustomerID = @CustID;
   SET @i = @i + 1;
END
```

---

# 🛠 Step 6: Watch Automatic Plan Correction Kick In

Check tuning recommendations:

```sql
SELECT reason, state, details, create_time, last_execute_time
FROM sys.dm_db_tuning_recommendations;
```

👉 You should see something like:

* **Reason**: Plan regression detected
* **State**: Accepted
* **Details**: Plan 2 regressed, forcing Plan 1 (last good plan)

---

# 🛠 Step 7: Verify Query is Using Forced Plan

Run again:

```sql
DECLARE @CustID INT = 99999;
SELECT * FROM Orders WHERE CustomerID = @CustID;

SELECT qsqt.query_sql_text, qsp.plan_id, qsp.is_forced_plan
FROM sys.query_store_query_text qsqt
JOIN sys.query_store_query qsq ON qsqt.query_text_id = qsq.query_text_id
JOIN sys.query_store_plan qsp ON qsq.query_id = qsp.query_id
WHERE qsqt.query_sql_text LIKE '%Orders WHERE CustomerID%';
```

👉 You’ll see:

* `is_forced_plan = 1` → SQL forced the last good plan.

---

# 📚 Use Case in Real Life

* **Scenario**: A monthly report suddenly runs 30 minutes instead of 2 minutes because SQL sniffed bad parameter and picked wrong plan.
* **Without APC**: DBA wakes up at 3 AM, checks Query Store, forces plan manually.
* **With APC**: SQL auto-detects regression, rolls back to last good plan within minutes, report completes in 2 minutes, no DBA intervention needed.

---

# ✅ Summary

* **Query Store** must be ON.
* **Automatic Plan Correction** must be enabled.
* SQL Server continuously monitors queries.
* If a new plan regresses, SQL **forces the last good plan** automatically.

---

