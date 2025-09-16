**How to handle Disk I/O issues?**

**How to proactively address Disk I/O issue?**

---

#### 🔎 **1. How to Handle Disk I/O Issues (Reactive Approach)**

When Disk I/O is already a problem (slow queries, blocking, timeouts), you need to **identify the bottleneck** and mitigate impact.

#### Where to Check:

* **Wait Stats** → look for I/O-related waits.

  ```sql
  SELECT wait_type, wait_time_ms/1000.0 AS Wait_Sec, 
         signal_wait_time_ms/1000.0 AS Signal_Sec, waiting_tasks_count
  FROM sys.dm_os_wait_stats
  WHERE wait_type LIKE 'PAGEIOLATCH%' OR wait_type LIKE 'WRITELOG'
  ORDER BY wait_time_ms DESC;
  ```
* **File Stats** → measure stall times (I/O latency).

  ```sql
  SELECT DB_NAME(database_id) AS DBName, 
         file_id, io_stall_read_ms, num_of_reads,
         io_stall_write_ms, num_of_writes, 
         io_stall_read_ms/NULLIF(num_of_reads,0) AS Avg_Read_ms,
         io_stall_write_ms/NULLIF(num_of_writes,0) AS Avg_Write_ms
  FROM sys.dm_io_virtual_file_stats(NULL, NULL);
  ```

📌 **Key metrics to focus on**:

* `Avg_Read_ms` > 20 ms → I/O bottleneck
* `Avg_Write_ms` > 10 ms → I/O bottleneck
* `PAGEIOLATCH_*` waits → Slow data file reads
* `WRITELOG` waits → Slow transaction log writes

---

# 🔎 **2. How to Proactively Address Disk I/O Issues (Preventive Approach)**

#### Bullet Points – Proactive Measures:

* ✅ Place **Data, Log, and TempDB** on separate disks.
* ✅ Enable **Instant File Initialization (IFI)** for data files (not logs).
* ✅ Pre-size database and log files → avoid autogrowth overhead.
* ✅ Use **fixed-size autogrowth in MB**, not %.
* ✅ Use **SSD / NVMe** storage for logs and TempDB.
* ✅ Configure multiple **TempDB data files** (1 per logical CPU up to 8).
* ✅ Regularly monitor **I/O latency** via DMV queries or monitoring tools.
* ✅ Archive/purge historical data to reduce table/index bloat.
* ✅ Rebuild/reorganize indexes to reduce random I/O.
* ✅ Use compression (row/page) → fewer reads/writes.

---

# 🔎 **3. Sample Output – 10 Records (from DMV)**

Example output from `sys.dm_io_virtual_file_stats` (simulated):

| DBName      | file\_id | io\_stall\_read\_ms | num\_of\_reads | io\_stall\_write\_ms | num\_of\_writes | Avg\_Read\_ms | Avg\_Write\_ms |
| ----------- | -------- | ------------------- | -------------- | -------------------- | --------------- | ------------- | -------------- |
| AdventureDB | 1        | 25000               | 5000           | 12000                | 4000            | 5.0           | 3.0            |
| AdventureDB | 2        | 78000               | 10000          | 34000                | 2000            | 7.8           | 17.0           |
| SalesDB     | 1        | 450000              | 10000          | 200000               | 5000            | 45.0          | 40.0           |
| SalesDB     | 2        | 120000              | 6000           | 90000                | 2500            | 20.0          | 36.0           |
| FinanceDB   | 1        | 5000                | 2000           | 10000                | 3000            | 2.5           | 3.3            |
| FinanceDB   | 2        | 90000               | 8000           | 70000                | 3500            | 11.3          | 20.0           |
| HRDB        | 1        | 250000              | 5000           | 150000               | 2000            | 50.0          | 75.0           |
| HRDB        | 2        | 8000                | 4000           | 5000                 | 2000            | 2.0           | 2.5            |
| TempDB      | 1        | 300000              | 7000           | 250000               | 6000            | 42.8          | 41.6           |
| TempDB      | 2        | 25000               | 3000           | 30000                | 2000            | 8.3           | 15.0           |

📌 **Interpretation**:

* `SalesDB` and `HRDB` show **high read/write latency (40–75 ms)** → Disk I/O bottleneck.
* `TempDB` also has high latency → move to faster disk & add multiple files.
* `FinanceDB` is healthy (< 5 ms latency).

---

#### 📚 **4. Use Case Example**

**Scenario:**

* Company has a reporting server (`ReportsDB`) with large fact tables (billions of rows).
* Users complain reports are slow.

**Check Results:**

* DMV shows `ReportsDB.mdf` has Avg\_Read\_ms = 55 ms (high latency).
* Wait Stats show `PAGEIOLATCH_SH` dominating.

**Fix (Reactive):**

* Rebuild heavily fragmented indexes → reduce I/O.
* Move `ReportsDB.mdf` to SSD storage.
* Increase read-ahead memory (optimize queries).

**Proactive Changes:**

* Partition large tables to spread I/O.
* Add additional storage throughput (IOPS).
* Schedule index maintenance and statistics updates.
* Regular monitoring alerts when I/O > 20 ms.

---

✅ **Summary (Checklist for Disk I/O Issues):**

* **Reactive (handle issue)** → check wait stats, DMV latency, move files, rebuild indexes, add storage.
* **Proactive (prevent issue)** → separate disks, SSD/NVMe, IFI, pre-size files, proper autogrowth, TempDB optimization, data archiving.
* **Where to check** → `sys.dm_io_virtual_file_stats`, `sys.dm_os_wait_stats`, Perfmon counters (`Disk sec/Read`, `Disk sec/Write`).

---


