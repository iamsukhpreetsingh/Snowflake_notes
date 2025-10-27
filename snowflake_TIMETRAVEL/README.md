# Snowflake Time Travel: A Comprehensive Guide with Examples

Snowflake’s **Time Travel** feature is one of its most powerful capabilities for data recovery, auditing, and historical analysis. It allows users to access historical data at any point within a defined retention period—without requiring backups or complex recovery procedures. This article dives deep into Snowflake Time Travel, covering its architecture, configuration, practical usage, limitations, and real-world examples.

---

## 1. What is Snowflake Time Travel?

**Time Travel** enables you to:
- Query data as it existed at a specific point in the past.
- Restore tables, schemas, or entire databases that were accidentally dropped or corrupted.
- Audit changes (e.g., who updated a record and when).
- Recover from accidental `DELETE`, `UPDATE`, or `DROP` operations using simple SQL.

Unlike traditional databases that require manual backups or point-in-time recovery setups, Snowflake handles this **automatically** using immutable micro-partitions.

---

## 2. How Time Travel Works Internally

### Micro-Partitions & Immutability
Snowflake stores data in **micro-partitions** (50–500 MB each) in cloud storage (S3, GCS, or Azure Blob). These partitions are **immutable**—meaning once written, they cannot be modified.

When you run a DML operation (e.g., `UPDATE`, `DELETE`, `INSERT`):
1. Snowflake **does not overwrite** existing micro-partitions.
2. Instead, it creates **new micro-partitions** reflecting the change.
3. The **old partitions are retained** for the duration of the **data retention period** (Time Travel window).

> 🔑 **Key Insight**: Time Travel works because Snowflake keeps historical versions of micro-partitions—not by maintaining transaction logs like traditional RDBMS.

### Data Lifecycle: Time Travel → Fail-Safe
Snowflake’s continuous data protection follows this flow:

```
Current Data → Time Travel (configurable: 0–90 days) → Fail-Safe (non-configurable: 7 days)
```

- **Time Travel**: User-accessible. You can query or restore data.
- **Fail-Safe**: Only accessible by Snowflake Support (e.g., during disaster recovery). **Not user-accessible**.

> ⚠️ **Important**: Fail-Safe is **only available for permanent objects** (tables, schemas, databases). Transient and temporary objects skip Fail-Safe entirely.

---

## 3. Configuring Time Travel Retention

### Default Retention Periods
| Object Type      | Default Retention | Max Retention |
|------------------|-------------------|---------------|
| Permanent Table  | 1 day             | 90 days       |
| Transient Table  | 0 days            | 0 days        |
| Temporary Table  | 0 days            | 0 days        |

> 💡 **Note**: The actual max retention (90 days) is only available in **Enterprise Edition or higher**. Standard Edition caps at **1 day**, even for permanent tables.

### Setting Retention at Different Levels

#### a) At Table Level
```sql
-- Set retention to 10 days for a specific table
ALTER TABLE youtube_learning.raw_layer.aws_customer_load 
SET DATA_RETENTION_TIME_IN_DAYS = 10;
```

#### b) At Schema Level
```sql
-- Set retention to 15 days for the entire schema
ALTER SCHEMA youtube_learning.raw_layer 
SET DATA_RETENTION_TIME_IN_DAYS = 15;
```
> ✅ All tables in this schema **inherit** the 15-day retention—unless explicitly overridden.

#### c) At Database Level
```sql
-- Set retention for the whole database
ALTER DATABASE youtube_learning 
SET DATA_RETENTION_TIME_IN_DAYS = 30;
```

> 🔄 **Inheritance Rule**:  
> Table → inherits from Schema → inherits from Database  
> Explicit table-level settings **override** schema/database defaults.

#### d) Disabling Time Travel
Set retention to `0`:
```sql
ALTER TABLE aws_customer_load SET DATA_RETENTION_TIME_IN_DAYS = 0;
```
> 🔒 **Warning**: Once disabled, you **cannot** recover data via Time Travel—even if the operation just happened.

---

## 4. Accessing Historical Data: 3 Methods

You can query historical data using three Time Travel clauses:

### Method 1: `AT(OFFSET => -N)`
Go back **N seconds** from now.

```sql
-- View data as it was 200 seconds ago
SELECT * FROM aws_customer_load 
AT(OFFSET => -200);
```

> ✅ Best for **recent mistakes** (e.g., you just ran a bad `UPDATE`).

---

### Method 2: `BEFORE(STATEMENT => '<query_id>')`
View data **before a specific query** executed.

#### Step 1: Find the problematic query ID
- Go to **Snowflake Web UI → History → Query History**.
- Filter by table name (`aws_customer_load`) or SQL text (`UPDATE`).
- Copy the **Query ID** of the bad operation.

#### Step 2: Query before that statement
```sql
-- Replace with actual Query ID
SELECT * FROM aws_customer_load 
BEFORE(STATEMENT => '01abc123-def456_gh7890');
```

> ✅ Ideal when you know **which query caused the issue** but not the exact time.

---

### Method 3: `AT(TIMESTAMP => '...')`
Go back to an **exact timestamp**.

```sql
-- View data as it was on June 29, 2025 at 4:20 PM UTC
SELECT * FROM aws_customer_load 
AT(TIMESTAMP => '2025-06-29 16:20:00'::TIMESTAMP);
```

> ⚠️ **Time Zone Note**:  
> Snowflake uses **UTC by default**. If your session uses a different time zone (e.g., `Asia/Kolkata`), ensure timestamps are converted correctly.  
> Use `CONVERT_TIMEZONE()` if needed:
> ```sql
> AT(TIMESTAMP => CONVERT_TIMEZONE('Asia/Kolkata', 'UTC', '2025-06-29 16:20:00')::TIMESTAMP)
> ```

---

## 5. Restoring Dropped or Corrupted Objects

### Restore a Dropped Table
```sql
UNDROP TABLE aws_customer_load;
```

### Restore a Dropped Schema
```sql
UNDROP SCHEMA raw_layer;
```

### Restore a Dropped Database
```sql
UNDROP DATABASE youtube_learning;
```

> ⏳ **Limitation**: You can only `UNDROP` within the **retention period**. After that, the object enters Fail-Safe (inaccessible to users).

---

## 6. Recovering from Accidental Data Changes

### Scenario: Bad `UPDATE` Without `WHERE` Clause
```sql
-- Oops! Updated ALL rows
UPDATE aws_customer_load SET id = 1000;
```

### Recovery Options:

#### Option A: Recreate Table from Historical Snapshot
```sql
CREATE OR REPLACE TABLE aws_customer_load AS
SELECT * FROM aws_customer_load BEFORE(STATEMENT => '01abc123...');
```

#### Option B: Use Zero-Copy Cloning (Next Topic)
> 🔄 Covered in the next video (as mentioned in the transcript), but briefly:
> ```sql
> CREATE TABLE aws_customer_load_restored 
> CLONE aws_customer_load BEFORE(STATEMENT => '01abc123...');
> ```

---

## 7. Monitoring Storage Usage: Time Travel Bytes

To see how much storage is used by Time Travel and Fail-Safe:

### Query Storage Metrics
```sql
SELECT 
    TABLE_NAME,
    ACTIVE_BYTES / (1024*1024*1024) AS active_gb,
    TIME_TRAVEL_BYTES / (1024*1024*1024) AS time_travel_gb,
    FAILSAFE_BYTES / (1024*1024*1024) AS failsafe_gb
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE TABLE_NAME = 'AWS_CUSTOMER_LOAD';
```

> 📊 This helps estimate **storage costs**, as Time Travel data **is billed**.

---

## 8. Key Limitations & Best Practices

### Limitations
- **Fail-Safe is not user-accessible**—only Snowflake Support can recover from it.
- **Transient/Temporary tables have 0-day retention** → no Time Travel.
- **Standard Edition**: Max 1-day retention (even if you set 90).
- **Storage costs increase** with longer retention periods.

### Best Practices
1. **Set appropriate retention** based on business needs (e.g., 7–30 days for critical tables).
2. **Use transient tables** for ETL staging if you don’t need recovery.
3. **Monitor `TABLE_STORAGE_METRICS`** to avoid unexpected costs.
4. **Train teams** to use `BEFORE(STATEMENT => ...)` for precise recovery.
5. **Combine with Zero-Copy Cloning** for safe, instant backups.

---

## Conclusion

Snowflake Time Travel transforms data recovery from a **panic-inducing crisis** into a **routine SQL operation**. By leveraging immutable storage, configurable retention, and intuitive syntax (`AT`, `BEFORE`, `UNDROP`), it provides enterprise-grade data protection without operational overhead.

Whether you’re fixing a mistaken `UPDATE`, auditing historical changes, or restoring a dropped schema, Time Travel ensures your data is **never truly lost**—as long as you act within the retention window.

> 🔜 **Next Step**: Learn about **Zero-Copy Cloning** to create instant, storage-efficient copies of tables at any point in time—perfect for testing, backups, and safe rollbacks.

---

*Note: All examples assume you have the necessary privileges (`USAGE`, `SELECT`, `OWNERSHIP`) on the objects involved.*
