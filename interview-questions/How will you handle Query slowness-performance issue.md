How will you handle Query slowness-performance issue?

Handling **query slowness / performance issues** in SQL Server, from **basic triage** to **advanced root-cause fixes**. 

# Immediate approach — triage vs deep investigation

* **Triage (first 0–10 min)**: find *is it system-wide* (CPU/IO/memory) or *single-query / blocking / plans*? Run quick checks below.
* **Contain (10–30 min)**: if production is unusable, apply short-term mitigations (kill runaway session, route non-critical workloads away, set resource limits).
* **Investigate (30–240 min)**: capture plans, waits, stats, top consumers; find root cause.
* **Remediate (hours–days)**: add index, update stats, change code, change server config, or hardware as needed.
* **Prevent (ongoing)**: monitoring, baselining, Query Store, CI for SQL, capacity planning.

---

# 1) Immediate triage queries (run these first on the affected server)

Run these *right away* to get the current picture.

### A. Currently running requests (longest first)

```sql
SELECT r.session_id, r.status, r.command, r.cpu_time, r.total_elapsed_time,
       r.reads, r.writes, DB_NAME(r.database_id) AS database_name,
       s.login_name, s.host_name, s.program_name,
       SUBSTRING(t.text, (r.statement_start_offset/2)+1,
         ((CASE r.statement_end_offset WHEN -1 THEN DATALENGTH(t.text) ELSE r.statement_end_offset END - r.statement_start_offset)/2)+1
       ) AS current_statement,
       qp.query_plan
FROM sys.dm_exec_requests r
JOIN sys.dm_exec_sessions s ON r.session_id = s.session_id
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
CROSS APPLY sys.dm_exec_query_plan(r.plan_handle) qp
WHERE r.session_id > 50 -- ignore system
ORDER BY r.total_elapsed_time DESC;
```

**What to look for:** long `total_elapsed_time`, huge `reads/writes`, blocking (see `blocking_session_id`), or excessive CPU.

---

### B. Top resource-consuming queries (historical)

```sql
SELECT TOP 10
  qs.plan_handle, qs.sql_handle,
  qs.total_worker_time/1000 AS total_cpu_ms,
  qs.execution_count,
  (qs.total_worker_time/1000)/NULLIF(qs.execution_count,0) AS avg_cpu_ms,
  qs.total_logical_reads,
  qs.total_logical_reads/NULLIF(qs.execution_count,0) AS avg_logical_reads,
  LEFT(st.text,400) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY qs.total_worker_time DESC;
```

**What to look for:** heavy CPU or logical reads; large average values imply expensive queries.

---

### C. Top waits overall (since last restart)

```sql
SELECT TOP 10 wait_type,
  wait_time_ms/1000.0 AS wait_seconds,
  waiting_tasks_count,
  signal_wait_time_ms/1000.0 AS signal_seconds
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN (
  'CLR_SEMAPHORE','LAZYWRITER_SLEEP','RESOURCE_QUEUE','SLEEP_TASK',
  'BROKER_TASK_STOP','BROKER_TO_FLUSH','SQLTRACE_BUFFER_FLUSH',
  'XE_TIMER_EVENT','XE_DISPATCHER_JOIN','BROKER_EVENTHANDLER')
ORDER BY wait_time_ms DESC;
```

**Interpretation tips:**

* `PAGEIOLATCH_*` → slow storage reads (IO subsystem).
* `WRITELOG` → slow log writes (log disk problem).
* `CXPACKET` / `SOS_SCHEDULER_YIELD` → parallelism / CPU pressure.
* `LCK_M_*` → blocking/locking.
* `ASYNC_NETWORK_IO` → slow client/network or slow consumer.

---

### D. Blocking chain (who is blocking whom)

```sql
SELECT w.session_id AS blocked_session,
       w.blocking_session_id AS blocking_session,
       wt.wait_duration_ms, wt.wait_type,
       s.login_name, s.host_name,
       SUBSTRING(t.text, (r.statement_start_offset/2)+1, 200) AS statement
FROM sys.dm_exec_requests r
JOIN sys.dm_os_waiting_tasks wt ON r.session_id = wt.session_id
JOIN sys.dm_exec_sessions s ON r.session_id = s.session_id
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
LEFT JOIN (SELECT * FROM sys.dm_exec_requests) w ON w.session_id = r.session_id
WHERE wt.blocking_session_id IS NOT NULL
ORDER BY wt.wait_duration_ms DESC;
```

**Action:** identify blocker SPID and either investigate or `KILL <spid>` if safe.

---

### E. Disk I/O health (file-level)

```sql
SELECT DB_NAME(vfs.database_id) AS dbname,
       vfs.file_id,
       vfs.io_stall_read_ms, vfs.num_of_reads,
       vfs.io_stall_write_ms, vfs.num_of_writes,
       CASE WHEN vfs.num_of_reads>0 THEN CAST(vfs.io_stall_read_ms*1.0/vfs.num_of_reads AS DECIMAL(10,2)) ELSE NULL END AS avg_read_ms,
       CASE WHEN vfs.num_of_writes>0 THEN CAST(vfs.io_stall_write_ms*1.0/vfs.num_of_writes AS DECIMAL(10,2)) ELSE NULL END AS avg_write_ms
FROM sys.dm_io_virtual_file_stats(NULL,NULL) vfs
ORDER BY (CASE WHEN vfs.num_of_reads>0 THEN vfs.io_stall_read_ms*1.0/vfs.num_of_reads ELSE 0 END
           +
           CASE WHEN vfs.num_of_writes>0 THEN vfs.io_stall_write_ms*1.0/vfs.num_of_writes ELSE 0 END) DESC;
```

**Rule of thumb:** avg\_read\_ms > 20ms and avg\_write\_ms > 10ms are red flags for OLTP.

---

# 2) Quick containment & hotfixes (use with caution)

* **Kill runaway session:** `KILL <spid>` (if non-critical and blocking/consuming).
* **Reduce concurrency** temporarily: use Resource Governor to limit a runaway workload.
* **Restart problematic service** only if you must and after understanding side effects.
* **Add temp capacity**: move backups, clear old files to reduce IO pressure if disk full.
* **Force last good plan** (Query Store) if a plan regression is known and Query Store enabled.

---

# 3) Find the root cause — structured checklist (categories)

## A. Blocking / application design

**Diagnosis**

* Use queries in section 1.D and `sys.dm_tran_locks` / `sys.dm_os_waiting_tasks`.
* Use Extended Events or Profiler short trace to capture deadlocks.

**Remedy**

* Fix long transactions in app code (commit more frequently).
* Add appropriate indexes to reduce lock duration (reduce scans).
* Use snapshot isolation/READ\_COMMITTED\_SNAPSHOT if appropriate to reduce blocking.
* Use smaller transactions / shorter locks, avoid user interaction inside transactions.

---

## B. Poor execution plan (bad plan / parameter sniffing)

**Diagnosis**

* Compare *estimated* vs *actual* plan (missing/incorrect estimates).
* Check Query Store for plan changes, `sys.dm_exec_query_stats` and `sys.dm_exec_cached_plans`.
* Identify parameter sniffing symptoms: first execution param leads to bad cached plan for other values.

**Remedy**

* Update statistics with `FULLSCAN` or with Sampling if huge table:

  ```sql
  UPDATE STATISTICS dbo.TableName(IndexOrStatsName) WITH FULLSCAN;
  ```
* Use `OPTION (RECOMPILE)` for queries with widely varying parameter cardinality (cost at runtime, but higher CPU for compile).
* Use `OPTIMIZE FOR (@param = <value>)` or `OPTIMIZE FOR UNKNOWN` or local variables to avoid bad sniffing.
* Use plan forcing via Query Store: `EXEC sp_query_store_force_plan @query_id = ..., @plan_id = ...;`
* Consider plan guides for stable behavior.

---

## C. Missing / bad indexes or fragmentation

**Diagnosis**

* Missing index DMV: run `sys.dm_db_missing_index_details` query below.
* Index usage: `sys.dm_db_index_usage_stats` to see unused / heavily scanned indexes.
* Fragmentation: `sys.dm_db_index_physical_stats`.

**Queries**

```sql
-- missing index suggestions
SELECT TOP 50
  DB_NAME(mid.database_id) AS database_name,
  OBJECT_NAME(mid.object_id, mid.database_id) AS table_name,
  mid.equality_columns, mid.inequality_columns, mid.included_columns,
  migs.user_seeks, migs.user_scans, migs.avg_user_impact
FROM sys.dm_db_missing_index_groups mig
JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
ORDER BY migs.avg_user_impact DESC;

-- index usage
SELECT DB_NAME(database_id) AS database_name,
       OBJECT_NAME(object_id, database_id) AS object_name,
       index_id, user_seeks, user_scans, user_lookups, user_updates
FROM sys.dm_db_index_usage_stats
ORDER BY (user_seeks+user_scans) DESC;
```

**Remedy**

* Create covering indexes where helpful (include columns to avoid lookups).
* Avoid blind index creation—simulate effect (use missing index DMVs and check impact).
* Rebuild or reorganize indexes based on fragmentation and page\_count thresholds:

  * `avg_fragmentation_in_percent > 30% AND page_count > 1000` → REBUILD.
  * 5–30% → REORGANIZE.
* Consider filtered indexes for selective predicates.

---

## D. Outdated or bad statistics

**Diagnosis**

* `STATS_DATE(object_id, stats_id)` shows last update.
* Query runtime shows big difference between estimated and actual rows.

**Remedy**

* `UPDATE STATISTICS dbo.Table WITH FULLSCAN;` for critical tables (or `sp_updatestats` for broader).
* Consider `AUTO_CREATE_STATISTICS` and `AUTO_UPDATE_STATISTICS` settings (ON by default).

---

## E. Storage / I/O subsystem

**Diagnosis**

* Check I/O stats (see section 1.E).
* Perfmon counters to check: `PhysicalDisk\Avg. Disk sec/Read`, `Avg. Disk sec/Write`, `Avg. Disk Queue Length`, `\LogicalDisk\% Disk Time`.
* High `PAGEIOLATCH` waits indicate slow reads; `WRITELOG` indicates slow log disk.

**Remedy**

* Move data/log/tempdb to faster disks (SSD/NVMe).
* Increase striping/IOPS or lower latency on SAN.
* Reconfigure autogrowth to sensible fixed MB values to avoid growing too often.
* Offload backups to separate disks.

---

## F. CPU pressure / parallelism

**Diagnosis**

* `SOS_SCHEDULER_YIELD`, `CXPACKET`, `SOS_SCHEDULER_YIELD` waits.
* Top CPU queries (section 1.B).

**Remedy**

* Tune `max degree of parallelism (MAXDOP)` and `Cost Threshold for Parallelism`.
* For specific queries, use `OPTION (MAXDOP 1)` to force serial plan.
* Optimize problematic queries to reduce work (index, rewrite).
* Check for scalar UDFs — inline table-valued functions or rewrite to set-based operations.

---

## G. TempDB contention

**Diagnosis**

* Waits like `PAGELATCH_EX` on `tempdb` allocation pages or high tempdb IO.
* Use `sys.dm_db_file_space_usage` and `sys.dm_db_session_space_usage`.

**Remedy**

* Create multiple tempdb data files (one per logical NUMA core up to 8, then monitor).
* Ensure files are equal size and autogrowth same setting.
* Put tempdb on fast storage.

---

## H. Network / client issues

**Diagnosis**

* `ASYNC_NETWORK_IO` wait can indicate slow client consumption.
* Check network latency and application connection pooling.

**Remedy**

* Investigate application side, improve client processing, enforce connection pooling, increase network bandwidth.

---

# 4) Advanced diagnostics (capture & replay / trace)

### A. Use Query Store (recommended)

* Enable Query Store for history, plan comparison, easy forcing:

```sql
ALTER DATABASE YourDB SET QUERY_STORE = ON;
```

* Use Query Store reports for regressed queries and force last good plan.

### B. Extended Events to capture slow queries (create session)

```sql
CREATE EVENT SESSION [LongRunningQueries] ON SERVER
ADD EVENT sqlserver.sql_batch_completed(
    ACTION(sqlserver.sql_text, sqlserver.session_id, sqlserver.username)
    WHERE (duration > 5000000) -- microseconds = 5 sec
),
ADD EVENT sqlserver.rpc_completed(
    ACTION(sqlserver.sql_text, sqlserver.session_id, sqlserver.username)
    WHERE (duration > 5000000)
)
ADD TARGET package0.event_file(SET filename=N'C:\XE\LongRunning.xel');
ALTER EVENT SESSION [LongRunningQueries] ON SERVER STATE = START;
```

### C. Capture full actual plan and statistics for offline analysis

* Use `sys.query_store_query` / `sys.query_store_plan` to get plans and statistics.
* Save actual execution plan in SSMS (include runtime statistics) to analyze operator costs, missing indexes, warnings.

---

# 5) Example “incident playbook” — step-by-step

1. **Run triage scripts** (section 1). Capture outputs & save them.
2. **Identify top 3 suspects**: (a) blocking session(s), (b) highest CPU/reads query, (c) IO hot file.
3. **If blocking**: find blocker — check what it’s doing; if safe, `KILL <spid>`; otherwise work with app owner.
4. **If a single query**: capture actual plan, check statistics & indexes, check parameter sniffing. Try `OPTION (RECOMPILE)` on test to see improvement.
5. **If IO issue**: check avg\_read\_ms/avg\_write\_ms; involve storage team if high. Move files or add IOPS.
6. **If regression suspected**: check Query Store for last plan changes and consider forcing last good plan.
7. **Quick wins**: update stats on hot tables, rebuild highly fragmented indexes, apply missing index suggestions carefully.
8. **If widespread CPU**: review parallelism, check for runaway ad-hoc compilation, enable `optimize for ad hoc workloads` if necessary.
9. **Document** what you did, timeline, why chosen, and follow-up permanent fix (index, rewrite, config).

---

# 6) Long-term prevention & monitoring (proactive)

* Enable Query Store for query baselining + automatic plan correction.
* Capture wait stats and perf counters historically (monitoring tools like PMM, SCOM, SQL Sentry, Redgate).
* Baseline typical workloads and set alerts for deviations.
* Use CI/CD for DB code & schema changes; code review for expensive queries.
* Regular index and statistics maintenance (not over-aggressive).
* Capacity planning for CPU, memory, I/O; ensure headroom.

---

# 7) Sample outputs (10 records each) — so you know what to expect

### A. Sample Top 10 Queries (by total CPU)

| total\_cpu\_ms | execution\_count | avg\_cpu\_ms | total\_logical\_reads | avg\_logical\_reads | query\_text (truncated)          |
| -------------: | ---------------: | -----------: | --------------------: | ------------------: | -------------------------------- |
|      1,250,000 |              500 |         2500 |            12,500,000 |              25,000 | SELECT \* FROM Orders WHERE ...  |
|        820,000 |               20 |       41,000 |             6,400,000 |             320,000 | exec usp\_GenerateInvoice ...    |
|        410,000 |             1000 |          410 |             4,100,000 |               4,100 | SELECT col1,col2 FROM Lookup ... |
|        205,000 |                5 |       41,000 |             1,000,000 |             200,000 | INSERT INTO BigTable ...         |
|        180,000 |              600 |          300 |             1,800,000 |               3,000 | UPDATE Inventory SET ...         |
|        120,000 |              400 |          300 |               800,000 |               2,000 | SELECT ... JOIN ...              |
|         90,000 |               30 |        3,000 |               300,000 |              10,000 | SELECT ReportData ...            |
|         60,000 |              100 |          600 |               150,000 |               1,500 | DELETE FROM TempTable ...        |
|         40,000 |              250 |          160 |                60,000 |                 240 | SELECT SmallLookup ...           |
|         20,000 |               10 |        2,000 |                20,000 |               2,000 | EXEC sp\_DoWork ...              |

---

### B. Sample Wait Stats Top 10

| wait\_type            | wait\_seconds | waiting\_tasks\_count | signal\_seconds |
| --------------------- | ------------: | --------------------: | --------------: |
| PAGEIOLATCH\_SH       |        12,345 |                   150 |           1,234 |
| WRITELOG              |         5,600 |                    40 |             300 |
| CXPACKET              |         4,500 |                   320 |             200 |
| LCK\_M\_X             |         3,200 |                    45 |              30 |
| ASYNC\_NETWORK\_IO    |         1,800 |                   600 |              20 |
| RESOURCE\_SEMAPHORE   |         1,200 |                    50 |             150 |
| PAGELATCH\_EX         |           900 |                   120 |             100 |
| SOS\_SCHEDULER\_YIELD |           500 |                    30 |             400 |
| BACKUPIO              |           120 |                     2 |              10 |
| OLEDB                 |            60 |                     5 |               5 |

**Interpretation:** PAGEIOLATCH and WRITELOG dominating → storage issue. CXPACKET → look at parallelism.

---

### C. Sample File I/O Stats (10 files)

| dbname    | file\_id | num\_of\_reads | avg\_read\_ms | num\_of\_writes | avg\_write\_ms |
| --------- | -------: | -------------: | ------------: | --------------: | -------------: |
| SalesDB   |        1 |        100,000 |          45.0 |          36,000 |           12.0 |
| SalesDB   |  2 (log) |         10,000 |           2.0 |          80,000 |           50.0 |
| TempDB    |        1 |         80,000 |          40.5 |          60,000 |           41.6 |
| FinanceDB |        1 |         30,000 |           5.0 |          20,000 |            4.0 |
| HRDB      |        1 |         20,000 |          50.0 |          10,000 |           75.0 |
| ArchiveDB |        1 |          5,000 |           8.3 |           2,500 |           10.0 |
| DWDB      |        1 |        150,000 |          12.0 |          20,000 |           15.0 |
| AppDB     |        1 |         40,000 |           4.5 |          15,000 |            3.2 |
| LogDB     |        1 |          1,000 |           1.0 |          50,000 |            0.9 |
| MiscDB    |        1 |          2,000 |           3.0 |           1,500 |            2.5 |

**Interpretation:** log file with avg\_write\_ms 50 ms → log disk slow; TempDB also slow.

---

# 8) Advanced tuning techniques & when to use them

* **Columnstore Indexes** — for large analytic queries / warehousing. Great when scans of huge tables are bottleneck.
* **In-Memory OLTP (Hekaton)** — for very high OLTP throughput, low-latency inserts/updates on specific tables (requires design changes).
* **Plan Forcing / Query Store Automatic Plan Correction** — when plan regression occurs unexpectedly.
* **Parameterization & Plan Guides** — to stabilize plans where necessary.
* **Resource Governor** — to isolate workloads (limit a runaway ETL from choking OLTP).
* **Statistics histogram analysis** — to see skew; sometimes filtered stats or ascending-key problems need specific tuning (e.g., trace flags or application changes).

---

# 9) Safety tips & cautions

* **Do not** run `DBCC FREEPROCCACHE` globally in production (it flushes all cached plans). Prefer clearing a single plan with `DBCC FREEPROCCACHE(plan_handle)` if necessary.
* **Avoid** frequent forced recompiles (`OPTION (RECOMPILE)`) on very frequently executed statements (compile cost).
* **Back up** any certificate or key before changing TDE (we covered this earlier).
* **Test** changes in a staging environment with real-ish data volumes before applying to prod.

---

# 10) Wrap-up — prioritized checklist (short)

1. Run triage queries (running requests, top CPU/reads, waits, blocking).
2. If blocking: identify & kill if safe; examine app code for long transactions.
3. If single query: capture actual plan → check missing indexes + stats → try update stats / index.
4. If IO wait dominated: check file-level I/O → involve storage team / move files / add IOPS.
5. If CPU: find top CPU queries → tune queries / reduce parallelism where appropriate.
6. If tempdb: add equal-sized files on fast storage.
7. Use Query Store + Extended Events for continuous diagnosis and ability to force plans or rollback regressions.
8. Implement monitoring and baselines to detect regressions early.

---

If you want, I can:

* produce a **single “one-click” Triage Script** that runs all the immediate checks (and saves results to a table or file), or
* create a **playbook PDF** you can hand to your on-call DBAs, or
* build a **lab demo** showing parameter sniffing → bad plan → fix via `OPTION (RECOMPILE)` and Query Store forcing.


