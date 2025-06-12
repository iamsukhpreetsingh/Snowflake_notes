# Snowflake Snowpipe: Automated Continuous Data Ingestion from Cloud Storage

## 🚀 What is Snowpipe?

**Snowpipe** is a near real-time data ingestion service that continuously loads data from files stored in external cloud storage into Snowflake tables. It uses **event-driven architecture**, meaning as soon as a new file arrives in your bucket, Snowflake automatically processes and loads the data.

### Key Features of Snowpipe:
- **Automated ingestion**: No need to manually run `COPY INTO` commands.
- **Serverless**: Fully managed by Snowflake; no infrastructure to maintain.
- **Event-based**: Triggers on file upload events (e.g., S3 object creation).
- **Scalable**: Handles thousands of files per second without performance degradation.
- **Fault-tolerant**: Automatically retries failed loads and supports backpressure.

---

## 🧩 How Does Snowpipe Work?

Here’s a high-level flow of how Snowpipe operates:

1. A file is uploaded to an **external stage** (e.g., S3 bucket).
2. The cloud provider (AWS, GCP, etc.) sends an **event notification** to Snowflake via SQS, SNS, or Event Grid.
3. Snowflake triggers the **Snowpipe pipeline**.
4. Snowflake reads the file from the external stage.
5. Data is transformed and loaded into the target Snowflake table.
6. Metadata about the load is recorded for monitoring and auditing.

---

## 🔧 Prerequisites for Using Snowpipe

Before setting up Snowpipe, ensure you have:
- A **Snowflake account** with appropriate permissions.
- An **external stage** configured (e.g., pointing to an S3 bucket).
- A **file format** defined (CSV, Parquet, JSON, etc.).
- A **target table** already created in Snowflake.
- **Storage integration** set up between Snowflake and your cloud provider (e.g., AWS S3).

> 💡 *Note*: If you haven’t already configured external stages or storage integrations, refer to Part 14 of this series for detailed setup instructions.

---

## 🛠️ Step-by-Step Guide to Configuring Snowpipe with AWS S3

Let’s walk through setting up Snowpipe using **Amazon S3** as the source.

### 1. **Upload Files to S3 Bucket**
Ensure your data files are uploaded to your S3 bucket. For example:
- File name: `customer_data.csv`
- Location: `s3://my-snowflake-bucket/customer_data.csv`

### 2. **Create Target Table in Snowflake**
Make sure the table exists and matches the structure of your incoming data.

```sql
CREATE OR REPLACE TABLE AWS_CUSTOMER_LOAD (
    CUSTOMER_ID INT,
    NAME STRING,
    PHONE STRING,
    ADDRESS STRING,
    COMPANY_NAME STRING,
    RUN_BY STRING
);
```

### 3. **Define File Format**
Create a file format for parsing your CSV/JSON/Parquet files.

```sql
CREATE OR REPLACE FILE FORMAT CSV_TYPE
TYPE = CSV
SKIP_HEADER = 1;
```

### 4. **Create External Stage (if not already done)**

```sql
CREATE OR REPLACE STAGE MY_S3_STAGE
URL = 's3://my-snowflake-bucket/'
CREDENTIALS = (AWS_KEY_ID='...' AWS_SECRET_KEY='...')
FILE_FORMAT = CSV_TYPE;
```

### 5. **Create Snowpipe**

Now create the Snowpipe object that will monitor the stage and load data automatically.

```sql
CREATE PIPE AWS_PIPE
AUTO_INGEST = TRUE
AS
COPY INTO AWS_CUSTOMER_LOAD
FROM @MY_S3_STAGE
FILE_FORMAT = CSV_TYPE;
```

This command creates a pipe named `AWS_PIPE` that listens for new files in the specified S3 bucket and copies them into the `AWS_CUSTOMER_LOAD` table.

---

## ⚙️ Configure Event Notification in AWS

To enable automatic triggering, you must configure **event notifications** in your AWS S3 bucket.

### Steps:
1. Go to your **S3 bucket** in the AWS Console.
2. Navigate to **Properties > Events > Create Event Notification**.
3. Set the following:
   - **Event name**: e.g., `snowpipe-alert`
   - **Events**: Select `All object create events`
   - **Destination**: Choose **SQS queue**
   - **Queue ARN**: Use the SQS queue provided by Snowflake when creating the pipe.

You can retrieve the SQS ARN from Snowflake using:

```sql
SHOW PIPES;
```

Look for the `NOTIFICATION_CHANNEL` column — this contains the SQS ARN needed for configuration.

4. Save the event notification.

Once configured, any new file uploaded to the S3 bucket will trigger Snowflake to process and load the file automatically.

---

## 🔍 Monitoring Snowpipe Performance

After setting up Snowpipe, you should regularly monitor its status and performance.

### 1. **Check Pipe Status**
Use the `SYSTEM$PIPE_STATUS` function to check if Snowpipe is running and has pending files.

```sql
SELECT SYSTEM$PIPE_STATUS('AWS_PIPE');
```

Example output:
```json
{
  "executionState": "RUNNING",
  "pendingFileCount": 0,
  "numOutstandingMessagesOnChannel": 1
}
```

### 2. **Query Copy History**
Use the `COPY_HISTORY` view to see recent load operations.

```sql
SELECT *
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'AWS_CUSTOMER_LOAD',
    START_TIME => DATEADD(HOURS, -1, CURRENT_TIMESTAMP())
));
```

This gives details like:
- File name
- Load start/end time
- Number of rows processed
- Load status (Loaded, Failed)

### 3. **View Pipe Usage History**
Monitor credit usage for Snowpipe processing.

```sql
SELECT *
FROM TABLE(INFORMATION_SCHEMA.PIPE_USAGE_HISTORY(
    DATE_RANGE_START => DATEADD(DAYS, -7, CURRENT_TIMESTAMP()),
    PIPE_NAME => 'AWS_PIPE'
));
```

This helps track cost and optimize resource usage.

---

## 🛡️ Error Handling and Troubleshooting

Even though Snowpipe is highly reliable, issues may occur due to malformed data, schema mismatches, or network failures.

### 1. **Check for Errors in Copy History**
Review the `ERROR_MESSAGE` field in the `COPY_HISTORY` view.

```sql
SELECT ERROR_MESSAGE
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(...));
```

### 2. **Set Up Error Notifications**
Configure alerts to notify users when a load fails:
- Use **Snowflake Alerts** to send email/SMS notifications.
- Integrate with **Snowflake Actions** and **Third-party Tools** (e.g., Slack, PagerDuty).

### 3. **Manually Retry Failed Loads**
If a file fails, you can re-upload it to the stage and Snowpipe will automatically retry the load.

Alternatively, use `COPY INTO` to manually reload a specific file:

```sql
COPY INTO AWS_CUSTOMER_LOAD
FROM @MY_S3_STAGE/customer_data.csv
FILE_FORMAT = CSV_TYPE;
```

---

## 💰 Understanding Snowpipe Credit Consumption

Snowpipe runs as a **serverless compute task**, and Snowflake charges based on **credits consumed** during data loading.

### Key Points:
- **Credit usage** depends on:
  - File size
  - Frequency of uploads
  - Complexity of transformations
- You can **monitor credit usage** using the `PIPE_USAGE_HISTORY` view.
- Snowflake provides **free tier** credits for Snowpipe under certain conditions.

---

## ✅ Best Practices for Using Snowpipe

1. **Use Proper File Formats** – Ensure files match the expected schema and format.
2. **Enable Compression** – Compress large files to reduce load times and costs.
3. **Partition Files** – Break large datasets into smaller files for faster ingestion.
4. **Avoid Duplicate Uploads** – Snowflake tracks processed files for up to 64 days; avoid uploading identical files unless necessary.
5. **Monitor Regularly** – Check copy history and error logs to ensure smooth operation.
6. **Optimize Cost** – Use `PURGE = TRUE` in `COPY INTO` or configure lifecycle policies in S3 to delete old files.

---

## 🎯 Real-World Use Cases

- **Log Analytics**: Continuously ingest application logs from S3 into Snowflake for analysis.
- **IoT Data Ingestion**: Stream sensor data from edge devices into a centralized data lake and warehouse.
- **Data Lakehouse Architecture**: Build a hybrid architecture where raw data resides in S3 but is queried and transformed in Snowflake.
- **Real-Time Dashboards**: Power live dashboards with continuously updated data from cloud storage.
- **ETL Automation**: Eliminate manual scripting by automating ingestion pipelines.

---

## 🧪 Example Scenario: Loading Customer Data from S3

### Step 1: Upload Updated File to S3
You update `customer_data.csv` with new records and upload it to your S3 bucket.

### Step 2: Snowflake Detects New File
The S3 event notification triggers Snowflake to process the new file.

### Step 3: Data is Loaded Automatically
Snowflake parses the file using the defined format and inserts new records into the `AWS_CUSTOMER_LOAD` table.

### Step 4: Monitor Load in Snowflake
You check the `COPY_HISTORY` view and confirm that the file was successfully loaded.

---

## 📌 Summary of Key Commands

| Command | Description |
|--------|-------------|
| `CREATE PIPE` | Creates a new Snowpipe object |
| `SHOW PIPES` | Lists all existing pipes |
| `SYSTEM$PIPE_STATUS` | Checks current status of a pipe |
| `COPY_HISTORY` | View historical copy operations |
| `PIPE_USAGE_HISTORY` | Track credit usage by Snowpipe |

---

## 🎯 Conclusion

Snowpipe revolutionizes how data is ingested into Snowflake by enabling **automated, continuous, and scalable data loading** from cloud storage. Whether you're building a real-time analytics dashboard, implementing a data lakehouse architecture, or streamlining ETL workflows, Snowpipe offers a robust, serverless solution that integrates seamlessly with your cloud ecosystem.

By understanding how to configure Snowpipe, monitor performance, and handle errors, you can build resilient and efficient data pipelines that keep your analytics up-to-date with minimal effort.

CREDITS: https://www.youtube.com/watch?v=TPmtS-MDcsc&list=PL__gObEGy1Y7klsW7vc2TM2Cmt6BwRkzh&index=21
