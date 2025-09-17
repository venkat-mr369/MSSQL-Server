
### 🔑 Locks, Blocking & DeadLock

---

#### 🔑 Core Concepts

### 1. **Lock**

* A **lock** is a mechanism SQL Server uses to maintain **data consistency** and **isolation**.
* When a transaction reads/writes data, SQL Server applies locks on rows, pages, or tables.
* Locks prevent conflicting operations from happening simultaneously.

---

### 2. **Blocking**

* **Blocking** happens when one transaction holds a lock on a resource and another transaction has to **wait** for that lock to be released.
* It is **not an error** — just a delay until the blocking transaction commits/rolls back.

---

### 3. **Deadlock**

* **Deadlock** occurs when **two or more transactions are waiting for each other’s locks**, creating a circular dependency.
* SQL Server detects it and **kills one transaction** (the victim).

---

### 🔒 Types of Locks 

| Lock Type                       | Purpose                                                                                        | Example Use Case                                                      |
| ------------------------------- | ---------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| **Shared (S)**                  | Allows read-only operations. Multiple shared locks can coexist.                                | `SELECT * FROM Employee`                                              |
| **Exclusive (X)**               | For writes (INSERT/UPDATE/DELETE). No other locks can access the resource.                     | `UPDATE Employee SET Salary=20000 WHERE EmpID=1`                      |
| **Update (U)**                  | Prevents deadlock during updates (first step before converting to exclusive).                  | `UPDATE Employee SET Salary=...`                                      |
| **Intent (IS, IX, IU)**         | Lock hierarchy indicator at table level before acquiring row/page locks.                       | If row is locked exclusively, SQL adds an **IX lock** at table level. |
| **Schema Locks (Sch-S, Sch-M)** | Prevent schema changes while query runs (Sch-S) or block queries during schema change (Sch-M). | `ALTER TABLE Employee ADD DOB DATE`                                   |

---

### ⚡ Examples with Your `Employee` Table

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

### ✅ Example 1: **Shared Lock (S)**

Transaction 1:

```sql
BEGIN TRAN
SELECT * FROM Employee WHERE EmpID = 1; -- Shared Lock on row
-- not committed yet
```

Transaction 2 (runs in another session):

```sql
SELECT * FROM Employee WHERE EmpID = 1; -- Allowed, shared lock compatible
```

👉 Multiple shared locks can coexist.

---

### ✅ Example 2: **Exclusive Lock (X) & Blocking**

Transaction 1:

```sql
BEGIN TRAN
UPDATE Employee SET Salary = 20000 WHERE EmpID = 1; -- Exclusive Lock
-- not committed yet
```

Transaction 2 (runs in another session):

```sql
SELECT * FROM Employee WHERE EmpID = 1; -- BLOCKED until T1 commits/rolls back
```

👉 This is **blocking**.

---

### ✅ Example 3: **Update Lock (U)**

Transaction 1:

```sql
BEGIN TRAN
UPDATE Employee SET Salary = Salary + 1000 WHERE EmpID = 2;
-- SQL Server first places an Update Lock (U), then converts to Exclusive (X).
```

👉 Prevents two transactions from both holding shared locks and later converting them to exclusive (deadlock prevention).

---

### ✅ Example 4: **Intent Locks**

When you update a row:

```sql
BEGIN TRAN
UPDATE Employee SET Salary = 25000 WHERE EmpID = 3;
```

* Row level: **X Lock** on EmpID=3.
* Table level: **IX Lock** (intent exclusive).
  👉 Intent locks help SQL Server manage hierarchical locking efficiently.

---

### ✅ Example 5: **Deadlock**

Transaction 1:

```sql
BEGIN TRAN
UPDATE Employee SET Salary = 30000 WHERE EmpID = 1; -- Locks row 1
-- waiting for row 2
UPDATE Employee SET Salary = 35000 WHERE EmpID = 2;
```

Transaction 2 (simultaneously):

```sql
BEGIN TRAN
UPDATE Employee SET Salary = 40000 WHERE EmpID = 2; -- Locks row 2
-- waiting for row 1
UPDATE Employee SET Salary = 45000 WHERE EmpID = 1;
```

👉 **Deadlock occurs** (T1 waits for row 2, T2 waits for row 1).
SQL Server chooses one transaction as a **victim**.

---

### 📊 Summary of Differences

| Concept      | Definition                             | Outcome                                   |
| ------------ | -------------------------------------- | ----------------------------------------- |
| **Lock**     | Mechanism to control concurrent access | Ensures ACID consistency                  |
| **Blocking** | One session waits for another’s lock   | Delay, but resolves after commit/rollback |
| **Deadlock** | Circular waiting between transactions  | SQL Server kills one transaction          |

---
