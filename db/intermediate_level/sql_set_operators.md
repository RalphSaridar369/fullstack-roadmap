# SQL Set Operators

SQL **set operators** are used to combine the results of two or more `SELECT` queries into a single result set.

They are useful when you want to compare datasets, merge results, or find differences between tables.

---

## Rules for Using Set Operators

Before using any set operator, the following rules **must be satisfied**:

1. Each `SELECT` statement must have the **same number of columns**.
2. Corresponding columns must have **compatible data types**.
3. Columns are matched **by position**, not by name.
4. The `ORDER BY` clause can only be used **once**, at the end of the final query.

---

## 1. UNION

### Description

`UNION` combines the results of two or more `SELECT` statements **and removes duplicate rows**.

### Syntax

```sql
SELECT column1, column2
FROM table1
UNION
SELECT column1, column2
FROM table2;
```

### Example

```sql
SELECT email FROM customers
UNION
SELECT email FROM suppliers;
```

**Result:** A list of unique email addresses from both tables.

---

## 2. UNION ALL

### Description

`UNION ALL` combines result sets **without removing duplicates**.

### Syntax

```sql
SELECT column1, column2
FROM table1
UNION ALL
SELECT column1, column2
FROM table2;
```

### Example

```sql
SELECT city FROM customers
UNION ALL
SELECT city FROM suppliers;
```

**Result:** All cities from both tables, including duplicates.

### Performance Note

`UNION ALL` is **faster** than `UNION` because it does not perform duplicate elimination.

---

## 3. INTERSECT

### Description

`INTERSECT` returns only the rows that **exist in both result sets**.

> ⚠️ Not supported in MySQL (supported in PostgreSQL, Oracle, SQL Server).

### Syntax

```sql
SELECT column1
FROM table1
INTERSECT
SELECT column1
FROM table2;
```

### Example

```sql
SELECT user_id FROM orders_2024
INTERSECT
SELECT user_id FROM orders_2025;
```

**Result:** Users who placed orders in both years.

---

## 4. EXCEPT (or MINUS)

### Description

`EXCEPT` returns rows from the **first query that do not exist in the second query**.

- `EXCEPT` → PostgreSQL, SQL Server
- `MINUS` → Oracle

### Syntax

```sql
SELECT column1
FROM table1
EXCEPT
SELECT column1
FROM table2;
```

### Example

```sql
SELECT product_id FROM products
EXCEPT
SELECT product_id FROM discontinued_products;
```

**Result:** Products that are not discontinued.

---

## NULL Handling

- Set operators treat `NULL` values as **equal** when comparing rows.
- This behavior differs from normal `WHERE` comparisons.

---

## Using ORDER BY with Set Operators

You can only use `ORDER BY` **once**, after the final query:

```sql
SELECT name FROM employees
UNION
SELECT name FROM contractors
ORDER BY name;
```

---

## Summary Table

| Operator     | Removes Duplicates | Description |
|-------------|-------------------|-------------|
| UNION       | ✅ Yes            | Combines results uniquely |
| UNION ALL   | ❌ No             | Combines all results |
| INTERSECT   | ✅ Yes            | Common rows only |
| EXCEPT      | ✅ Yes            | Rows in first, not second |

---

## When to Use Set Operators

- Merging similar datasets
- Comparing data between tables
- Finding overlaps or differences
- Reporting and analytics

---

## Tip

If your database does not support `INTERSECT` or `EXCEPT`, similar results can be achieved using `JOIN`, `IN`, or `NOT IN` clauses.

---

Happy querying 🚀

