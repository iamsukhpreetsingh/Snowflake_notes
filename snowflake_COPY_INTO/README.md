# Snowflake COPY INTO Command

## 📥 What is the COPY INTO Command?

The `COPY INTO` command is used to **load data from files stored in a stage** into a **Snowflake table**. These files can be in formats like CSV, JSON, Parquet, ORC, Avro, etc., and are typically staged using internal or external stages.

### 🔧 Basic Syntax
```sql
COPY INTO <target_table>
FROM <stage_name>
FILE_FORMAT = (TYPE = '<file_type>');
```

You can also specify additional options depending on your use case.

---

## 📁 Prerequisites for Using COPY INTO

Before executing the `COPY INTO` command, ensure:
- The **target table exists** with appropriate schema.
- The **file(s)** have been uploaded to a **named internal or external stage**.
- The **file format** has been defined and associated with the stage or specified inline.

---

## 🛠️ Step-by-Step Process

### 1. **Create Target Table**
Ensure the target table matches the structure of the incoming data.

Example:
```sql
CREATE OR REPLACE TABLE dim_customer (
    CustomerID INT,
    Name VARCHAR,
    Address VARCHAR,
    CompanyName VARCHAR
);
```

### 2. **Create File Format**
Define how the file should be parsed.

Example:
```sql
CREATE OR REPLACE FILE FORMAT ingest_data
TYPE = CSV
SKIP_HEADER = 1
ERROR_ON_COLUMN_COUNT_MISMATCH = FALSE;
```

### 3. **Create Stage**
Link the stage with the file format.

Example:
```sql
CREATE OR REPLACE STAGE stage_csv
FILE_FORMAT = ingest_data;
```

### 4. **Upload Files to Stage**
Using SnowSQL or UI, upload files to the stage.

Example:
```bash
PUT file:///path/to/dim_customer.csv @stage_csv;
```

### 5. **List Files in Stage**
Verify that files were uploaded successfully.

```sql
LIST @stage_csv;
```

---

## ⚙️ Running the COPY INTO Command

### Basic Load
```sql
COPY INTO dim_customer
FROM @stage_csv;
```

This loads all files from the stage into the table based on the associated file format.

---

## 🛡️ Handling Errors During Data Load

When loading large datasets, it’s common to encounter malformed or invalid records. Snowflake provides several mechanisms to handle these scenarios gracefully.

### 1. **Validation Mode**
Use `VALIDATION_MODE = RETURN_ALL_ERRORS` to validate the entire file and return all error messages without inserting any data.

Example:
```sql
COPY INTO dim_customer
FROM @stage_csv
VALIDATION_MODE = RETURN_ALL_ERRORS;
```

This helps identify issues like:
- Incorrect data types
- Mismatched column counts
- Invalid date formats

### 2. **On Error Mode**
Use `ON_ERROR` to control behavior when errors occur during the load.

Options:
- `ABORT_STATEMENT`: Default; stops the load on first error.
- `CONTINUE`: Skips bad rows and inserts valid ones.
- `SKIP_FILE`: Skips the entire file if any error occurs.

Example:
```sql
COPY INTO dim_customer
FROM @stage_csv
ON_ERROR = CONTINUE;
```

This will insert all valid rows and skip corrupt ones.

---

## 🔁 Re-loading Files with FORCE Option

By default, Snowflake keeps track of which files have already been loaded (for up to 64 days). To re-load a file (e.g., after fixing data), use the `FORCE = TRUE` option.

Example:
```sql
COPY INTO dim_customer
FROM @stage_csv
FORCE = TRUE;
```

⚠️ **Note**: This may result in duplicate data if not handled carefully.

---

## 🗑️ Purging Files After Successful Load

To automatically delete files from the stage after they’ve been successfully loaded, use the `PURGE = TRUE` option.

Example:
```sql
COPY INTO dim_customer
FROM @stage_csv
PURGE = TRUE;
```

This helps reduce storage costs by removing processed files.

---

## 🔄 Matching Columns by Name

If your source file columns are not in the same order as your target table, use `MATCH_BY_COLUMN_NAME`.

Example:
```sql
COPY INTO employee_data
FROM @stage_csv
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;
```

Supported modes:
- `CASE_SENSITIVE`
- `CASE_INSENSITIVE`
- `NONE` (default)

This ensures correct mapping even if column order differs.

---

## 📄 Including Metadata Columns

Snowflake allows you to include metadata about the file being loaded, such as:
- `FILENAME`
- `ROW_NUMBER`
- `ETL_LOAD_TIME`
- `LAST_MODIFIED`

To include metadata, use `INCLUDE` and define corresponding columns in the target table.

Example:
```sql
CREATE OR REPLACE TABLE employee_data (
    EmpID INT,
    Name VARCHAR,
    FileName VARCHAR,
    LoadTime TIMESTAMP
);

COPY INTO employee_data(EmpID, Name, FileName, LoadTime)
FROM (
    SELECT $1, $2, METADATA$FILENAME, CURRENT_TIMESTAMP()
    FROM @stage_csv
);
```

Alternatively, use shorthand:
```sql
COPY INTO employee_data
FROM @stage_csv
INCLUDE = (METADATA$FILENAME AS FileName, CURRENT_TIMESTAMP() AS LoadTime);
```

---

## 📂 Copying Specific Files

If multiple files exist in a stage but you want to load only a specific one, use the `PATTERN` or `FILES` parameter.

Example:
```sql
COPY INTO employee_data
FROM @stage_csv
FILES = ('employee_load.csv');
```

Or match using regex:
```sql
COPY INTO employee_data
FROM @stage_csv
PATTERN = '.*employee.*csv';
```

---

## 🧪 Example Use Case: Loading Employee Data

### 1. Define File Format
```sql
CREATE OR REPLACE FILE FORMAT emp_format
TYPE = CSV
SKIP_HEADER = 1
DATE_FORMAT = 'DDMMYYYY';
```

### 2. Create Stage
```sql
CREATE OR REPLACE STAGE emp_stage
FILE_FORMAT = emp_format;
```

### 3. Upload File
```bash
PUT file:///path/to/employee_load.csv @emp_stage;
```

### 4. Load Data with Metadata
```sql
COPY INTO employee_data(EmpID, Name, JoiningDate, Country, FileName, LoadTime)
FROM (
    SELECT $1, $2, $3, $4, METADATA$FILENAME, CURRENT_TIMESTAMP()
    FROM @emp_stage
)
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE
PURGE = TRUE;
```

This example:
- Loads employee data
- Matches columns by name (case-insensitive)
- Includes file name and load timestamp
- Automatically deletes the file after successful load

---

## 📊 Monitoring and Verifying the Load

After running `COPY INTO`, verify the results:

### Check Loaded Rows
```sql
SELECT COUNT(*) FROM employee_data;
```

### View File Metadata
```sql
SELECT FileName, LoadTime FROM employee_data;
```

### List Remaining Files in Stage
```sql
LIST @emp_stage;
```

---

## 🧼 Cleaning Up

If needed, manually remove files from the stage:
```sql
REMOVE @emp_stage PATTERN='.*';
```

Or truncate the target table:
```sql
TRUNCATE TABLE employee_data;
```

---

## 📌 Summary of Key Parameters

| Parameter | Description |
|----------|-------------|
| `VALIDATION_MODE` | Validates file content and returns all errors |
| `ON_ERROR` | Controls behavior on encountering errors (`ABORT`, `CONTINUE`, `SKIP_FILE`) |
| `FORCE` | Forces reloading of previously loaded files |
| `PURGE` | Deletes files after successful load |
| `MATCH_BY_COLUMN_NAME` | Maps columns by name instead of position |
| `INCLUDE` | Adds metadata fields like filename, row number, etc. |
| `FILES` / `PATTERN` | Specifies which files to load |

---

## ✅ Best Practices

1. **Always Validate First** – Use `VALIDATION_MODE` before full load.
2. **Use PURGE Carefully** – Ensure data is successfully loaded before deleting files.
3. **Match Columns by Name** – Avoid mismatches due to column order changes.
4. **Include Metadata** – Track ETL timestamps and file sources for auditing.
5. **Handle Errors Gracefully** – Use `ON_ERROR = CONTINUE` for partial success.

---

## 🎯 Conclusion

The `COPY INTO` command is a cornerstone of data ingestion in Snowflake. Whether you're loading structured or semi-structured data from internal or external stages, mastering its parameters and behaviors is crucial for building robust, scalable, and error-resilient data pipelines.
