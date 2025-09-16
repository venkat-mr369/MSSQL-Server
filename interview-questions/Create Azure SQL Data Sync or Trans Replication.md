**deep dive** into **when Azure SQL Data Sync or Transactional Replication are supported vs not supported** for migration to **Azure SQL Database (PaaS)**.

---

# 🔹 **Transactional Replication to Azure SQL Database**

✅ **Supported Cases**

* Source: **SQL Server 2016 SP1 or later** (publisher/distributor).
* Target: **Azure SQL Database (PaaS)** as **subscriber** only.
* Schema & objects supported in replication:

  * **Tables** (with primary keys).
  * Indexed views (some restrictions).
  * Stored procedures (as schema only).
  * Most scalar datatypes (int, varchar, datetime, etc.).

❌ **Not Supported Cases**

* No **merge replication**.
* No **peer-to-peer replication**.
* No **Azure SQL Database as publisher or distributor** → it can only be a subscriber.
* Unsupported objects:

  * Tables without primary keys.
  * Certain datatypes (`geography`, `hierarchyid`, `sql_variant`, CLR UDTs).
  * FILESTREAM/FileTable.
  * Cross-database transactions.
* Heavy OLTP workloads with lots of schema changes (replication breaks if schema changes not replicated).

👉 **Summary:** Use transactional replication only if your DB has **replication-friendly schema** (PKs on all tables, supported datatypes, no unsupported features).

---

# 🔹 **Azure SQL Data Sync**

✅ **Supported Cases**

* Works with **SQL Server 2012 SP1 or later** (on-prem) as member.
* Can do:

  * **One-way sync (On-prem → Azure SQL DB)** → migration scenario.
  * **Two-way sync** (keep both in sync temporarily).
  * **Multiple databases** (hub-and-spoke topology).
* Supported for **OLTP-style workloads** (incremental changes, relatively small batches).

❌ **Not Supported / Not Ideal**

* Not good for **very large DBs (100s of GBs / TBs)** because sync works row-by-row.
* No support for:

  * CDC (Change Data Capture).
  * Large bulk operations (can lag behind).
  * All SQL features (e.g., memory-optimized tables, FILESTREAM, In-Memory OLTP).
  * Complex objects (triggers, cross-database dependencies).
* Schema requirements:

  * Must have **primary keys** on tables to be synced.
  * Limited support for computed columns.
  * Identity columns can cause conflicts.

👉 **Summary:** Data Sync is best if you want **incremental sync for a subset of tables** and can tolerate limitations. It’s not recommended for full-scale large DB migrations (DMS is better).

---

# 🔹 When to Choose Which?

| Feature            | Transactional Replication                       | Azure SQL Data Sync                   |
| ------------------ | ----------------------------------------------- | ------------------------------------- |
| **Target**         | Azure SQL DB (subscriber only)                  | Azure SQL DB (hub)                    |
| **Direction**      | One-way (on-prem → Azure)                       | One-way or Two-way                    |
| **Works best for** | Large OLTP DBs with replication-friendly schema | Selective table sync, hybrid apps     |
| **Downtime**       | Seconds–minutes                                 | Minutes at cutover                    |
| **Limitations**    | No merge, limited datatypes, PK required        | Row-based sync, not good for TB-scale |
| **Use case**       | Near-real-time migration with low downtime      | Partial migrations or hybrid cloud    |

---

# 🔹 Final Check

* If your DB has **all tables with PKs** and supported datatypes → ✅ you can use **Transactional Replication**.
* If your DB is **too complex, lacks PKs, or very large** → ❌ replication may not work → use **DMS Online** instead.
* If you only need **subset of tables** synced → ✅ **Azure Data Sync** is okay.

---

⚡ **Bottom Line:**

* **Transactional Replication = great if schema is compatible**.
* **Azure Data Sync = lightweight but limited** (best for smaller incremental sync scenarios).
* **Azure DMS Online = safest for all cases** (recommended for large migrations).

---
