**DDL Auditing Framework** that can track schema changes **without hurting performance**. 
That’s very important because DDL triggers and heavy logging can slow things down if not designed carefully.

Here’s a **step-by-step framework** combining **Extended Events (lightweight)** + **central logging table** + **reporting query**.

---

# 🛠 Step 1: Create a Central Audit Table

This is where we store changes after capturing them via XE.

```sql
USE DBAudit;
GO
CREATE TABLE dbo.DDL_AuditLog
(
    AuditID       BIGINT IDENTITY(1,1) PRIMARY KEY,
    EventTime     DATETIME2,
    DatabaseName  NVARCHAR(128),
    ObjectName    NVARCHAR(256),
    ObjectType    NVARCHAR(128),
    EventType     NVARCHAR(128),
    TSQLCommand   NVARCHAR(MAX),
    LoginName     NVARCHAR(128),
    HostName      NVARCHAR(128),
    Application   NVARCHAR(256)
);
```

---

# 🛠 Step 2: Create an Extended Events Session (Lightweight Capture)

Extended Events capture DDL activity with **minimal overhead**.

```sql
CREATE EVENT SESSION [DDL_Audit_XE]
ON SERVER
ADD EVENT sqlserver.object_created(
    ACTION(sqlserver.client_hostname, sqlserver.client_app_name, sqlserver.sql_text, sqlserver.server_principal_name, sqlserver.database_name)
),
ADD EVENT sqlserver.object_altered(
    ACTION(sqlserver.client_hostname, sqlserver.client_app_name, sqlserver.sql_text, sqlserver.server_principal_name, sqlserver.database_name)
),
ADD EVENT sqlserver.object_deleted(
    ACTION(sqlserver.client_hostname, sqlserver.client_app_name, sqlserver.sql_text, sqlserver.server_principal_name, sqlserver.database_name)
)
ADD TARGET package0.event_file
(
    SET filename = 'D:\XEvents\DDL_Audit.xel',
        max_file_size = 50, 
        max_rollover_files = 5
);
GO

ALTER EVENT SESSION [DDL_Audit_XE] ON SERVER STATE = START;
```

👉 This writes DDL changes to `.xel` files (rotating 50 MB files, low overhead).

---

# 🛠 Step 3: Load XE Data into the Audit Table

You can run a SQL Agent Job every X minutes to load XE events into the audit table.

```sql
INSERT INTO dbo.DDL_AuditLog (EventTime, DatabaseName, ObjectName, ObjectType, EventType,
                              TSQLCommand, LoginName, HostName, Application)
SELECT 
    DATEADD(HOUR, DATEDIFF(HOUR, GETUTCDATE(), SYSDATETIMEOFFSET()), 
        event_data.value('(event/@timestamp)[1]', 'DATETIME2')) AS EventTime,
    event_data.value('(event/action[@name="database_name"]/value)[1]', 'NVARCHAR(128)') AS DatabaseName,
    event_data.value('(event/data[@name="object_name"]/value)[1]', 'NVARCHAR(256)') AS ObjectName,
    event_data.value('(event/data[@name="object_type"]/text)[1]', 'NVARCHAR(128)') AS ObjectType,
    event_data.value('(event/@name)[1]', 'NVARCHAR(128)') AS EventType,
    event_data.value('(event/action[@name="sql_text"]/value)[1]', 'NVARCHAR(MAX)') AS TSQLCommand,
    event_data.value('(event/action[@name="server_principal_name"]/value)[1]', 'NVARCHAR(128)') AS LoginName,
    event_data.value('(event/action[@name="client_hostname"]/value)[1]', 'NVARCHAR(128)') AS HostName,
    event_data.value('(event/action[@name="client_app_name"]/value)[1]', 'NVARCHAR(256)') AS Application
FROM 
(
    SELECT CAST(event_data AS XML) event_data
    FROM sys.fn_xe_file_target_read_file('D:\XEvents\DDL_Audit*.xel', NULL, NULL, NULL)
) AS X;
```

---

# 🛠 Step 4: Query the Audit Table (Reporting)

Now you can report on the last 10 schema changes, for example:

```sql
SELECT TOP 10 
    EventTime, DatabaseName, ObjectName, EventType, 
    LoginName, HostName, Application, TSQLCommand
FROM dbo.DDL_AuditLog
ORDER BY EventTime DESC;
```

### ✅ Sample Output (10 records):

| EventTime           | DatabaseName | ObjectName     | EventType         | LoginName                                   | HostName  | Application | TSQLCommand                    |
| ------------------- | ------------ | -------------- | ----------------- | ------------------------------------------- | --------- | ----------- | ------------------------------ |
| 2025-09-16 12:05:01 | SalesDB      | Orders         | CREATE\_TABLE     | [dev1@example.com](mailto:dev1@example.com) | DevPC01   | SSMS        | CREATE TABLE Orders (...)      |
| 2025-09-16 12:10:22 | HRDB         | EmpDetails     | ALTER\_TABLE      | [dev2@example.com](mailto:dev2@example.com) | DevPC02   | SSMS        | ALTER TABLE EmpDetails ADD ... |
| 2025-09-16 12:20:45 | FinanceDB    | Accounts       | DROP\_TABLE       | [dba@example.com](mailto:dba@example.com)   | AdminPC01 | SSMS        | DROP TABLE Accounts            |
| 2025-09-16 12:30:15 | HRDB         | usp\_GetEmp    | CREATE\_PROCEDURE | [dev1@example.com](mailto:dev1@example.com) | DevPC01   | VSCode      | CREATE PROC usp\_GetEmp ...    |
| 2025-09-16 12:35:44 | HRDB         | usp\_GetEmp    | ALTER\_PROCEDURE  | [dev2@example.com](mailto:dev2@example.com) | DevPC03   | SSMS        | ALTER PROC usp\_GetEmp ...     |
| 2025-09-16 12:40:10 | SalesDB      | vw\_Orders     | CREATE\_VIEW      | [dev3@example.com](mailto:dev3@example.com) | DevPC05   | SSMS        | CREATE VIEW vw\_Orders AS ...  |
| 2025-09-16 12:45:20 | SalesDB      | vw\_Orders     | DROP\_VIEW        | [dev3@example.com](mailto:dev3@example.com) | DevPC05   | SSMS        | DROP VIEW vw\_Orders           |
| 2025-09-16 12:55:05 | FinanceDB    | IX\_Accounts   | CREATE\_INDEX     | [dba@example.com](mailto:dba@example.com)   | AdminPC01 | SSMS        | CREATE INDEX IX\_Accounts ...  |
| 2025-09-16 13:05:50 | HRDB         | usp\_GetEmp    | ALTER\_PROCEDURE  | [dev2@example.com](mailto:dev2@example.com) | DevPC03   | SSMS        | ALTER PROC usp\_GetEmp ...     |
| 2025-09-16 13:15:32 | SalesDB      | usp\_GetOrders | DROP\_PROCEDURE   | [dba@example.com](mailto:dba@example.com)   | AdminPC02 | SSMS        | DROP PROC usp\_GetOrders       |

---

# 📚 **Use Cases**

* **Accidental Drops** → Find who dropped a table/proc.
* **Unauthorized Changes** → Detect schema changes outside of change management.
* **Auditing / Compliance** → Provide auditors with full change logs.
* **DevOps Integration** → Validate that deployments happened as expected.

---

# ✅ Summary

* **Extended Events** → Lightweight, minimal performance impact.
* **Central Audit Table** → Easy reporting, long-term history.
* **SQL Agent Job** → Automates loading `.xel` files into audit table.
* **Reports** → Quick insight into last N changes.

---
