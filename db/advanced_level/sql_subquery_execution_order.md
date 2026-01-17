# Subquery Execution Order Guide

## Core Principle: Inside-Out Execution

Subqueries execute from **innermost to outermost**, but the pattern differs based on whether they're correlated or independent.

---

## 1. Non-Correlated Subqueries

**Execute once, then substitute result into outer query.**

### Example: Simple Scalar Subquery
```sql
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

**Execution Order:**
1. Inner query runs: `SELECT AVG(salary) FROM employees` → Result: 75000
2. Result substituted: `WHERE salary > 75000`
3. Outer query executes with substituted value

### Example: Nested Non-Correlated
```sql
SELECT name
FROM employees
WHERE department_id IN (
    SELECT id FROM departments
    WHERE location IN (
        SELECT city FROM offices WHERE country = 'USA'
    )
);
```

**Execution Order:**
1. Innermost: `SELECT city FROM offices WHERE country = 'USA'` → ['NYC', 'Boston']
2. Middle: `SELECT id FROM departments WHERE location IN ('NYC', 'Boston')` → [5, 12]
3. Outer: `SELECT name FROM employees WHERE department_id IN (5, 12)`

---

## 2. Correlated Subqueries

**Execute once for EACH row in the outer query.**

### Example: Basic Correlated
```sql
SELECT e1.name, e1.salary
FROM employees e1
WHERE e1.salary > (SELECT AVG(e2.salary) 
                   FROM employees e2 
                   WHERE e2.department_id = e1.department_id);
```

**Execution Order:**
```
For Row 1 (e1.department_id = 5):
  → Execute subquery: AVG salary for dept 5 → 70000
  → Compare: e1.salary > 70000 → Include/Exclude

For Row 2 (e1.department_id = 5):
  → Execute subquery again (or use cache)
  → Compare and decide

For Row 3 (e1.department_id = 8):
  → Execute subquery: AVG salary for dept 8 → 85000
  → Compare and decide

... (repeats for EVERY row)
```

**Performance:** If 1000 rows exist, subquery may execute up to 1000 times!

### Example: EXISTS (Correlated with Short-Circuit)
```sql
SELECT d.name
FROM departments d
WHERE EXISTS (SELECT 1 FROM employees e 
              WHERE e.department_id = d.id AND e.salary > 100000);
```

**Execution Order:**
```
For each department:
  → Check if ANY employee matches condition
  → STOP at first match (doesn't scan all employees)
  → Return TRUE/FALSE
```

---

## 3. Execution by Location

### In SELECT Clause
```sql
SELECT name, 
       (SELECT AVG(salary) FROM employees) as avg_salary
FROM employees e;
```
- Non-correlated: Executes once, result cached
- Correlated: Executes per row

### In FROM Clause (Derived Table)
```sql
SELECT * 
FROM (SELECT department_id, AVG(salary) as avg_sal
      FROM employees 
      GROUP BY department_id) dept_avg;
```
**Execution:** Subquery runs first, creates temporary table, then outer query uses it.

### In WHERE Clause
```sql
SELECT name 
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```
**Execution:** Subquery runs first, result used in WHERE filter.

### In HAVING Clause
```sql
SELECT department_id, AVG(salary) 
FROM employees
GROUP BY department_id
HAVING AVG(salary) > (SELECT AVG(salary) FROM employees);
```
**Execution Order:**
1. Subquery executes: overall average salary
2. Main query groups data
3. HAVING filters groups using subquery result

---

## 4. Complex Example

```sql
SELECT e.name,
       (SELECT COUNT(*) FROM employees WHERE department_id = e.department_id) as dept_count
FROM employees e
WHERE e.salary > (SELECT AVG(salary) FROM employees)
  AND e.department_id IN (SELECT id FROM departments WHERE budget > 500000);
```

**Execution Order:**
1. **Pre-execution (non-correlated):**
   - `SELECT AVG(salary) FROM employees` → 75000
   - `SELECT id FROM departments WHERE budget > 500000` → [3, 7, 12]

2. **Main query scan:**
   - Filter: `WHERE salary > 75000 AND department_id IN (3, 7, 12)`

3. **For each qualifying row:**
   - Execute SELECT subquery: `COUNT(*) for that department`
   - Build result row

---

## 5. Performance Comparison

### Execution Counts (for 1000-row table)

| Type | Executions | Example |
|------|-----------|---------|
| Non-correlated | 1 time | `WHERE salary > (SELECT AVG(salary)...)` |
| Correlated | Up to N times | `WHERE salary > (SELECT AVG... WHERE dept = e1.dept)` |
| Nested correlated | N × M times | Correlated subquery containing another correlated subquery |

### Optimization Tips

**❌ SLOW - Correlated (1000 executions):**
```sql
SELECT name FROM employees e1
WHERE salary > (SELECT AVG(salary) FROM employees e2 
                WHERE e2.department_id = e1.department_id);
```

**✅ FAST - JOIN (1 execution):**
```sql
SELECT e.name FROM employees e
JOIN (SELECT department_id, AVG(salary) as avg_sal
      FROM employees GROUP BY department_id) d
ON e.department_id = d.department_id
WHERE e.salary > d.avg_sal;
```

**✅ Use EXISTS instead of IN for correlated checks:**
```sql
-- Better performance (stops at first match)
WHERE EXISTS (SELECT 1 FROM departments d 
              WHERE d.id = e.department_id AND d.budget > 500000)

-- vs IN (may scan entire result set)
WHERE department_id IN (SELECT id FROM departments WHERE budget > 500000)
```

---

## Quick Reference

**Non-Correlated:** Execute once → substitute → use result  
**Correlated:** Execute per outer row → compare → repeat  
**Nested:** Innermost first → work outward  
**Optimization:** Prefer JOINs over correlated subqueries when possible  

**Key Rule:** The more times a subquery executes, the more important indexing and optimization become!