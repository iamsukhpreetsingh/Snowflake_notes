# Snowflake CHANGES Clause vs. Streams: A Practical Guide
 
Snowflake offers two powerful ways to track data changes over time: **Streams** and the newer **CHANGES clause**. While both serve similar purposes, they differ significantly in flexibility, use cases, and ease of use.
 
In this article, we’ll break down everything you need to know about the **CHANGES clause**, compare it with **Streams**, and show you **real-world examples** so you can decide which one fits your needs.
 
---
 
## 🤔 What Problem Are We Solving?
 
Before Snowflake introduced the `CHANGES` clause, the only way to capture data modifications (INSERTs, UPDATEs, DELETEs) was using **Streams**. But Streams come with limitations:
 
1. **Single consumer**: Once you read from a stream, the offset moves forward—other users can’t read the same changes.

2. **No time-based filtering**: You can’t easily say “show me all changes between 2 PM and 3 PM.”

3. **Rigid structure**: If you initially create an *append-only* stream but later need to capture *updates/deletes*, you must recreate it.

4. **Partial consumption is destructive**: If you query only some records (e.g., `WHERE id = 8`), the *entire stream* is consumed and cleared.
 
Enter the **CHANGES clause**—a **declarative**, **non-destructive**, and **time-aware** alternative.
 
---
 
## 🔍 What Is the CHANGES Clause?
 
The `CHANGES` clause lets you **query historical changes** on a table **without using a Stream**. It reads from Snowflake’s internal **change tracking metadata**, which logs every DML operation (INSERT/UPDATE/DELETE) within the table’s time travel window.
 
> ✅ **Key idea**: `CHANGES` is like a “read-only stream” that never gets consumed and supports time ranges, filtering, and multiple users.
 
---
 
## 🛠️ Prerequisites: Enable Change Tracking
 
To use `CHANGES`, your table must have **change tracking enabled**.
 
### Option 1: Create a Stream (automatically enables change tracking)

```sql

CREATE STREAM netflix_stream ON TABLE netflix;

```
 
### Option 2: Explicitly enable change tracking (no stream needed)

```sql

ALTER TABLE customer SET CHANGE_TRACKING = TRUE;

```
 
✅ Verify it’s on:

```sql

SHOW TABLES LIKE 'netflix';

-- Look for "change_tracking = ON"

```
 
> 💡 Note: Change tracking is **off by default**. You must enable it *before* changes occur to capture them.
 
---
 
## 📜 Basic Syntax of CHANGES
 
```sql

SELECT * FROM TABLE(

  CHANGES(INFORMATION => { 'DEFAULT' | 'APPEND_ONLY' },

          AT => { TIMESTAMP => <timestamp> | STREAM => '<stream_name>' },

          END => { TIMESTAMP => <timestamp> } -- optional

         )

) AS c

WHERE ...;

```
 
### Parameters Explained:
 
| Parameter | Description |

|--------|-------------|

| `INFORMATION => 'DEFAULT'` | Captures **INSERT, UPDATE, DELETE** (shows before/after via `METADATA$ACTION`) |

| `INFORMATION => 'APPEND_ONLY'` | Captures **INSERTs only** (like an append-only stream) |

| `AT => TIMESTAMP => ...` | Start time for change capture |

| `END => TIMESTAMP => ...` | End time (optional) — defines a **time window** |

| `AT => STREAM => 'my_stream'` | Use a stream’s offset as the starting point |
 
---
 
## 🧪 Practical Examples
 
### Sample Table Setup

```sql

CREATE OR REPLACE TABLE netflix (

  show_id INT,

  title STRING

);
 
INSERT INTO netflix VALUES (1, 'Harry Potter'), (8, 'Iron Man');
 
-- Enable change tracking

ALTER TABLE netflix SET CHANGE_TRACKING = TRUE;

```
 
### Simulate Changes

```sql

-- Update show_id

UPDATE netflix SET show_id = 2 WHERE show_id = 1;
 
-- Fix title casing

UPDATE netflix SET title = 'Iron Man 2' WHERE show_id = 8;

```
 
Now, 4 change records exist internally (2 per UPDATE: one DELETE of old row, one INSERT of new row).
 
---
 
## ✅ Use Case 1: Multiple Users, Independent Reads
 
**Problem**:  

Vicki wants changes where `show_id = 8`. Raj wants all changes. Neither should affect the other.
 
### ❌ With Streams (problematic):

```sql

-- Vicki runs:

CREATE TABLE vicki_changes AS 

SELECT * FROM netflix_stream WHERE show_id = 8;
 
-- Now Raj runs:

SELECT * FROM netflix_stream; -- Returns EMPTY! Stream is consumed.

```
 
### ✅ With CHANGES (ideal):

```sql

-- Vicki:

CREATE OR REPLACE TABLE vicki_changes AS

SELECT * FROM TABLE(

  CHANGES(INFORMATION => 'DEFAULT', AT => STREAM => 'netflix_stream')

)

WHERE show_id = 8;
 
-- Raj (at the same time or later):

CREATE OR REPLACE TABLE raj_changes AS

SELECT * FROM TABLE(

  CHANGES(INFORMATION => 'DEFAULT', AT => STREAM => 'netflix_stream')

);

```
 
✅ **Both get full data**. No interference. Stream remains untouched.
 
> 🔁 The `netflix_stream` still contains all 4 records because `CHANGES` **does not consume** the stream—it only uses its offset as a reference point.
 
---
 
## ✅ Use Case 2: Time-Bound Change Capture
 
**Requirement**:  

“Show me all changes between 10:00 PM and 10:30 PM.”
 
### With CHANGES:

```sql

-- Set time variables

SET start_time = '2025-10-27 22:00:00'::TIMESTAMP;

SET end_time = '2025-10-27 22:30:00'::TIMESTAMP;
 
-- Capture changes in that window

CREATE OR REPLACE TABLE changes_30min AS

SELECT * FROM TABLE(

  CHANGES(

    INFORMATION => 'DEFAULT',

    AT => TIMESTAMP => $start_time,

    END => TIMESTAMP => $end_time

  )

);

```
 
✅ Only changes within that 30-minute window are returned—**no stream needed**.
 
> 💡 This is **impossible with standard streams**, which only track from last consumption point.
 
---
 
## ✅ Use Case 3: Flexible Change Types (No Stream Recreation)
 
**Scenario**:  

Initially, you thought you only needed INSERTs. Later, you realize you need UPDATES/DELETES too.
 
### ❌ With Streams:

- You must **drop and recreate** the stream as `DEFAULT` (not `APPEND_ONLY`).

- Lose historical change context.
 
### ✅ With CHANGES:

Just change the query:
 
```sql

-- Originally (inserts only):

SELECT * FROM TABLE(CHANGES(INFORMATION => 'APPEND_ONLY', ...));
 
-- Later (full changes):

SELECT * FROM TABLE(CHANGES(INFORMATION => 'DEFAULT', ...));

```
 
✅ Same table. Same history. No reconfiguration.
 
---
 
## ⚖️ CHANGES vs. Streams: When to Use Which?
 
| Feature | **CHANGES Clause** | **Streams** |

|-------|------------------|-----------|

| **Multiple consumers** | ✅ Yes (non-destructive) | ❌ No (consumes offset) |

| **Time-range queries** | ✅ Yes (`AT` + `END`) | ❌ No |

| **Partial reads** | ✅ Safe (no side effects) | ❌ Consumes entire stream |

| **Automated pipelines** | ❌ Manual querying | ✅ Perfect for Tasks |

| **Low-latency ETL** | ⚠️ Polling required | ✅ Native integration |

| **Storage overhead** | ❌ None (reads metadata) | ✅ Stores offset state |
 
### 🎯 Recommendation:
 
- Use **CHANGES** when you need:

  - Ad-hoc auditing

  - Time-bound analysis

  - Multiple teams querying the same changes

  - Flexibility in change types
 
- Use **Streams** when you need:

  - Automated, continuous data pipelines (e.g., with Tasks)

  - Guaranteed exactly-once processing

  - Integration into ELT workflows
 
> 💡 Pro Tip: You can **combine both**! Use a stream to define a logical starting point, then use `CHANGES(... AT => STREAM => 'my_stream')` for flexible querying.
 
---
 
## 🔒 Important Notes
 
1. **Retention**: Changes are only available within the table’s **Time Travel retention period** (1–90 days, depending on edition).

2. **Performance**: `CHANGES` queries may be slower on large tables with many changes.

3. **Cost**: No extra storage cost—uses existing Time Travel metadata.

4. **Views**: You can also use `CHANGES` on **views** (if base tables have change tracking enabled).
 
---
 
## 🧩 Real-World Analogy
 
Think of it like this:
 
- **Stream** = A **conveyor belt** in a factory. Once an item passes, it’s gone. Only one worker can take it.

- **CHANGES** = A **security camera recording**. Anyone can rewind, pause, or watch a specific time window—without affecting others.
 
---
 
## ✅ Summary
 
The **CHANGES clause** is a **game-changer** for:

- Auditing

- Debugging

- Multi-user analytics

- Time-bound investigations
 
It doesn’t replace Streams—but **complements** them by adding **flexibility** where Streams are rigid.
 
> 🚀 **Best Practice**: Enable `CHANGE_TRACKING = TRUE` on critical tables *proactively*. You’ll thank yourself later when you need to audit “what changed and when?”
 
Now you can confidently choose the right tool for your data change tracking needs!
 
