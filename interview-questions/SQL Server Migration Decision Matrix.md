
---

# 🧾 SQL Server Migration Decision Matrix

| Criteria                 | Azure SQL Database                                | Azure SQL Managed Instance                                             | SQL Server on Azure VM (IaaS)                        |
| ------------------------ | ------------------------------------------------- | ---------------------------------------------------------------------- | ---------------------------------------------------- |
| **Best For**             | Modern apps, SaaS-style DB-only workloads         | Lift & shift with minimal code changes, needs cross-DB features        | Full control, legacy apps, unsupported features      |
| **Database Size**        | < 4 TB (Standard) / up to 100 TB (Hyperscale)     | Up to 16 TB per DB                                                     | No limit (depends on VM storage)                     |
| **SQL Features Needed**  | No SQL Agent, limited cross-DB, no linked servers | Full cross-DB queries, Agent Jobs, Linked Servers, CLR, Service Broker | 100% feature parity with on-prem                     |
| **HA/DR**                | Built-in HA, Geo-replication, Failover groups     | Built-in HA, Auto-failover groups, Zone-redundant                      | Manual (Always On AG, Failover Cluster)              |
| **Downtime Tolerance**   | Moderate → use DMS Online or Bacpac               | Minimal downtime → use DMS or Managed Instance Link                    | Flexible (backup/restore, log shipping, replication) |
| **Admin Responsibility** | Microsoft manages infra, backups, patching        | Microsoft manages infra + HA, DBA manages DB configs                   | Full admin required (patching, backups, HA config)   |
| **Cost**                 | Lowest TCO for new apps                           | Mid (slightly higher than DB), but less admin                          | Highest (VM, storage, licensing)                     |
| **When to Choose**       | Small-medium apps, new developments, SaaS         | Most migrations from on-prem (compatibility + HA)                      | Legacy workloads, 3rd party apps requiring OS access |

---

# 🔀 Migration Method Decision Guide

| Database Size | Downtime Allowed | Recommended Migration Method                    |
| ------------- | ---------------- | ----------------------------------------------- |
| < 100 GB      | Few hours        | **BACPAC Export/Import** (simple, schema+data)  |
| < 1 TB        | 1–4 hours        | **Backup/Restore via Azure Blob**               |
| 1–4 TB        | Few hours        | **Azure Database Migration Service (Offline)**  |
| 1–4 TB        | Minimal downtime | **Azure Database Migration Service (Online)**   |
| 4–16 TB       | Minimal downtime | **Managed Instance Link (SQL 2022 → Azure MI)** |
| > 16 TB       | Low downtime     | **Log Shipping / Replication → Azure VM**       |

---

# 📚 Use Case Examples

### **Case 1: Banking OLTP (2 TB, cross-database queries, <1 hr downtime)**

* Target: **Azure SQL Managed Instance**
* Migration: **DMS Online**
* HA/DR: Auto-failover group across 2 regions
* Why: Full feature support, minimal downtime, compliance ready.

---

### **Case 2: SaaS App (300 GB, single DB, no legacy features, weekend downtime ok)**

* Target: **Azure SQL Database**
* Migration: **BACPAC Export/Import**
* HA/DR: Active Geo-Replication
* Why: Cost-effective, no complex HA needed, small size.

---

### **Case 3: Legacy ERP (11 TB, SQL Server 2012, needs replication + CLR + SSRS)**

* Target: **SQL Server on Azure VM**
* Migration: **Backup/Restore + Log Shipping**
* HA/DR: AlwaysOn AG in IaaS VMs
* Why: Feature parity, very large size, custom OS-level dependencies.

---

# ✅ Summary

* **If you need full feature support with less admin → Azure SQL Managed Instance**.
* **If DB-only workload, simple apps → Azure SQL Database**.
* **If legacy/unsupported features or very large DB → SQL Server on Azure VM**.
* **Choose migration method** based on **size + downtime tolerance** (BACPAC, Backup/Restore, DMS, MI Link, Log Shipping).

---

👉 Do you want me to also prepare a **step-by-step migration checklist** (tasks before/during/after migration) that DBAs can follow like a runbook during actual migration?
