# JOIN vs EXISTS vs Subquery Performance Guide

## Performance Matrix

| Query Type | Executions | Use Case | Speed |
|------------|-----------|----------|-------|
| Non-correlated IN | 1 time | Small result set | ⚡⚡⚡ Fast |
| Non-correlated subquery | 1 time | Single value needed | ⚡⚡⚡ Fast |
| JOIN (indexed) | 1 operation | Need data from both tables | ⚡⚡⚡ Fastest |
| EXISTS (correlated) | N times (short-circuit) | Existence check | ⚡⚡ Good |
| IN (correlated) | N times | List membership | ⚠️ Moderate |
| Correlated subquery | N times | Per-row calculation | ⚠️ Slow |

---

## 1. Non-Correlated Subqueries

### Non-Correlated IN

```sql
SELECT e.name
FROM employees e
WHERE e.department_id IN (SELECT id FROM departments WHERE budget > 500000);
```

**Execution:**
1. Inner query runs **once**: `SELECT id FROM departments WHERE budget > 500000` → [3, 7, 12]
2. Transforms to: `WHERE e.department_id IN (3, 7, 12)`
3. Scans employees table once

**Performance:** 
- Subquery executions: **1**
- Total operations: **~20,000** (scan employees)
- Time: **~15ms**

**When to use:**
- ✅ Subquery returns small list (< 100 values)
- ✅ Subquery is independent of outer query
- ✅ Only filtering, not retrieving data

---

### Non-Correlated Scalar Subquery

```sql
SELECT e.name, e.salary
FROM employees e
WHERE e.salary > (SELECT AVG(salary) FROM employees);
```

**Execution:**
1. Inner query runs **once**: `SELECT AVG(salary) FROM employees` → 75000
2. Transforms to: `WHERE e.salary > 75000`
3. Scans employees table once

**Performance:**
- Subquery executions: **1**
- Total operations: **~20,000** (scan employees)
- Time: **~10ms**

**When to use:**
- ✅ Need single aggregate value
- ✅ Value doesn't depend on outer row
- ✅ Simple comparison

---

## 2. Correlated EXISTS

### Basic Correlated EXISTS

```sql
SELECT d.name
FROM departments d
WHERE EXISTS (SELECT 1 FROM employees e 
              WHERE e.department_id = d.id 
              AND e.salary > 100000);
```

**Execution:**
```
For dept 1 (id=1):
  → Check first employee in dept 1
  → Salary = 120000 > 100000 ✓
  → STOP (found match, return TRUE)
  
For dept 2 (id=2):
  → Check first employee: 80000 ❌
  → Check second employee: 95000 ❌
  → Check third employee: 110000 ✓
  → STOP (found match)
  
For dept 3 (id=3):
  → Check all 50 employees in dept 3
  → None match
  → Return FALSE
```

**Performance:**
- Worst case: **50 departments × average checks per dept**
- Best case: **50 checks** (first employee always matches)
- Average: **~500 checks** (stops early)
- Time: **~8ms**

**Why it's fast:**
- ✅ Short-circuits at first match
- ✅ Only returns TRUE/FALSE (no data)
- ✅ Can use indexes efficiently

---

### EXISTS vs JOIN Comparison

**Scenario:** Find departments with high earners

**EXISTS (Correlated):**
```sql
SELECT d.name
FROM departments d
WHERE EXISTS (SELECT 1 FROM employees e 
              WHERE e.department_id = d.id 
              AND e.salary > 100000);
```
- Checks: ~500 employees (short-circuits)
- Time: **8ms**

**JOIN:**
```sql
SELECT DISTINCT d.name
FROM departments d
JOIN employees e ON e.department_id = d.id
WHERE e.salary > 100000;
```
- Processes: All 300 high-earners across all departments
- DISTINCT removes duplicates
- Time: **25ms**

**Winner: EXISTS** (3x faster because it short-circuits)

---

## 3. Correlated IN

### Correlated IN (Slow)

```sql
SELECT e.name
FROM employees e
WHERE e.department_id IN (SELECT d.id FROM departments d 
                          WHERE d.id = e.department_id 
                          AND d.budget > 500000);
```

**Execution:**
```
For employee 1 (dept_id = 5):
  → Run: SELECT d.id FROM departments WHERE d.id = 5 AND d.budget > 500000
  → Result: [5] or []
  → Check if 5 IN [5]
  
For employee 2 (dept_id = 5):
  → Run same query again (may be cached)
  → Check membership

... (20,000 times)
```

**Performance:**
- Executions: **Up to 20,000** (one per employee)
- With caching: **~50** (one per unique department)
- Time: **~200ms** (without caching), **~30ms** (with caching)

**Problem:**
- ❌ Doesn't short-circuit like EXISTS
- ❌ Returns actual values (not just TRUE/FALSE)
- ❌ Less efficient than EXISTS for same goal

**Better alternative:** Use EXISTS or JOIN

---

## 4. Correlated Scalar Subquery

### Correlated Scalar Subquery

```sql
SELECT e.name, e.salary,
       (SELECT AVG(salary) FROM employees e2 
        WHERE e2.department_id = e.department_id) as dept_avg
FROM employees e;
```

**Execution:**
```
For employee 1 (dept_id = 5):
  → Run: SELECT AVG(salary) FROM employees WHERE department_id = 5
  → Result: 70000
  → Output: {name, salary, 70000}

For employee 2 (dept_id = 5):
  → Run same query (likely cached)
  → Result: 70000
  
For employee 3 (dept_id = 8):
  → Run: SELECT AVG(salary) FROM employees WHERE department_id = 8
  → Result: 85000
```

**Performance:**
- Worst case: **20,000 executions**
- With caching: **~50 executions** (one per department)
- Time: **~100ms** (without optimization), **~40ms** (with caching)

**When to use:**
- ✅ Need per-row calculated value
- ✅ Value depends on outer row
- ⚠️ Consider JOIN alternative for better performance

---

## 5. JOIN Performance

### Simple JOIN

```sql
SELECT e.name, d.name, d.budget
FROM employees e
JOIN departments d ON e.department_id = d.id;
```

**Execution (Hash Join):**
1. Build hash table from departments (50 rows): **50 operations**
2. Scan employees and lookup in hash: **20,000 operations**
3. Total: **20,050 operations**

**Performance:**
- Algorithm: Hash join (O(n + m))
- Time: **~12ms**

**Why it's fast:**
- ✅ Single optimized operation
- ✅ Hash lookup is O(1)
- ✅ No repeated queries

---

### JOIN with WHERE vs EXISTS

**Find employees in high-budget departments**

**JOIN:**
```sql
SELECT e.name
FROM employees e
JOIN departments d ON e.department_id = d.id
WHERE d.budget > 500000;
```
- Operations: Hash join + filter
- Result: All employees in matching departments
- Time: **~15ms**

**EXISTS:**
```sql
SELECT e.name
FROM employees e
WHERE EXISTS (SELECT 1 FROM departments d 
              WHERE d.id = e.department_id 
              AND d.budget > 500000);
```
- Operations: 20,000 existence checks (with index)
- Result: Same as JOIN
- Time: **~25ms**

**Winner: JOIN** (faster for this use case because we're filtering, not just checking existence)

---

## 6. Performance Comparison Table

### Dataset: 50 departments, 20,000 employees

| Query | Type | Executions | Operations | Time | Best For |
|-------|------|-----------|-----------|------|----------|
| `dept_id IN (SELECT...)` | Non-correlated | 1 | ~20,000 | 15ms | Small result list |
| `salary > (SELECT AVG...)` | Non-correlated | 1 | ~20,000 | 10ms | Single aggregate |
| `EXISTS (SELECT... e.dept_id)` | Correlated | ~500 | ~500 | 8ms | Existence check |
| `IN (SELECT... WHERE id=e.id)` | Correlated | 20,000 | ~60,000 | 200ms | ❌ Avoid |
| `(SELECT AVG... WHERE dept=e.dept)` | Correlated scalar | ~50 (cached) | ~60,000 | 40ms | Per-row calculation |
| `JOIN departments` | JOIN | 1 | ~20,050 | 12ms | Need data from both |
| `JOIN + DISTINCT` | JOIN | 1 | ~20,050 + sort | 25ms | Remove duplicates |

---

## 7. Optimization Strategies

### Converting Correlated to Non-Correlated

**❌ SLOW - Correlated:**
```sql
SELECT e.name
FROM employees e
WHERE e.salary > (SELECT AVG(salary) FROM employees e2 
                  WHERE e2.department_id = e.department_id);
```
Time: **~100ms**

**✅ FAST - JOIN:**
```sql
SELECT e.name
FROM employees e
JOIN (SELECT department_id, AVG(salary) as avg_sal
      FROM employees
      GROUP BY department_id) dept_avg
ON e.department_id = dept_avg.department_id
WHERE e.salary > dept_avg.avg_sal;
```
Time: **~20ms**

**Improvement: 5x faster**

---

### Using EXISTS Instead of IN

**❌ SLOWER - Correlated IN:**
```sql
SELECT e.name
FROM employees e
WHERE e.department_id IN (SELECT d.id FROM departments d 
                          WHERE d.budget > 500000);
```

**✅ FASTER - Non-correlated IN:**
```sql
SELECT e.name
FROM employees e
WHERE e.department_id IN (SELECT id FROM departments 
                          WHERE budget > 500000);
```

**✅ FASTEST - JOIN:**
```sql
SELECT e.name
FROM employees e
JOIN departments d ON e.department_id = d.id
WHERE d.budget > 500000;
```

---

### EXISTS for Pure Existence Checks

**❌ SLOWER - JOIN with DISTINCT:**
```sql
SELECT DISTINCT d.name
FROM departments d
JOIN employees e ON e.department_id = d.id
WHERE e.status = 'active';
```
- Processes all active employees
- Removes duplicates
- Time: **~30ms**

**✅ FASTER - EXISTS:**
```sql
SELECT d.name
FROM departments d
WHERE EXISTS (SELECT 1 FROM employees e 
              WHERE e.department_id = d.id 
              AND e.status = 'active');
```
- Stops at first active employee per department
- No duplicates
- Time: **~8ms**

**Improvement: 4x faster**

---

## 8. When Each Performs Best

### Use Non-Correlated Subquery When:
- Subquery returns single value or small list
- Value doesn't depend on outer row
- Simple filtering condition

**Example:**
```sql
WHERE salary > (SELECT AVG(salary) FROM employees)
WHERE dept_id IN (1, 5, 9)  -- Small, static list
```

---

### Use EXISTS When:
- Only checking if related records exist
- Don't need data from related table
- One-to-many relationship (short-circuit benefit)
- Exclusion queries (NOT EXISTS)

**Example:**
```sql
-- Find departments with employees
WHERE EXISTS (SELECT 1 FROM employees WHERE dept_id = d.id)

-- Find employees without projects
WHERE NOT EXISTS (SELECT 1 FROM projects WHERE emp_id = e.id)
```

---

### Use JOIN When:
- Need columns from both tables
- Processing most/all rows from both tables
- One-to-one or many-to-one relationship
- Complex filtering on multiple tables

**Example:**
```sql
SELECT e.name, d.name, d.budget
FROM employees e
JOIN departments d ON e.department_id = d.id
WHERE d.budget > 500000 AND e.salary > 80000;
```

---

### Avoid Correlated IN/Subquery When:
- Can be rewritten as JOIN
- Can be rewritten as non-correlated
- High execution count expected

**❌ Avoid:**
```sql
WHERE dept_id IN (SELECT id FROM depts WHERE id = e.dept_id)
```

**✅ Use instead:**
```sql
JOIN departments d ON d.id = e.dept_id
```

---

## 9. Real-World Scenarios

### Scenario 1: Finding Orphan Records

**Task:** Find departments with no employees

**❌ JOIN (awkward):**
```sql
SELECT d.name
FROM departments d
LEFT JOIN employees e ON e.department_id = d.id
WHERE e.id IS NULL;
```
Time: **~35ms**

**✅ NOT EXISTS (clean):**
```sql
SELECT d.name
FROM departments d
WHERE NOT EXISTS (SELECT 1 FROM employees e 
                  WHERE e.department_id = d.id);
```
Time: **~5ms**

**Winner: NOT EXISTS** (7x faster + clearer intent)

---

### Scenario 2: Filtering with Aggregates

**Task:** Employees earning above department average

**❌ Correlated subquery (slow):**
```sql
SELECT name FROM employees e1
WHERE salary > (SELECT AVG(salary) FROM employees e2 
                WHERE e2.department_id = e1.department_id);
```
Time: **~100ms**

**✅ JOIN with derived table:**
```sql
SELECT e.name
FROM employees e
JOIN (SELECT department_id, AVG(salary) as avg_sal
      FROM employees GROUP BY department_id) d
ON e.department_id = d.department_id
WHERE e.salary > d.avg_sal;
```
Time: **~25ms**

**Winner: JOIN** (4x faster)

---

### Scenario 3: Multiple Existence Checks

**Task:** Employees with projects AND certifications

**❌ Multiple JOINs (slow):**
```sql
SELECT DISTINCT e.name
FROM employees e
JOIN projects p ON p.employee_id = e.id
JOIN certifications c ON c.employee_id = e.id;
```
- Creates cartesian product
- Many duplicates
- Time: **~200ms**

**✅ Multiple EXISTS:**
```sql
SELECT e.name
FROM employees e
WHERE EXISTS (SELECT 1 FROM projects p WHERE p.employee_id = e.id)
  AND EXISTS (SELECT 1 FROM certifications c WHERE c.employee_id = e.id);
```
- Two quick checks per employee
- No duplicates
- Time: **~30ms**

**Winner: EXISTS** (7x faster)

---

## 10. Quick Decision Guide

```
Need data from both tables?
├─ YES → Use JOIN
└─ NO → Continue

Just checking existence?
├─ YES → Use EXISTS
└─ NO → Continue

Subquery returns single value/small list?
├─ YES → Use non-correlated IN/subquery
└─ NO → Continue

Can you rewrite as JOIN?
├─ YES → Use JOIN (probably faster)
└─ NO → Use correlated subquery (optimize with indexes)
```

---

## Key Takeaways

1. **Non-correlated subqueries** = 1 execution = Fast
2. **EXISTS** = Short-circuits = Best for existence checks
3. **JOIN** = Single optimized operation = Best for retrieving data
4. **Correlated IN/subquery** = N executions = Avoid when possible
5. **Indexes are crucial** for correlated queries to perform well
6. **Always test** with realistic data volumes - theory ≠ practice!