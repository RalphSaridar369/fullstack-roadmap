# 🔹 SQL DISTINCT Clause

The `DISTINCT` clause is used to **remove duplicate rows** from the results of a SQL query. It is commonly used in **reporting, analytics, and when querying unique values**.

---

## 1️⃣ Basic Syntax

```sql
SELECT DISTINCT column_name
FROM table_name;
```

- Returns **unique values** in `column_name`
- Can be applied to **one or multiple columns**

---

## 2️⃣ Examples

### a) Single Column

Suppose we have a **customers table**:

| id  | country |
|-----|---------|
| 1   | USA     |
| 2   | USA     |
| 3   | UK      |
| 4   | UK      |
| 5   | France  |

Query for unique countries:

```sql
SELECT DISTINCT country
FROM customers;
```

### Result

| country |
|---------|
| USA     |
| UK      |
| France  |

---

### b) Multiple Columns

If you want unique combinations of columns:

```sql
SELECT DISTINCT country, id
FROM customers;
```

- Only **rows with unique (country, id) pairs** are returned

---

### c) Using DISTINCT with COUNT

To **count unique values** in a column:

```sql
SELECT COUNT(DISTINCT country) AS unique_countries
FROM customers;
```

### Result

| unique_countries |
|-----------------|
| 3               |

---

### d) DISTINCT with GROUP BY and HAVING

You can combine `DISTINCT` with `GROUP BY` and `HAVING` for advanced filtering.

Suppose we have an **orders table**:

| id  | country | amount |
|-----|---------|--------|
| 1   | USA     | 100    |
| 2   | USA     | 50     |
| 3   | UK      | 200    |
| 4   | UK      | 150    |
| 5   | USA     | 75     |

We want **countries with more than 2 orders**, counting only **distinct amounts**:

```sql
SELECT country, COUNT(DISTINCT amount) AS distinct_order_amounts
FROM orders
GROUP BY country
HAVING COUNT(DISTINCT amount) > 2;
```

### Result

| country | distinct_order_amounts |
|---------|-----------------------|
| USA     | 3                     |

---

### ⚠️ Notes and Common Mistakes

- ❌ `DISTINCT` applies to **all selected columns**, not just one, when multiple columns are selected
- ❌ Avoid using `DISTINCT` unnecessarily on large tables (can impact performance)
- ✅ Can be combined with `ORDER BY`, `WHERE`, `GROUP BY`, and `HAVING`

---

This is useful when you need **clean, non-redundant datasets** for analysis or reporting.

