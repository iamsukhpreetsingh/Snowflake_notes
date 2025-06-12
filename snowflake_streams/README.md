## 🧩 What Are Snowflake Streams?

A **Stream** in Snowflake is a **read-only object** that records **data manipulation language (DML) changes** made to a source table or external table. These changes include:

- `INSERT` operations (new rows added)
- `UPDATE` operations (rows modified)
- `DELETE` operations (rows removed)

This is especially useful for:
- Building **change data capture (CDC)** pipelines
- Implementing **real-time ETL/ELT workflows**
- Creating **audit trails**
- Enabling **incremental data loading**
  
However, it's important to note that **streams do not store the actual data**. Instead, they store **metadata** about these changes, such as:

- The type of operation (`INSERT`, `UPDATE`, `DELETE`)
- The row before and after the change
- A unique identifier for each change (`ROW_ID`)
- The timestamp of the change

This makes streams **lightweight and efficient**, consuming minimal storage while providing valuable insights into data modifications.

---

## 🔍 How Do Streams Work?

### 1. **Create a Stream on a Table**

To begin tracking changes, you must first create a stream on a table.

#### Syntax:
```sql
CREATE OR REPLACE STREAM stream_name ON TABLE table_name;
```

#### Example:
```sql
CREATE OR REPLACE STREAM stream_emp ON TABLE employee_data;
```

This creates a stream named `stream_emp` that tracks all DML changes made to the `employee_data` table.

---

### 2. **Metadata Columns in Streams**

When you query a stream, you'll see several **metadata columns** that help interpret the changes:

| Column Name         | Description |
|---------------------|-------------|
| `METADATA$ACTION`   | Indicates the type of DML operation (`INSERT`, `UPDATE`, `DELETE`) |
| `METADATA$ISUPDATE` | Boolean indicating if the row was updated |
| `METADATA$ROW_ID`   | Unique identifier for the row in the source table |

These metadata fields allow you to determine what changed and when.

---

## ⚙️ Types of Streams

Snowflake supports different types of streams depending on your use case:

### 1. **Standard Streams**
Tracks both `INSERT`, `UPDATE`, and `DELETE` operations.

```sql
CREATE OR REPLACE STREAM stream_emp ON TABLE employee_data;
```

### 2. **Insert-Only Streams**
Captures only new rows inserted into the table. Useful for **external tables** or **Iceberg tables** where updates and deletes are not supported.

```sql
CREATE OR REPLACE STREAM stream_emp_insert_only ON TABLE employee_data INSERT_ONLY = TRUE;
```

> ❗ Note: Insert-only streams cannot be used with internal tables if you need to track updates or deletions.

---

## 📈 Use Cases for Streams

### 1. **Change Data Capture (CDC)**
Use streams to detect and process only the data that has changed since the last load. This is ideal for feeding downstream systems like data lakes, BI tools, or other databases.

### 2. **Audit Logging**
Maintain a historical record of all changes made to sensitive or critical data. For example, tracking who deleted a customer record and when.

### 3. **Incremental Data Pipelines**
Instead of reloading entire datasets, streams enable incremental processing by identifying only the affected rows.

### 4. **Real-Time Dashboards**
Build dashboards that reflect live updates by continuously querying streams and pushing changes to visualization tools.

### 5. **Data Synchronization**
Keep multiple systems in sync by using stream-based triggers to update secondary systems whenever the primary system changes.

---

## 🛠️ Working with Streams: Step-by-Step Guide

### Step 1: Create the Source Table
Ensure you have a source table to track changes.

```sql
CREATE OR REPLACE TABLE employee_data (
    emp_id INT,
    name STRING,
    department STRING,
    salary NUMBER
);
```

### Step 2: Insert Initial Data
Populate the table with some sample data.

```sql
INSERT INTO employee_data VALUES
(101, 'John Doe', 'HR', 60000),
(102, 'Jane Smith', 'Engineering', 80000),
(103, 'Ani', 'Marketing', 70000);
```

### Step 3: Create a Stream
Now create a stream to monitor changes.

```sql
CREATE OR REPLACE STREAM stream_emp ON TABLE employee_data;
```

### Step 4: Make Changes to the Table
Simulate real-world activity with DML operations.

```sql
-- Update a record
UPDATE employee_data SET salary = 85000 WHERE name = 'Jane Smith';

-- Delete a record
DELETE FROM employee_data WHERE name = 'Ani';
```

### Step 5: Query the Stream
Check what changes were captured.

```sql
SELECT * FROM stream_emp;
```

You’ll see output like:

| EMP_ID | NAME       | DEPARTMENT     | SALARY | METADATA$ACTION | METADATA$ISUPDATE | METADATA$ROW_ID |
|--------|------------|----------------|--------|------------------|--------------------|------------------|
| 102    | Jane Smith | Engineering    | 85000  | INSERT           | TRUE               | ...              |
| 102    | Jane Smith | Engineering    | 85000  | DELETE           | TRUE               | ...              |
| 103    | Ani        | Marketing      | NULL   | DELETE           | FALSE              | ...              |

This shows that two changes occurred: an update and a deletion.

---

## 🔄 Consuming Stream Data

Once you’ve captured changes, you can process them in various ways:

### 1. **Move Changes to Another Table**
Insert the changes into a logging or audit table.

```sql
INSERT INTO audit_table
SELECT *, CURRENT_TIMESTAMP() AS change_time
FROM stream_emp;
```

### 2. **Trigger Actions Using Tasks**
Automate responses to changes using **Snowflake Tasks**.

Example:
```sql
CREATE TASK task_process_changes
  WAREHOUSE = my_wh
  SCHEDULE = '1 minute'
AS
INSERT INTO audit_table
SELECT *, CURRENT_TIMESTAMP()
FROM stream_emp;
```

### 3. **Integrate with External Systems**
Use **Snowpipe** or **Snowflake Connectors** to send change data to external platforms like Kafka, AWS Lambda, or Google Cloud Functions.

---

## 🧹 Managing Stream Retention

### 1. **Stream Window**
By default, a stream retains change data for up to **14 days**. After that, the data is automatically purged.

You can check the retention period using:

```sql
SHOW STREAMS;
```

Look for the `RETENTION_TIME` column.

### 2. **Consuming and Resetting Streams**
After processing the changes, it's essential to reset the stream to avoid reprocessing old data.

```sql
ALTER STREAM stream_emp SET CONSUMED = TRUE;
```

This marks the current point in time as the **last consumed position**. Any future queries will only show changes made after this point.

---

## 🧪 Advanced Stream Options

### 1. **Applying Filters**
You can filter which changes are captured based on specific conditions.

```sql
CREATE OR REPLACE STREAM stream_high_salary
ON TABLE employee_data
WHERE salary > 75000;
```

This stream will only capture changes for employees earning more than $75,000.

### 2. **Tracking Only Inserts**
As mentioned earlier, insert-only streams are ideal for external tables.

```sql
CREATE OR REPLACE STREAM stream_inserts_only
ON TABLE sales_data
INSERT_ONLY = TRUE;
```

### 3. **Using Streams with External Tables**
While standard streams don’t support external tables, insert-only streams do.

```sql
CREATE OR REPLACE STREAM stream_external_sales
ON TABLE ext_sales_data
INSERT_ONLY = TRUE;
```

---

## 📊 Monitoring and Troubleshooting

### 1. **Check Stream Status**
Use the following command to inspect stream properties:

```sql
DESCRIBE STREAM stream_emp;
```

### 2. **List All Streams**
See all available streams in your account:

```sql
SHOW STREAMS;
```

### 3. **Query Change History**
If you want to look back at historical changes beyond the stream window, consider maintaining a **snapshot table** or using **Time Travel**.

---

## 🧠 Best Practices

1. **Use Streams with Tasks** – Automate change detection and processing.
2. **Reset Streams After Consumption** – Avoid duplicate processing.
3. **Limit Stream Scope** – Filter streams to relevant data only.
4. **Monitor Stream Retention** – Ensure you’re capturing changes within the 14-day window.
5. **Combine with Time Travel** – Use Time Travel for long-term recovery of lost data.
6. **Track Only Necessary Changes** – Don’t overburden your system with unnecessary change logs.

---

## 🎯 Real-World Scenario: Employee Audit Trail

### Problem:
Your HR team wants to know who deleted an employee record and when.

### Solution:
Set up a stream and audit table to log deletions.

#### Steps:

1. **Create Audit Table**
```sql
CREATE OR REPLACE TABLE employee_audit (
    emp_id INT,
    name STRING,
    action STRING,
    change_time TIMESTAMP
);
```

2. **Create Stream**
```sql
CREATE OR REPLACE STREAM stream_emp_delete ON TABLE employee_data;
```

3. **Capture Deletions**
```sql
INSERT INTO employee_audit (emp_id, name, action, change_time)
SELECT emp_id, name, 'DELETE', CURRENT_TIMESTAMP()
FROM stream_emp_delete
WHERE METADATA$ACTION = 'DELETE';
```

Now, every deletion is logged in the audit table for compliance and review.

---

## 📌 Summary of Key Commands

| Command | Description |
|--------|-------------|
| `CREATE STREAM` | Creates a stream to track DML changes |
| `SELECT * FROM stream_name` | Queries captured changes |
| `ALTER STREAM ... SET CONSUMED = TRUE` | Resets the stream after processing |
| `DESCRIBE STREAM` | Shows stream metadata |
| `SHOW STREAMS` | Lists all available streams |

---

## 🎯 Conclusion

Snowflake Streams offer a **powerful mechanism** to track and respond to data changes in real-time. Whether you're building CDC pipelines, implementing auditing capabilities, or enabling incremental data loads, streams provide a lightweight and efficient way to keep your data ecosystem synchronized and responsive.

Understanding how to configure, query, and manage streams is essential for anyone working with dynamic data environments in Snowflake.

CREDITS: https://www.youtube.com/watch?v=Ng580kGhzf8&list=PL__gObEGy1Y7klsW7vc2TM2Cmt6BwRkzh&index=13
