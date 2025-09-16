How to migrate the logins from Onprem to Azure MI ?
---

## 🔹 Why this is important

* In SQL Server, **logins** are at the **server level**, while **users** are at the **database level**.
* If you just migrate databases (via backup/restore, DMS, or log shipping), **users can become “orphaned”** if their associated logins don’t exist at the server level on the target.
* To avoid orphaned users, you must **migrate logins** with **same SID** to Azure SQL MI.

---

## 🔹 Step 1: Understand What We Need to Move

Each login contains:

1. **Login Name** – (`sa`, `app_user`, `domain\svc_sql`, etc.)
2. **Type** – SQL login / Windows login / Contained database user
3. **SID** – Unique security identifier (must remain the same across environments)
4. **Password Hash** – Needed for SQL logins
5. **Server-level Roles** – sysadmin, securityadmin, etc.
6. **Server-level Permissions**
7. **Default database & language**

👉 **Windows logins/groups** are tricky because Azure SQL MI does not support traditional AD logins, but supports **Azure AD authentication**.

---

## 🔹 Step 2: Generate Scripts for Logins in On-Prem

On the source SQL Server (on-prem), use Microsoft’s **sp\_help\_revlogin** or improved scripts.

### Install helper script (on on-prem):

```sql
USE master;
GO
-- Microsoft-provided script (improved sp_help_revlogin)
-- This script generates CREATE LOGIN with SID & hashed password
-- Copy this from Microsoft Docs: https://aka.ms/sphelprevlogin
```

Run:

```sql
EXEC sp_help_revlogin;
```

✅ Output: It generates `CREATE LOGIN` statements with:

* Same SID
* Password hash
* Default DB
* Server roles

---

## 🔹 Step 3: Adapt Scripts for Azure SQL MI

Azure SQL MI is almost fully SQL Server-compatible, but:

* **Windows Logins (domain-based)** must be migrated to **Azure AD logins**. Example:

  ```sql
  CREATE LOGIN [myuser@mydomain.com] FROM EXTERNAL PROVIDER;
  ```
* SQL Logins will migrate fine using the generated scripts. Example:

  ```sql
  CREATE LOGIN [app_user] 
  WITH PASSWORD = 0x0200EFD435...(hashed value)..., 
       SID = 0x010600000000...(same as on-prem)..., 
       DEFAULT_DATABASE = [mydb], 
       CHECK_POLICY = OFF, CHECK_EXPIRATION = OFF;
  ```
* Server roles: Manually apply if needed (`ALTER SERVER ROLE sysadmin ADD MEMBER [user];`).

---

## 🔹 Step 4: Apply the Script on Azure MI

* Connect to Azure SQL MI with `sqlcmd` or SSMS.
* Run the `CREATE LOGIN` scripts.
* Verify with:

  ```sql
  SELECT name, sid, type_desc, create_date 
  FROM sys.server_principals
  WHERE name NOT LIKE '##%';
  ```

---

## 🔹 Step 5: Fix Orphaned Users (if any)

After database migration, check for orphaned users:

```sql
USE [yourdb];
EXEC sp_change_users_login 'Report';
```

Or (modern way):

```sql
SELECT dp.name AS DBUser, sp.name AS ServerLogin
FROM sys.database_principals dp
LEFT JOIN sys.server_principals sp
  ON dp.sid = sp.sid
WHERE dp.authentication_type = 1;
```

If mismatched:

```sql
ALTER USER [db_user] WITH LOGIN = [login_name];
```

---

## 🔹 Step 6: Test

1. Try connecting with the migrated login (SQL auth / Azure AD).
2. Validate roles:

   ```sql
   SELECT IS_SRVROLEMEMBER('sysadmin', 'loginname');
   ```

---

## 🔹 Step 7: Special Cases

1. **Contained Database Users** → They don’t need server logins; they move with DB.

   ```sql
   CREATE USER [ContainedUser] WITH PASSWORD = 'StrongPass@123';
   ```
2. **Cross-database access** → Ensure all logins exist on MI.
3. **Jobs (SQL Agent)** → If you migrate jobs, make sure job owner logins exist on MI.
4. **Linked servers** → Recreate with migrated logins.

---

## 🔹 Step 8: Automation for Multiple Logins

If you have **hundreds of logins**:

* Use **sp\_help\_revlogin** output to a file.
* Run script in MI.
* Verify with `sys.sql_logins` and `sys.server_principals`.

---

## 🔹 Example End-to-End

On-prem:

```sql
EXEC sp_help_revlogin;
```

Output (example):

```sql
CREATE LOGIN [app_user] 
WITH PASSWORD = 0x0200A9A5... HASHED, 
     SID = 0x123456789ABCDEF, 
     DEFAULT_DATABASE=[AppDB], 
     CHECK_POLICY=OFF, CHECK_EXPIRATION=OFF;
```

On Azure MI:

```sql
CREATE LOGIN [app_user] 
WITH PASSWORD = 0x0200A9A5... HASHED, 
     SID = 0x123456789ABCDEF, 
     DEFAULT_DATABASE=[AppDB], 
     CHECK_POLICY=OFF, CHECK_EXPIRATION=OFF;
```

Then map users:

```sql
ALTER USER [app_user] WITH LOGIN = [app_user];
```

---


Do you want me to also give you a **ready-to-use sp\_help\_revlogin script (updated for Azure MI)** so you don’t have to adapt manually?
