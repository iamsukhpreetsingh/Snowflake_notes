# Snowflake Dynamic Data Masking: A Complete Guide with Real-World Examples

In today’s data-driven world, **protecting sensitive information**—like credit card numbers, SSNs, Aadhaar cards, or phone numbers—is not optional. It’s a **non-negotiable requirement** for compliance (GDPR, HIPAA, PCI-DSS) and ethical data handling.

Snowflake’s **Dynamic Data Masking (DDM)** is one of its most powerful security features. It allows you to **hide or obfuscate sensitive data at query time**—based on the **user’s role**—without altering the underlying data.

> 🔐 **Key Benefit**:  
> The same table can show **full data to admins**, **partially masked data to customer support**, and **completely hidden data to third parties**—all in real time.

This guide walks you through **everything you need to know**—from theory to hands-on implementation—with real-world analogies, SQL examples, and role-based access control.

---

## 🔍 What Is Dynamic Data Masking?

**Dynamic Data Masking (DDM)** is a **column-level security policy** that:
- **Does NOT modify stored data**.
- **Applies masking rules at query runtime**.
- Uses **Snowflake roles** to determine **who sees what**.

### Real-World Analogies
1. **E-commerce Delivery**:  
   When you order from Amazon/Flipkart, the delivery person **never sees your real phone number**. They call a **masked virtual number** instead.

2. **Bank Customer Care**:  
   When you call your bank, the agent asks for the **last 4 digits** of your card—not the full 16-digit number. Why? Because their role **doesn’t grant access** to full PII.

> ✅ **Snowflake DDM replicates this behavior in your data warehouse**.

---

## 🧱 Core Concepts

### 1. **Masking Policy**
A reusable SQL function that defines **how to mask data** based on conditions (usually `CURRENT_ROLE()`).

### 2. **Roles**
- `ACCOUNTADMIN`: Sees **unmasked data**.
- `CCP` (Customer Care Professional): Sees **partially masked data**.
- `SUPPLIER`: Sees **fully masked data** (e.g., `$$$$`).
- Others: See **custom messages** (e.g., `"You cannot see me"`).

### 3. **PII Columns**
Columns containing **Personally Identifiable Information**:
- Credit card numbers
- PAN/Aadhaar/SSN
- Phone numbers
- Email addresses

---

## 🛠️ Step-by-Step Implementation

### Step 1: Create a Sample Table
```sql
-- Create a customer table with sensitive data
CREATE OR REPLACE TABLE credit_card_customer (
    id INT,
    first_name STRING,
    last_name STRING,
    credit_card STRING,
    pan_card STRING,
    city STRING,
    country STRING
);

-- Insert sample data
INSERT INTO credit_card_customer VALUES
(1, 'Rahul', 'Sharma', '4532123456781234', 'ABCDE1234F', 'Mumbai', 'India'),
(2, 'Priya', 'Patel', '5412345678901234', 'FGHIJ5678K', 'Delhi', 'India');
```

---

### Step 2: Create Masking Policies

#### Policy 1: `pi_masking` (Multi-Role Masking)
```sql
CREATE OR REPLACE MASKING POLICY pi_masking AS (val STRING)
RETURNS STRING ->
  CASE
    WHEN CURRENT_ROLE() IN ('ACCOUNTADMIN') THEN val
    WHEN CURRENT_ROLE() IN ('CCP') THEN 
      RPAD('########', LENGTH(val) - 4, '#') || RIGHT(val, 4)
    WHEN CURRENT_ROLE() IN ('SUPPLIER') THEN '$$$$'
    ELSE 'You cannot see me'
  END;
```

> 💡 **Explanation**:
> - **Admins**: See full value (`4532123456781234`)
> - **CCP**: See last 4 digits (`##########1234`)
> - **Supplier**: See `$$$$`
> - **Others**: See `"You cannot see me"`

#### Policy 2: `pii_masking` (Alternative Format)
```sql
CREATE OR REPLACE MASKING POLICY pii_masking AS (val STRING)
RETURNS STRING ->
  CASE
    WHEN CURRENT_ROLE() = 'ACCOUNTADMIN' THEN val
    WHEN CURRENT_ROLE() = 'CCP' THEN '****' || RIGHT(val, 4)
    ELSE 'SENSITIVE DATA HIDDEN'
  END;
```

---

### Step 3: Apply Masking Policies to Columns

#### Option A: Apply During Table Creation
```sql
CREATE OR REPLACE TABLE credit_card_customer (
    id INT,
    first_name STRING,
    last_name STRING,
    credit_card STRING MASKING POLICY pi_masking,
    pan_card STRING MASKING POLICY pii_masking,
    city STRING,
    country STRING
);
```

#### Option B: Apply to Existing Table
```sql
-- Apply to credit_card column
ALTER TABLE credit_card_customer 
MODIFY COLUMN credit_card SET MASKING POLICY pi_masking;

-- Apply to pan_card column
ALTER TABLE credit_card_customer 
MODIFY COLUMN pan_card SET MASKING POLICY pii_masking;
```

> ✅ Verify with:
> ```sql
> DESCRIBE TABLE credit_card_customer;
> ```
> You’ll see `masking policy` listed under each protected column.

---

### Step 4: Create Roles and Users

#### Create Roles
```sql
-- Customer Care Role
CREATE ROLE ccp;
GRANT ROLE ccp TO USER customer_care_agent;

-- Supplier Role
CREATE ROLE supplier;
GRANT ROLE supplier TO USER delivery_agent;
```

#### Grant Necessary Privileges
```sql
-- Grant database/schema access
GRANT USAGE ON DATABASE my_db TO ROLE ccp;
GRANT USAGE ON SCHEMA my_db.public TO ROLE ccp;

-- Grant table access
GRANT SELECT ON TABLE credit_card_customer TO ROLE ccp;
GRANT SELECT ON TABLE credit_card_customer TO ROLE supplier;

-- Grant warehouse access
GRANT USAGE ON WAREHOUSE compute_wh TO ROLE ccp;
GRANT USAGE ON WAREHOUSE compute_wh TO ROLE supplier;
```

> ⚠️ **Critical**: Without these grants, users **won’t see the table at all**.

---

### Step 5: Test Role-Based Masking

#### As `ACCOUNTADMIN` (Full Access)
```sql
SELECT * FROM credit_card_customer;
```
**Output**:
```
ID | FIRST_NAME | CREDIT_CARD       | PAN_CARD
1  | Rahul      | 4532123456781234  | ABCDE1234F
```

#### As `CCP` (Partial Mask)
```sql
-- Switch role in Snowflake UI or via SQL
USE ROLE ccp;
SELECT * FROM credit_card_customer;
```
**Output**:
```
ID | FIRST_NAME | CREDIT_CARD       | PAN_CARD
1  | Rahul      | ##########1234    | ****1234F
```

#### As `SUPPLIER` (Full Mask)
```sql
USE ROLE supplier;
SELECT * FROM credit_card_customer;
```
**Output**:
```
ID | FIRST_NAME | CREDIT_CARD | PAN_CARD
1  | Rahul      | $$$$        | SENSITIVE DATA HIDDEN
```

#### As `SYSADMIN` (No Access Defined)
```sql
USE ROLE sysadmin;
SELECT * FROM credit_card_customer;
```
**Output**:
```
ID | FIRST_NAME | CREDIT_CARD       | PAN_CARD
1  | Rahul      | You cannot see me | SENSITIVE DATA HIDDEN
```

---

## 🔒 Advanced Security: Filtering Is Also Masked

Even if a user **knows the exact sensitive value**, they **cannot use it in filters**:

```sql
-- As CCP role
SELECT * FROM credit_card_customer 
WHERE credit_card = '4532123456781234';
```
✅ **Result**: **0 rows returned**  
> ❗ Snowflake **masks the column before applying the filter**, so the query sees `##########1234`, not the real value.

---

## 🧹 How to Remove (Unset) a Masking Policy

```sql
-- Remove from credit_card column
ALTER TABLE credit_card_customer 
MODIFY COLUMN credit_card UNSET MASKING POLICY;

-- Remove from pan_card column
ALTER TABLE credit_card_customer 
MODIFY COLUMN pan_card UNSET MASKING POLICY;

-- Drop the policy (only after unsetting from all columns)
DROP MASKING POLICY pi_masking;
DROP MASKING POLICY pii_masking;
```

> ⚠️ **Note**: You must have **`OWNERSHIP`** on the table to unset policies.

---

## 📊 Governance & Monitoring

Snowflake provides built-in governance dashboards:

1. Go to **Governance → Data Policies**
2. View:
   - **Most used masking policies**
   - **Columns with masking policies**
   - **Tables with row access policies**

> 🕒 **Note**: Dashboard updates may take a few minutes to reflect changes.

---

## 🎯 Best Practices

1. **Principle of Least Privilege**:  
   Only grant `ACCOUNTADMIN` to trusted users.

2. **Use Meaningful Role Names**:  
   `FINANCE_ANALYST`, `HR_MANAGER`, not `ROLE1`, `ROLE2`.

3. **Combine with Row Access Policies**:  
   For **row + column-level security** (e.g., HR sees only their region’s employees).

4. **Avoid Hardcoding Role Names**:  
   Use **role hierarchies** (e.g., `GRANT ROLE ccp TO ROLE support_team`).

5. **Test Thoroughly**:  
   Always validate masking behavior across all roles.

---

## ❓ Common Interview Question

> **Q**: *If a user knows the real value of a masked column, can they filter by it?*  
> **A**: **No**. Snowflake applies masking **before** query execution, so filters see the **masked value**, not the real one.

---

## ✅ Conclusion

Snowflake’s **Dynamic Data Masking** gives you **granular, role-based control** over sensitive data—without duplicating tables or writing complex application logic.

By implementing DDM:
- **Compliance** becomes effortless.
- **Data democratization** is safe.
- **User experience** remains seamless.

> 🔜 **Next Step**: Explore **Tag-Based Masking Policies** (automatically apply masking based on column tags like `PII`, `FINANCIAL`).

With this guide, you’re ready to **secure your data like a pro**—whether you’re preparing for interviews or building production systems.
