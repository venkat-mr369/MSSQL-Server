 **SPN (Service Principal Name) issues** in SQL Server, with the context:

* **Domain Name:** `example.com`
* **Service Account (SA):** `sqldb@example.com`
* **SQL Server Host:** `DevServer01`

Let’s go **step by step in depth** with an **easy-to-understand use case**.

---

# 🔎 What is an SPN in SQL Server?

* **SPN (Service Principal Name)** is an Active Directory (AD) identity used by **Kerberos authentication**.
* It tells AD:

  > “This SQL Server service running under `sqldb@example.com` is listening on `DevServer01` (and optionally port 1433).”

Without the right SPN, Kerberos fails → authentication falls back to **NTLM** or may **completely fail** (cannot connect with "Cannot generate SSPI context" errors).

---

# 🛑 Common SPN Issue Symptoms

* Users can connect locally, but not remotely.
* "Login failed for user" errors with Windows Authentication.
* SQL Error logs show `"SSPI handshake failed"` or `"Cannot generate SSPI context"`.
* Kerberos double-hop issues (e.g., linked servers, SSRS to SQL).

---

# 🛠 Step 1: Identify Current SPNs

Run from CMD/PowerShell **on SQL Server node**:

```bash
setspn -L sqldb@example.com
```

This lists SPNs registered for the SQL service account.

For SQL Server, SPNs should look like:

```
MSSQLSvc/DevServer01:1433
MSSQLSvc/DevServer01.example.com:1433
```

---

# 🛠 Step 2: Register Missing SPNs

If missing, register with `setspn`.
Assume SQL runs on `DevServer01`, default instance on TCP 1433, service account = `sqldb@example.com`.

Run (as Domain Admin):

```bash
setspn -S MSSQLSvc/DevServer01:1433 sqldb@example.com
setspn -S MSSQLSvc/DevServer01.example.com:1433 sqldb@example.com
```

* `-S` ensures no duplicates (use `-A` if you want to force add).
* Use **NetBIOS name** (`DevServer01`) and **FQDN** (`DevServer01.example.com`) → both are needed.

---

# 🛠 Step 3: Verify SPN Registration

After adding, verify:

```bash
setspn -L sqldb@example.com
```

Expected output:

```
MSSQLSvc/DevServer01:1433
MSSQLSvc/DevServer01.example.com:1433
```

Check duplicate SPNs:

```bash
setspn -X
```

---

# 🛠 Step 4: Validate Kerberos is Actually Used

Run inside SQL:

```sql
SELECT auth_scheme 
FROM sys.dm_exec_connections
WHERE session_id = @@SPID;
```

* If it returns **KERBEROS** → SPN is working.
* If it returns **NTLM** → still an SPN issue or AD delegation problem.

---

# 📚 Use Case Example – Double-Hop Issue (SSRS → SQL)

1. You have **SSRS running on AppServer01**, connecting to **SQL Server on DevServer01**.
2. A user accesses SSRS via browser (first hop: user → AppServer01).
3. SSRS tries to connect to SQL with **Windows pass-through (Kerberos delegation)**.
4. If the SPN for `DevServer01` is missing, Kerberos can’t authenticate the 2nd hop → authentication fails.

**Fix**: Register correct SPN for `sqldb@example.com` on `DevServer01`. After this, SSRS can successfully delegate the credentials.

---

# 🚨 Best Practices for SPN Management

* Use **dedicated domain accounts** (like `sqldb@example.com`), not LocalSystem/NetworkService.
* Always register both **NetBIOS** and **FQDN**.
* Monitor for **duplicate SPNs** (they cause Kerberos failures).
* Automate SPN registration during SQL installation or patching.
* If using **failover clusters or AG listeners**, SPNs must also be created for listener names.

---

✅ **Summary (for your example):**

* Domain: `example.com`
* SQL Service Account: `sqldb@example.com`
* SQL Host: `DevServer01`

Correct SPNs to register:

```bash
setspn -S MSSQLSvc/DevServer01:1433 sqldb@example.com
setspn -S MSSQLSvc/DevServer01.example.com:1433 sqldb@example.com
```

Then confirm Kerberos is used via `auth_scheme`.

---


