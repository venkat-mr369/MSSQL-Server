
**step-by-step assessment plan** for SQL Server **Always On Availability Groups (AG)** with this setup:

* **Region 1 (East)**

  * **Node1** (Primary)
  * **Node2** (Secondary – synchronous commit, automatic failover)

* **Region 2 (West)**

  * **Node3** (DR – asynchronous commit, manual failover)

This is a **classic high availability + disaster recovery (HA+DR)** design.
I’ll break the **assessment steps** into **phases** with **deep details**, so you can map Node1, Node2, and Node3 properly.

---

## 🔍 General Steps for Assessment (Detailed)

### **1. Infrastructure & Network Assessment**

* **Node1 (East – Primary)**

  * Verify OS, SQL Server edition (Enterprise required for multi-node AG).
  * Check CPU, RAM, Storage IOPS.
  * Ensure local disks are high performance (SSD/NVMe).

* **Node2 (East – HA Partner)**

  * Same build/version as Node1.
  * Network latency between Node1 and Node2 must be **<1ms** for synchronous commit.
  * Validate that this node is in **same subnet/region** for automatic failover.

* **Node3 (West – DR Node)**

  * Same SQL Server edition/version/patch level.
  * Validate network bandwidth between East ↔ West (latency usually **>10ms**).
  * Should be **async commit** to avoid latency impact on OLTP.

✅ **Checks:**

* Firewall ports open (1433, 5022 for HADR endpoint).
* DNS/Active Directory replication across regions.
* Validate Windows Failover Cluster communication across subnets.

---

### **2. Windows Failover Cluster (WSFC) Readiness**

* Run **Cluster Validation Wizard** on Node1, Node2, Node3.
* Decide on **quorum configuration**:

  * With 3 nodes across 2 regions → Use **Node Majority + File Share Witness**.
  * Place the witness in Region1 (East) or a neutral 3rd region (cloud).

✅ **Checks:**

* All nodes must see the witness.
* Proper heartbeat and cluster communication between East ↔ West.

---

### **3. SQL Server Configuration Assessment**

* Verify **SQL Server Always On feature enabled** on all nodes.

* Endpoints:

  * Node1: `TCP://node1.east:5022`
  * Node2: `TCP://node2.east:5022`
  * Node3: `TCP://node3.west:5022`

* Security:

  * Use same SQL service account across nodes (or domain account).
  * Endpoints must allow mutual authentication.

* Database readiness:

  * All DBs must be **Full Recovery Model**.
  * Initial full backup + log backup restored (WITH NORECOVERY) on Node2 and Node3.

---

### **4. Availability Group Design**

* **Node1 (Primary, East):**

  * Synchronous commit, automatic failover with Node2.
  * Role: Production OLTP.

* **Node2 (Secondary, East):**

  * Synchronous commit, automatic failover.
  * Can be used for **read-only routing** / backups.

* **Node3 (DR, West):**

  * Asynchronous commit, manual failover.
  * Used for DR testing or reporting workload.

✅ **Checks:**

* AG listener DNS entries replicate across regions.
* Application connection strings use **MultiSubnetFailover=True**.
* Read-only routing list configured for reporting workloads.

---

### **5. Failover & DR Testing Plan**

* **Node1 → Node2 (in East):**

  * Test automatic failover.
  * Check data consistency and client reconnection.

* **Node1 → Node3 (East → West):**

  * Test manual failover (planned/unplanned).
  * Validate RPO (data loss) when async.
  * Measure failover time (RTO).

* **Node2/Node3 → Node1 back failback:**

  * Validate that primary role can be restored without issues.

---

### **6. Monitoring & Maintenance**

* Configure **AG dashboard** for health.
* Enable **Extended Events** for failover tracking.
* Monitor network latency (Node1/2 ↔ Node3).
* Configure backups:

  * Primary DB backups OR offload to Node2/Node3.

---

### **7. Risk & Gap Analysis**

* **Quorum Risk:** If Region1 goes down fully (Node1+Node2), DR Node3 cannot form quorum unless witness is properly placed.
* **Split Brain Risk:** Ensure cluster heartbeat properly configured.
* **Licensing:** Enterprise edition required for multi-region AG.

---

✅ **Summary Mapping (East vs West)**

| Node  | Region | Commit Mode       | Failover Type | Usage                |
| ----- | ------ | ----------------- | ------------- | -------------------- |
| Node1 | East   | Primary (Sync)    | Automatic     | OLTP / Main Prod     |
| Node2 | East   | Secondary (Sync)  | Automatic     | HA partner / Reports |
| Node3 | West   | Secondary (Async) | Manual        | DR / Reporting       |

---


