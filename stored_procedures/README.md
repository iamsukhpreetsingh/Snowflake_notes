# Snowflake Stored Procedures - Comprehensive SQL Guide

Core Components Covered:

- Basic procedure structure and syntax
- Parameters and data types
- Variable declaration and assignment
- Control flow (IF-THEN-ELSE, CASE statements)
- Loops (FOR, WHILE, LOOP)
- Cursors for row-by-row processing
- Exception handling with TRY-CATCH
- Dynamic SQL execution
- Result sets and table returns
- Transaction management
- Procedure calling syntax


## 1. Basic Stored Procedure Structure

### CREATE OR REPLACE PROCEDURE
Creates or replaces a stored procedure in Snowflake.

**Syntax:**
```sql
CREATE OR REPLACE PROCEDURE procedure_name(parameter_list)
RETURNS return_type
LANGUAGE SQL
AS
$$
DECLARE
    -- Variable declarations
BEGIN
    -- Procedure body
    RETURN return_value;
END;
$$;
```

**Example:**
```sql
CREATE OR REPLACE PROCEDURE get_employee_count()
RETURNS INTEGER
LANGUAGE SQL
AS
$$
DECLARE
    emp_count INTEGER;
BEGIN
    SELECT COUNT(*) INTO emp_count FROM employees;
    RETURN emp_count;
END;
$$;
```

## 2. Parameters and Data Types

### Input Parameters
Define parameters that the procedure accepts.

**Syntax:**
```sql
CREATE OR REPLACE PROCEDURE proc_name(
    param1 DATA_TYPE,
    param2 DATA_TYPE DEFAULT default_value
)
```

**Example:**
```sql
CREATE OR REPLACE PROCEDURE update_salary(
    employee_id INTEGER,
    new_salary DECIMAL(10,2),
    department VARCHAR(50) DEFAULT 'GENERAL'
)
RETURNS STRING
LANGUAGE SQL
AS
$$
BEGIN
    UPDATE employees 
    SET salary = new_salary, dept = department 
    WHERE id = employee_id;
    RETURN 'Salary updated successfully';
END;
$$;
```

### Common Data Types
- `INTEGER`, `NUMBER`, `DECIMAL(p,s)`
- `VARCHAR(n)`, `STRING`, `TEXT`
- `DATE`, `TIMESTAMP`, `TIME`
- `BOOLEAN`
- `VARIANT`, `OBJECT`, `ARRAY`

## 3. Variable Declaration and Assignment

### DECLARE Section
Declare variables before the BEGIN block.

**Syntax:**
```sql
DECLARE
    variable_name DATA_TYPE;
    variable_name2 DATA_TYPE := initial_value;
```

**Example:**
```sql
CREATE OR REPLACE PROCEDURE calculate_bonus()
RETURNS DECIMAL(10,2)
LANGUAGE SQL
AS
$$
DECLARE
    total_sales DECIMAL(10,2);
    bonus_rate DECIMAL(3,2) := 0.05;
    calculated_bonus DECIMAL(10,2);
BEGIN
    SELECT SUM(sales_amount) INTO total_sales FROM sales;
    calculated_bonus := total_sales * bonus_rate;
    RETURN calculated_bonus;
END;
$$;
```

### Variable Assignment
- `:=` operator for direct assignment
- `INTO` clause with SELECT statements

## 4. Control Flow Statements

### IF-THEN-ELSE
Conditional execution of code blocks.

**Syntax:**
```sql
IF condition THEN
    -- statements
ELSEIF condition THEN
    -- statements
ELSE
    -- statements
END IF;
```

**Example:**
```sql
CREATE OR REPLACE PROCEDURE categorize_employee(emp_id INTEGER)
RETURNS STRING
LANGUAGE SQL
AS
$$
DECLARE
    emp_salary DECIMAL(10,2);
    category STRING;
BEGIN
    SELECT salary INTO emp_salary FROM employees WHERE id = emp_id;
    
    IF emp_salary > 100000 THEN
        category := 'Senior';
    ELSEIF emp_salary > 50000 THEN
        category := 'Mid-level';
    ELSE
        category := 'Junior';
    END IF;
    
    RETURN category;
END;
$$;
```

### CASE Statement
Multi-way conditional logic.

**Syntax:**
```sql
CASE
    WHEN condition1 THEN result1
    WHEN condition2 THEN result2
    ELSE default_result
END CASE;
```

**Example:**
```sql
DECLARE
    grade CHAR(1) := 'B';
    description STRING;
BEGIN
    CASE grade
        WHEN 'A' THEN description := 'Excellent';
        WHEN 'B' THEN description := 'Good';
        WHEN 'C' THEN description := 'Average';
        ELSE description := 'Below Average';
    END CASE;
END;
```

## 5. Loops

### FOR Loop
Iterate over a range of values.

**Syntax:**
```sql
FOR counter IN start TO end DO
    -- statements
END FOR;
```

**Example:**
```sql
CREATE OR REPLACE PROCEDURE generate_sequence(n INTEGER)
RETURNS STRING
LANGUAGE SQL
AS
$$
DECLARE
    result STRING := '';
    i INTEGER;
BEGIN
    FOR i IN 1 TO n DO
        result := result || i || ',';
    END FOR;
    RETURN TRIM(result, ',');
END;
$$;
```

### WHILE Loop
Execute statements while a condition is true.

**Syntax:**
```sql
WHILE condition DO
    -- statements
END WHILE;
```

**Example:**
```sql
DECLARE
    counter INTEGER := 1;
    sum_value INTEGER := 0;
BEGIN
    WHILE counter <= 10 DO
        sum_value := sum_value + counter;
        counter := counter + 1;
    END WHILE;
    RETURN sum_value;
END;
```

### LOOP with EXIT
Infinite loop with explicit exit condition.

**Syntax:**
```sql
LOOP
    -- statements
    EXIT WHEN condition;
END LOOP;
```

## 6. Cursors

### Cursor Declaration and Usage
Handle result sets row by row.

**Syntax:**
```sql
DECLARE
    cursor_name CURSOR FOR select_statement;
BEGIN
    OPEN cursor_name;
    FETCH cursor_name INTO variables;
    CLOSE cursor_name;
END;
```

**Example:**
```sql
CREATE OR REPLACE PROCEDURE process_employees()
RETURNS STRING
LANGUAGE SQL
AS
$$
DECLARE
    emp_cursor CURSOR FOR SELECT id, name, salary FROM employees;
    emp_id INTEGER;
    emp_name STRING;
    emp_salary DECIMAL(10,2);
    processed_count INTEGER := 0;
BEGIN
    OPEN emp_cursor;
    
    LOOP
        FETCH emp_cursor INTO emp_id, emp_name, emp_salary;
        EXIT WHEN SQLNOTFOUND;
        
        -- Process each employee
        IF emp_salary < 50000 THEN
            UPDATE employees SET salary = salary * 1.1 WHERE id = emp_id;
        END IF;
        
        processed_count := processed_count + 1;
    END LOOP;
    
    CLOSE emp_cursor;
    RETURN 'Processed ' || processed_count || ' employees';
END;
$$;
```

## 7. Exception Handling

### TRY-CATCH Blocks
Handle runtime errors gracefully.

**Syntax:**
```sql
BEGIN
    -- code that might throw exception
EXCEPTION
    WHEN exception_type THEN
        -- error handling code
END;
```

**Example:**
```sql
CREATE OR REPLACE PROCEDURE safe_divide(a DECIMAL, b DECIMAL)
RETURNS DECIMAL
LANGUAGE SQL
AS
$$
DECLARE
    result DECIMAL;
BEGIN
    BEGIN
        result := a / b;
    EXCEPTION
        WHEN DIVISION_BY_ZERO THEN
            result := 0;
            -- Log error or handle as needed
    END;
    RETURN result;
END;
$$;
```

## 8. Dynamic SQL

### EXECUTE IMMEDIATE
Execute dynamically constructed SQL statements.

**Syntax:**
```sql
EXECUTE IMMEDIATE sql_string;
EXECUTE IMMEDIATE sql_string INTO variable;
```

**Example:**
```sql
CREATE OR REPLACE PROCEDURE dynamic_table_query(table_name STRING, column_name STRING)
RETURNS INTEGER
LANGUAGE SQL
AS
$$
DECLARE
    sql_stmt STRING;
    record_count INTEGER;
BEGIN
    sql_stmt := 'SELECT COUNT(*) FROM ' || table_name || 
                ' WHERE ' || column_name || ' IS NOT NULL';
    
    EXECUTE IMMEDIATE sql_stmt INTO record_count;
    
    RETURN record_count;
END;
$$;
```

## 9. Result Sets

### Returning Result Sets
Return query results as a result set.

**Syntax:**
```sql
CREATE OR REPLACE PROCEDURE proc_name()
RETURNS TABLE(column_list)
LANGUAGE SQL
AS
$$
DECLARE
    res RESULTSET;
BEGIN
    res := (SELECT columns FROM table WHERE conditions);
    RETURN TABLE(res);
END;
$$;
```

**Example:**
```sql
CREATE OR REPLACE PROCEDURE get_high_salary_employees(min_salary DECIMAL)
RETURNS TABLE(id INTEGER, name STRING, salary DECIMAL)
LANGUAGE SQL
AS
$$
DECLARE
    res RESULTSET;
BEGIN
    res := (SELECT id, name, salary 
            FROM employees 
            WHERE salary >= min_salary
            ORDER BY salary DESC);
    RETURN TABLE(res);
END;
$$;
```

## 10. Transaction Management

### Transaction Control
Manage transactions within stored procedures.

**Example:**
```sql
CREATE OR REPLACE PROCEDURE transfer_funds(
    from_account INTEGER,
    to_account INTEGER,
    amount DECIMAL(10,2)
)
RETURNS STRING
LANGUAGE SQL
AS
$$
DECLARE
    from_balance DECIMAL(10,2);
BEGIN
    -- Start transaction (implicit)
    
    -- Check balance
    SELECT balance INTO from_balance 
    FROM accounts 
    WHERE account_id = from_account;
    
    IF from_balance < amount THEN
        RETURN 'Insufficient funds';
    END IF;
    
    -- Perform transfer
    UPDATE accounts 
    SET balance = balance - amount 
    WHERE account_id = from_account;
    
    UPDATE accounts 
    SET balance = balance + amount 
    WHERE account_id = to_account;
    
    -- Transaction commits automatically on successful completion
    RETURN 'Transfer completed successfully';
    
EXCEPTION
    WHEN OTHERS THEN
        -- Transaction rolls back automatically on error
        RETURN 'Transfer failed: ' || SQLERRM;
END;
$$;
```

## 11. Calling Stored Procedures

### CALL Statement
Execute stored procedures.

**Syntax:**
```sql
CALL procedure_name(parameters);
```

**Examples:**
```sql
-- Simple procedure call
CALL get_employee_count();

-- Procedure with parameters
CALL update_salary(123, 75000.00, 'ENGINEERING');

-- Capturing return value
SELECT * FROM TABLE(get_high_salary_employees(80000));
```

## 12. Best Practices

### Error Handling
Always implement proper error handling for robust procedures.

**Example:**
```sql
CREATE OR REPLACE PROCEDURE robust_procedure()
RETURNS STRING
LANGUAGE SQL
AS
$$
DECLARE
    error_msg STRING;
BEGIN
    -- Main logic here
    
    RETURN 'Success';
    
EXCEPTION
    WHEN OTHERS THEN
        error_msg := 'Error in robust_procedure: ' || SQLERRM;
        -- Log error to error table
        INSERT INTO error_log (procedure_name, error_message, error_time)
        VALUES ('robust_procedure', error_msg, CURRENT_TIMESTAMP());
        
        RETURN error_msg;
END;
$$;
```

### Performance Tips
- Use appropriate data types
- Minimize dynamic SQL when possible
- Use bulk operations instead of row-by-row processing
- Consider using temporary tables for complex operations

### Security Considerations
- Validate input parameters
- Use parameterized queries to prevent SQL injection
- Implement proper access controls
- Audit sensitive operations

## 13. Common Built-in Functions

### String Functions
- `UPPER()`, `LOWER()`, `TRIM()`
- `SUBSTRING()`, `LENGTH()`, `CONCAT()`
- `REPLACE()`, `SPLIT_PART()`

### Date Functions
- `CURRENT_DATE()`, `CURRENT_TIMESTAMP()`
- `DATEADD()`, `DATEDIFF()`
- `EXTRACT()`, `DATE_TRUNC()`

### Conversion Functions
- `TO_NUMBER()`, `TO_DATE()`, `TO_TIMESTAMP()`
- `CAST()`, `TRY_CAST()`

### System Functions
- `SQLCODE`, `SQLERRM` (error information)
- `SQLNOTFOUND` (cursor status)
- `USER`, `CURRENT_ROLE()`

