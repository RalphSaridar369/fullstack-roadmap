# Correlated vs Non-Correlated Subqueries

## Overview

Subqueries (also called nested queries or inner queries) are SQL queries embedded within another query. They can be classified into two main categories based on their relationship with the outer query: **non-correlated** and **correlated** subqueries.

## Non-Correlated Subqueries

### Definition
A non-correlated subquery is independent of the outer query. It can be executed on its own and doesn't reference any columns from the outer query. The subquery executes once, and its result is used by the outer query.

### Characteristics
- Executes independently of the outer query
- Runs only once for the entire outer query
- Generally more efficient than correlated subqueries
- The inner query doesn't depend on values from the outer query

### Example

```sql
-- Find all employees who earn more than the average salary
SELECT employee_name, salary
FROM employees
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
);
```

In this example, the subquery `(SELECT AVG(salary) FROM employees)` runs once, calculates the average salary, and returns a single value that the outer query uses for comparison.

### Common Use Cases
- Finding values above or below an aggregate (average, max, min)
- Using IN clauses with a fixed list of values
- EXISTS checks that don't depend on outer query values
- Generating lookup tables or reference data

### Additional Example

```sql
-- Find products from categories that have more than 10 products
SELECT product_name, category_id
FROM products
WHERE category_id IN (
    SELECT category_id
    FROM products
    GROUP BY category_id
    HAVING COUNT(*) > 10
);
```

## Correlated Subqueries

### Definition
A correlated subquery depends on the outer query for its values. It references columns from the outer query and executes once for each row processed by the outer query.

### Characteristics
- Depends on values from the outer query
- Executes repeatedly (once per row in the outer query)
- Generally less efficient than non-correlated subqueries
- Cannot be executed independently

### Example

```sql
-- Find employees who earn more than the average salary in their department
SELECT e1.employee_name, e1.salary, e1.department_id
FROM employees e1
WHERE e1.salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.department_id = e1.department_id
);
```

Notice how the subquery references `e1.department_id` from the outer query. For each employee in the outer query, the subquery recalculates the average salary for that specific department.

### Common Use Cases
- Row-by-row comparisons with aggregates
- Checking for existence of related records with specific conditions
- Finding records that match complex criteria involving the same table
- Calculating running totals or rankings

### Additional Example

```sql
-- Find customers who have placed orders in the last 30 days
SELECT customer_name
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
    AND o.order_date >= CURRENT_DATE - INTERVAL '30 days'
);
```

## Performance Comparison

### Non-Correlated Subqueries
- **Execution**: Once for the entire query
- **Performance**: Generally faster, especially with large datasets
- **Optimization**: Easier for query optimizers to handle

### Correlated Subqueries
- **Execution**: Once per row in the outer query
- **Performance**: Can be slow with large datasets (O(n×m) complexity)
- **Optimization**: More challenging, though modern optimizers can sometimes transform them into joins

## Performance Optimization Tips

Many correlated subqueries can be rewritten as joins or window functions for better performance:

```sql
-- Correlated subquery (slower)
SELECT e1.employee_name, e1.salary
FROM employees e1
WHERE e1.salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.department_id = e1.department_id
);

-- Rewritten as a join (faster)
SELECT e.employee_name, e.salary
FROM employees e
INNER JOIN (
    SELECT department_id, AVG(salary) as avg_salary
    FROM employees
    GROUP BY department_id
) dept_avg ON e.department_id = dept_avg.department_id
WHERE e.salary > dept_avg.avg_salary;
```

## Quick Identification Guide

**It's a Non-Correlated Subquery if:**
- The inner query has no references to the outer query tables
- You can copy the subquery and run it independently
- The subquery result doesn't change based on outer query rows

**It's a Correlated Subquery if:**
- The inner query references columns from the outer query
- The subquery cannot run independently
- The subquery result varies based on the current row being processed in the outer query

## Summary

Choose non-correlated subqueries when possible for better performance. Use correlated subqueries when you need row-by-row comparisons that cannot be easily expressed as joins. Always test and analyze query performance with your specific data and database system, as modern query optimizers may handle these differently than expected.