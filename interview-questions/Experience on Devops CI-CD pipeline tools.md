how CI/CD pipeline tools for **databases** (Liquibase, Flyway, DBmaestro, Azure DevOps, DACPAC, etc.) actually work.
We’ll use a **sample `emp` table** scenario with schema updates (update row, insert new row) to demonstrate **how DevOps CI/CD handles database changes safely**.

---

# 🔹 1. **Database DevOps CI/CD: Core Idea**

* **Application CI/CD** → compile → test → deploy.
* **Database CI/CD** → schema migration scripts + seed/data migration scripts → version controlled → tested → deployed automatically.
* **Key focus**: preserve data integrity, allow rollback, avoid downtime.

---

# 🔹 2. **Example Scenario**

We have an **`emp` table** on SQL Server / Azure SQL:

```sql
CREATE TABLE emp (
    emp_id INT PRIMARY KEY,
    emp_name NVARCHAR(50),
    dept NVARCHAR(50)
);
```

Now we want to:

1. Update employee name `"Kiran"` to `"Kiran Kumar"`.
2. Insert new employee `"Venkat"`.

---

# 🔹 3. **How Different CI/CD Tools Handle This**

---

## ✅ **A. Flyway Example**

Flyway uses **versioned migration scripts** (`V1__init.sql`, `V2__update_emp.sql`, etc.).

**Migration Script: `V2__update_emp_insert_venkat.sql`**

```sql
-- Update existing record
UPDATE emp SET emp_name = 'Kiran Kumar' WHERE emp_name = 'Kiran';

-- Insert new record
INSERT INTO emp (emp_id, emp_name, dept)
VALUES (101, 'Venkat', 'IT');
```

* Script is checked into **Git**.
* Flyway CLI runs inside CI/CD pipeline:

  ```bash
  flyway migrate -url=jdbc:sqlserver://mydbserver -user=sa -password=*** -locations=filesystem:sql/migrations
  ```
* Flyway tracks applied versions in a metadata table (`flyway_schema_history`).
* ✅ Rollback → you can create `undo` scripts (`U2__undo_update_insert.sql`).

---

## ✅ **B. Liquibase Example**

Liquibase supports **SQL, XML, YAML, JSON changelogs**.

**YAML Changelog (`emp_changes.yaml`):**

```yaml
databaseChangeLog:
  - changeSet:
      id: 1
      author: devops
      changes:
        - update:
            tableName: emp
            columns:
              - column:
                  name: emp_name
                  value: "Kiran Kumar"
            where: "emp_name = 'Kiran'"

  - changeSet:
      id: 2
      author: devops
      changes:
        - insert:
            tableName: emp
            columns:
              - column:
                  name: emp_id
                  value: "101"
              - column:
                  name: emp_name
                  value: "Venkat"
              - column:
                  name: dept
                  value: "IT"
```

Run in CI/CD (Jenkins, Azure DevOps, GitHub Actions):

```bash
liquibase update --changelog-file=emp_changes.yaml
```

* ✅ Rollback supported:

  ```yaml
  rollback: "DELETE FROM emp WHERE emp_id = 101;"
  ```

---

## ✅ **C. Azure DevOps + DACPAC (SSDT)**

* Developers maintain schema in **SSDT project**.

* Changes tracked in Git.

* Pipeline builds **DACPAC**.

* Deployment uses:

  ```bash
  SqlPackage.exe /Action:Publish /SourceFile:EmpDB.dacpac /TargetServerName:mydbserver.database.windows.net /TargetDatabaseName:EmpDB /TargetUser:sqladmin /TargetPassword:***
  ```

* DACPAC compares **desired state (schema in project)** vs **target DB** → generates migration script automatically.

For data updates like Kiran→Kiran Kumar or Venkat insert, you can add **post-deployment script**:

```sql
-- PostDeploy.sql
UPDATE emp SET emp_name = 'Kiran Kumar' WHERE emp_name = 'Kiran';
INSERT INTO emp (emp_id, emp_name, dept) VALUES (101, 'Venkat', 'IT');
```

---

## ✅ **D. DBmaestro Example**

DBmaestro provides:

* Change control (approvals).
* Rollback support.
* Policy enforcement.

Steps:

1. Developer commits script:

   ```sql
   UPDATE emp SET emp_name = 'Kiran Kumar' WHERE emp_name = 'Kiran';
   INSERT INTO emp (emp_id, emp_name, dept) VALUES (101, 'Venkat', 'IT');
   ```
2. Pipeline runs DBmaestro plugin in Jenkins/Azure DevOps.
3. DBmaestro enforces:

   * Review required? ✅
   * Rollback script exists? ✅
   * Compliance check passed? ✅
4. If approved → script runs on target DB.

---

# 🔹 4. **Best Practices (Applied to Our Example)**

* **Version every change** → Kiran’s update & Venkat’s insert should each have their own migration script/changeSet.
* **Automated Testing**:

  ```sql
  SELECT COUNT(*) FROM emp WHERE emp_name = 'Kiran Kumar';  -- expected 1
  SELECT COUNT(*) FROM emp WHERE emp_name = 'Venkat';       -- expected 1
  ```

  → can be part of post-deployment test stage.
* **Rollback Planning**:

  ```sql
  UPDATE emp SET emp_name = 'Kiran' WHERE emp_name = 'Kiran Kumar';
  DELETE FROM emp WHERE emp_id = 101;
  ```
* **Idempotent scripts**: Make sure applying twice doesn’t break:

  ```sql
  IF NOT EXISTS (SELECT 1 FROM emp WHERE emp_id = 101)
    INSERT INTO emp (emp_id, emp_name, dept) VALUES (101, 'Venkat', 'IT');
  ```

---

# 🔹 5. **Comparison Table for Our Example**

| Tool                  | How it Runs the Update/Insert    | Rollback                  | CI/CD Integration             |
| --------------------- | -------------------------------- | ------------------------- | ----------------------------- |
| Flyway                | SQL migration script (`V2__...`) | Undo script               | Jenkins, GitLab, Azure DevOps |
| Liquibase             | YAML/JSON/SQL changelog          | Built-in rollback         | All major                     |
| Azure DevOps + DACPAC | PostDeploy.sql in DACPAC         | Manual script             | Native Azure Pipelines        |
| DBmaestro             | Script w/ approval workflow      | Automatic rollback script | Enterprise tools              |

---

✅ **Summary:**

* For our `emp` example → **Flyway** and **Liquibase** give you clear, versioned migration scripts.
* **DACPAC** automates schema comparison + deployment (good for Microsoft shops).
* **DBmaestro** adds **governance + rollback automation** (good for regulated industries).

---
