Configuring **tempdb** properly is one of the most critical things for SQL Server performance, especially in production. 
Let’s break it down step by step for your **16-core CPU system**.

---

### 🔎 1. How Many TempDB Data Files?

Microsoft guidance (and real-world tuning experience):

* Start with **1 file per logical CPU core**, up to **8 files**.
* Beyond 8, add in multiples of 4 only if you still see **PAGELATCH contention** (on allocation pages, e.g., `PAGELATCH_UP` / `PAGELATCH_EX` waits on `tempdb`).
* With **16 cores** → start with **8 data files**.
* Monitor after deployment; if contention persists, increase to **12 or 16**.

👉 Rule of Thumb: **Min(#cores, 8)** to start. Scale further if needed.

---

### 🔎 2. Default File Size for Production

The “right” size depends on workload, but **best practices** are:

* **Do not leave default (8 MB)** → it causes frequent autogrowth.
* Set **initial size large enough** to handle workload without constant growth.
* **Production guideline**:

  * Start with **1–2 GB per file** for OLTP workloads.
  * For heavier workloads (DW/reporting), **4–8 GB per file**.
  * Ensure **equal size across all files**.
* Set **autogrowth** to a **fixed size** (e.g., 512 MB or 1 GB), not % growth.

👉 For 8 files on a 16-core box, a **safe starting config** =

* 8 data files @ **2 GB each**
* 1 log file @ **4 GB (or more depending on workload)**

---

### 🔎 3. Example Config Script for 16-Core Server

```sql
USE master;
GO
ALTER DATABASE tempdb MODIFY FILE (NAME = tempdev, SIZE = 2048MB, FILEGROWTH = 512MB);
GO
-- Add 7 more data files, each equal size
ALTER DATABASE tempdb ADD FILE (NAME = tempdev2, FILENAME = 'E:\SQLData\tempdb2.ndf', SIZE = 2048MB, FILEGROWTH = 512MB);
ALTER DATABASE tempdb ADD FILE (NAME = tempdev3, FILENAME = 'E:\SQLData\tempdb3.ndf', SIZE = 2048MB, FILEGROWTH = 512MB);
ALTER DATABASE tempdb ADD FILE (NAME = tempdev4, FILENAME = 'E:\SQLData\tempdb4.ndf', SIZE = 2048MB, FILEGROWTH = 512MB);
ALTER DATABASE tempdb ADD FILE (NAME = tempdev5, FILENAME = 'E:\SQLData\tempdb5.ndf', SIZE = 2048MB, FILEGROWTH = 512MB);
ALTER DATABASE tempdb ADD FILE (NAME = tempdev6, FILENAME = 'E:\SQLData\tempdb6.ndf', SIZE = 2048MB, FILEGROWTH = 512MB);
ALTER DATABASE tempdb ADD FILE (NAME = tempdev7, FILENAME = 'E:\SQLData\tempdb7.ndf', SIZE = 2048MB, FILEGROWTH = 512MB);
ALTER DATABASE tempdb ADD FILE (NAME = tempdev8, FILENAME = 'E:\SQLData\tempdb8.ndf', SIZE = 2048MB, FILEGROWTH = 512MB);
GO
-- Configure log file
ALTER DATABASE tempdb MODIFY FILE (NAME = templog, SIZE = 4096MB, FILEGROWTH = 1024MB);
GO
```

---

### 🔎 4. Best Practices for TempDB

* Place tempdb on the **fastest disk** (SSD/NVMe).
* Use **dedicated drive** for tempdb if possible.
* Keep all data files the **same size and autogrowth**.
* Use **trace flag 1117 and 1118** (pre-SQL 2016) for uniform file growth and to avoid mixed extents (in SQL 2016+, this is default).
* Monitor for **PAGELATCH contention** (`sys.dm_os_wait_stats`) to decide if you need >8 files.

---

### ✅ Recommended Starting Point for Your 16-Core Server

* **# of tempdb data files**: **8**
* **Initial size per file**: **2 GB** (adjust upwards if workload is heavy)
* **Log file size**: **4 GB (or bigger depending on workload)**
* **Autogrowth**: Fixed **512 MB–1 GB**, no % growth

---
