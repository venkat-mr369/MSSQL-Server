**Classic SQL Server performance pain point**: **slowness with Linked Server queries / OPENQUERY**.
Let’s break it down **deeply** — root causes, why it happens, and **ways to overcome it**.

---

## 🔹 1. Why Linked Server Queries Can Be Slow

When you query via a linked server (`SELECT … FROM [LinkedServer].DB.Schema.Table` or `OPENQUERY`), SQL Server has to coordinate **two engines**:

* **Local SQL Server** (where query is executed).
* **Remote SQL Server / Oracle / MySQL / other OLE DB provider** (the data source).

Performance issues arise because:

### **A. Data Movement / Network Bottlenecks**

* If you run:

  ```sql
  SELECT * FROM LinkedServer.DB.Schema.Table
  ```

  → SQL Server pulls **all rows across the network** before filtering/joining.
* Big network transfer = slow.

---

### **B. Poor Query Plan Delegation**

* SQL Server tries to **decide which part of the query to push down** to the remote server.
* Sometimes, it pushes only part (or nothing), leading to:

  * Large dataset pulled locally.
  * Filtering/joining done **on local server**, not remotely.

---

### **C. OLE DB Provider Settings**

* Default provider settings may prevent efficient query delegation:

  * `Enable Promotion of Distributed Transactions` → triggers MSDTC overhead.
  * `RPC` / `RPC OUT` disabled → prevents remote procedure execution.
  * `Collation Compatibility` mismatch → forces local sorting/conversions.

---

### **D. OpenQuery vs 4-Part Naming**

* `OPENQUERY` executes **entire query remotely** and returns only result set.
* `LinkedServer.Table` syntax may pull **raw data**, then filter locally.
* Example:

  ```sql
  -- This fetches all rows to local, then filters
  SELECT * FROM LinkedServer.DB.dbo.BigTable WHERE Col = 'X';

  -- This pushes filter to remote
  SELECT * FROM OPENQUERY(LinkedServer, 'SELECT * FROM BigTable WHERE Col = ''X''');
  ```

---

### **E. Distributed Transaction Overhead**

* If you update/join remote + local DBs, SQL Server uses **MSDTC** → heavy, slow.

---

### **F. Statistics / Cardinality Mismatch**

* Local SQL Server has **no statistics** about remote tables.
* Optimizer guesses row counts → wrong query plan → poor performance.

---

## 🔹 2. How to Overcome Linked Server Slowness

### ✅ **A. Push Work to Remote (Reduce Data Movement)**

* Use `OPENQUERY` (preferred) to ensure filtering/aggregation happens remotely:

  ```sql
  SELECT * 
  FROM OPENQUERY(LinkedServer, 'SELECT Col1, Col2 FROM BigTable WHERE Col2 = ''X''');
  ```

* Or wrap logic in **stored procedure on remote server** and call via:

  ```sql
  EXEC LinkedServer.DB.dbo.RemoteProc @param = 'X';
  ```

---

### ✅ **B. Optimize Provider Settings**

Run:

```sql
EXEC sp_serveroption 'LinkedServer', 'rpc out', 'true';
EXEC sp_serveroption 'LinkedServer', 'rpc', 'true';
EXEC sp_serveroption 'LinkedServer', 'collation compatible', 'true';
EXEC sp_serveroption 'LinkedServer', 'use remote collation', 'true';
```

* `rpc/rpc out` = allows execution of remote procs.
* `collation compatible` = avoids unnecessary conversions.
* Check `Enable Promotion of Distributed Transactions` (set off if not updating both servers).

---

### ✅ **C. Limit Columns & Rows**

* Avoid `SELECT *`.
* Pull only required columns.
* Apply WHERE, JOIN, GROUP BY on **remote side**.

---

### ✅ **D. Temp Tables for Large Joins**

Instead of joining directly across linked servers:

```sql
-- Bad
SELECT a.*, b.* 
FROM LocalTable a
JOIN LinkedServer.DB.dbo.BigTable b ON a.ID = b.ID;
```

Do this:

```sql
SELECT * 
INTO #RemoteData
FROM OPENQUERY(LinkedServer, 'SELECT ID, ColX FROM BigTable WHERE SomeFilter = ''Y''');

SELECT a.*, r.ColX
FROM LocalTable a
JOIN #RemoteData r ON a.ID = r.ID;
```

👉 Reduces network cost & optimizer confusion.

---

### ✅ **E. Consider Replication / ETL for Heavy Loads**

* If queries are frequent/heavy → better to replicate/sync remote data locally:

  * **Transactional Replication**
  * **Change Data Capture (CDC)**
  * **ETL (SSIS, ADF, etc.)**

This avoids hitting linked server in real-time.

---

### ✅ **F. Indexing on Remote Side**

* Ensure remote tables are indexed properly (on join/filter columns).
* Since query executes remotely in `OPENQUERY`, good indexing helps.

---

### ✅ **G. Monitor & Troubleshoot**

* Use `SET STATISTICS IO, TIME ON` to measure.
* Use **Profiler/Extended Events** on remote server to see what query actually runs.
* Sometimes SQL rewrites the query in unexpected ways → check actual remote SQL.

---

## 🔹 3. Example – Bad vs Good

❌ **Bad (slow):**

```sql
SELECT a.OrderID, b.CustomerName
FROM Orders a
JOIN LinkedServer.Sales.dbo.Customers b
   ON a.CustomerID = b.CustomerID
WHERE b.City = 'New York';
```

* Pulls entire Customers table to local → filters locally.

✅ **Good (faster):**

```sql
SELECT a.OrderID, b.CustomerName
FROM Orders a
JOIN OPENQUERY(LinkedServer,
    'SELECT CustomerID, CustomerName FROM Customers WHERE City = ''New York''') b
   ON a.CustomerID = b.CustomerID;
```

* Remote server filters `City = 'New York'` → smaller dataset pulled.

---

## 🔹 4. Golden Rules for Linked Server Performance

1. **Push down filters/joins** (use OPENQUERY or remote procs).
2. \*\*Avoid SELECT \*\*\* → fetch only needed cols.
3. **Tune OLE DB provider options**.
4. **Use temp tables** for intermediate joins.
5. **Use replication/ETL** if workload is high-frequency.
6. **Monitor actual query executed remotely** (Profiler/Extended Events).

---

**step-by-step troubleshooting checklist** **Linked Server / OPENQUERY slowness**.

---

## 🟢 **Linked Server Slowness Troubleshooting Checklist**

---

## 🔹 Step 1: Identify Query Pattern

* **Check if query uses 4-part naming** (`LinkedServer.DB.Schema.Table`) OR `OPENQUERY`.

  * If **4-part name** → SQL Server may **pull all rows** locally → slow.
  * If `OPENQUERY` → filtering happens remotely → usually better.

✅ **Action:** If using 4-part naming → rewrite into `OPENQUERY`.

---

## 🔹 Step 2: Check What Query Actually Runs on Remote

* Run the local query with **Profiler/Extended Events** enabled on **remote server**.
* Confirm whether filtering/join is happening **remotely** or **locally**.

✅ **Action:**

* If remote server is returning too many rows → **rewrite query to push filtering down**.
* Example:

  ```sql
  -- BAD: filter done locally
  SELECT * FROM LinkedServer.DB.dbo.BigTable WHERE Col = 'X';

  -- GOOD: filter done remotely
  SELECT * FROM OPENQUERY(LinkedServer, 'SELECT * FROM BigTable WHERE Col = ''X''');
  ```

---

## 🔹 Step 3: Measure Data Movement

* Use:

  ```sql
  SET STATISTICS IO, TIME ON;
  ```
* Check how many rows/bytes are being pulled across the network.

✅ **Action:**

* If huge dataset → reduce columns, push filters.
* Avoid `SELECT *`.

---

## 🔹 Step 4: Check Provider Settings

Run:

```sql
EXEC sp_serveroption 'LinkedServer', 'rpc out', 'true';
EXEC sp_serveroption 'LinkedServer', 'rpc', 'true';
EXEC sp_serveroption 'LinkedServer', 'collation compatible', 'true';
EXEC sp_serveroption 'LinkedServer', 'use remote collation', 'true';
```

✅ **Action:**

* Enable `RPC OUT` → allows calling remote stored procedures.
* Enable `Collation Compatible` → avoids local sorting/conversions.
* Disable `Enable Promotion of Distributed Transactions` if you don’t need updates across both servers.

---

## 🔹 Step 5: Check for Distributed Transactions

* If query **updates/inserts** across linked servers → SQL Server promotes to **MSDTC transaction** (very slow).

✅ **Action:**

* Avoid cross-server transactions.
* Instead:

  * Extract remote data into a temp table.
  * Do updates locally.

---

## 🔹 Step 6: Check Statistics & Execution Plan

* Local SQL Server has **no statistics** for remote tables → optimizer guesses row counts.
* This may cause **bad plans** (nested loops instead of hash joins, etc.).

✅ **Action:**

* Break query into steps (use temp tables).
* Example:

  ```sql
  SELECT * 
  INTO #RemoteData
  FROM OPENQUERY(LinkedServer, 'SELECT ID, ColX FROM BigTable WHERE Filter = ''Y''');

  SELECT a.*, r.ColX
  FROM LocalTable a
  JOIN #RemoteData r ON a.ID = r.ID;
  ```

---

## 🔹 Step 7: Check Remote Server Health

* Is the remote SQL Server/Oracle/MySQL under CPU or IO pressure?
* Is there network latency between servers?

✅ **Action:**

* Run same query **directly on remote server** → if slow there too → fix indexes, tune query remotely.

---

## 🔹 Step 8: Consider Alternatives (If Query is Frequent/Heavy)

* If query is **one-time** → tuning is fine.
* If query is **frequent/heavy**:

  * Use **replication** (keep remote data locally).
  * Use **ETL (SSIS, ADF, CDC)** to sync data.
  * Use **linked server only for light lookups**, not heavy joins.

---

# 🔹 Quick Diagnostic Flow

1. **Is it 4-part naming or OPENQUERY?**

   * Use `OPENQUERY` → pushes filter to remote.
2. **What query runs remotely?**

   * Capture in Profiler → check row count.
3. **How much data is moving?**

   * Use `SET STATISTICS IO`.
4. **Provider settings tuned?**

   * Enable `rpc`, `rpc out`, `collation compatible`.
5. **Distributed transaction?**

   * Avoid MSDTC unless required.
6. **Bad query plan?**

   * Use temp tables to force optimizer.
7. **Remote server health OK?**

   * If slow remotely too → tune indexes.
8. **Is workload frequent?**

   * If yes → replicate/ETL instead of linked server.

---

✅ **Summary:**

* **Root causes**: excessive data movement, bad delegation, MSDTC, wrong provider settings, or remote server slowness.
* **Fixes**: use `OPENQUERY` / remote procs, limit columns/rows, tune provider options, use temp tables, or replace with replication/ETL for heavy workloads.

---


