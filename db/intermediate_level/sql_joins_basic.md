# SQL Joins – Basic Overview

SQL **JOINs** are used to combine rows from two or more tables based on a related column.

---

## The 4 Basic SQL Joins

### 1. INNER JOIN
Returns only rows that have matching values in **both tables**.

```sql
SELECT *
FROM orders o
INNER JOIN users u ON o.user_id = u.id;
```

---

### 2. LEFT JOIN (LEFT OUTER JOIN)
Returns **all rows from the left table** and matching rows from the right table.
Unmatched rows from the right table return `NULL`.

```sql
SELECT *
FROM users u
LEFT JOIN orders o ON o.user_id = u.id;
```

---

### 3. RIGHT JOIN (RIGHT OUTER JOIN)
Returns **all rows from the right table** and matching rows from the left table.
Unmatched rows from the left table return `NULL`.

```sql
SELECT *
FROM orders o
RIGHT JOIN users u ON o.user_id = u.id;
```

---

### 4. FULL JOIN (FULL OUTER JOIN)
Returns **all rows from both tables**.
Unmatched rows on either side return `NULL`.

```sql
SELECT *
FROM users u
FULL JOIN orders o ON o.user_id = u.id;
```

---

## Summary

- SQL has **4 basic joins**:
  - INNER JOIN
  - LEFT JOIN
  - RIGHT JOIN
  - FULL JOIN

These joins form the foundation for most relational database queries.

---

## Notes

- `LEFT JOIN` is more commonly used than `RIGHT JOIN`
- `FULL JOIN` is not supported in some databases (e.g. MySQL)
- Other joins like `CROSS JOIN` and `SELF JOIN` exist but are not considered basic

