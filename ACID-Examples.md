
**Explanation of ACID properties in Microsoft SQL Server (MSSQL)** with **step-by-step examples and use cases**. 

Using **two sessions (sunil\_sessionwindow1 and sunil\_sessionwindow2)** to demonstrate **concurrency and transactions**, 


Let’s break it down carefully:

---

### 🔹 **ACID Properties in SQL Server with Examples**

ACID = **Atomicity, Consistency, Isolation, Durability**

---

#### 1. **Setup: Employee Table**

```sql
CREATE TABLE Employee (
    EmpID INT PRIMARY KEY IDENTITY(1,1),
    EmpName VARCHAR(50),
    Salary INT
);

INSERT INTO Employee (EmpName, Salary) VALUES
('Sriram', 10000),
('Devi', 12000),
('Jaanu', 18000);
```

Now we have 3 employees:

| EmpID | EmpName | Salary |
| ----- | ------- | ------ |
| 1     | Sriram  | 10000  |
| 2     | Devi    | 12000  |
| 3     | Jaanu   | 18000  |

---

#### 2. **Atomicity Example (All or Nothing)**

👉 **Definition:** A transaction is atomic → either all operations succeed or none.

**Case:** Increase Devi’s salary to 15,000 and Jaanu’s to 20,000 **together**. If one fails, rollback both.

#### sunil\_sessionwindow1

```sql
BEGIN TRANSACTION;

UPDATE Employee SET Salary = 15000 WHERE EmpName = 'Devi';
UPDATE Employee SET Salary = 20000 WHERE EmpName = 'Jaanu';

-- Simulate error
-- SELECT 1/0;  -- division by zero (forces error)

ROLLBACK; -- Undo if any step fails
```

✅ Result → Both salaries remain unchanged because transaction rolled back.

---

#### 3. **Consistency Example (Valid State Transition)**

👉 **Definition:** Data must move from one **valid state** to another. Constraints (PK, FK, CHECK) enforce this.

**Case:** Suppose salaries must be **greater than 5,000**.

```sql
ALTER TABLE Employee
ADD CONSTRAINT chk_salary CHECK (Salary > 5000);
```

#### sunil\_sessionwindow1

```sql
BEGIN TRANSACTION;
UPDATE Employee SET Salary = 4000 WHERE EmpName = 'Sriram';
COMMIT;
```

❌ This will **fail** because it violates consistency rule (`chk_salary`).
✅ Table remains unchanged → database consistent.

---

#### 4. **Isolation Example (Concurrent Transactions)**

👉 **Definition:** Multiple transactions should not interfere with each other. Levels = READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, SERIALIZABLE, SNAPSHOT.

### Case: Two sessions update salaries

#### sunil\_sessionwindow1 (T1)

```sql
BEGIN TRANSACTION;
UPDATE Employee SET Salary = 13000 WHERE EmpName = 'Sriram';
-- Do not commit yet
```

#### sunil\_sessionwindow2 (T2)

```sql
SELECT * FROM Employee WHERE EmpName = 'Sriram';
```

* If **READ COMMITTED (default):** T2 waits until T1 commits.
* If **READ UNCOMMITTED:** T2 sees **dirty read** (uncommitted 13000).
* If **REPEATABLE READ:** T2 won’t allow other updates until commit.
* If **SERIALIZABLE:** Strictest, prevents phantom reads.

#### sunil\_sessionwindow1 (commit)

```sql
COMMIT;
```

Now T2 will see the updated salary.

---

## 5. **Durability Example (Changes Survive Crash)**

👉 **Definition:** Once committed, data is permanent (even after crash/restart).

### sunil\_sessionwindow1

```sql
BEGIN TRANSACTION;
UPDATE Employee SET Salary = 16000 WHERE EmpName = 'Devi';
COMMIT;
```

✅ After commit → SQL Server writes to **transaction log**.
Even if server crashes → upon restart, log is replayed → Devi’s salary remains 16000.

---

# 🔹 **Use Case Demo with Names**

### Before Transaction

| EmpID | EmpName | Salary |
| ----- | ------- | ------ |
| 1     | Sriram  | 10000  |
| 2     | Devi    | 12000  |
| 3     | Jaanu   | 18000  |

### sunil\_sessionwindow1 (Atomic Transaction)

```sql
BEGIN TRANSACTION;
UPDATE Employee SET Salary = 11000 WHERE EmpName = 'Sriram';
UPDATE Employee SET Salary = 14000 WHERE EmpName = 'Devi';
UPDATE Employee SET Salary = 19000 WHERE EmpName = 'Jaanu';
COMMIT;
```

✅ After Commit

| EmpID | EmpName | Salary |
| ----- | ------- | ------ |
| 1     | Sriram  | 11000  |
| 2     | Devi    | 14000  |
| 3     | Jaanu   | 19000  |

---

### 🔹 **Summary**

* **Atomicity:** All-or-nothing (Rollback on error).
* **Consistency:** DB rules always maintained.
* **Isolation:** Concurrent users don’t corrupt data.
* **Durability:** Committed changes survive crash.

---
**Deep dive into REPEATABLE READ, SERIALIZABLE, and SNAPSHOT isolation levels** in SQL Server using our **Employee table** (`Sriram, Devi, Jaanu`).

We’ll again use **two sessions** (`sunil_sessionwindow1` and `sunil_sessionwindow2`) so you can run these step by step in SSMS.

---

### 🔹 Step 1: Verify Table Setup

Run once:

```sql
DROP TABLE IF EXISTS Employee;

CREATE TABLE Employee (
    EmpID INT PRIMARY KEY IDENTITY(1,1),
    EmpName VARCHAR(50),
    Salary INT
);

INSERT INTO Employee (EmpName, Salary) VALUES
('Sriram', 10000),
('Devi', 12000),
('Jaanu', 18000);
```

---

### 🔹 Step 2: REPEATABLE READ Demo

👉 **Guarantee:** If you read a row once, you’ll get the same value if you re-read it.
👉 **Limitation:** New rows (phantoms) can still appear.

---

### sunil\_sessionwindow1

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRANSACTION;

SELECT * FROM Employee WHERE EmpName = 'Devi';
-- Salary = 12000

WAITFOR DELAY '00:00:10';  -- pause to allow Session2
SELECT * FROM Employee WHERE EmpName = 'Devi';

COMMIT;
```

### sunil\_sessionwindow2 (during 10-sec pause)

```sql
UPDATE Employee SET Salary = 14000 WHERE EmpName = 'Devi';
COMMIT;
```

🔹 Result: Session2 will **block** until Session1 commits.
Session1’s two SELECTs always return **12000**.
✅ Prevents **Non-Repeatable Read**.
❌ But if Session2 inserts a **new row**, Session1 could still see it → Phantom Read allowed.

---

### 🔹 Step 3: SERIALIZABLE Demo

👉 **Guarantee:** Prevents **dirty, non-repeatable, and phantom reads**.
👉 **How:** Adds range locks, so no new rows can be inserted that affect your query.

---

### sunil\_sessionwindow1

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRANSACTION;

SELECT * FROM Employee WHERE Salary > 10000;
-- Expect Devi & Jaanu

WAITFOR DELAY '00:00:10'; -- pause
SELECT * FROM Employee WHERE Salary > 10000;

COMMIT;
```

### sunil\_sessionwindow2 (during 10-sec pause)

```sql
INSERT INTO Employee (EmpName, Salary) VALUES ('Anitha', 15000);
COMMIT;
```

🔹 Result: Session2 will be **blocked** because Session1’s range query locked rows with `Salary > 10000`.
Session1’s two SELECTs return the **same result set (Devi & Jaanu only)**.
✅ Prevents **Phantom Reads**.

---

### 🔹 Step 4: SNAPSHOT Isolation Demo

👉 **Guarantee:** Each transaction sees a **consistent snapshot** of data at the start of the transaction.
👉 **How:** Uses **row versioning** in `tempdb`.
👉 **Benefit:** Non-blocking reads (no waiting on locks).

---

### Enable Snapshot Isolation (run once in the database)

```sql
ALTER DATABASE YourDatabaseName SET ALLOW_SNAPSHOT_ISOLATION ON;
```

---

### sunil\_sessionwindow1

```sql
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
BEGIN TRANSACTION;

SELECT * FROM Employee WHERE EmpName = 'Sriram';
-- Suppose salary = 10000

WAITFOR DELAY '00:00:10'; -- pause
SELECT * FROM Employee WHERE EmpName = 'Sriram';

COMMIT;
```

### sunil\_sessionwindow2 (during 10-sec pause)

```sql
UPDATE Employee SET Salary = 16000 WHERE EmpName = 'Sriram';
COMMIT;
```

🔹 Result:

* Session1 sees **10000 both times** (snapshot at transaction start).
* Session2 commits **16000**, but Session1 doesn’t see it until after commit.
  ✅ Non-blocking, consistent view of data.
  ✅ Avoids dirty, non-repeatable, phantom reads.
  ✅ Great for reporting queries.

---

### 🔹 Comparison Summary

| Isolation Level  | Dirty Read | Non-Repeatable Read | Phantom Read | Blocking | Notes                              |
| ---------------- | ---------- | ------------------- | ------------ | -------- | ---------------------------------- |
| READ UNCOMMITTED | ✅ Allowed  | ✅ Allowed           | ✅ Allowed    | ❌ None   | Fast but unsafe                    |
| READ COMMITTED   | ❌ No       | ✅ Allowed           | ✅ Allowed    | ✅ Some   | Default in SQL Server              |
| REPEATABLE READ  | ❌ No       | ❌ No                | ✅ Allowed    | ✅ More   | Prevents row changes, not new rows |
| SERIALIZABLE     | ❌ No       | ❌ No                | ❌ No         | ✅ High   | Strictest, may reduce concurrency  |
| SNAPSHOT         | ❌ No       | ❌ No                | ❌ No         | ❌ None   | Uses row versioning, best balance  |

---

Got it 👍. Let’s walk through a **Phantom Read** example in SQL Server with a clear step-by-step demo.

---

### 🔹 What is Phantom Read?

* A **phantom read** happens when one transaction reads a set of rows using a condition (e.g., `WHERE salary > 10000`)
* Later, another transaction **inserts or deletes rows** that satisfy the same condition.
* If the first transaction re-executes the same query, it sees **new “phantom” rows** that were not there before.

---

### 📌 Example Setup

```sql
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

### 🔹 Transaction Simulation

#### 🖥️ Session 1 (T1)

```sql
-- Transaction 1 starts
BEGIN TRANSACTION;

-- Read employees with salary > 10000
SELECT * FROM Employee WHERE Salary > 10000;
-- Returns: Devi (12000), Jaanu (18000)
```

#### 🖥️ Session 2 (T2)

```sql
-- Transaction 2 starts
BEGIN TRANSACTION;

-- Insert a new employee who satisfies the condition
INSERT INTO Employee VALUES (4, 'Kiran', 15000);

COMMIT;  -- commit new row
```

#### 🖥️ Session 1 (T1) Again

```sql
-- Transaction 1 re-reads
SELECT * FROM Employee WHERE Salary > 10000;
-- Returns: Devi (12000), Jaanu (18000), Kiran (15000) 👈 Phantom row appears
```

---

### 🔹 Why this happens?

Because the default isolation level in SQL Server is **READ COMMITTED**,

* It prevents **dirty reads** (can’t see uncommitted data)
* But it allows **phantoms** since inserts/deletes can sneak in.

---

### 🔹 How to Prevent Phantom Reads?

* Use **SERIALIZABLE** isolation level:

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRANSACTION;
SELECT * FROM Employee WHERE Salary > 10000;
```

This places a **range lock**, blocking inserts into the qualifying range until T1 finishes.

---
 **Phantom Read example** again but this time with **`sunil_sessionwindow1`** and **`sunil_sessionwindow2`** 
 
 so you can simulate it in SQL Server Management Studio (SSMS) by opening two query windows.

---

#### 🛠️ Step 1: Table Setup

Run this only once:

```sql
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

#### 🖥️ Session 1 → `sunil_sessionwindow1`

```sql
-- Use default READ COMMITTED isolation level (phantoms allowed)
-- Transaction starts
BEGIN TRANSACTION;

-- First read
SELECT * FROM Employee WHERE Salary > 10000;
-- Expected: Devi (12000), Jaanu (18000)

-- Keep transaction open, don’t commit yet
```

---

#### 🖥️ Session 2 → `sunil_sessionwindow2`

```sql
BEGIN TRANSACTION;

-- Insert a new employee who qualifies for the same condition
INSERT INTO Employee VALUES (4, 'Kiran', 15000);

COMMIT;  -- commit immediately so session1 can see it
```

---

#### 🖥️ Back to Session 1 → `sunil_sessionwindow1`

```sql
-- Re-run the same query
SELECT * FROM Employee WHERE Salary > 10000;

-- Now result includes:
-- Devi (12000), Jaanu (18000), and 👻 Phantom row Kiran (15000)
```

---

#### 🔒 Prevent Phantom Read (Serializable)

If you want to block `sunil_sessionwindow2` from inserting until `sunil_sessionwindow1` finishes:

### Session 1 (`sunil_sessionwindow1`)

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRANSACTION;

SELECT * FROM Employee WHERE Salary > 10000;
-- Locks the range so no phantom inserts allowed
```

### Session 2 (`sunil_sessionwindow2`)

```sql
BEGIN TRANSACTION;
INSERT INTO Employee VALUES (5, 'Ravi', 16000);
-- This will block until Session 1 commits/rollbacks
```

---

✅ That’s a **true Phantom Read** demo with two windows (`sunil_sessionwindow1`, `sunil_sessionwindow2`).




