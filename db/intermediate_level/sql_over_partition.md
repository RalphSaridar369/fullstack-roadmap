# OVER and PARTITION BY in SQL

Window functions with OVER and PARTITION BY clauses allow you to perform calculations across sets of rows that are related to the current row, without collapsing the result set like GROUP BY does. These are powerful tools for analytics, ranking, running totals, and complex calculations.

## Understanding Window Functions

### What Are Window Functions?

Window functions perform calculations across a set of rows (a "window") that are somehow related to the current row. Unlike aggregate functions with GROUP BY, window functions:
- **Retain all rows** in the result set
- Perform calculations across a "window" of rows
- Can access data from other rows without joining

### The OVER Clause

The OVER clause defines the window of rows for the function to operate on.

**Basic Syntax:**
```sql
function_name() OVER (
    [PARTITION BY partition_expression]
    [ORDER BY sort_expression]
    [ROWS or RANGE frame_clause]
)
```

## Basic Components

### 1. OVER Clause Alone

Without PARTITION BY, the window is the entire result set.

```sql
-- Calculate percentage of total sales
SELECT 
    product_name,
    sales_amount,
    SUM(sales_amount) OVER () AS total_sales,
    ROUND(sales_amount * 100.0 / SUM(sales_amount) OVER (), 2) AS percentage_of_total
FROM product_sales;
```

### 2. PARTITION BY Clause

Divides the result set into partitions (groups). The window function is applied separately to each partition.

```sql
-- Calculate percentage within each category
SELECT 
    category,
    product_name,
    sales_amount,
    SUM(sales_amount) OVER (PARTITION BY category) AS category_total,
    ROUND(sales_amount * 100.0 / SUM(sales_amount) OVER (PARTITION BY category), 2) AS pct_of_category
FROM product_sales;
```

### 3. ORDER BY Clause

Defines the logical order of rows within each partition.

```sql
-- Running total within each category
SELECT 
    category,
    product_name,
    sales_amount,
    SUM(sales_amount) OVER (PARTITION BY category ORDER BY sale_date) AS running_total
FROM product_sales;
```

## Common Window Functions

### Aggregate Functions

**SUM, AVG, COUNT, MIN, MAX**
```sql
SELECT 
    department,
    employee_name,
    salary,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg_salary,
    MAX(salary) OVER (PARTITION BY department) AS dept_max_salary,
    COUNT(*) OVER (PARTITION BY department) AS dept_employee_count
FROM employees;
```

### Ranking Functions

**ROW_NUMBER()** - Assigns unique sequential integers
```sql
SELECT 
    department,
    employee_name,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS salary_rank
FROM employees;
```

**RANK()** - Assigns rank with gaps for ties
```sql
SELECT 
    department,
    employee_name,
    salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS salary_rank
FROM employees;
-- If two employees have same salary, both get rank 1, next is rank 3
```

**DENSE_RANK()** - Assigns rank without gaps for ties
```sql
SELECT 
    department,
    employee_name,
    salary,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS salary_rank
FROM employees;
-- If two employees have same salary, both get rank 1, next is rank 2
```

**NTILE(n)** - Divides rows into n groups
```sql
SELECT 
    employee_name,
    salary,
    NTILE(4) OVER (ORDER BY salary DESC) AS salary_quartile
FROM employees;
-- Divides employees into 4 equal groups based on salary
```

### Value Functions

**LAG()** - Accesses data from previous row
```sql
SELECT 
    sale_date,
    daily_revenue,
    LAG(daily_revenue, 1) OVER (ORDER BY sale_date) AS previous_day_revenue,
    daily_revenue - LAG(daily_revenue, 1) OVER (ORDER BY sale_date) AS day_over_day_change
FROM daily_sales;
```

**LEAD()** - Accesses data from next row
```sql
SELECT 
    sale_date,
    daily_revenue,
    LEAD(daily_revenue, 1) OVER (ORDER BY sale_date) AS next_day_revenue,
    LEAD(daily_revenue, 1) OVER (ORDER BY sale_date) - daily_revenue AS next_day_change
FROM daily_sales;
```

**FIRST_VALUE()** - Returns first value in window
```sql
SELECT 
    department,
    employee_name,
    salary,
    FIRST_VALUE(employee_name) OVER (
        PARTITION BY department 
        ORDER BY salary DESC
    ) AS highest_paid_employee
FROM employees;
```

**LAST_VALUE()** - Returns last value in window
```sql
SELECT 
    department,
    employee_name,
    salary,
    LAST_VALUE(employee_name) OVER (
        PARTITION BY department 
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS lowest_paid_employee
FROM employees;
```

## Use Cases

### 1. Running Totals and Cumulative Calculations

**Running Total**
```sql
SELECT 
    order_date,
    order_amount,
    SUM(order_amount) OVER (ORDER BY order_date) AS running_total
FROM orders
ORDER BY order_date;
```

**Running Total by Category**
```sql
SELECT 
    category,
    order_date,
    order_amount,
    SUM(order_amount) OVER (
        PARTITION BY category 
        ORDER BY order_date
    ) AS category_running_total
FROM orders
ORDER BY category, order_date;
```

**Year-to-Date Totals**
```sql
SELECT 
    YEAR(order_date) AS year,
    MONTH(order_date) AS month,
    SUM(order_amount) AS monthly_total,
    SUM(SUM(order_amount)) OVER (
        PARTITION BY YEAR(order_date) 
        ORDER BY MONTH(order_date)
    ) AS ytd_total
FROM orders
GROUP BY YEAR(order_date), MONTH(order_date);
```

### 2. Ranking and Top N Queries

**Top 3 Products per Category**
```sql
WITH ranked_products AS (
    SELECT 
        category,
        product_name,
        sales_amount,
        ROW_NUMBER() OVER (
            PARTITION BY category 
            ORDER BY sales_amount DESC
        ) AS rank_in_category
    FROM product_sales
)
SELECT 
    category,
    product_name,
    sales_amount
FROM ranked_products
WHERE rank_in_category <= 3;
```

**Quartile Analysis**
```sql
SELECT 
    customer_name,
    total_purchases,
    NTILE(4) OVER (ORDER BY total_purchases DESC) AS customer_quartile,
    CASE NTILE(4) OVER (ORDER BY total_purchases DESC)
        WHEN 1 THEN 'Top 25%'
        WHEN 2 THEN '25-50%'
        WHEN 3 THEN '50-75%'
        ELSE 'Bottom 25%'
    END AS quartile_label
FROM customer_summary;
```

**Percentile Calculation**
```sql
SELECT 
    product_name,
    price,
    PERCENT_RANK() OVER (ORDER BY price) AS price_percentile,
    ROUND(PERCENT_RANK() OVER (ORDER BY price) * 100, 2) AS percentile_rank
FROM products;
```

### 3. Comparative Analysis

**Compare to Average**
```sql
SELECT 
    department,
    employee_name,
    salary,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg,
    salary - AVG(salary) OVER (PARTITION BY department) AS diff_from_avg,
    ROUND((salary / AVG(salary) OVER (PARTITION BY department) - 1) * 100, 2) AS pct_diff_from_avg
FROM employees;
```

**Compare to Previous Period**
```sql
SELECT 
    product_name,
    sale_month,
    monthly_sales,
    LAG(monthly_sales, 1) OVER (
        PARTITION BY product_name 
        ORDER BY sale_month
    ) AS previous_month_sales,
    ROUND(
        (monthly_sales - LAG(monthly_sales, 1) OVER (
            PARTITION BY product_name 
            ORDER BY sale_month
        )) * 100.0 / 
        LAG(monthly_sales, 1) OVER (
            PARTITION BY product_name 
            ORDER BY sale_month
        ), 2
    ) AS month_over_month_growth_pct
FROM monthly_product_sales;
```

**Compare to Best Performer**
```sql
SELECT 
    department,
    employee_name,
    sales_total,
    MAX(sales_total) OVER (PARTITION BY department) AS dept_best_sales,
    ROUND(sales_total * 100.0 / MAX(sales_total) OVER (PARTITION BY department), 2) AS pct_of_best
FROM employee_sales;
```

### 4. Moving Averages and Trends

**7-Day Moving Average**
```sql
SELECT 
    sale_date,
    daily_revenue,
    AVG(daily_revenue) OVER (
        ORDER BY sale_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7_day
FROM daily_sales
ORDER BY sale_date;
```

**3-Month Moving Average by Product**
```sql
SELECT 
    product_name,
    sale_month,
    monthly_sales,
    AVG(monthly_sales) OVER (
        PARTITION BY product_name
        ORDER BY sale_month
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3_month
FROM monthly_product_sales;
```

**Centered Moving Average**
```sql
SELECT 
    sale_date,
    daily_revenue,
    AVG(daily_revenue) OVER (
        ORDER BY sale_date
        ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING
    ) AS centered_moving_avg_7_day
FROM daily_sales;
```

### 5. Gap and Island Detection

**Finding Gaps in Sequences**
```sql
WITH numbered_orders AS (
    SELECT 
        order_id,
        order_date,
        ROW_NUMBER() OVER (ORDER BY order_date) AS row_num,
        LAG(order_date) OVER (ORDER BY order_date) AS prev_order_date
    FROM orders
)
SELECT 
    order_id,
    order_date,
    prev_order_date,
    DATEDIFF(day, prev_order_date, order_date) AS days_since_last_order
FROM numbered_orders
WHERE DATEDIFF(day, prev_order_date, order_date) > 7;
```

**Finding Consecutive Sequences**
```sql
WITH grouped_data AS (
    SELECT 
        event_date,
        event_type,
        ROW_NUMBER() OVER (ORDER BY event_date) -
        ROW_NUMBER() OVER (PARTITION BY event_type ORDER BY event_date) AS grp
    FROM events
)
SELECT 
    event_type,
    MIN(event_date) AS sequence_start,
    MAX(event_date) AS sequence_end,
    COUNT(*) AS consecutive_days
FROM grouped_data
GROUP BY event_type, grp
HAVING COUNT(*) >= 3;
```

### 6. First and Last in Group

**First Purchase per Customer**
```sql
SELECT DISTINCT
    customer_id,
    FIRST_VALUE(order_date) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
    ) AS first_order_date,
    FIRST_VALUE(order_amount) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
    ) AS first_order_amount
FROM orders;
```

**Most Recent Activity**
```sql
SELECT DISTINCT
    customer_id,
    customer_name,
    LAST_VALUE(activity_date) OVER (
        PARTITION BY customer_id 
        ORDER BY activity_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_activity_date,
    LAST_VALUE(activity_type) OVER (
        PARTITION BY customer_id 
        ORDER BY activity_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_activity_type
FROM customer_activities;
```

### 7. Deduplication

**Remove Duplicates, Keep Most Recent**
```sql
WITH ranked_records AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id, product_id 
            ORDER BY purchase_date DESC
        ) AS rn
    FROM purchases
)
SELECT 
    customer_id,
    product_id,
    purchase_date,
    amount
FROM ranked_records
WHERE rn = 1;
```

**Identify Duplicates**
```sql
SELECT 
    email,
    customer_name,
    COUNT(*) OVER (PARTITION BY email) AS duplicate_count,
    ROW_NUMBER() OVER (PARTITION BY email ORDER BY created_date) AS occurrence_number
FROM customers
WHERE COUNT(*) OVER (PARTITION BY email) > 1;
```

### 8. Pagination

**Row Numbers for Pagination**
```sql
SELECT 
    ROW_NUMBER() OVER (ORDER BY product_name) AS row_num,
    product_id,
    product_name,
    price
FROM products
-- Application logic: WHERE row_num BETWEEN 21 AND 30 for page 3 (10 per page)
```

### 9. Cohort Analysis

**Customer Cohorts by Registration Month**
```sql
WITH customer_cohorts AS (
    SELECT 
        customer_id,
        registration_date,
        DATE_TRUNC('month', registration_date) AS cohort_month,
        order_date,
        DATE_TRUNC('month', order_date) AS order_month
    FROM customers
    JOIN orders USING (customer_id)
)
SELECT 
    cohort_month,
    order_month,
    COUNT(DISTINCT customer_id) AS active_customers,
    FIRST_VALUE(COUNT(DISTINCT customer_id)) OVER (
        PARTITION BY cohort_month 
        ORDER BY order_month
    ) AS cohort_size,
    ROUND(
        COUNT(DISTINCT customer_id) * 100.0 / 
        FIRST_VALUE(COUNT(DISTINCT customer_id)) OVER (
            PARTITION BY cohort_month 
            ORDER BY order_month
        ), 2
    ) AS retention_rate
FROM customer_cohorts
GROUP BY cohort_month, order_month;
```

### 10. Performance Metrics

**Employee Performance Scoring**
```sql
SELECT 
    employee_name,
    department,
    sales_total,
    customer_satisfaction,
    
    -- Rank within department
    RANK() OVER (PARTITION BY department ORDER BY sales_total DESC) AS sales_rank,
    
    -- Percentile performance
    PERCENT_RANK() OVER (PARTITION BY department ORDER BY sales_total) AS sales_percentile,
    
    -- Compare to department average
    AVG(sales_total) OVER (PARTITION BY department) AS dept_avg_sales,
    
    -- Overall performance score
    (PERCENT_RANK() OVER (PARTITION BY department ORDER BY sales_total) * 0.6 +
     PERCENT_RANK() OVER (PARTITION BY department ORDER BY customer_satisfaction) * 0.4) AS performance_score
FROM employee_metrics;
```

## Frame Specifications

### Understanding Frame Clauses

Frame clauses define which rows are included in the window relative to the current row.

**ROWS vs RANGE:**
- **ROWS**: Physical offset (number of rows)
- **RANGE**: Logical offset (based on value)

### Common Frame Specifications

**Unbounded Preceding to Current**
```sql
SUM(amount) OVER (
    ORDER BY date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
)
-- Running total from first row to current row
```

**Unbounded on Both Sides**
```sql
SUM(amount) OVER (
    ORDER BY date
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
-- Total across entire partition
```

**Specific Row Range**
```sql
AVG(amount) OVER (
    ORDER BY date
    ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING
)
-- Average of 7 rows (3 before, current, 3 after)
```

**Only Previous Rows**
```sql
MAX(amount) OVER (
    ORDER BY date
    ROWS BETWEEN 5 PRECEDING AND 1 PRECEDING
)
-- Maximum of 5 rows before current (excluding current)
```

### Frame Examples

**Moving Sum (Fixed Window)**
```sql
SELECT 
    sale_date,
    daily_sales,
    SUM(daily_sales) OVER (
        ORDER BY sale_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS last_7_days_total
FROM daily_sales;
```

**Range-Based Frame**
```sql
SELECT 
    sale_date,
    daily_sales,
    SUM(daily_sales) OVER (
        ORDER BY sale_date
        RANGE BETWEEN INTERVAL '7' DAY PRECEDING AND CURRENT ROW
    ) AS last_7_days_total
FROM daily_sales;
```

## PARTITION BY vs GROUP BY

### Key Differences

**GROUP BY:**
- Collapses rows into groups
- Returns one row per group
- Requires aggregate functions
- Reduces result set size

**PARTITION BY:**
- Keeps all rows
- Performs calculations across partitions
- Returns original number of rows
- Used with window functions

### Comparison Example

**GROUP BY:**
```sql
SELECT 
    department,
    AVG(salary) AS avg_salary
FROM employees
GROUP BY department;
-- Returns 3 rows (one per department)
```

**PARTITION BY:**
```sql
SELECT 
    department,
    employee_name,
    salary,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg_salary
FROM employees;
-- Returns all employee rows with department average included
```

### Combining Both

```sql
SELECT 
    department,
    hire_year,
    COUNT(*) AS hires_that_year,
    AVG(salary) AS avg_starting_salary,
    AVG(AVG(salary)) OVER (PARTITION BY department) AS dept_overall_avg
FROM employees
GROUP BY department, hire_year;
```

## Performance Considerations

### Optimization Tips

**1. Use Appropriate Indexes**
```sql
-- Index on PARTITION BY and ORDER BY columns
CREATE INDEX idx_sales_category_date 
ON sales(category, sale_date);
```

**2. Limit Window Size**
```sql
-- Better: Fixed frame
ROWS BETWEEN 6 PRECEDING AND CURRENT ROW

-- Avoid: Large unbounded frames when not necessary
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
```

**3. Reuse Window Definitions**
```sql
SELECT 
    product_name,
    sales,
    ROW_NUMBER() OVER w AS rn,
    RANK() OVER w AS rnk,
    AVG(sales) OVER w AS avg_sales
FROM product_sales
WINDOW w AS (PARTITION BY category ORDER BY sales DESC);
```

**4. Use CTEs for Complex Logic**
```sql
-- Better: Break into steps
WITH ranked AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) AS rn
    FROM products
)
SELECT *
FROM ranked
WHERE rn <= 5;

-- Avoid: Multiple window functions in WHERE
SELECT *
FROM (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) AS rn
    FROM products
) x
WHERE rn <= 5;
```

## Common Patterns

### Running Difference
```sql
SELECT 
    sale_date,
    revenue,
    revenue - LAG(revenue) OVER (ORDER BY sale_date) AS day_over_day_diff
FROM daily_sales;
```

### Percent of Total
```sql
SELECT 
    product_name,
    sales,
    ROUND(sales * 100.0 / SUM(sales) OVER (), 2) AS pct_of_total
FROM product_sales;
```

### Dense Numbering Without Gaps
```sql
SELECT 
    order_id,
    DENSE_RANK() OVER (ORDER BY order_date) AS day_number
FROM orders;
```

### Conditional Aggregation in Window
```sql
SELECT 
    employee_name,
    salary,
    department,
    AVG(CASE WHEN salary > 50000 THEN salary END) OVER (PARTITION BY department) AS high_earner_avg
FROM employees;
```

## Best Practices

1. **Name your windows** - Use WINDOW clause for reusability

2. **Be explicit with frames** - Specify ROWS or RANGE to avoid defaults

3. **Consider NULL handling** - Window functions handle NULLs differently than regular aggregates

4. **Use CTEs for readability** - Break complex window functions into steps

5. **Test with edge cases** - Verify behavior with ties, NULLs, and single-row partitions

6. **Document complex logic** - Add comments explaining business rules

7. **Profile performance** - Monitor query plans for large datasets

8. **Avoid redundant calculations** - Calculate once, reference multiple times

## Common Mistakes

```sql
-- WRONG: Using WHERE with window function
SELECT *
FROM products
WHERE ROW_NUMBER() OVER (ORDER BY price) <= 10;

-- CORRECT: Use subquery or CTE
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (ORDER BY price) AS rn
    FROM products
)
SELECT * FROM ranked WHERE rn <= 10;

-- WRONG: Forgetting ROWS specification with LAST_VALUE
LAST_VALUE(x) OVER (ORDER BY y)  -- Only sees current row!

-- CORRECT: Include proper frame
LAST_VALUE(x) OVER (
    ORDER BY y
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)

-- WRONG: ORDER BY without PARTITION on non-deterministic data
ROW_NUMBER() OVER (ORDER BY category)  -- Arbitrary ordering within category

-- CORRECT: Add tiebreaker
ROW_NUMBER() OVER (ORDER BY category, product_id)
```

## Summary

Window functions with OVER and PARTITION BY are powerful SQL features that enable:
- Running totals and moving averages
- Ranking and top-N queries
- Comparative analysis without self-joins
- Complex analytics while preserving detail
- Efficient deduplication
- Cohort and trend analysis

Master these concepts to write sophisticated analytical queries that perform well and are easier to maintain than complex join-based alternatives.