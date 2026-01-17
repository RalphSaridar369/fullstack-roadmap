# Common Table Expressions (CTEs) Guide

## What is a CTE?

A **Common Table Expression (CTE)** is a temporary named result set that you can reference within a SELECT, INSERT, UPDATE, or DELETE statement. Think of it as a temporary view that exists only for the duration of a query.

### Basic Syntax
```sql
WITH cte_name AS (
    SELECT column1, column2
    FROM table_name
    WHERE condition
)
SELECT * FROM cte_name;
```

---

## When to Use CTEs

### 1. **Improve Query Readability**

**Without CTE (hard to read):**
```sql
SELECT e.name, e.salary
FROM employees e
WHERE e.salary > (SELECT AVG(salary) 
                  FROM employees 
                  WHERE department_id = e.department_id)
  AND e.department_id IN (SELECT id 
                          FROM departments 
                          WHERE budget > 500000);
```

**With CTE (clear and organized):**
```sql
WITH high_budget_depts AS (
    SELECT id
    FROM departments
    WHERE budget > 500000
),
dept_averages AS (
    SELECT department_id, AVG(salary) as avg_salary
    FROM employees
    GROUP BY department_id
)
SELECT e.name, e.salary
FROM employees e
JOIN dept_averages da ON e.department_id = da.department_id
WHERE e.salary > da.avg_salary
  AND e.department_id IN (SELECT id FROM high_budget_depts);
```

### 2. **Reuse the Same Subquery Multiple Times**

**Without CTE (subquery repeated):**
```sql
SELECT 
    (SELECT AVG(salary) FROM employees) as company_avg,
    name,
    salary,
    salary - (SELECT AVG(salary) FROM employees) as difference
FROM employees;
```

**With CTE (defined once, used multiple times):**
```sql
WITH company_stats AS (
    SELECT AVG(salary) as avg_salary
    FROM employees
)
SELECT 
    cs.avg_salary as company_avg,
    e.name,
    e.salary,
    e.salary - cs.avg_salary as difference
FROM employees e
CROSS JOIN company_stats cs;
```

### 3. **Break Down Complex Queries into Steps**

```sql
WITH 
-- Step 1: Get active employees
active_employees AS (
    SELECT id, name, department_id, salary
    FROM employees
    WHERE status = 'active'
),
-- Step 2: Calculate department averages
dept_stats AS (
    SELECT 
        department_id,
        AVG(salary) as avg_salary,
        COUNT(*) as employee_count
    FROM active_employees
    GROUP BY department_id
),
-- Step 3: Find high-performing departments
high_performing_depts AS (
    SELECT department_id
    FROM dept_stats
    WHERE avg_salary > 75000 
      AND employee_count > 10
)
-- Final query
SELECT 
    ae.name,
    ae.salary,
    ds.avg_salary,
    ds.employee_count
FROM active_employees ae
JOIN dept_stats ds ON ae.department_id = ds.department_id
WHERE ae.department_id IN (SELECT department_id FROM high_performing_depts);
```

### 4. **Recursive Queries (Hierarchical Data)**

CTEs are the ONLY way to write recursive queries in SQL.

**Example: Employee hierarchy (who reports to whom)**
```sql
WITH RECURSIVE employee_hierarchy AS (
    -- Base case: top-level managers (no manager)
    SELECT id, name, manager_id, 1 as level
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case: employees who report to current level
    SELECT e.id, e.name, e.manager_id, eh.level + 1
    FROM employees e
    JOIN employee_hierarchy eh ON e.manager_id = eh.id
)
SELECT * FROM employee_hierarchy
ORDER BY level, name;
```

**Example: Organizational depth chart**
```sql
WITH RECURSIVE org_chart AS (
    -- Start with CEO
    SELECT 
        id,
        name,
        manager_id,
        name as path,
        0 as depth
    FROM employees
    WHERE title = 'CEO'
    
    UNION ALL
    
    -- Add direct reports
    SELECT 
        e.id,
        e.name,
        e.manager_id,
        oc.path || ' -> ' || e.name,
        oc.depth + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT depth, path, name
FROM org_chart
ORDER BY depth, name;
```

---

## Why Use CTEs?

### ✅ Advantages

**1. Readability**
- Gives meaningful names to intermediate results
- Breaks complex logic into understandable steps
- Makes queries self-documenting

**2. Maintainability**
- Easier to modify one CTE than finding subqueries scattered throughout
- Changes in one place affect all references
- Debugging is simpler

**3. Reusability (within same query)**
- Reference the same CTE multiple times
- No need to repeat complex subqueries

**4. Recursion**
- Only way to handle hierarchical/tree data in SQL
- Traverse parent-child relationships
- Generate sequences

**5. Better than Subqueries for Complex Logic**
```sql
-- Instead of nested subqueries:
SELECT * FROM (
    SELECT * FROM (
        SELECT * FROM table1 WHERE condition1
    ) sub1 WHERE condition2
) sub2 WHERE condition3;

-- Use CTEs:
WITH 
step1 AS (SELECT * FROM table1 WHERE condition1),
step2 AS (SELECT * FROM step1 WHERE condition2),
step3 AS (SELECT * FROM step2 WHERE condition3)
SELECT * FROM step3;
```

### ⚠️ Limitations

**1. Scope**
- CTEs only exist for a single query
- Cannot be referenced in subsequent queries

**2. Performance (Database-Dependent)**
- Some databases don't optimize CTEs well
- May execute CTE multiple times if referenced multiple times
- Some databases materialize CTEs (store temporarily), others inline them

**3. Not a Stored Object**
- Defined fresh each time query runs
- For permanent reusable logic, use VIEWs instead

---

## CTE vs Subquery vs View vs Temp Table

| Feature | CTE | Subquery | View | Temp Table |
|---------|-----|----------|------|------------|
| **Reusable in same query** | ✅ Yes | ❌ No (must repeat) | ✅ Yes | ✅ Yes |
| **Reusable across queries** | ❌ No | ❌ No | ✅ Yes | ✅ Yes |
| **Recursion support** | ✅ Yes | ❌ No | ✅ Yes | ❌ No |
| **Readability** | ✅ Excellent | ⚠️ Poor (nested) | ✅ Good | ✅ Good |
| **Performance** | ⚠️ Varies | ⚠️ Varies | ⚠️ Varies | ✅ Good (indexed) |
| **Persistence** | ❌ Query only | ❌ Query only | ✅ Permanent | ⚠️ Session only |

---

## Practical Examples

### Example 1: Data Analysis Pipeline
```sql
WITH 
-- Step 1: Filter to last quarter
recent_sales AS (
    SELECT *
    FROM sales
    WHERE sale_date >= DATE_SUB(CURDATE(), INTERVAL 3 MONTH)
),
-- Step 2: Aggregate by product
product_totals AS (
    SELECT 
        product_id,
        SUM(amount) as total_sales,
        COUNT(*) as num_transactions
    FROM recent_sales
    GROUP BY product_id
),
-- Step 3: Rank products
ranked_products AS (
    SELECT 
        product_id,
        total_sales,
        num_transactions,
        RANK() OVER (ORDER BY total_sales DESC) as sales_rank
    FROM product_totals
)
-- Final: Top 10 products with details
SELECT 
    p.name,
    rp.total_sales,
    rp.num_transactions,
    rp.sales_rank
FROM ranked_products rp
JOIN products p ON rp.product_id = p.id
WHERE rp.sales_rank <= 10;
```

### Example 2: Running Totals and Comparisons
```sql
WITH monthly_sales AS (
    SELECT 
        DATE_FORMAT(sale_date, '%Y-%m') as month,
        SUM(amount) as total
    FROM sales
    GROUP BY DATE_FORMAT(sale_date, '%Y-%m')
)
SELECT 
    month,
    total,
    SUM(total) OVER (ORDER BY month) as running_total,
    LAG(total) OVER (ORDER BY month) as prev_month,
    total - LAG(total) OVER (ORDER BY month) as month_change
FROM monthly_sales
ORDER BY month;
```

### Example 3: Finding Gaps in Sequences
```sql
WITH RECURSIVE date_series AS (
    -- Start date
    SELECT DATE('2024-01-01') as date
    
    UNION ALL
    
    -- Generate next dates
    SELECT DATE_ADD(date, INTERVAL 1 DAY)
    FROM date_series
    WHERE date < '2024-12-31'
),
actual_sales AS (
    SELECT DISTINCT sale_date
    FROM sales
    WHERE sale_date BETWEEN '2024-01-01' AND '2024-12-31'
)
-- Find missing dates
SELECT ds.date as missing_date
FROM date_series ds
LEFT JOIN actual_sales as ON ds.date = as.sale_date
WHERE as.sale_date IS NULL;
```

### Example 4: Multiple CTEs Working Together
```sql
WITH 
-- Get department budgets
dept_budgets AS (
    SELECT id, name, budget
    FROM departments
),
-- Calculate employee costs per department
dept_costs AS (
    SELECT 
        department_id,
        SUM(salary) as total_salaries,
        COUNT(*) as employee_count
    FROM employees
    GROUP BY department_id
),
-- Calculate utilization
dept_utilization AS (
    SELECT 
        db.id,
        db.name,
        db.budget,
        COALESCE(dc.total_salaries, 0) as costs,
        COALESCE(dc.employee_count, 0) as employees,
        ROUND((COALESCE(dc.total_salaries, 0) / db.budget) * 100, 2) as utilization_pct
    FROM dept_budgets db
    LEFT JOIN dept_costs dc ON db.id = dc.department_id
)
SELECT 
    name,
    budget,
    costs,
    employees,
    utilization_pct,
    CASE 
        WHEN utilization_pct > 90 THEN 'Over Budget Risk'
        WHEN utilization_pct > 75 THEN 'High Utilization'
        WHEN utilization_pct > 50 THEN 'Moderate'
        ELSE 'Low Utilization'
    END as status
FROM dept_utilization
ORDER BY utilization_pct DESC;
```

---

## Best Practices

### ✅ DO:
- Use descriptive names for CTEs
- Order CTEs logically (building on each other)
- Comment complex CTEs
- Use CTEs for queries you'll need to reference multiple times in the same query
- Use recursive CTEs for hierarchical data

### ❌ DON'T:
- Create overly complex CTEs (break them down)
- Use CTEs when a simple subquery is clearer
- Forget that CTEs don't persist beyond the query
- Assume CTEs are always faster (test performance)

---

## Quick Decision Guide

**Use CTE when:**
- Query has multiple logical steps
- Same subquery needed multiple times
- Working with hierarchical data (recursive)
- Readability is important
- Team collaboration on complex queries

**Use Subquery when:**
- Simple, one-time calculation
- Single level of nesting
- Performance-critical (some databases optimize better)

**Use View when:**
- Logic needed across multiple queries
- Need to restrict access to underlying tables
- Want to save the query permanently

**Use Temp Table when:**
- Need indexes on intermediate results
- Very large intermediate result sets
- Need to reference results across multiple queries

---

## Summary

**CTEs are like temporary, named scratch pads for your query.**

- They make complex queries readable
- They let you build logic step-by-step
- They're essential for recursive operations
- They exist only for one query
- They're perfect for breaking down data analysis into clear stages