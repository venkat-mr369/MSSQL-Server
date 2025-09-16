Excellent question 👍 — you’re asking specifically about **Data in Transit (SSL/TLS for SQL Server)** and how **CNAMEs** (DNS aliases) play into it.

Let’s do a **deep dive** into **why CNAMES matter**, **how to configure them**, and **how to handle certificates (CN/SANs)** so clients can connect securely.

---

# 🔎 Why CNAMES in SQL Server TLS?

* Clients often connect using a **CNAME (DNS alias)** instead of the real server hostname.

  * Example: Instead of `SQLNode1.corp.example.com`, clients use `SQLProd.corp.example.com`.
* Benefits:
  ✅ Easier migrations (point alias to new server without changing apps).
  ✅ Consistent connection string across environments.
* Problem:

  * During TLS handshake, SQL presents a certificate with a **Subject (CN) or SAN (Subject Alternative Name)**.
  * If the alias (`SQLProd.corp.example.com`) is not on the certificate, clients get **certificate mismatch errors**.

---

# 🛠 Step 1: Check Current Certificate

1. Open `certlm.msc` → Local Computer → Personal → Certificates.
2. Look at the SQL Server cert bound to the SQL instance.

   * Check the **CN (Common Name)**.
   * Check **SANs** (Subject Alternative Names).

👉 If you only see `CN=SQLNode1.corp.example.com`, then connections via `SQLProd.corp.example.com` will fail TLS validation.

---

# 🛠 Step 2: Add CNAMES in DNS

* On your **DNS server**, create CNAMEs for SQL Server.
  Example:

  ```
  SQLProd.corp.example.com → SQLNode1.corp.example.com
  SQLDR.corp.example.com   → SQLNode2.corp.example.com
  ```

Now clients can use `SQLProd` instead of the real host.

---

# 🛠 Step 3: Update / Request Certificate with CNAMES (SANs)

The certificate must cover all the hostnames (real + CNAMEs) that clients use.

### Option A — Internal Enterprise CA (best for corporate)

Request cert with SANs:

```
CN = SQLProd.corp.example.com
SAN = SQLProd.corp.example.com, SQLNode1.corp.example.com
```

### Option B — Self-signed (for lab/testing)

Use PowerShell:

```powershell
New-SelfSignedCertificate `
  -DnsName "SQLProd.corp.example.com","SQLNode1.corp.example.com" `
  -CertStoreLocation "cert:\LocalMachine\My" `
  -KeyExportPolicy Exportable `
  -KeyLength 2048 `
  -NotAfter (Get-Date).AddYears(3) `
  -TextExtension "2.5.29.37={text}1.3.6.1.5.5.7.3.1"
```

👉 This creates a cert valid for **both the CNAME and real hostname**.

---

# 🛠 Step 4: Bind Certificate to SQL Server

1. Open **SQL Server Configuration Manager**.
2. Go to **SQL Server Network Configuration → Protocols for MSSQLSERVER**.
3. On the **Certificate tab**, select the new cert.
4. On the **Flags tab**, set **Force Encryption = Yes** (optional, after testing).
5. Restart SQL Server service.

---

# 🛠 Step 5: Test Connection

From client:

```bash
sqlcmd -S SQLProd.corp.example.com -E
```

Check encryption:

```sql
SELECT session_id, encrypt_option, auth_scheme
FROM sys.dm_exec_connections
WHERE session_id = @@SPID;
```

Expected:

```
encrypt_option = TRUE
auth_scheme    = KERBEROS
```

---

# 📚 Use Case Example

* **Scenario:**

  * Apps connect to `SQLProd.corp.example.com` (CNAME).
  * Underlying physical server = `SQLNode1.corp.example.com`.
* **Without SAN:** TLS fails (`certificate name mismatch`).
* **With SAN including CNAME:** TLS succeeds.
* **Benefit:** Later, when you move prod to `SQLNode2`, you just repoint CNAME → clients keep working securely with no connection string change.

---

# ✅ Summary

* **CNAMES help**: make SQL migration/HA easier.
* **But**: TLS cert must have the **CNAMEs listed in SANs** (or CN).
* **Steps**:

  1. Create DNS CNAME.
  2. Issue cert with SANs (real name + CNAME).
  3. Bind cert to SQL via Config Manager.
  4. Restart SQL & test with CNAME.

---
