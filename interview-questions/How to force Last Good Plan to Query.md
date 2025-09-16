**forcing SQL Server to use the Last Good Plan** for a query. 

This is a **Query Store feature** in introduced in SQL Server 2016 that helps stabilize query performance when SQL Server suddenly generates a bad execution plan.

---

### 🔎 What is “Last Good Plan”?

* SQL Server chooses an **execution plan** for every query.
* Sometimes it picks a **bad plan** (parameter sniffing, wrong statistics, skewed data).
* The query suddenly runs slow, even though it was fast before.
* **Last Good Plan** lets you **force SQL Server to reuse the previously good plan** instead of the new bad one.

---

#### 🛠 How to Enable Query Store (needed first)

```sql
ALTER DATABASE SalesDB 
SET QUERY_STORE = ON;
```

Check it:

```sql
SELECT actual_state_desc, desired_state_desc
FROM sys.database_query_store_options;
```

---

### 🛠 How to Force Last Good Plan

1. Identify the query that has regression:

   ```sql
   SELECT TOP 10 
       qsq.query_id, 
       qsp.plan_id, 
       qsq.object_id, 
       qsq.query_sql_text,
       rs.avg_duration, 
       rs.count_executions
   FROM sys.query_store_query qsq
   JOIN sys.query_store_plan qsp ON qsq.query_id = qsp.query_id
   JOIN sys.query_store_runtime_stats rs ON qsp.plan_id = rs.plan_id
   WHERE qsq.query_sql_text LIKE '%SELECT * FROM Orders%';
   ```

2. Suppose you see **Plan 10** was good, **Plan 12** is bad.
   Force SQL to always use **Plan 10**:

   ```sql
   EXEC sp_query_store_force_plan @query_id = 101, @plan_id = 10;
   ```

3. Check forced plans:

   ```sql
   SELECT * 
   FROM sys.query_store_plan 
   WHERE is_forced_plan = 1;
   ```

4. To unforce:

   ```sql
   EXEC sp_query_store_unforce_plan @query_id = 101, @plan_id = 10;
   ```

---

### 📚 Use Case Example 

👉 **Scenario**:

* You have a query:

  ```sql
  SELECT * FROM Orders WHERE CustomerID = @CustID;
  ```
* Normally runs in **50 ms**.
* One day, a query with `@CustID = 99999` makes SQL choose a **bad plan** (full table scan).
* Now the query takes **10 seconds** for all customers.

👉 **What you do**:

* Query Store shows that **Plan A** was always fast (index seek).
* SQL picked **Plan B** (table scan) yesterday.
* You force SQL to always use **Plan A (last good plan)**.

👉 **Result**:

* Query is back to running in **50 ms** for all customers.
* Users are happy again.

---

### ✅ Summary 

* SQL Server sometimes picks a **bad plan**.
* Query Store remembers the **old good plans**.
* You can **force SQL** to keep using the **last good one**.
* This is helpful when query performance suddenly changes.

---

⚡ Proactive Tip:
SQL Server 2022 introduced **"Query Store Hints"** and **"Automatic Plan Correction"** — SQL can automatically force last good plans when regression is detected.

---
