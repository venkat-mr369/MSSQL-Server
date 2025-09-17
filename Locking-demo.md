**Step-by-Step blocking & Deadlock demo** with your `Employee` table. I’ll show you:

1. **Setup**
2. **Blocking demo** (using two SSMS query windows)
3. **Deadlock demo**
4. **Monitoring locks** (using system DMVs)

---

# 1️⃣ Setup

```sql
USE tempdb;
GO

IF OBJECT_ID('Employee') IS NOT NULL DROP TABLE Employee;
GO

CREATE TABLE Employee (
    EmpID INT PRIMARY KEY,
    EmpName VARCHAR(50),
    Salary INT
);

INSERT INTO Employee VALUES
(1, 'Sriram', 10000),
(2, 'Devi', 12000),
(3, 'Jaanu', 18000);
```

---

# 2️⃣ Blocking Example

⚡ Open **two query windows** in SSMS.

### **Window 1 (Transaction 1)**

```sql
BEGIN TRAN
UPDATE Employee SET Salary = 20000 WHERE EmpID = 1; -- Holds Exclusive Lock (X)
-- Do not commit yet
```

### **Window 2 (Transaction 2)**

```sql
SELECT * FROM Employee WHERE EmpID = 1;
```

👉 Window 2 will **hang (blocked)** until Window 1 commits/rolls back.

### To resolve:

```sql
-- In Window 1
COMMIT;
```

✅ Window 2 query completes immediately.

---

# 3️⃣ Deadlock Example

⚡ Again use **two query windows**.

### **Window 1 (Transaction 1)**

```sql
BEGIN TRAN
UPDATE Employee SET Salary = 30000 WHERE EmpID = 1;  -- Locks row 1
-- Now wait a few seconds, then run:
UPDATE Employee SET Salary = 35000 WHERE EmpID = 2;  -- Wants row 2
```

### **Window 2 (Transaction 2)**

```sql
BEGIN TRAN
UPDATE Employee SET Salary = 40000 WHERE EmpID = 2;  -- Locks row 2
-- Now wait a few seconds, then run:
UPDATE Employee SET Salary = 45000 WHERE EmpID = 1;  -- Wants row 1
```

👉 Both transactions are waiting for each other.
SQL Server detects **deadlock** and kills one transaction with an error like:

```
Msg 1205, Level 13, State 51, Line 1
Transaction (Process ID 52) was deadlocked on resources...
```

---

# 4️⃣ Monitoring Locks & Blocking

You can see what’s happening under the hood.

```sql
-- Shows current locks
SELECT 
    request_session_id,
    resource_type,
    resource_description,
    request_mode,
    request_status
FROM sys.dm_tran_locks;
```

```sql
-- Shows blocking sessions
SELECT 
    blocking_session_id,
    session_id,
    wait_type,
    wait_time,
    wait_resource
FROM sys.dm_exec_requests
WHERE blocking_session_id <> 0;
```

```sql
-- See active transactions
DBCC OPENTRAN;
```

---

# 🔎 Expected Observations

* **Blocking** → Second session waits until first session commits.
* **Deadlock** → One transaction gets killed, the other continues.
* **Locks DMV** shows Shared (S), Exclusive (X), Update (U), Intent (IX/IS).

---
