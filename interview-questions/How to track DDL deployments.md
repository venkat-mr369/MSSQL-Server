
#### How to track DDL deployments ? 

Tracking **DDL (Data Definition Language) deployments** in SQL Server (like `CREATE`, `ALTER`, `DROP` on tables, views, procs, etc.) is very important for auditing, troubleshooting, and compliance.

Let’s go **deep dive** into the different ways you can track DDL deployments, with **use cases** and **options**.

---

# 🔎 **1. SQL Server DDL Triggers**

* **What it is:** A server- or database-level trigger that fires automatically whenever a DDL statement runs.
* **Best for:** Auditing schema changes in real-time.

### Example:

```sql
CREATE TABLE DDL_AuditLog
(
    EventTime   DATETIME,
    LoginName   NVARCHAR(100),
    UserName    NVARCHAR(100),
    DatabaseName NVARCHAR(100),
    ObjectName  NVARCHAR(200),
    EventType   NVARCHAR(100),
    TSQLCommand NVARCHAR(MAX)
);

CREATE TRIGGER trgDDLAudit
ON DATABASE
FOR CREATE_TABLE, ALTER_TABLE, DROP_TABLE,
    CREATE_PROCEDURE, ALTER_PROCEDURE, DROP_PROCEDURE
AS
BEGIN
    INSERT INTO DDL_AuditLog
    SELECT 
        GETDATE(),
        ORIGINAL_LOGIN(),
        USER_NAME(),
        DB_NAME(),
        EVENTDATA().value('(/EVENT_INSTANCE/ObjectName)[1]', 'NVARCHAR(200)'),
        EVENTDATA().value('(/EVENT_INSTANCE/EventType)[1]', 'NVARCHAR(100)'),
        EVENTDATA().value('(/EVENT_INSTANCE/TSQLCommand)[1]', 'NVARCHAR(MAX)');
END;
```

✅ **Use Case:** Developer drops a table by mistake. You can check `DDL_AuditLog` to see who, when, and what command executed.

---

# 🔎 **2. SQL Server Audit (Enterprise / Std Editions)**

* **What it is:** Built-in auditing feature to capture schema changes into logs or security log.
* **Best for:** Compliance-driven environments (SOX, HIPAA, GDPR).

### Example:

```sql
-- Create an audit
CREATE SERVER AUDIT AuditDDLEvents
TO FILE (FILEPATH = 'D:\SQLAudit\');
ALTER SERVER AUDIT AuditDDLEvents WITH (STATE = ON);

-- Create audit specification
CREATE DATABASE AUDIT SPECIFICATION DDL_AuditSpec
FOR SERVER AUDIT AuditDDLEvents
ADD (SCHEMA_OBJECT_CHANGE_GROUP);
ALTER DATABASE AUDIT SPECIFICATION DDL_AuditSpec WITH (STATE = ON);
```

✅ **Use Case:** Regulated environment → DBA must prove every schema change is tracked in immutable logs.

---

# 🔎 **3. Default Trace (Built-in)**

* **What it is:** SQL Server by default runs a trace that captures schema changes.
* **Best for:** Quick troubleshooting (but limited history).

### Query Example:

```sql
SELECT 
    TE.name AS EventName,
    T.DatabaseName,
    T.ObjectName,
    T.HostName,
    T.ApplicationName,
    T.LoginName,
    T.StartTime
FROM sys.fn_trace_gettable(CONVERT(VARCHAR(150), (SELECT TOP 1 value 
    FROM sys.fn_trace_getinfo(NULL) WHERE property = 2)), DEFAULT) T
JOIN sys.trace_events TE ON T.EventClass = TE.trace_event_id
WHERE TE.name IN ('Object:Created','Object:Altered','Object:Deleted')
ORDER BY T.StartTime DESC;
```

✅ **Use Case:** Yesterday a table disappeared. Check default trace logs to see who dropped it.

---

# 🔎 **4. Extended Events (Modern Alternative to Profiler/Trace)**

* **What it is:** Lightweight monitoring that can capture DDL activity.
* **Best for:** Performance-sensitive systems, fine-grained auditing.

### Example:

```sql
CREATE EVENT SESSION DDL_Audit_XE
ON SERVER
ADD EVENT sqlserver.object_altered,
ADD EVENT sqlserver.object_created,
ADD EVENT sqlserver.object_deleted
ADD TARGET package0.event_file (SET filename='D:\XEvents\DDLAudit.xel');
ALTER EVENT SESSION DDL_Audit_XE ON SERVER STATE = START;
```

✅ **Use Case:** Track `ALTER TABLE` changes in a high-traffic production server without overhead.

---

# 🔎 **5. Source Control / DevOps Tools**

* **What it is:** Track DDL deployments as part of CI/CD pipelines (Git, Azure DevOps, Liquibase, Flyway, Redgate).
* **Best for:** Proactive deployment tracking, version control of schema.

✅ **Use Case:** Every developer change is checked into Git. SQL schema migrations are deployed via CI/CD, and rollback scripts are tracked automatically.

---

# 🔎 **6. Manual Change Management Table**

* **What it is:** A custom solution where DBAs log every approved schema change before deployment.
* **Best for:** Environments without automation tools.

### Example Table:

```sql
CREATE TABLE SchemaChangeLog (
    ChangeID INT IDENTITY PRIMARY KEY,
    ChangeDate DATETIME DEFAULT GETDATE(),
    ChangedBy NVARCHAR(100),
    ChangeDesc NVARCHAR(500),
    ScriptApplied NVARCHAR(MAX)
);
```

✅ **Use Case:** Small shops → DBA manually inserts an entry when running `ALTER TABLE` scripts.

---

# 📊 **Sample Output (10 Records from DDL Audit Table)**

| EventTime           | LoginName                                   | UserName | DatabaseName | ObjectName | EventType         | TSQLCommand                            |
| ------------------- | ------------------------------------------- | -------- | ------------ | ---------- | ----------------- | -------------------------------------- |
| 2025-09-16 11:00:05 | [dev1@example.com](mailto:dev1@example.com) | dbo      | SalesDB      | Orders     | CREATE\_TABLE     | CREATE TABLE Orders (...)              |
| 2025-09-16 11:05:10 | [dev2@example.com](mailto:dev2@example.com) | dbo      | SalesDB      | Orders     | ALTER\_TABLE      | ALTER TABLE Orders ADD OrderDate ...   |
| 2025-09-16 11:15:12 | [dba@example.com](mailto:dba@example.com)   | dbo      | FinanceDB    | Accounts   | DROP\_TABLE       | DROP TABLE Accounts                    |
| 2025-09-16 11:20:40 | [dev1@example.com](mailto:dev1@example.com) | dbo      | HRDB         | EmpDetails | CREATE\_PROCEDURE | CREATE PROC usp\_GetEmpDetails ...     |
| 2025-09-16 11:35:30 | [dev3@example.com](mailto:dev3@example.com) | dbo      | SalesDB      | Orders     | ALTER\_TABLE      | ALTER TABLE Orders ALTER COLUMN ...    |
| 2025-09-16 11:40:11 | [dba@example.com](mailto:dba@example.com)   | dbo      | SalesDB      | vw\_Orders | CREATE\_VIEW      | CREATE VIEW vw\_Orders AS ...          |
| 2025-09-16 11:50:25 | [dev2@example.com](mailto:dev2@example.com) | dbo      | SalesDB      | Orders     | DROP\_VIEW        | DROP VIEW vw\_Orders                   |
| 2025-09-16 12:05:10 | [dev1@example.com](mailto:dev1@example.com) | dbo      | FinanceDB    | Accounts   | CREATE\_INDEX     | CREATE INDEX IX\_Accounts\_Name ON ... |
| 2025-09-16 12:10:44 | [dev3@example.com](mailto:dev3@example.com) | dbo      | HRDB         | EmpDetails | ALTER\_PROCEDURE  | ALTER PROC usp\_GetEmpDetails ...      |
| 2025-09-16 12:20:55 | [dba@example.com](mailto:dba@example.com)   | dbo      | SalesDB      | Orders     | DROP\_PROCEDURE   | DROP PROC usp\_GetOrders               |

---

# ✅ **Summary – Multiple Options**

* **DDL Triggers** → Custom logging, flexible.
* **SQL Audit** → Compliance-grade, enterprise environments.
* **Default Trace** → Quick, limited history.
* **Extended Events** → Lightweight, modern.
* **Source Control / DevOps** → CI/CD integration, proactive.
* **Manual Change Log** → Simple environments without automation.

---
