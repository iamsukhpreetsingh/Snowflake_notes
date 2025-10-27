# Snowflake Access Control: A Practical Guide to Roles, Privileges, and Security

Snowflake’s **Access Control** system ensures that the right people access the right data—no more, no less. Think of it as a **high-tech digital library**: books (data) are organized into sections (databases, schemas, tables), and only authorized users can enter specific sections based on their role.

In this guide, we’ll break down Snowflake’s **Role-Based Access Control (RBAC)** model, explain core concepts, demonstrate how to implement it via SQL and UI, and share best practices—all based on real-world usage patterns.

---

## 1. Why Access Control Matters

Without proper access control:
- Sensitive data (e.g., PII, financial records) could be exposed.
- Users might accidentally (or intentionally) delete or corrupt data.
- Compliance (GDPR, HIPAA, SOC 2) becomes impossible to enforce.

Snowflake solves this with a **hierarchical, role-based permission system** that separates **identity (users)** from **permissions (roles)**.

---

## 2. Core Concepts of Snowflake Access Control

### 🔐 **Securable Objects**
These are resources you can protect:
- **Accounts**, **Warehouses**, **Databases**, **Schemas**, **Tables**, **Views**, **Stages**, etc.

> You **grant privileges** on these objects to **roles**—not directly to users.

---

### 👤 **Users**
- Represent people or applications.
- **Cannot have privileges directly**.
- Must be assigned one or more **roles**.
- Every user automatically gets the **`PUBLIC`** role (a pseudo-role with minimal/no access by default).

---

### 🎭 **Roles**
- **Entities that hold privileges**.
- Users **assume roles** to gain access.
- Roles can **inherit** from other roles (hierarchy).
- Two types:
  - **System-defined roles** (built-in)
  - **Custom roles** (user-created)

---

### ⚙️ **Privileges**
Permissions granted on securable objects. Examples:
- `SELECT`, `INSERT`, `UPDATE`, `DELETE` (on tables)
- `USAGE` (on databases, schemas, warehouses)
- `CREATE SCHEMA`, `CREATE TABLE`, `MONITOR` (on databases/schemas)
- `OPERATE`, `MODIFY` (on warehouses)

> Privileges are **granted to roles**, not users.

---

## 3. Snowflake’s System-Defined Roles (Hierarchy)

Snowflake provides a **predefined role hierarchy**:

```
ACCOUNTADMIN
│
├── SECURITYADMIN
│   └── USERADMIN
│
└── SYSADMIN
    └── (All custom roles should report here)
```

| Role | Responsibilities |
|------|------------------|
| **`ACCOUNTADMIN`** | ⭐ **Top-level role**. Can do **everything**. Equivalent to a "super admin". Should be granted to **very few people**. |
| **`SECURITYADMIN`** | Manages **users, roles, and grants**. Can grant/revoke **any privilege globally**. Inherits `USERADMIN` capabilities. |
| **`USERADMIN`** | Creates and manages **users and roles** (but cannot grant object privileges). |
| **`SYSADMIN`** | Creates and manages **databases, schemas, warehouses, and other objects**. **Best practice**: All custom roles should be granted to `SYSADMIN`. |
| **`PUBLIC`** | Automatically granted to **every user and role**. Use sparingly (e.g., for public datasets). |

> 💡 **Golden Rule**:  
> **Never assign object privileges directly to `PUBLIC`** unless you intend for **everyone** to access it.

---

## 4. How to Implement Access Control: Step-by-Step

### Step 1: Create a Custom Role
```sql
-- Create a role for BI/reporting users
CREATE ROLE bi_role;
```

> ✅ **Best Practice**: Grant custom roles to `SYSADMIN` to maintain hierarchy:
> ```sql
> GRANT ROLE bi_role TO ROLE SYSADMIN;
> ```

---

### Step 2: Create a User and Assign Role
```sql
-- Create user
CREATE USER katrina 
  PASSWORD = 'SecurePass123!' 
  MUST_CHANGE_PASSWORD = FALSE
  DEFAULT_ROLE = bi_role
  DEFAULT_WAREHOUSE = dev_warehouse;

-- Assign role to user
GRANT ROLE bi_role TO USER katrina;
```

> 🔐 The user `katrina` now has **two roles**: `bi_role` (default) + `PUBLIC`.

---

### Step 3: Grant Privileges to the Role

#### a) Grant Database Access
```sql
-- Allow role to "use" the database
GRANT USAGE ON DATABASE youtube_learning TO ROLE bi_role;
```

#### b) Grant Schema Access
```sql
-- Allow role to "use" the schema
GRANT USAGE ON SCHEMA youtube_learning.raw_layer TO ROLE bi_role;
```

#### c) Grant Table Access
```sql
-- Grant SELECT on all existing tables
GRANT SELECT ON ALL TABLES IN SCHEMA youtube_learning.raw_layer TO ROLE bi_role;
```

#### d) Grant Future Table Access (Critical!)
```sql
-- Automatically grant SELECT on tables created in the future
GRANT SELECT ON FUTURE TABLES IN SCHEMA youtube_learning.raw_layer TO ROLE bi_role;
```

> ✅ Without **future grants**, new tables won’t be accessible—forcing manual updates.

---

### Step 4: Grant Warehouse Access
```sql
-- Allow role to use the warehouse
GRANT USAGE ON WAREHOUSE dev_warehouse TO ROLE bi_role;

-- Optional: Allow role to resize/suspend warehouse
GRANT OPERATE ON WAREHOUSE dev_warehouse TO ROLE bi_role;
```

---

## 5. Verifying & Auditing Access

### Check Grants on a Database
```sql
SHOW GRANTS ON DATABASE youtube_learning;
```
Output includes:
- `granted_to`: Which role received the privilege
- `privilege`: Type of access (e.g., `USAGE`)
- `granted_by`: Who granted it (e.g., `ACCOUNTADMIN`)

### Check User’s Roles
```sql
SHOW GRANTS TO USER katrina;
```

### Check Role’s Privileges
```sql
SHOW GRANTS OF ROLE bi_role;
```

---

## 6. Common Pitfalls & Solutions

### ❌ Problem: User can see database but not tables
**Cause**: Missing `USAGE` on schema or `SELECT` on tables.  
**Fix**:
```sql
GRANT USAGE ON SCHEMA db.schema TO ROLE my_role;
GRANT SELECT ON ALL TABLES IN SCHEMA db.schema TO ROLE my_role;
```

---

### ❌ Problem: New tables aren’t accessible
**Cause**: Forgot **future grants**.  
**Fix**:
```sql
GRANT SELECT ON FUTURE TABLES IN SCHEMA db.schema TO ROLE my_role;
```

---

### ❌ Problem: User can’t run queries
**Cause**: Missing warehouse `USAGE`.  
**Fix**:
```sql
GRANT USAGE ON WAREHOUSE my_warehouse TO ROLE my_role;
```

---

## 7. Best Practices for Access Control

1. **Follow the Principle of Least Privilege (PoLP)**  
   → Only grant the minimum access needed.

2. **Use Custom Roles, Not Direct User Grants**  
   → Easier to manage teams (e.g., `analyst_role`, `bi_role`, `etl_role`).

3. **Always Use Future Grants**  
   → Avoid manual privilege updates when new objects are created.

4. **Restrict `ACCOUNTADMIN`**  
   → Limit to 1–2 trusted individuals. Use `SECURITYADMIN`/`SYSADMIN` for daily tasks.

5. **Avoid `PUBLIC` Role for Sensitive Data**  
   → It’s a global role—any user inherits it.

6. **Use Secure Views for Extra Protection**  
   → Hide underlying table logic when sharing data.

---

## 8. Real-World Example: BI Team Setup

```sql
-- 1. Create role
CREATE ROLE powerbi_users;
GRANT ROLE powerbi_users TO ROLE SYSADMIN;

-- 2. Grant database/schema access
GRANT USAGE ON DATABASE analytics TO ROLE powerbi_users;
GRANT USAGE ON SCHEMA analytics.reporting TO ROLE powerbi_users;

-- 3. Grant table access (existing + future)
GRANT SELECT ON ALL TABLES IN SCHEMA analytics.reporting TO ROLE powerbi_users;
GRANT SELECT ON FUTURE TABLES IN SCHEMA analytics.reporting TO ROLE powerbi_users;

-- 4. Grant warehouse access
GRANT USAGE ON WAREHOUSE bi_warehouse TO ROLE powerbi_users;

-- 5. Assign to users
GRANT ROLE powerbi_users TO USER analyst1;
GRANT ROLE powerbi_users TO USER analyst2;
```

Now, both analysts can:
- Query all current/future tables in `analytics.reporting`
- Use the `bi_warehouse`
- **Cannot** drop tables, modify data, or access other schemas

---

## Conclusion

Snowflake’s access control is **powerful, flexible, and secure**—but only if implemented correctly. By leveraging **roles**, **privileges**, and **future grants**, you can:
- Enforce data security
- Simplify user management
- Maintain compliance
- Scale permissions across teams

> 🔑 **Remember**:  
> **Users → Roles → Privileges → Objects**  
> Never skip the role layer!

With this foundation, you’re ready to build a **secure, maintainable, and scalable** data platform in Snowflake.
