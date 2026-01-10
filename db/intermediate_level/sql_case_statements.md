# CASE Statements in SQL

CASE statements are SQL's way of implementing conditional logic, similar to IF-THEN-ELSE statements in programming languages. They allow you to return different values based on conditions, making them essential for data transformation, categorization, and complex business logic.

## Overview

CASE statements evaluate conditions in order and return a value when the first condition is met. Once a condition is true, CASE stops evaluating and returns the result. If no conditions are met, it returns the value in the ELSE clause (or NULL if no ELSE is specified).

## Two Forms of CASE Statements

### 1. Simple CASE Statement

Compares an expression to a set of simple values.

**Syntax:**
```sql
CASE expression
    WHEN value1 THEN result1
    WHEN value2 THEN result2
    WHEN value3 THEN result3
    ELSE default_result
END
```

**Example:**
```sql
SELECT 
    product_name,
    category_id,
    CASE category_id
        WHEN 1 THEN 'Electronics'
        WHEN 2 THEN 'Clothing'
        WHEN 3 THEN 'Books'
        ELSE 'Other'
    END AS category_name
FROM products;
```

### 2. Searched CASE Statement

Evaluates a set of Boolean expressions (more flexible and commonly used).

**Syntax:**
```sql
CASE
    WHEN condition1 THEN result1
    WHEN condition2 THEN result2
    WHEN condition3 THEN result3
    ELSE default_result
END
```

**Example:**
```sql
SELECT 
    product_name,
    price,
    CASE
        WHEN price < 10 THEN 'Budget'
        WHEN price >= 10 AND price < 50 THEN 'Standard'
        WHEN price >= 50 AND price < 100 THEN 'Premium'
        ELSE 'Luxury'
    END AS price_category
FROM products;
```

## Basic Examples

### Single Condition
```sql
SELECT 
    employee_name,
    salary,
    CASE
        WHEN salary > 100000 THEN 'High Earner'
        ELSE 'Standard'
    END AS salary_tier
FROM employees;
```

### Multiple Conditions
```sql
SELECT 
    student_name,
    score,
    CASE
        WHEN score >= 90 THEN 'A'
        WHEN score >= 80 THEN 'B'
        WHEN score >= 70 THEN 'C'
        WHEN score >= 60 THEN 'D'
        ELSE 'F'
    END AS grade
FROM student_scores;
```

### With NULL Handling
```sql
SELECT 
    customer_name,
    email,
    CASE
        WHEN email IS NULL THEN 'No Email Provided'
        WHEN email LIKE '%@gmail.com' THEN 'Gmail User'
        WHEN email LIKE '%@yahoo.com' THEN 'Yahoo User'
        ELSE 'Other Email Provider'
    END AS email_type
FROM customers;
```

## Use Cases

### 1. Data Categorization

**Age Groups**
```sql
SELECT 
    customer_name,
    age,
    CASE
        WHEN age < 18 THEN 'Minor'
        WHEN age BETWEEN 18 AND 25 THEN 'Young Adult'
        WHEN age BETWEEN 26 AND 40 THEN 'Adult'
        WHEN age BETWEEN 41 AND 60 THEN 'Middle Aged'
        ELSE 'Senior'
    END AS age_group
FROM customers;
```

**Sales Performance Tiers**
```sql
SELECT 
    salesperson_name,
    total_sales,
    CASE
        WHEN total_sales >= 1000000 THEN 'Diamond'
        WHEN total_sales >= 500000 THEN 'Platinum'
        WHEN total_sales >= 250000 THEN 'Gold'
        WHEN total_sales >= 100000 THEN 'Silver'
        ELSE 'Bronze'
    END AS performance_tier
FROM sales_performance;
```

**Product Classification**
```sql
SELECT 
    product_name,
    stock_quantity,
    CASE
        WHEN stock_quantity = 0 THEN 'Out of Stock'
        WHEN stock_quantity <= 10 THEN 'Low Stock'
        WHEN stock_quantity <= 50 THEN 'Adequate Stock'
        ELSE 'Well Stocked'
    END AS stock_status
FROM inventory;
```

### 2. Conditional Aggregation

**Pivot-Style Reporting**
```sql
SELECT 
    department,
    COUNT(*) AS total_employees,
    SUM(CASE WHEN gender = 'M' THEN 1 ELSE 0 END) AS male_count,
    SUM(CASE WHEN gender = 'F' THEN 1 ELSE 0 END) AS female_count,
    SUM(CASE WHEN salary > 80000 THEN 1 ELSE 0 END) AS high_earners
FROM employees
GROUP BY department;
```

**Conditional Sums**
```sql
SELECT 
    product_category,
    SUM(CASE WHEN sale_type = 'Online' THEN sale_amount ELSE 0 END) AS online_sales,
    SUM(CASE WHEN sale_type = 'In-Store' THEN sale_amount ELSE 0 END) AS instore_sales,
    SUM(CASE WHEN sale_type = 'Phone' THEN sale_amount ELSE 0 END) AS phone_sales,
    SUM(sale_amount) AS total_sales
FROM sales
GROUP BY product_category;
```

**Conditional Averages**
```sql
SELECT 
    department,
    AVG(CASE WHEN employment_type = 'Full-Time' THEN salary END) AS avg_fulltime_salary,
    AVG(CASE WHEN employment_type = 'Part-Time' THEN salary END) AS avg_parttime_salary,
    AVG(CASE WHEN employment_type = 'Contract' THEN salary END) AS avg_contract_salary
FROM employees
GROUP BY department;
```

### 3. Custom Sorting

**Prioritized Ordering**
```sql
SELECT 
    task_name,
    priority,
    due_date
FROM tasks
ORDER BY 
    CASE priority
        WHEN 'Critical' THEN 1
        WHEN 'High' THEN 2
        WHEN 'Medium' THEN 3
        WHEN 'Low' THEN 4
        ELSE 5
    END,
    due_date;
```

**Custom Business Logic Sorting**
```sql
SELECT 
    order_id,
    order_status,
    order_date
FROM orders
ORDER BY 
    CASE order_status
        WHEN 'Urgent' THEN 1
        WHEN 'Pending' THEN 2
        WHEN 'Processing' THEN 3
        WHEN 'Completed' THEN 4
        WHEN 'Cancelled' THEN 5
    END,
    order_date DESC;
```

**NULL Handling in Sorting**
```sql
SELECT 
    employee_name,
    bonus
FROM employees
ORDER BY 
    CASE 
        WHEN bonus IS NULL THEN 1
        ELSE 0
    END,
    bonus DESC;
```

### 4. Conditional Updates

**Dynamic Price Adjustments**
```sql
UPDATE products
SET price = CASE
    WHEN category = 'Electronics' THEN price * 1.10
    WHEN category = 'Clothing' THEN price * 1.05
    WHEN category = 'Books' THEN price * 1.03
    ELSE price
END;
```

**Status Updates Based on Multiple Conditions**
```sql
UPDATE orders
SET order_status = CASE
    WHEN shipped_date IS NOT NULL AND delivered_date IS NULL THEN 'In Transit'
    WHEN delivered_date IS NOT NULL THEN 'Delivered'
    WHEN payment_confirmed = 1 AND shipped_date IS NULL THEN 'Processing'
    ELSE 'Pending'
END;
```

**Conditional Field Updates**
```sql
UPDATE employees
SET 
    salary = CASE
        WHEN performance_rating >= 4.5 THEN salary * 1.15
        WHEN performance_rating >= 3.5 THEN salary * 1.10
        WHEN performance_rating >= 2.5 THEN salary * 1.05
        ELSE salary
    END,
    bonus = CASE
        WHEN performance_rating >= 4.5 THEN salary * 0.20
        WHEN performance_rating >= 3.5 THEN salary * 0.10
        ELSE 0
    END
WHERE review_year = 2024;
```

### 5. Data Validation and Quality Checks

**Identifying Data Issues**
```sql
SELECT 
    customer_id,
    customer_name,
    email,
    phone,
    CASE
        WHEN email IS NULL AND phone IS NULL THEN 'Critical: No Contact Info'
        WHEN email IS NULL THEN 'Warning: No Email'
        WHEN phone IS NULL THEN 'Warning: No Phone'
        WHEN email NOT LIKE '%@%' THEN 'Error: Invalid Email Format'
        ELSE 'Valid'
    END AS data_quality_status
FROM customers;
```

**Flagging Anomalies**
```sql
SELECT 
    order_id,
    order_date,
    ship_date,
    total_amount,
    CASE
        WHEN ship_date < order_date THEN 'Error: Ship before Order'
        WHEN total_amount <= 0 THEN 'Error: Invalid Amount'
        WHEN total_amount > 10000 THEN 'Alert: High Value Order'
        WHEN DATEDIFF(day, order_date, ship_date) > 30 THEN 'Warning: Long Processing Time'
        ELSE 'Normal'
    END AS order_flag
FROM orders;
```

### 6. Complex Business Logic

**Commission Calculation**
```sql
SELECT 
    salesperson_name,
    total_sales,
    CASE
        WHEN total_sales >= 1000000 THEN total_sales * 0.10
        WHEN total_sales >= 500000 THEN total_sales * 0.08
        WHEN total_sales >= 250000 THEN total_sales * 0.06
        WHEN total_sales >= 100000 THEN total_sales * 0.04
        ELSE total_sales * 0.02
    END AS commission,
    CASE
        WHEN total_sales >= 1000000 THEN 5000
        WHEN total_sales >= 500000 THEN 3000
        WHEN total_sales >= 250000 THEN 1500
        ELSE 0
    END AS bonus
FROM sales_summary;
```

**Shipping Cost Calculation**
```sql
SELECT 
    order_id,
    weight,
    destination_country,
    CASE
        WHEN destination_country = 'USA' AND weight <= 1 THEN 5.00
        WHEN destination_country = 'USA' AND weight <= 5 THEN 10.00
        WHEN destination_country = 'USA' THEN 15.00
        WHEN destination_country IN ('Canada', 'Mexico') AND weight <= 1 THEN 8.00
        WHEN destination_country IN ('Canada', 'Mexico') AND weight <= 5 THEN 15.00
        WHEN destination_country IN ('Canada', 'Mexico') THEN 25.00
        WHEN weight <= 1 THEN 15.00
        WHEN weight <= 5 THEN 30.00
        ELSE 50.00
    END AS shipping_cost
FROM orders;
```

**Customer Segmentation**
```sql
SELECT 
    customer_id,
    customer_name,
    total_purchases,
    years_as_customer,
    average_order_value,
    CASE
        WHEN total_purchases >= 50 AND average_order_value >= 200 THEN 'VIP'
        WHEN total_purchases >= 20 AND years_as_customer >= 2 THEN 'Loyal'
        WHEN total_purchases >= 10 OR average_order_value >= 150 THEN 'Regular'
        WHEN total_purchases >= 1 THEN 'Occasional'
        ELSE 'New'
    END AS customer_segment
FROM customer_metrics;
```

### 7. Dynamic Column Selection

**Conditional Value Display**
```sql
SELECT 
    employee_name,
    department,
    CASE
        WHEN department = 'Sales' THEN commission
        WHEN department = 'Management' THEN bonus
        ELSE overtime_pay
    END AS variable_compensation
FROM employees;
```

**Context-Dependent Information**
```sql
SELECT 
    product_name,
    product_type,
    CASE product_type
        WHEN 'Digital' THEN download_link
        WHEN 'Physical' THEN warehouse_location
        WHEN 'Service' THEN service_schedule
        ELSE 'N/A'
    END AS fulfillment_info
FROM products;
```

### 8. Reporting and Analytics

**Year-over-Year Comparison**
```sql
SELECT 
    product_id,
    product_name,
    current_year_sales,
    previous_year_sales,
    CASE
        WHEN previous_year_sales = 0 THEN 'New Product'
        WHEN current_year_sales > previous_year_sales * 1.2 THEN 'Strong Growth'
        WHEN current_year_sales > previous_year_sales THEN 'Moderate Growth'
        WHEN current_year_sales = previous_year_sales THEN 'Flat'
        WHEN current_year_sales > previous_year_sales * 0.8 THEN 'Slight Decline'
        ELSE 'Significant Decline'
    END AS performance_trend
FROM product_sales_comparison;
```

**KPI Status Indicators**
```sql
SELECT 
    department,
    actual_revenue,
    target_revenue,
    CASE
        WHEN actual_revenue >= target_revenue * 1.1 THEN '🟢 Exceeding Target'
        WHEN actual_revenue >= target_revenue THEN '🟢 Meeting Target'
        WHEN actual_revenue >= target_revenue * 0.9 THEN '🟡 Near Target'
        ELSE '🔴 Below Target'
    END AS performance_status
FROM department_performance;
```

### 9. Date and Time Logic

**Business Days Calculation**
```sql
SELECT 
    task_id,
    task_name,
    due_date,
    CASE DATEPART(WEEKDAY, due_date)
        WHEN 1 THEN 'Sunday - Weekend'
        WHEN 7 THEN 'Saturday - Weekend'
        ELSE 'Weekday'
    END AS day_type
FROM tasks;
```

**Seasonal Categorization**
```sql
SELECT 
    order_date,
    order_amount,
    CASE
        WHEN MONTH(order_date) IN (12, 1, 2) THEN 'Winter'
        WHEN MONTH(order_date) IN (3, 4, 5) THEN 'Spring'
        WHEN MONTH(order_date) IN (6, 7, 8) THEN 'Summer'
        ELSE 'Fall'
    END AS season
FROM orders;
```

**Time-Based Pricing**
```sql
SELECT 
    service_id,
    service_name,
    base_price,
    CASE
        WHEN DATEPART(HOUR, service_time) BETWEEN 6 AND 9 THEN base_price * 1.5  -- Morning rush
        WHEN DATEPART(HOUR, service_time) BETWEEN 17 AND 20 THEN base_price * 1.5  -- Evening rush
        WHEN DATEPART(WEEKDAY, service_time) IN (1, 7) THEN base_price * 1.25  -- Weekend
        ELSE base_price
    END AS final_price
FROM services;
```

### 10. Nested CASE Statements

**Complex Multi-Level Logic**
```sql
SELECT 
    employee_name,
    department,
    years_of_service,
    performance_rating,
    CASE department
        WHEN 'Sales' THEN
            CASE
                WHEN performance_rating >= 4.5 AND years_of_service >= 5 THEN 'Senior Sales Executive'
                WHEN performance_rating >= 4.0 THEN 'Sales Executive'
                WHEN performance_rating >= 3.0 THEN 'Sales Representative'
                ELSE 'Junior Sales Representative'
            END
        WHEN 'Engineering' THEN
            CASE
                WHEN years_of_service >= 10 THEN 'Principal Engineer'
                WHEN years_of_service >= 5 THEN 'Senior Engineer'
                WHEN years_of_service >= 2 THEN 'Engineer'
                ELSE 'Junior Engineer'
            END
        ELSE 'Staff'
    END AS job_title
FROM employees;
```

## Advanced Patterns

### CASE in JOIN Conditions

**Dynamic Joining**
```sql
SELECT 
    o.order_id,
    o.customer_type,
    o.customer_id,
    CASE o.customer_type
        WHEN 'Individual' THEN i.customer_name
        WHEN 'Business' THEN b.company_name
    END AS customer_name
FROM orders o
LEFT JOIN individual_customers i ON 
    o.customer_type = 'Individual' AND o.customer_id = i.customer_id
LEFT JOIN business_customers b ON 
    o.customer_type = 'Business' AND o.customer_id = b.customer_id;
```

### CASE in WHERE Clause

**Conditional Filtering**
```sql
SELECT 
    product_id,
    product_name,
    price,
    category
FROM products
WHERE 
    CASE 
        WHEN category = 'Electronics' THEN price <= 1000
        WHEN category = 'Clothing' THEN price <= 200
        ELSE price <= 500
    END = 1;  -- True condition
```

### CASE with Subqueries

**Dynamic Threshold Comparison**
```sql
SELECT 
    employee_name,
    salary,
    department,
    CASE
        WHEN salary > (SELECT AVG(salary) * 1.5 FROM employees WHERE department = e.department) 
            THEN 'Well Above Average'
        WHEN salary > (SELECT AVG(salary) FROM employees WHERE department = e.department) 
            THEN 'Above Average'
        WHEN salary >= (SELECT AVG(salary) * 0.8 FROM employees WHERE department = e.department) 
            THEN 'Average'
        ELSE 'Below Average'
    END AS salary_position
FROM employees e;
```

## Performance Considerations

### Order of Conditions Matters
```sql
-- GOOD: Most common condition first
CASE
    WHEN status = 'Active' THEN 'A'  -- 80% of records
    WHEN status = 'Pending' THEN 'P'  -- 15% of records
    WHEN status = 'Inactive' THEN 'I'  -- 5% of records
END

-- LESS EFFICIENT: Rare condition first
CASE
    WHEN status = 'Inactive' THEN 'I'  -- 5% of records
    WHEN status = 'Pending' THEN 'P'  -- 15% of records
    WHEN status = 'Active' THEN 'A'  -- 80% of records
END
```

### Avoid Redundant Evaluation
```sql
-- INEFFICIENT: Calculation repeated
CASE
    WHEN quantity * price > 1000 THEN quantity * price * 0.9
    WHEN quantity * price > 500 THEN quantity * price * 0.95
    ELSE quantity * price
END

-- BETTER: Use subquery or CTE
WITH calc AS (
    SELECT quantity * price AS total
    FROM orders
)
SELECT 
    CASE
        WHEN total > 1000 THEN total * 0.9
        WHEN total > 500 THEN total * 0.95
        ELSE total
    END
FROM calc;
```

### Indexing Considerations
```sql
-- CASE in WHERE may prevent index usage
WHERE CASE WHEN status = 'A' THEN 1 ELSE 0 END = 1

-- BETTER: Rewrite to use simple conditions
WHERE status = 'A'
```

## Best Practices

1. **Always include ELSE** - Avoid NULL returns by specifying ELSE clause

2. **Order conditions logically** - Put most common conditions first for performance

3. **Keep it readable** - Use proper indentation and line breaks

4. **Consider data types** - Ensure all THEN clauses return compatible data types

5. **Avoid excessive nesting** - Deep nesting reduces readability; consider breaking into CTEs

6. **Document complex logic** - Add comments for business rules

7. **Test edge cases** - Include NULL, empty strings, and boundary values in testing

8. **Use simple CASE when appropriate** - If just comparing one value, simple CASE is cleaner

## Common Mistakes to Avoid

```sql
-- WRONG: Forgetting END keyword
SELECT CASE WHEN x > 5 THEN 'High'

-- CORRECT
SELECT CASE WHEN x > 5 THEN 'High' END

-- WRONG: Mixing data types
CASE
    WHEN status = 'A' THEN 1
    WHEN status = 'B' THEN 'Two'  -- String instead of number
END

-- WRONG: Non-exclusive conditions
CASE
    WHEN price > 50 THEN 'Expensive'
    WHEN price > 100 THEN 'Very Expensive'  -- Never reached!
END

-- CORRECT: Proper order
CASE
    WHEN price > 100 THEN 'Very Expensive'
    WHEN price > 50 THEN 'Expensive'
END
```

## Summary

CASE statements are versatile tools for:
- Categorizing and segmenting data
- Implementing business logic
- Creating custom sorts
- Performing conditional aggregations
- Data validation and quality checks
- Dynamic reporting and analytics
- Conditional updates

They make SQL powerful enough to handle complex logic directly in queries, reducing the need for application-level processing. Master CASE statements to write more efficient, readable, and maintainable SQL code.