# 🗄️ SQL Aggregation Functions

SQL aggregation functions allow you to **summarize or compute values** over a set of rows.  
They are widely used in **reporting, analytics, and dashboards**.

---

## 1️⃣ Core Aggregation Functions

| Function   | What It Does | Example |
|-----------|--------------|---------|
| `SUM()`   | Adds numeric values | `SUM(amount)` |
| `COUNT()` | Counts rows or non-null values | `COUNT(*)` or `COUNT(id)` |
| `AVG()`   | Computes average of numeric values | `AVG(amount)` |
| `MIN()`   | Finds the minimum value | `MIN(price)` |
| `MAX()`   | Finds the maximum value | `MAX(price)` |

---

## 2️⃣ GROUP BY Examples

### a) Single-column GROUP BY

Suppose we have this **orders table**:

| id  | country | amount |
|-----|---------|--------|
| 1   | USA     | 100    |
| 2   | USA     | 50     |
| 3   | UK      | 200    |
| 4   | UK      | 150    |

We want the **total sales per country**:

SELECT country, SUM(amount) AS total_sales
FROM orders
GROUP BY country;

### b) Multi-column GROUP BY

SELECT country, category, SUM(amount) AS total_sales
FROM orders
GROUP BY country, category;