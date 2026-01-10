# IS NULL and IS NOT NULL in SQL

IS NULL and IS NOT NULL are SQL operators used to test whether a value is NULL or not. They are essential for handling missing or undefined data in databases.

## Understanding NULL

### What is NULL?

NULL represents the absence of a value or an unknown value. It is NOT:
- An empty string ('')
- Zero (0)
- False
- A space (' ')

NULL means "no value" or "value unknown".

### Why NULL Matters

NULL values are common in databases and occur when:
- Data hasn't been entered yet
- Data is not applicable
- Data is unknown or missing
- A calculation results in an undefined value
- An outer join finds no matching record

## Basic Syntax

### IS NULL
Tests if a value is NULL:
```sql
SELECT column1, column2
FROM table_name
WHERE column1 IS NULL;
```

### IS NOT NULL
Tests if a value is NOT NULL (has a value):
```sql
SELECT column1, column2
FROM table_name
WHERE column1 IS NOT NULL;
```

## Why You Cannot Use = NULL

### Common Mistake
```sql
-- WRONG: This will NOT work as expected
SELECT * FROM customers WHERE email = NULL;

-- WRONG: This will also NOT work
SELECT * FROM customers WHERE email != NULL;
SELECT * FROM customers WHERE email <> NULL;
```

### Why It Doesn't Work

NULL represents an unknown value. You cannot compare an unknown value to anything using standard comparison operators. The result of any comparison with NULL is UNKNOWN (not TRUE or FALSE).

```sql
-- All of these return UNKNOWN (treated as FALSE in WHERE clauses)
NULL = NULL    -- UNKNOWN
NULL <> NULL   -- UNKNOWN
NULL > 5       -- UNKNOWN
5 = NULL       -- UNKNOWN
```

### Correct Approach
```sql
-- CORRECT: Use IS NULL
SELECT * FROM customers WHERE email IS NULL;

-- CORRECT: Use IS NOT NULL
SELECT * FROM customers WHERE email IS NOT NULL;
```

## Basic Examples

### Finding Missing Data
```sql
-- Find customers without email addresses
SELECT customer_id, customer_name, email
FROM customers
WHERE email IS NULL;
```

### Finding Complete Records
```sql
-- Find customers with email addresses
SELECT customer_id, customer_name, email
FROM customers
WHERE email IS NOT NULL;
```

### Multiple NULL Checks
```sql
-- Find customers missing either phone or email
SELECT customer_id, customer_name
FROM customers
WHERE phone IS NULL OR email IS NULL;

-- Find customers with both phone and email
SELECT customer_id, customer_name
FROM customers
WHERE phone IS NOT NULL AND email IS NOT NULL;
```

## Use Cases

### 1. Data Quality Checks

**Identifying Incomplete Records**
```sql
-- Find orders missing critical information
SELECT order_id, customer_id, order_date, shipping_address
FROM orders
WHERE shipping_address IS NULL
   OR order_date IS NULL;
```

**Data Completeness Report**
```sql
-- Count complete vs incomplete customer records
SELECT 
    COUNT(*) AS total_customers,
    COUNT(email) AS customers_with_email,
    SUM(CASE WHEN email IS NULL THEN 1 ELSE 0 END) AS customers_without_email,
    SUM(CASE WHEN phone IS NULL THEN 1 ELSE 0 END) AS customers_without_phone
FROM customers;
```

### 2. Optional Fields Handling

**Filtering Optional Data**
```sql
-- Find products with optional descriptions provided
SELECT product_id, product_name, description
FROM products
WHERE description IS NOT NULL;
```

**Setting Defaults for Display**
```sql
-- Show description or 'No description' if NULL
SELECT 
    product_id,
    product_name,
    COALESCE(description, 'No description available') AS product_description
FROM products;
```

### 3. Join Operations

**Finding Unmatched Records (LEFT JOIN)**
```sql
-- Find customers who have never placed an order
SELECT c.customer_id, c.customer_name
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;
```

**Finding Matched Records Only**
```sql
-- Find customers who have placed at least one order
SELECT DISTINCT c.customer_id, c.customer_name
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NOT NULL;
```

### 4. Data Cleaning

**Finding Records to Clean**
```sql
-- Find employees with missing contact information
SELECT 
    employee_id,
    employee_name,
    CASE WHEN email IS NULL THEN 'Missing email' ELSE 'OK' END AS email_status,
    CASE WHEN phone IS NULL THEN 'Missing phone' ELSE 'OK' END AS phone_status
FROM employees
WHERE email IS NULL OR phone IS NULL;
```

**Updating NULL Values**
```sql
-- Set default values for NULL fields
UPDATE products
SET category = 'Uncategorized'
WHERE category IS NULL;
```

### 5. Business Logic Implementation

**Conditional Processing**
```sql
-- Find orders requiring follow-up (missing tracking number)
SELECT 
    order_id,
    customer_name,
    order_date,
    DATEDIFF(day, order_date, GETDATE()) AS days_since_order
FROM orders
WHERE tracking_number IS NULL
  AND order_status = 'Shipped';
```

**Priority Assignment**
```sql
-- Assign priority based on missing information
SELECT 
    customer_id,
    customer_name,
    CASE 
        WHEN email IS NULL AND phone IS NULL THEN 'High Priority - No Contact Info'
        WHEN email IS NULL THEN 'Medium Priority - No Email'
        WHEN phone IS NULL THEN 'Low Priority - No Phone'
        ELSE 'Complete'
    END AS data_quality_priority
FROM customers;
```

### 6. Reporting and Analytics

**Excluding Incomplete Data**
```sql
-- Calculate average salary only for employees with salary data
SELECT 
    department,
    AVG(salary) AS avg_salary,
    COUNT(*) AS total_employees,
    COUNT(salary) AS employees_with_salary
FROM employees
WHERE salary IS NOT NULL
GROUP BY department;
```

**Including All Data with Flags**
```sql
-- Include all records but flag incomplete ones
SELECT 
    product_id,
    product_name,
    price,
    CASE 
        WHEN price IS NULL THEN 'Price Missing'
        ELSE 'Price Available'
    END AS price_status
FROM products;
```

### 7. Temporal Data Handling

**Finding Active Records**
```sql
-- Find currently active employees (no end date)
SELECT employee_id, employee_name, start_date
FROM employees
WHERE end_date IS NULL;
```

**Finding Completed Records**
```sql
-- Find completed projects
SELECT project_id, project_name, start_date, end_date
FROM projects
WHERE end_date IS NOT NULL;
```

**Finding Pending Items**
```sql
-- Find pending approvals (approval date not set)
SELECT 
    request_id,
    requester_name,
    request_date,
    DATEDIFF(day, request_date, GETDATE()) AS days_pending
FROM approval_requests
WHERE approval_date IS NULL
ORDER BY request_date;
```

## Advanced Patterns

### Combining with Other Conditions

**Complex Filtering**
```sql
-- Find high-value orders with missing shipping info
SELECT order_id, total_amount, shipping_address
FROM orders
WHERE total_amount > 1000
  AND shipping_address IS NULL
  AND order_status = 'Pending';
```

### NULL in Subqueries

**Finding Orphaned Records**
```sql
-- Find products not in any category
SELECT p.product_id, p.product_name
FROM products p
WHERE p.category_id IS NULL
   OR p.category_id NOT IN (SELECT category_id FROM categories);
```

### NULL in Aggregations

**Counting NULL Values**
```sql
-- Count records with and without specific fields
SELECT 
    COUNT(*) AS total_records,
    COUNT(column_name) AS non_null_count,
    COUNT(*) - COUNT(column_name) AS null_count,
    ROUND(COUNT(column_name) * 100.0 / COUNT(*), 2) AS completion_percentage
FROM table_name;
```

## NULL Behavior in Calculations

### Arithmetic Operations
```sql
-- NULL in calculations results in NULL
SELECT 
    quantity,
    price,
    quantity * price AS total  -- NULL if either is NULL
FROM order_items;

-- Safe calculation with NULL handling
SELECT 
    quantity,
    price,
    COALESCE(quantity, 0) * COALESCE(price, 0) AS total
FROM order_items;
```

### String Concatenation
```sql
-- NULL in concatenation (behavior varies by database)
SELECT 
    first_name,
    middle_name,
    last_name,
    first_name + ' ' + middle_name + ' ' + last_name AS full_name  -- NULL if any part is NULL (SQL Server)
FROM employees;

-- Safe concatenation
SELECT 
    first_name,
    middle_name,
    last_name,
    CONCAT(first_name, ' ', COALESCE(middle_name + ' ', ''), last_name) AS full_name
FROM employees;
```

## Performance Considerations

### Indexing and NULL

**NULL Values and Indexes**
- Most databases can index NULL values
- Some databases (older Oracle versions) don't index NULL values
- IS NULL and IS NOT NULL can use indexes

```sql
-- This can use an index on email column
SELECT * FROM customers WHERE email IS NULL;
```

**Optimizing NULL Checks**
```sql
-- Consider adding computed columns or filtered indexes
CREATE INDEX idx_customers_missing_email 
ON customers(customer_id) 
WHERE email IS NULL;
```

## Common Patterns and Idioms

### The "Not Exists" Pattern
```sql
-- Find customers with no orders using NULL check
SELECT c.*
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;

-- Alternative using NOT EXISTS
SELECT c.*
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);
```

### The "Coalesce Default" Pattern
```sql
-- Provide defaults for NULL values
SELECT 
    employee_name,
    COALESCE(bonus, 0) AS bonus,
    salary + COALESCE(bonus, 0) AS total_compensation
FROM employees;
```

### The "NULL-Safe Comparison" Pattern
```sql
-- Compare values that might be NULL
SELECT *
FROM table1 t1
JOIN table2 t2 ON 
    (t1.column1 = t2.column1 OR (t1.column1 IS NULL AND t2.column1 IS NULL));
```

## Best Practices

1. **Always use IS NULL and IS NOT NULL** - Never use = NULL or != NULL

2. **Consider NOT NULL constraints** - Define columns as NOT NULL in schema when appropriate

3. **Provide default values** - Use DEFAULT constraints or COALESCE in queries

4. **Document NULL semantics** - Clearly document what NULL means in each context

5. **Be consistent** - Establish team conventions for handling NULL values

6. **Test NULL scenarios** - Always test queries with NULL values in test data

7. **Handle NULLs in calculations** - Use COALESCE or CASE to avoid NULL propagation

8. **Use ISNULL/COALESCE wisely** - Choose between performance and portability

## Common Mistakes to Avoid

```sql
-- WRONG: Using equality operators
WHERE column = NULL

-- CORRECT: Using IS NULL
WHERE column IS NULL

-- WRONG: Forgetting NULL in aggregates
COUNT(*)  -- Counts all rows including NULLs
COUNT(column)  -- Counts only non-NULL values

-- WRONG: Not handling NULL in ORDER BY
ORDER BY column  -- NULL values sort first or last (database-dependent)

-- CORRECT: Explicit NULL handling
ORDER BY COALESCE(column, 'ZZZ')
```

## Summary

- Use **IS NULL** to find missing or unknown values
- Use **IS NOT NULL** to find records with values
- Never use `= NULL` or `!= NULL` - they don't work
- NULL affects calculations, concatenations, and comparisons
- Handle NULLs explicitly in your queries using COALESCE or CASE
- Understand that COUNT(*) and COUNT(column) behave differently with NULLs
- Use NULL checks for data quality, joins, and business logic

NULL handling is fundamental to SQL development. Proper use of IS NULL and IS NOT NULL ensures robust, reliable queries.