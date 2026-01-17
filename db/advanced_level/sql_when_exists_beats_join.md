# When EXISTS Outperforms JOIN

## The Core Advantage: Short-Circuit + No Duplicates

EXISTS wins when:
1. You only need to know IF something exists (not the data itself)
2. One-to-many relationships create duplicates in JOINs
3. Early matches are common (short-circuit benefit)

---

## Scenario 1: Existence Checks

**Task:** Find departments that have at least one employee

### ❌ JOIN (slower)
```sql
SELECT DISTINCT d.name
FROM departments d
JOIN employees e ON e.department_id = d.id;
```

**What happens:**
- Dept 1 has 500 employees → Creates 500 rows
- Dept 2 has 300 employees → Creates 300 rows
- Total: 20,000 rows created
- DISTINCT removes duplicates → Back to 50 departments

**Cost:** Process 20,000 rows + expensive DISTINCT operation  
**Time: ~45ms**

### ✅ EXISTS (faster)
```sql
SELECT d.name
FROM departments d
WHERE EXISTS (SELECT 1 FROM employees e WHERE e.department_id = d.id);
```

**What happens:**
- Dept 1: Check first employee → Found! STOP
- Dept 2: Check first employee → Found! STOP
- Total: ~50 checks (one per department)

**Cost:** 50 quick existence checks  
**Time: ~8ms**

**Winner: EXISTS (5-6x faster)**

---

## Scenario 2: Short-Circuit Benefit

**Task:** Find departments with high earners (salary > 150k)

### Setup
- 50 departments
- 20,000 employees total
- Only 200 employees earn > 150k
- High earners are distributed: some depts have them early, some late, some not at all

### ❌ JOIN
```sql
SELECT DISTINCT d.name
FROM departments d
JOIN employees e ON e.department_id = d.id
WHERE e.salary > 150000;
```

**What happens:**
- Scans all 20,000 employees
- Finds all 200 high earners
- JOINs them with departments
- Creates 200 rows
- DISTINCT reduces to ~30 departments

**Cost:** Full scan of 20,000 + JOIN + DISTINCT  
**Time: ~35ms**

### ✅ EXISTS
```sql
SELECT d.name
FROM departments d
WHERE EXISTS (SELECT 1 FROM employees e 
              WHERE e.department_id = d.id AND e.salary > 150000);
```

**What happens:**
```
Dept 1: 400 employees, 1st employee earns 160k → STOP (1 check)
Dept 2: 350 employees, 50th employee earns 155k → STOP (50 checks)
Dept 3: 300 employees, none earn >150k → Check all 300
...
Average: ~80 checks per department
Total: ~4,000 checks
```

**Cost:** ~4,000 targeted checks (with index)  
**Time: ~12ms**

**Winner: EXISTS (3x faster)**

---

## Scenario 3: Multiple Unrelated Existence Checks

**Task:** Find employees who have projects AND certifications AND reviews

### ❌ Multiple JOINs (very slow)
```sql
SELECT DISTINCT e.name
FROM employees e
JOIN projects p ON p.employee_id = e.id
JOIN certifications c ON c.employee_id = e.id
JOIN reviews r ON r.employee_id = e.id;
```

**What happens - Cartesian explosion:**
```
Employee with:
- 5 projects
- 3 certifications  
- 10 reviews

Creates: 5 × 3 × 10 = 150 duplicate rows for ONE employee!

20,000 employees with similar data = millions of intermediate rows
Then DISTINCT removes duplicates
```

**Cost:** Massive intermediate result set  
**Time: ~500ms**

### ✅ Multiple EXISTS
```sql
SELECT e.name
FROM employees e
WHERE EXISTS (SELECT 1 FROM projects p WHERE p.employee_id = e.id)
  AND EXISTS (SELECT 1 FROM certifications c WHERE c.employee_id = e.id)
  AND EXISTS (SELECT 1 FROM reviews r WHERE r.employee_id = e.id);
```

**What happens:**
```
For each employee:
  Check projects → Found? TRUE (stop checking projects)
  Check certifications → Found? TRUE (stop checking certs)
  Check reviews → Found? TRUE (stop checking reviews)
  All TRUE → Include employee
```

**Cost:** 3 quick checks per employee (short-circuit)  
**Time: ~40ms**

**Winner: EXISTS (12x faster)**

---

## Scenario 4: One-to-Many with Large "Many"

**Task:** Find customers who made at least one purchase

### Setup
- 10,000 customers
- 500,000 orders
- Average 50 orders per customer
- Some customers have 1000+ orders

### ❌ JOIN
```sql
SELECT DISTINCT c.name
FROM customers c
JOIN orders o ON o.customer_id = c.id;
```

**What happens:**
```
Customer 1: 1000 orders → 1000 rows
Customer 2: 50 orders → 50 rows
Customer 3: 500 orders → 500 rows
...
Total: 500,000 rows created
DISTINCT: Reduce back to 10,000 customers
```

**Cost:** Process 500,000 rows + DISTINCT on 500k rows  
**Time: ~800ms**

### ✅ EXISTS
```sql
SELECT c.name
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

**What happens:**
```
Customer 1 (1000 orders): Check 1st order → Found! STOP
Customer 2 (50 orders): Check 1st order → Found! STOP
Customer 3 (500 orders): Check 1st order → Found! STOP
...
Total: ~10,000 checks (one per customer)
```

**Cost:** 10,000 quick checks  
**Time: ~25ms**

**Winner: EXISTS (32x faster!)**

---

## Scenario 5: NOT EXISTS (Finding Missing Records)

**Task:** Find customers who have NEVER made a purchase

### ❌ LEFT JOIN
```sql
SELECT c.name
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.id IS NULL;
```

**What happens:**
- Performs LEFT JOIN with all 500,000 orders
- Creates massive result set
- Filters for NULL orders

**Cost:** Full LEFT JOIN scan  
**Time: ~600ms**

### ✅ NOT EXISTS
```sql
SELECT c.name
FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

**What happens:**
```
Customer 1: Check for any order → Found → Exclude (stop)
Customer 2: Check for any order → Found → Exclude (stop)
Customer 3: Check for any order → None found → Include
```

**Cost:** Quick existence checks  
**Time: ~30ms**

**Winner: NOT EXISTS (20x faster)**

---

## Scenario 6: Skewed Data Distribution

**Task:** Find products that are in stock (inventory > 0)

### Setup
- 100,000 products
- 80% have inventory (80,000)
- 20% out of stock (20,000)
- Inventory table has multiple warehouse entries per product

### ❌ JOIN
```sql
SELECT DISTINCT p.name
FROM products p
JOIN inventory i ON i.product_id = p.id
WHERE i.quantity > 0;
```

**What happens:**
```
Product with inventory in 10 warehouses → 10 rows
80,000 products × avg 8 warehouses = 640,000 rows
Then DISTINCT reduces to 80,000 products
```

**Cost:** Process 640,000 rows + DISTINCT  
**Time: ~900ms**

### ✅ EXISTS
```sql
SELECT p.name
FROM products p
WHERE EXISTS (SELECT 1 FROM inventory i 
              WHERE i.product_id = p.id AND i.quantity > 0);
```

**What happens:**
```
For each product:
  Check first warehouse → Has stock? STOP
  Average: 1-2 checks per product (most have stock in first warehouse)
Total: ~120,000 checks
```

**Cost:** 120,000 quick checks  
**Time: ~80ms**

**Winner: EXISTS (11x faster)**

---

## Performance Summary Table

| Scenario | JOIN Time | EXISTS Time | Speedup | Why EXISTS Wins |
|----------|-----------|-------------|---------|-----------------|
| Existence check | 45ms | 8ms | **5.6x** | No duplicates to remove |
| High earner search | 35ms | 12ms | **2.9x** | Short-circuit on match |
| Multiple conditions | 500ms | 40ms | **12.5x** | Avoids cartesian product |
| Many orders per customer | 800ms | 25ms | **32x** | Stops at first order |
| Finding missing records | 600ms | 30ms | **20x** | More efficient anti-join |
| Skewed inventory | 900ms | 80ms | **11x** | Early short-circuit |

---

## When JOIN Wins Instead

EXISTS is NOT always better. Use JOIN when:

### ✅ Need Data from Both Tables
```sql
-- Need customer name AND order total
SELECT c.name, o.total
FROM customers c
JOIN orders o ON o.customer_id = c.id;
```
Can't use EXISTS here - you need the order data!

### ✅ One-to-One or Many-to-One
```sql
-- Each employee has exactly one department
SELECT e.name, d.name
FROM employees e
JOIN departments d ON e.department_id = d.id;
```
No duplicates created, JOIN is optimal

### ✅ Need to Aggregate Join Results
```sql
-- Total orders per customer
SELECT c.name, COUNT(o.id)
FROM customers c
JOIN orders o ON o.customer_id = c.id
GROUP BY c.name;
```
EXISTS can't count - need actual rows

---

## Key Principles

**EXISTS wins when:**
1. ✅ Only checking IF records exist (boolean check)
2. ✅ One-to-many relationship (avoids duplicates)
3. ✅ Early matches likely (short-circuit benefit)
4. ✅ Don't need data from related table
5. ✅ Multiple independent existence checks

**JOIN wins when:**
1. ✅ Need columns from both tables
2. ✅ One-to-one relationships
3. ✅ Aggregating related data
4. ✅ Complex multi-table queries
5. ✅ Processing most rows anyway

---

## Rule of Thumb

**Ask yourself:** "Do I need the related data, or just to know if it exists?"

- **Need data** → Use JOIN
- **Just existence** → Use EXISTS

**The golden question:** Would I need DISTINCT after my JOIN?
- **YES** → Use EXISTS instead (probably 5-10x faster)
- **NO** → JOIN is fine