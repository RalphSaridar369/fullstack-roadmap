# 🔹 SQL LIMIT and OFFSET

The `LIMIT` and `OFFSET` clauses are used to **control the number of rows returned** by a query and to **skip a certain number of rows**. These are commonly used for **pagination** in applications.

---

## 1️⃣ Basic Syntax

```sql
SELECT column1, column2
FROM table_name
LIMIT row_count OFFSET skip_count;
```

- `LIMIT row_count` → number of rows to return
- `OFFSET skip_count` → number of rows to skip before starting to return rows
- In some databases (e.g., MySQL, PostgreSQL), you can also write:

```sql
SELECT column1, column2
FROM table_name
LIMIT skip_count, row_count; -- MySQL style
```

---

## 2️⃣ Examples

Suppose we have a **customers table**:

| id  | name     |
|-----|----------|
| 1   | Alice    |
| 2   | Bob      |
| 3   | Charlie  |
| 4   | Diana    |
| 5   | Edward   |

### a) Using LIMIT

Return the first 3 rows:

```sql
SELECT *
FROM customers
LIMIT 3;
```

### Result

| id  | name    |
|-----|---------|
| 1   | Alice   |
| 2   | Bob     |
| 3   | Charlie |

---

### b) Using LIMIT with OFFSET

Skip the first 2 rows and return the next 2:

```sql
SELECT *
FROM customers
LIMIT 2 OFFSET 2;
```

### Result

| id  | name    |
|-----|---------|
| 3   | Charlie |
| 4   | Diana   |

---

### c) MySQL-style LIMIT with offset

Equivalent query in MySQL shorthand:

```sql
SELECT *
FROM customers
LIMIT 2, 2; -- skip 2, return 2
```

### Result

| id  | name    |
|-----|---------|
| 3   | Charlie |
| 4   | Diana   |

---

### ⚠️ Notes and Best Practices

- ✅ Commonly used for **pagination** in web apps
- ❌ Avoid using without `ORDER BY` if **consistent ordering is required**
- ✅ Works with **any SELECT query** including joins and subqueries
- Some databases (like SQL Server) use `TOP` or `OFFSET FETCH NEXT` instead of `LIMIT/OFFSET`

```sql
-- SQL Server example
SELECT *
FROM customers
ORDER BY id
OFFSET 2 ROWS FETCH NEXT 2 ROWS ONLY;
```

---

Using `LIMIT` and `OFFSET` makes queries more efficient when dealing with **large datasets** or when implementing **paginated results**.

