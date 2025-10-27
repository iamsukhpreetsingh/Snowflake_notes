# Snowflake Views: A Complete Guide to Standard, Materialized, and Secure Views

In Snowflake, **views** are powerful virtual constructs that simplify data access, enforce security, and abstract complexity—without duplicating data (in most cases). This guide dives deep into the **three types of views** in Snowflake—**Standard (Non-Materialized)**, **Materialized**, and **Secure Views**—with practical examples, use cases, limitations, and performance considerations.

---

## 1. What is a View in Snowflake?

A **view** is a **virtual table** defined by a SQL query. It does **not store data** (except for Materialized Views) but dynamically retrieves data from underlying base tables when queried.

### Common Use Cases:
- **Row-level security**: Filter rows (e.g., only US customers).
- **Column-level security**: Hide sensitive columns (e.g., account balance, phone number).
- **Simplify complex logic**: Expose joins, aggregations, or transformations as a single object.
- **Data abstraction**: Shield end users (e.g., analysts, BI tools) from table complexity.

---

## 2. Standard (Non-Materialized) Views

### Definition
A standard view executes its underlying query **at runtime** every time it’s accessed. No data is stored.

### Syntax
```sql
CREATE [OR REPLACE] VIEW view_name AS
SELECT ...
FROM ...
[WHERE ...];
```

### Example: Row-Level Security
Suppose you only want to expose customers from nation keys `13` and `15`:
```sql
CREATE OR REPLACE VIEW vw_us_customers AS
SELECT *
FROM snowflake_sample_data.tpch_sf1.customer
WHERE c_nationkey IN (13, 15);
```

> ✅ End users query `vw_us_customers` without knowing the filter logic.

### Example: Column-Level Security Using `EXCLUDE`
Hide sensitive columns like `c_acctbal` and `c_phone`:
```sql
CREATE OR REPLACE VIEW vwc_customer AS
SELECT * EXCLUDE (c_acctbal, c_phone)
FROM snowflake_sample_data.tpch_sf1.customer;
```

> 🔑 **`EXCLUDE` is a Snowflake-specific feature**—ideal when you have 100+ columns and want to omit just a few.

### Characteristics
| Feature | Standard View |
|--------|----------------|
| **Data Storage** | ❌ None |
| **Storage Cost** | $0 |
| **Compute Cost** | Incurred **only when queried** |
| **Data Freshness** | Always real-time |
| **Multi-Table Support** | ✅ Yes (joins, subqueries) |
| **Aggregations** | ✅ Supported |
| **Performance** | Slower for complex/large queries |

> 💡 **Best for**: Security, abstraction, and dynamic data access where real-time freshness is critical.

---

## 3. Materialized Views

### Definition
A **Materialized View (MV)** **stores the result set physically** in Snowflake’s storage layer—like a table—but is automatically maintained by Snowflake.

### Syntax
```sql
CREATE [OR REPLACE] MATERIALIZED VIEW mv_name AS
SELECT ...
FROM table_name
[WHERE ...]
[GROUP BY ...];
```

### Example
```sql
CREATE OR REPLACE MATERIALIZED VIEW mvw_customer_summary AS
SELECT 
    c_nationkey,
    COUNT(*) AS customer_count
FROM snowflake_sample_data.tpch_sf1.customer
GROUP BY c_nationkey;
```

### Key Advantages
- **Faster query performance**: Data is pre-computed and stored.
- **Automatic refresh**: Snowflake updates the MV **in the background** when the base table changes (serverless maintenance).
- **Ideal for aggregations** on large, infrequently updated tables.

### Critical Limitations ⚠️
1. **Single-table only**:  
   ❌ **Cannot reference more than one table** (no joins, no subqueries from other tables).
   ```sql
   -- ❌ This will FAIL
   CREATE MATERIALIZED VIEW mv_bad AS
   SELECT c.c_name, n.n_name
   FROM customer c
   JOIN nation n ON c.c_nationkey = n.n_nationkey;
   -- Error: "More than one table referenced"
   ```

2. **Limited SQL support**:  
   No window functions, CTEs, or complex expressions in some cases.

3. **Storage cost**: You pay for the stored result set.

### When to Use Materialized Views
✅ Use when:
- Querying **aggregations** (SUM, COUNT, AVG) on **one large table**.
- Base table **changes infrequently**.
- You accept **storage costs** for **compute savings**.
- Querying **external tables** (to avoid repeated cloud storage scans).

❌ Avoid when:
- You need joins or multi-table logic.
- Data changes very frequently (high refresh overhead).
- Storage cost is a concern.

### Monitoring Materialized Views
Check view type using:
```sql
SHOW VIEWS IN SCHEMA youtube_learning.raw_layer;
```
Look for the `is_materialized` column → `true` = Materialized View.

---

## 4. Secure Views

### Definition
A **Secure View** **hides the view definition** from non-owners, enhancing data privacy and preventing exposure of underlying logic or sensitive table structures.

### Syntax
```sql
CREATE [OR REPLACE] SECURE VIEW secure_view_name AS
SELECT ...
FROM ...;
```

### Example
```sql
CREATE OR REPLACE SECURE VIEW svw1 AS
SELECT * EXCLUDE (c_acctbal, c_phone)
FROM snowflake_sample_data.tpch_sf1.customer;
```

### Why Use Secure Views?
1. **Hide query logic**:  
   Non-owners **cannot see** the base tables, filters, or columns used.
2. **Prevent metadata leakage**:  
   Even `SHOW VIEWS` or `GET DDL` returns **no definition** for non-owners.
3. **Required for Data Sharing**:  
   ❗ You **must use Secure Views** when sharing data via **Snowflake Data Sharing** or **Data Marketplace**.

### Trade-offs
| Aspect | Secure View |
|-------|-------------|
| **Performance** | ❌ Slower than non-secure views (due to additional security checks) |
| **Visibility** | ✅ Definition hidden from non-owners |
| **Use Case** | ✅ Mandatory for cross-account data sharing |

> 🚫 **Do NOT use Secure Views for every view**—only when **data privacy** or **sharing** is required.

### Verification
- As a non-owner, run:
  ```sql
  SELECT * FROM svw1; -- Works
  SHOW VIEWS LIKE 'svw1'; -- Shows view exists, but NO definition
  ```
- In **Query Profile**, the underlying tables **won’t appear** in the execution graph.

---

## 5. Comparison: Standard vs Materialized vs Secure Views

| Feature | Standard View | Materialized View | Secure View |
|--------|----------------|-------------------|-------------|
| **Stores Data** | ❌ | ✅ | ❌ |
| **Storage Cost** | $0 | ✅ (billed) | $0 |
| **Real-Time Data** | ✅ | ⚠️ Near real-time (auto-refresh) | ✅ |
| **Multi-Table Joins** | ✅ | ❌ | ✅ |
| **Aggregations** | ✅ | ✅ (best for single-table) | ✅ |
| **Hides Definition** | ❌ | ❌ | ✅ |
| **Data Sharing Ready** | ❌ | ❌ | ✅ |
| **Performance** | Slower (on-demand) | Fastest (precomputed) | Slower (security overhead) |
| **Auto Refresh** | N/A | ✅ (serverless) | N/A |

---

## 6. Best Practices

1. **Use Standard Views** for:
   - Row/column security.
   - Simplifying complex queries for BI users.
   - Dynamic, real-time reporting.

2. **Use Materialized Views** only when:
   - Querying **single-table aggregations**.
   - Base data is **stable**.
   - You’ve validated the **cost vs performance trade-off**.

3. **Use Secure Views** when:
   - Sharing data externally.
   - Protecting sensitive transformation logic.
   - Compliance requires metadata hiding.

4. **Avoid**:
   - Creating Secure Views unnecessarily (performance hit).
   - Assuming Materialized Views support joins (they don’t).
   - Forgetting that `EXCLUDE` only works in **SELECT \*** contexts.

---

## Conclusion

Snowflake’s view system offers **flexible, secure, and performant** ways to expose data:
- **Standard Views** = Security + Abstraction.
- **Materialized Views** = Performance (with constraints).
- **Secure Views** = Privacy + Sharing Compliance.

By choosing the right view type for your use case, you can **reduce complexity**, **enhance security**, and **optimize costs**—all while delivering reliable data to your consumers.

> 🔜 **Pro Tip**: Combine **Secure Standard Views** with **Row Access Policies** and **Column Masking Policies** for enterprise-grade data governance in Snowflake.
