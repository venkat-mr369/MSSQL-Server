Patching Sequence **Always On Availability Group (AOAG)** with **3 replicas**:

* **EastUS Node1** → Primary (Synchronous Commit)
* **EastUS Node2** → Secondary (Synchronous Commit)
* **WestUS Node3** → Secondary (Asynchronous Commit, DR site)

You want to **apply OS/SQL patches** across all 3 nodes in a production-safe sequence with **minimal downtime and no data loss**.

---

### 🔎 General Rule for AOAG Patching (Production Safe)

1. **Patch synchronous secondaries first** (one at a time).
2. **Failover Primary to patched secondary**.
3. **Patch old primary** (now secondary).
4. **Finally patch asynchronous replicas** (DR).

---

### 🛠 Step-by-Step Patching Sequence for Your 3-Node Setup

#### ✅ Step 1: Start with EastUS Node2 (Sync Secondary)

* Remove it from availability group for failover testing if needed (`ALTER AVAILABILITY GROUP ... MODIFY REPLICA ... WITH (AVAILABILITY_MODE = SECONDARY_ROLE)` if doing maintenance).
* Apply patch (OS/SQL CU).
* Reboot node.
* Verify synchronization:

  ```sql
  SELECT ag.name, ar.replica_server_name, drs.synchronization_state_desc
  FROM sys.availability_groups ag
  JOIN sys.dm_hadr_availability_replica_states drs ON ag.group_id = drs.group_id
  JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id;
  ```
* Confirm Node2 syncs back with Primary.

---

#### ✅ Step 2: Failover to EastUS Node2

* Perform a **planned failover** from Node1 → Node2.

  ```sql
  ALTER AVAILABILITY GROUP [SalesAG] FAILOVER;
  ```
* Node2 becomes **Primary**.
* Node1 is now a **Secondary (Sync)**.

---

#### ✅ Step 3: Patch EastUS Node1 (now Secondary)

* Apply OS/SQL patches.
* Reboot.
* Verify it resynchronizes as a secondary.

---

#### ✅ Step 4: Failback to Node1 (Optional but Common in Prod)

* If Node1 is your **preferred primary**, fail back:

  ```sql
  ALTER AVAILABILITY GROUP [SalesAG] FAILOVER;
  ```
* Node1 = Primary again.
* Node2 = Sync Secondary.

*(If business prefers Node2 as new primary, you can leave it there.)*

---

#### ✅ Step 5: Patch WestUS Node3 (Async DR)

* Since it’s asynchronous, patch it **last** (lowest risk).
* Apply patch.
* Reboot.
* Verify synchronization (state = `SYNCHRONIZING`).

---

### 🔎 Final State

* All nodes patched.
* Primary running (Node1 or Node2, based on business preference).
* Sync secondary healthy.
* Async secondary DR healthy.

---

### 📚 Why This Sequence?

* **Patch secondaries first** → ensures redundancy exists before touching the primary.
* **Failover to patched node** → you always patch the primary while it’s in secondary role, avoiding downtime.
* **Patch async last** → protects DR in case something goes wrong during primary/secondary patching.

---

### ✅ Summary – Recommended Patching Order

1. **EastUS Node2 (Sync Secondary)** → Patch first.
2. **Failover to Node2**, patch **EastUS Node1** (was primary).
3. (Optional) Fail back to Node1.
4. **Patch WestUS Node3 (Async Secondary)** last.

---
