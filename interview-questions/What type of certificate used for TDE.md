**Type of certificate used for TDE (Transparent Data Encryption)** in SQL Server.

---

# 🔎 What Type of Certificate is Used for TDE?

* SQL Server uses an **X.509 certificate** stored in the **master database**.
* This certificate is created and managed **inside SQL Server** (not a Windows/CA-issued certificate).
* The certificate is used to **encrypt the Database Encryption Key (DEK)**, which in turn encrypts the database.

👉 So:

* **TDE itself doesn’t use SSL/TLS certificates for communication.**
* Instead, it uses an **internal SQL Server certificate (X.509)** to protect keys.

---

# 🔑 Key Hierarchy (Arrow Flow)

```
Windows OS
   │
   ▼
Service Master Key (SMK)   ← Auto-created at SQL install
   │
   ▼
Database Master Key (DMK)  ← You create in master
   │
   ▼
TDE Certificate (X.509 in master DB)  ← You create for TDE
   │
   ▼
Database Encryption Key (DEK)  ← Created in user DB
   │
   ▼
Encrypted MDF, LDF, Backups
```

---

# ✅ Key Points About the Certificate

* **Where stored?** → In `master` database.
* **Type?** → X.509 certificate (SQL Server generated, not PKI).
* **Used for?** → Protecting the **DEK**.
* **Backup required?** → Yes, with a private key (`.cer` + `.pvk`).
* **External CA involvement?** → Not required for TDE.

---

# 📚 Example

```sql
USE master;
GO
CREATE CERTIFICATE TDECert
WITH SUBJECT = 'TDE Certificate for SalesDB';
```

* Creates an **X.509 certificate** inside master DB.
* You must back it up:

```sql
BACKUP CERTIFICATE TDECert 
TO FILE = 'D:\TDE\TDECert.cer'
WITH PRIVATE KEY (
    FILE = 'D:\TDE\TDECert.pvk',
    ENCRYPTION BY PASSWORD = 'P@ssw0rd_PrivateKey'
);
```

---
