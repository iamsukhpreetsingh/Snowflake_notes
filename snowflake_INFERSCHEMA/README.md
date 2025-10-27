# Snowflake INFER_SCHEMA: Create Tables On-the-Fly from Raw Data Files

In real-world data engineering, you often receive raw files—CSV, JSON, Parquet—from clients or upstream systems **without a predefined schema**. Manually inspecting 100+ columns across dozens of files is time-consuming and error-prone.

Snowflake solves this with a powerful feature: **`INFER_SCHEMA`**—a table function that **automatically detects column names, data types, and structure** from staged files, enabling you to **create tables dynamically**—no manual schema definition needed.

This guide explains how `INFER_SCHEMA` works, walks through a complete example, and shows you how to load data **without ever opening the source file**.

---

## 🧠 What Is `INFER_SCHEMA`?

`INFER_SCHEMA` is a **table function** in Snowflake that:
- Scans files in an **internal or external stage** (not table stages)
- Supports **Parquet, JSON, CSV, and Avro** (not XML)
- Analyzes sample records to **infer column names and data types**
- Returns a structured result you can use to **auto-generate table DDL**

> ✅ Use Case:  
> “Client dropped 50 Parquet files with 200 columns each into S3. I need a Snowflake table—fast.”

---

## 🔧 Syntax & Key Parameters

```sql
SELECT * FROM TABLE(
  INFER_SCHEMA(
    LOCATION => '@your_stage',
    FILE_FORMAT => 'your_file_format',
    FILES => 'file1.parquet,file2.parquet',  -- optional
    IGNORE_CASE => TRUE | FALSE,             -- default: FALSE
    MAX_FILE_COUNT => 10,                    -- default: 1
    MAX_RECORDS_PER_FILE => 10000            -- default: 1000
  )
);
```

### Parameter Breakdown:

| Parameter | Purpose | Best Practice |
|---------|--------|--------------|
| `LOCATION` | Stage name (e.g., `@my_s3_stage`) | Required |
| `FILE_FORMAT` | Predefined file format object | Required |
| `FILES` | Scan only specific files | Use for large directories |
| `IGNORE_CASE` | Normalize column names to uppercase | Set `TRUE` for consistency |
| `MAX_FILE_COUNT` | Limit # of files scanned | Avoid scanning 1000+ files |
| `MAX_RECORDS_PER_FILE` | Limit rows per file | Balance accuracy vs. speed |

> ⚠️ **Note**:  
> - Only works with **named stages** (internal or external)  
> - Does **not support table stages** (`my_table%`)

---

## 🚀 Step-by-Step: Auto-Create a Table from Parquet

### Step 1: Set Up External Stage & File Format

```sql
-- 1. Create file format for Parquet
CREATE OR REPLACE FILE FORMAT parquet_format
  TYPE = PARQUET;

-- 2. Create external stage (linked to S3)
CREATE OR REPLACE STAGE aws_ext_stage
  URL = 's3://your-bucket/data/'
  STORAGE_INTEGRATION = my_s3_integration
  FILE_FORMAT = parquet_format;
```

> 💡 Already have files in S3? Just upload them—no need to know the schema!

---

### Step 2: Use `INFER_SCHEMA` to Detect Structure

```sql
SELECT * FROM TABLE(
  INFER_SCHEMA(
    LOCATION => '@aws_ext_stage',
    FILE_FORMAT => 'parquet_format',
    IGNORE_CASE => TRUE
  )
);
```

#### Sample Output:
| COLUMN_NAME | TYPE | NULLABLE | EXPRESSION | FILE_NAME | ORDER_ID |
|-------------|------|----------|------------|-----------|----------|
| BIRTHDATE   | DATE | YES      | GET_IGNORE_CASE($1, 'birthdate') | user_data_1.parquet | 1 |
| EMAIL       | TEXT | YES      | GET_IGNORE_CASE($1, 'email')     | user_data_1.parquet | 2 |
| ID          | NUMBER | NO     | GET_IGNORE_CASE($1, 'id')        | user_data_1.parquet | 3 |

> 🔍 **Key Insight**:  
> - `IGNORE_CASE = TRUE` → All column names become **uppercase**  
> - `EXPRESSION` shows how to extract data (critical for `COPY INTO`)

---

### Step 3: Create Table Using Template

Use the `USING TEMPLATE` clause to auto-generate the table:

```sql
CREATE OR REPLACE TABLE user_data
  USING TEMPLATE (
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(*))
    FROM TABLE(
      INFER_SCHEMA(
        LOCATION => '@aws_ext_stage',
        FILE_FORMAT => 'parquet_format',
        IGNORE_CASE => TRUE
      )
    )
  );
```

✅ **Result**:  
- Table `user_data` created with **all inferred columns**  
- Data types: `DATE`, `TEXT`, `NUMBER`, etc.  
- No manual DDL writing!

> 📌 **Pro Tip**:  
> `ARRAY_AGG(OBJECT_CONSTRUCT(*))` converts the `INFER_SCHEMA` output into a format Snowflake can use for table creation.

---

### Step 4: Load Data with `COPY INTO`

Now load the data—**but handle case sensitivity**:

```sql
COPY INTO user_data
FROM @aws_ext_stage
FILE_FORMAT = (FORMAT_NAME = 'parquet_format')
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;
```

> ❗ **Why `MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE`?**  
> Because we used `IGNORE_CASE = TRUE`, column names in the table are uppercase, but Parquet metadata may be lowercase. This ensures columns **match correctly**.

✅ **Success**:  
- 1,000 rows loaded  
- Data appears in correct columns

```sql
SELECT * FROM user_data LIMIT 5;
-- Shows: ID, FIRST_NAME, LAST_NAME, EMAIL, BIRTHDATE, COUNTRY...
```

---

## 🆚 Manual vs. Auto Table Creation

| Approach | Steps | Time for 200 Columns | Risk of Error |
|--------|------|---------------------|--------------|
| **Manual** | 1. Inspect file<br>2. Write DDL<br>3. Create table<br>4. Load data | 30–60 mins | High (typos, wrong types) |
| **INFER_SCHEMA** | 1. Stage file<br>2. Run `INFER_SCHEMA`<br>3. `CREATE TABLE USING TEMPLATE`<br>4. `COPY INTO` | 2–5 mins | Very Low |

---

## ⚠️ Important Considerations

### 1. **Accuracy Depends on Sample Size**
- If you set `MAX_RECORDS_PER_FILE = 100`, but a column only appears after row 200 → **it will be missed**.
- **Fix**: Increase sample size or ensure files are representative.

### 2. **Data Type Inference Isn’t Perfect**
- A column with `"123"` and `"abc"` → inferred as `TEXT`
- A column with all numbers → inferred as `NUMBER`
- **Fix**: Review output before creating the table.

### 3. **Case Sensitivity Matters**
- Always pair `IGNORE_CASE = TRUE` with `MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE`
- Otherwise, columns won’t map → load fails.

---

## 💡 Advanced Use Cases

### A. **Infer from Multiple File Types**
```sql
-- For CSV
CREATE FILE FORMAT csv_ff TYPE = CSV SKIP_HEADER = 1;
SELECT * FROM TABLE(INFER_SCHEMA(LOCATION=>'@stage', FILE_FORMAT=>'csv_ff'));
```

### B. **Preview Schema Before Creating Table**
```sql
-- Just review the output—don’t create table yet
SELECT COLUMN_NAME, TYPE, NULLABLE 
FROM TABLE(INFER_SCHEMA(...))
ORDER BY ORDER_ID;
```

### C. **Automate with Stored Procedures**
Wrap the logic in a procedure to auto-ingest any new file format.

---

## ✅ Summary: Best Practices

1. **Always use named stages** (internal or external)
2. **Set `IGNORE_CASE = TRUE`** for consistent column names
3. **Use `MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE`** in `COPY INTO`
4. **Review `INFER_SCHEMA` output** before creating the table
5. **Adjust `MAX_RECORDS_PER_FILE`** based on data complexity

---

## 🔚 Final Thoughts

`INFER_SCHEMA` is a **game-changer** for rapid data ingestion:
- Eliminates manual schema definition
- Reduces human error
- Accelerates time-to-insight

Whether you’re building a data lakehouse, handling client data drops, or automating ETL pipelines, **dynamic table creation** with `INFER_SCHEMA` lets you focus on **analytics—not admin**.

> 🚀 **Try it today**: Drop a Parquet/JSON/CSV file into a stage and run `INFER_SCHEMA`. You’ll have a ready-to-query table in under a minute!
