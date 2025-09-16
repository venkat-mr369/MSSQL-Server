 **failure demo** of TDE (Transparent Data Encryption). 
 
 This will show you **exactly what happens when you try to restore a TDE-encrypted database backup on another server without the certificate**.

---

# 🛠 Failure Demo: Restore Without Certificate

### 🔹 Step 1: On ProdServer1 (where TDE is enabled)

```sql
-- Create Master Key
USE master;
GO
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'StrongP@ssword123!';

-- Create Certificate
CREATE CERTIFICATE TDECert
WITH SUBJECT = 'TDE Certificate for SalesDB';

-- Create DEK in SalesDB
USE SalesDB;
GO
CREATE DATABASE ENCRYPTION KEY
WITH ALGORITHM = AES_256
ENCRYPTION BY SERVER CERTIFICATE TDECert;
GO

-- Enable Encryption
ALTER DATABASE SalesDB SET ENCRYPTION ON;
GO

-- Backup database (encrypted by TDE)
BACKUP DATABASE SalesDB 
TO DISK = 'D:\Backups\SalesDB_TDE.bak';
```

✅ At this point, the backup is encrypted.

---

### 🔹 Step 2: On DevServer2 (new server, no certificate imported yet)

```sql
-- Try to restore the encrypted backup
RESTORE DATABASE SalesDB
FROM DISK = 'D:\Backups\SalesDB_TDE.bak'
WITH MOVE 'SalesDB' TO 'D:\SQLData\SalesDB.mdf',
     MOVE 'SalesDB_log' TO 'D:\SQLLog\SalesDB.ldf';
```

---

### ❌ Failure Output (Sample Error)

You will see an error like:

```
Msg 33111, Level 16, State 3, Line 1
Cannot find server certificate with thumbprint '0xA1B2C3D4...' in the master database.
```

or

```
Msg 33104, Level 16, State 4, Line 1
A database encryption key cannot be decrypted using the server certificate.
Restore of database 'SalesDB' aborted.
```

👉 **Why?**
Because DevServer2 doesn’t have the same TDE certificate & private key that were used on ProdServer1 to protect the Database Encryption Key (DEK).

---

# 🛠 Step 3: Fix by Importing Certificate

On **DevServer2**, import the certificate and private key before restore:

```sql
-- Create Master Key
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'StrongP@ssword123!';

-- Import Certificate
CREATE CERTIFICATE TDECert
FROM FILE = 'D:\TDE\TDECert.cer'
WITH PRIVATE KEY (
    FILE = 'D:\TDE\TDECert.pvk',
    DECRYPTION BY PASSWORD = 'P@ssw0rd_PrivateKey'
);
```

Now retry the restore → ✅ success.

---

# 📚 Use Case Example

* **Banking Production DB** (`ProdServer1`) has TDE enabled for compliance.
* DBA takes encrypted backup and gives it to **Development Team** for DevServer2 testing.
* Without certificate → Dev cannot restore (sensitive data stays safe).
* Only when DBA explicitly exports/imports the certificate → restore works.

👉 This ensures that **backups can’t be misused** if stolen or copied.

---

# ✅ Summary

* If you restore a TDE backup **without the certificate**, SQL Server will throw **Msg 33111/33104 errors**.
* You must import the **exact certificate + private key** used to encrypt the DEK on the source server.
* This is why **backing up the certificate and private key** is critical in any TDE-enabled environment.

---
