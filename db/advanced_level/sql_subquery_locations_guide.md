# Where to Use Subqueries in SQL

## Overview

Subqueries can be placed in various clauses of a SQL statement. Each location serves different purposes and has specific use cases. This guide covers all the common places where subqueries can be used in SQL queries.

---

## 1. Subqueries in the SELECT Clause

### Purpose
Used to calculate values or retrieve related data for each row in the result set. These are also called **scalar subqueries** when they return a single value.

### Syntax
```sql
SELECT column1, 
       (subquery) AS alias
FROM table;
```

### Examples

**Basic Example:**
```sql
SELECT 
    employee_name,
    salary,
    (SELECT AVG(salary) FROM employees) AS company_avg_salary,
    salary - (SELECT AVG(salary) FROM employees) AS difference_from_avg
FROM employees;
```

**Correlated Example:**
```sql
SELECT 
    department_name,
    (SELECT COUNT(*) 
     FROM employees e 
     WHERE e.department_id = d.department_id) AS employee_count
FROM departments d;
```

### Use Cases
- Calculating aggregates for comparison
- Retrieving single related values
- Performing calculations based on other table data
- Adding computed columns to results

### Important Notes
- Subquery must return a single value (one row, one column)
- Can significantly impact performance if correlated
- Executes for each row in the outer query if correlated

---

## 2. Subqueries in the FROM Clause

### Purpose
Used to create derived tables or inline views that act as temporary result sets. Often called **derived tables** or **inline views**.

### Syntax
```sql
SELECT columns
FROM (subquery) AS alias
WHERE conditions;
```

### Examples

**Basic Example:**
```sql
SELECT 
    dept_summary.department_id,
    dept_summary.avg_salary,
    dept_summary.employee_count
FROM (
    SELECT 
        department_id,
        AVG(salary) AS avg_salary,
        COUNT(*) AS employee_count
    FROM employees
    GROUP BY department_id
) AS dept_summary
WHERE dept_summary.avg_salary > 50000;
```

**Complex Example with Multiple Subqueries:**
```sql
SELECT 
    high_earners.department_id,
    high_earners.emp_count AS high_earner_count,
    all_emps.total_count AS total_employees
FROM (
    SELECT department_id, COUNT(*) AS emp_count
    FROM employees
    WHERE salary > 75000
    GROUP BY department_id
) AS high_earners
JOIN (
    SELECT department_id, COUNT(*) AS total_count
    FROM employees
    GROUP BY department_id
) AS all_emps ON high_earners.department_id = all_emps.department_id;
```

### Use Cases
- Pre-aggregating data before joining
- Breaking complex queries into logical steps
- Filtering aggregated results
- Creating reusable intermediate result sets

### Important Notes
- Must have an alias
- Can significantly improve query readability
- Often more efficient than complex joins
- Can be used with other subqueries or tables

---

## 3. Subqueries in the WHERE Clause

### Purpose
Used to filter rows based on values from other queries. This is one of the most common uses of subqueries.

### Syntax Variations

**With Comparison Operators:**
```sql
WHERE column operator (subquery)
```

**With IN/NOT IN:**
```sql
WHERE column IN (subquery)
WHERE column NOT IN (subquery)
```

**With EXISTS/NOT EXISTS:**
```sql
WHERE EXISTS (subquery)
WHERE NOT EXISTS (subquery)
```

**With ANY/ALL:**
```sql
WHERE column operator ANY (subquery)
WHERE column operator ALL (subquery)
```

### Examples

**Simple Comparison:**
```sql
-- Find employees earning more than the average
SELECT employee_name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

**Using IN:**
```sql
-- Find employees in departments located in New York
SELECT employee_name, department_id
FROM employees
WHERE department_id IN (
    SELECT department_id 
    FROM departments 
    WHERE location = 'New York'
);
```

**Using NOT IN:**
```sql
-- Find departments with no employees
SELECT department_name
FROM departments
WHERE department_id NOT IN (
    SELECT DISTINCT department_id 
    FROM employees 
    WHERE department_id IS NOT NULL
);
```

**Using EXISTS:**
```sql
-- Find customers who have placed at least one order
SELECT customer_name
FROM customers c
WHERE EXISTS (
    SELECT 1 
    FROM orders o 
    WHERE o.customer_id = c.customer_id
);
```

**Using NOT EXISTS:**
```sql
-- Find customers who have never placed an order
SELECT customer_name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 
    FROM orders o 
    WHERE o.customer_id = c.customer_id
);
```

**Using ANY:**
```sql
-- Find employees earning more than ANY employee in department 5
SELECT employee_name, salary
FROM employees
WHERE salary > ANY (
    SELECT salary 
    FROM employees 
    WHERE department_id = 5
);
```

**Using ALL:**
```sql
-- Find employees earning more than ALL employees in department 5
SELECT employee_name, salary
FROM employees
WHERE salary > ALL (
    SELECT salary 
    FROM employees 
    WHERE department_id = 5
);
```

### Use Cases
- Filtering based on aggregate values
- Finding records that exist/don't exist in other tables
- Complex conditional filtering
- Comparing against multiple values

### Performance Tips
- EXISTS is often faster than IN for large datasets
- NOT EXISTS is typically faster than NOT IN
- Avoid NOT IN with columns that can be NULL
- Consider using JOINs for better performance in some cases

---

## 4. Subqueries in JOIN Clauses

### Purpose
Used to join against derived tables or pre-aggregated data.

### Syntax
```sql
SELECT columns
FROM table1
[INNER|LEFT|RIGHT|FULL] JOIN (subquery) AS alias
ON join_condition;
```

### Examples

**INNER JOIN with Subquery:**
```sql
SELECT 
    e.employee_name,
    e.salary,
    dept_avg.avg_salary
FROM employees e
INNER JOIN (
    SELECT department_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
) AS dept_avg ON e.department_id = dept_avg.department_id
WHERE e.salary > dept_avg.avg_salary;
```

**LEFT JOIN with Subquery:**
```sql
SELECT 
    d.department_name,
    COALESCE(emp_count.total, 0) AS employee_count
FROM departments d
LEFT JOIN (
    SELECT department_id, COUNT(*) AS total
    FROM employees
    GROUP BY department_id
) AS emp_count ON d.department_id = emp_count.department_id;
```

**Multiple Subqueries in JOINs:**
```sql
SELECT 
    c.customer_name,
    order_summary.order_count,
    payment_summary.total_paid
FROM customers c
LEFT JOIN (
    SELECT customer_id, COUNT(*) AS order_count
    FROM orders
    GROUP BY customer_id
) AS order_summary ON c.customer_id = order_summary.customer_id
LEFT JOIN (
    SELECT customer_id, SUM(amount) AS total_paid
    FROM payments
    GROUP BY customer_id
) AS payment_summary ON c.customer_id = payment_summary.customer_id;
```

### Use Cases
- Joining pre-aggregated data
- Improving performance by reducing data before joining
- Creating complex analytical queries
- Combining data from multiple aggregations

---

## 5. Subqueries in the HAVING Clause

### Purpose
Used to filter grouped results based on aggregate conditions involving other queries.

### Syntax
```sql
SELECT columns
FROM table
GROUP BY columns
HAVING aggregate_function operator (subquery);
```

### Examples

**Basic Example:**
```sql
-- Find departments with average salary above company average
SELECT department_id, AVG(salary) AS dept_avg
FROM employees
GROUP BY department_id
HAVING AVG(salary) > (
    SELECT AVG(salary) 
    FROM employees
);
```

**Complex Example:**
```sql
-- Find products with total sales above average category sales
SELECT 
    product_id, 
    SUM(quantity * price) AS total_sales
FROM order_items
GROUP BY product_id
HAVING SUM(quantity * price) > (
    SELECT AVG(category_total)
    FROM (
        SELECT SUM(quantity * price) AS category_total
        FROM order_items oi
        JOIN products p ON oi.product_id = p.product_id
        GROUP BY p.category_id
    ) AS category_averages
);
```

### Use Cases
- Filtering groups based on aggregate comparisons
- Finding groups that exceed certain thresholds
- Comparing group-level metrics

---

## 6. Subqueries in INSERT Statements

### Purpose
Used to insert data from one or more tables into another table.

### Syntax
```sql
INSERT INTO table (columns)
SELECT columns
FROM (subquery);
```

### Examples

**Basic INSERT with Subquery:**
```sql
INSERT INTO high_performers (employee_id, employee_name, salary)
SELECT employee_id, employee_name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

**INSERT with Complex Subquery:**
```sql
INSERT INTO monthly_sales_summary (month, total_sales, order_count)
SELECT 
    DATE_TRUNC('month', order_date) AS month,
    SUM(total_amount) AS total_sales,
    COUNT(*) AS order_count
FROM orders
WHERE order_date >= DATE_TRUNC('month', CURRENT_DATE - INTERVAL '3 months')
GROUP BY DATE_TRUNC('month', order_date);
```

### Use Cases
- Copying filtered data to another table
- Creating summary tables
- Archiving data
- Populating reporting tables

---

## 7. Subqueries in UPDATE Statements

### Purpose
Used to update records based on values from other queries.

### Syntax
```sql
UPDATE table
SET column = (subquery)
WHERE conditions;
```

### Examples

**Update with Scalar Subquery:**
```sql
-- Update employee bonuses to 10% of department average
UPDATE employees e
SET bonus = (
    SELECT AVG(salary) * 0.10
    FROM employees e2
    WHERE e2.department_id = e.department_id
);
```

**Update with WHERE Subquery:**
```sql
-- Increase salary for employees in high-performing departments
UPDATE employees
SET salary = salary * 1.10
WHERE department_id IN (
    SELECT department_id
    FROM department_performance
    WHERE performance_rating > 4.5
);
```

**Complex Update:**
```sql
-- Update product prices based on category averages
UPDATE products p
SET price = (
    SELECT AVG(price) * 1.05
    FROM products p2
    WHERE p2.category_id = p.category_id
)
WHERE category_id IN (
    SELECT category_id
    FROM categories
    WHERE needs_price_update = TRUE
);
```

### Use Cases
- Updating based on calculated values
- Bulk updates with complex conditions
- Synchronizing data between tables

---

## 8. Subqueries in DELETE Statements

### Purpose
Used to delete records based on conditions from other queries.

### Syntax
```sql
DELETE FROM table
WHERE column IN (subquery);
```

### Examples

**Basic DELETE:**
```sql
-- Delete orders older than the average order age
DELETE FROM orders
WHERE order_date < (
    SELECT DATE_TRUNC('day', AVG(order_date))
    FROM orders
);
```

**DELETE with NOT IN:**
```sql
-- Delete products that have never been ordered
DELETE FROM products
WHERE product_id NOT IN (
    SELECT DISTINCT product_id 
    FROM order_items
);
```

**DELETE with EXISTS:**
```sql
-- Delete inactive customers with no recent orders
DELETE FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
    AND o.order_date >= CURRENT_DATE - INTERVAL '2 years'
);
```

### Use Cases
- Removing orphaned records
- Cleaning up old data
- Deleting based on complex conditions

---

## 9. Subqueries in CASE Expressions

### Purpose
Used within CASE statements to perform conditional logic based on subquery results.

### Syntax
```sql
SELECT 
    CASE 
        WHEN column operator (subquery) THEN value
        ELSE other_value
    END AS result
FROM table;
```

### Examples

**Basic CASE with Subquery:**
```sql
SELECT 
    employee_name,
    salary,
    CASE 
        WHEN salary > (SELECT AVG(salary) FROM employees) 
        THEN 'Above Average'
        WHEN salary = (SELECT AVG(salary) FROM employees)
        THEN 'Average'
        ELSE 'Below Average'
    END AS salary_category
FROM employees;
```

**Complex CASE:**
```sql
SELECT 
    product_name,
    price,
    CASE 
        WHEN price > (
            SELECT MAX(price) * 0.8 
            FROM products p2 
            WHERE p2.category_id = p.category_id
        ) THEN 'Premium'
        WHEN price > (
            SELECT AVG(price) 
            FROM products p2 
            WHERE p2.category_id = p.category_id
        ) THEN 'Mid-Range'
        ELSE 'Budget'
    END AS price_tier
FROM products p;
```

### Use Cases
- Categorizing data based on comparisons
- Creating dynamic labels
- Conditional calculations

---

## 10. Common Table Expressions (CTEs) with Subqueries

### Purpose
CTEs provide a more readable alternative to subqueries, especially for complex queries.

### Syntax
```sql
WITH cte_name AS (
    subquery
)
SELECT columns
FROM cte_name;
```

### Examples

**Single CTE:**
```sql
WITH department_averages AS (
    SELECT 
        department_id,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
)
SELECT 
    e.employee_name,
    e.salary,
    da.avg_salary
FROM employees e
JOIN department_averages da ON e.department_id = da.department_id
WHERE e.salary > da.avg_salary;
```

**Multiple CTEs:**
```sql
WITH 
high_value_customers AS (
    SELECT customer_id, SUM(order_total) AS lifetime_value
    FROM orders
    GROUP BY customer_id
    HAVING SUM(order_total) > 10000
),
recent_orders AS (
    SELECT customer_id, COUNT(*) AS order_count
    FROM orders
    WHERE order_date >= CURRENT_DATE - INTERVAL '6 months'
    GROUP BY customer_id
)
SELECT 
    c.customer_name,
    hvc.lifetime_value,
    COALESCE(ro.order_count, 0) AS recent_order_count
FROM customers c
JOIN high_value_customers hvc ON c.customer_id = hvc.customer_id
LEFT JOIN recent_orders ro ON c.customer_id = ro.customer_id;
```

**Recursive CTE:**
```sql
WITH RECURSIVE employee_hierarchy AS (
    -- Base case: top-level managers
    SELECT employee_id, employee_name, manager_id, 1 AS level
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case: employees with managers
    SELECT e.employee_id, e.employee_name, e.manager_id, eh.level + 1
    FROM employees e
    JOIN employee_hierarchy eh ON e.manager_id = eh.employee_id
)
SELECT * FROM employee_hierarchy
ORDER BY level, employee_name;
```

### Advantages over Subqueries
- More readable and maintainable
- Can be referenced multiple times
- Supports recursive queries
- Better for complex multi-step logic

---

## Performance Considerations

### When to Use Subqueries
- Simple, one-time calculations
- Logical clarity is more important than performance
- Working with small datasets
- When the alternative would be more complex

### When to Consider Alternatives
- Large datasets where performance matters
- When the same subquery is used multiple times (use CTE instead)
- When a JOIN would be more efficient
- When window functions can replace correlated subqueries

### Optimization Tips
1. **Use EXISTS instead of IN** for better performance with large datasets
2. **Avoid correlated subqueries** in the SELECT clause when possible
3. **Consider JOINs** as alternatives for better performance
4. **Use CTEs** for complex queries with multiple subqueries
5. **Index appropriately** on columns used in subquery conditions
6. **Test and analyze** with EXPLAIN PLAN to understand query execution

---

## Summary Table

| Location | Type | Common Use Cases | Performance Notes |
|----------|------|-----------------|-------------------|
| SELECT | Scalar | Calculated columns, related values | Can be slow if correlated |
| FROM | Derived table | Pre-aggregation, complex joins | Generally efficient |
| WHERE | Filter | Conditional filtering, existence checks | EXISTS faster than IN for large data |
| JOIN | Derived table | Pre-aggregated joins | Often more efficient than WHERE subqueries |
| HAVING | Filter | Group-level filtering | Similar to WHERE subqueries |
| INSERT | Data source | Bulk inserts from queries | Efficient for large operations |
| UPDATE | Value/Filter | Calculated updates, conditional updates | Can be slow with correlated subqueries |
| DELETE | Filter | Conditional deletion | Similar performance to WHERE subqueries |
| CASE | Conditional | Dynamic categorization | Performance depends on complexity |
| CTE | Named subquery | Complex multi-step queries | More readable, similar performance |

---

## Best Practices

1. **Name subqueries clearly** with meaningful aliases
2. **Keep subqueries simple** - break complex logic into CTEs
3. **Avoid nesting too deeply** - more than 2-3 levels becomes hard to read
4. **Consider readability** - sometimes a slightly less efficient query that's easier to understand is better
5. **Test performance** - always verify with your actual data and database
6. **Use appropriate indexes** on columns referenced in subqueries
7. **Prefer EXISTS over IN** when checking for existence
8. **Use CTEs** for complex queries that use the same subquery multiple times

---

## Conclusion

Subqueries are powerful tools that can be used in many parts of SQL statements. Understanding where and when to use them effectively will help you write more efficient and maintainable SQL code. Always consider readability, performance, and maintainability when choosing between subqueries, JOINs, and CTEs.