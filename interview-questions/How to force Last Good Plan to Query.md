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

Perfect 👍 — let’s now go into **Automatic Plan Correction (APC)** in SQL Server, which extends the idea of “forcing the last good plan” but does it **automatically** for you.

---

### 🔎 What is Automatic Plan Correction?

* SQL Server (2017+) can **detect query plan regressions**.
* If a query suddenly runs slower because of a new execution plan, SQL can **automatically force the last known good plan**.
* This is part of the **Automatic Tuning** feature.

👉 In simple English: SQL watches your queries, and if one slows down because of a bad plan, it says:

> “Hey, I’ll just reuse the plan that worked fine before.”

---

### 🛠 Step 1: Enable Query Store

Query Store must be ON (it stores history of plans).

```sql
ALTER DATABASE SalesDB 
SET QUERY_STORE = ON;
```

---

### 🛠 Step 2: Enable Automatic Tuning (Plan Correction)

At database level (SQL 2017+):

```sql
ALTER DATABASE SalesDB 
SET AUTOMATIC_TUNING ( FORCE_LAST_GOOD_PLAN = ON );
```

Check current state:

```sql
SELECT name, desired_state_desc, actual_state_desc
FROM sys.database_automatic_tuning_options;
```

---

### 🛠 Step 3: Use Case Example

👉 **Scenario:**

* You have a query:

  ```sql
  SELECT * FROM Orders WHERE CustomerID = @CustID;
  ```
* For most customers, SQL uses an **index seek** (fast = 50 ms).
* Suddenly, for `@CustID = 99999`, SQL chooses a **table scan** (slow = 10 seconds).

👉 **Without APC:**

* SQL would keep using the bad plan until DBA intervenes.

👉 **With APC (Automatic Plan Correction):**

1. SQL notices that the **average duration** of the query has increased compared to history.
2. SQL decides that the new plan is worse than the old one.
3. SQL **automatically reverts to the last good plan** (index seek).
4. In Query Store views, you’ll see the plan marked as “forced by automatic tuning.”

✅ **Result**: Query performance returns to normal (50 ms) without DBA action.

---

### 🛠 Step 4: How to See APC in Action

SQL logs plan corrections into DMV:

```sql
SELECT reason, state, details, create_time, last_execute_time
FROM sys.dm_db_tuning_recommendations;
```

You’ll see entries like:

* **Reason**: Plan regression detected
* **State**: Accepted
* **Details**: Forced plan ID = 10 (previous good plan)

---

### 📚 Use Case in Production

👉 **Example: Reporting Server**

* Users run monthly reports.
* Normally queries run in 1–2 minutes.
* This month, one report takes 30 minutes because SQL chose a bad plan.

👉 **With Automatic Plan Correction:**

* SQL auto-detects regression.
* SQL forces back the last good plan.
* Report runs in 2 minutes again — before users even complain.

---

### ✅ Summary

* **Manual Forcing** = You, the DBA, look at Query Store and force the good plan.
* **Automatic Plan Correction** = SQL detects slowdowns and **fixes it for you automatically**.
* **Best Use Cases**:

  * OLTP systems with recurring queries that must be stable.
  * Reporting systems where performance regressions are unacceptable.
  * Environments with limited DBA monitoring (SQL self-heals).

---


