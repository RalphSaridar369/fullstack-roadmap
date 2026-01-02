# 🔹 SQL Query Execution Order (with Joins)

Understanding the **logical execution order** of SQL clauses is crucial for writing correct and efficient queries. This includes `SELECT DISTINCT`, `FROM` (with joins), `WHERE`, `GROUP BY`, `HAVING`, `ORDER BY`, `LIMIT`, and `OFFSET`.

---

## 1️⃣ Logical Execution Order with Joins

1. **FROM / JOIN** → determine the source tables and perform any joins
2. **WHERE** → filter rows **before aggregation**
3. **GROUP BY** → group the filtered rows
4. **HAVING** → filter groups **after aggregation**
5. **SELECT** → choose columns, compute expressions, apply `DISTINCT` if used
6. **ORDER BY** → sort the result set
7. **LIMIT / OFFSET** → restrict the number of rows returned

> Note: `DISTINCT` is applied **after aggregation and selection**, not before `WHERE` or `GROUP BY`.

---

## 2️⃣ Example with JOIN

Suppose we have two tables:

**customers**
| id | name    | country |
|----|---------|---------|
| 1  | Alice   | USA     |
| 2  | Bob     | USA     |
| 3  | Charlie | UK      |

**orders**
| id | customer_id | amount |
|----|-------------|--------|
| 1  | 1           | 100    |
| 2  | 1           | 50     |
| 3  | 2           | 200    |
| 4  | 3           | 150    |

Query to get **total order amount per country** for orders > 50, sorted descending:

```sql
SELECT DISTINCT c.country, SUM(o.amount) AS total_sales
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE o.amount > 50
GROUP BY c.country
HAVING SUM(o.amount) > 100
ORDER BY total_sales DESC
LIMIT 2 OFFSET 0;
```

### Logical Execution Steps:
1. **FROM / JOIN**: join `customers` and `orders` on `customer_id`

| customer_id | name    | country | order_id | amount |
|-------------|---------|---------|----------|--------|
| 1           | Alice   | USA     | 1        | 100    |
| 1           | Alice   | USA     | 2        | 50     |
| 2           | Bob     | USA     | 3        | 200    |
| 3           | Charlie | UK      | 4        | 150    |

2. **WHERE**: filter orders with `amount > 50`

| customer_id | name    | country | order_id | amount |
|-------------|---------|---------|----------|--------|
| 1           | Alice   | USA     | 1        | 100    |
| 2           | Bob     | USA     | 3        | 200    |
| 3           | Charlie | UK      | 4        | 150    |

3. **GROUP BY**: group by `country`

| country | total_sales |
|---------|-------------|
| USA     | 300         |
| UK      | 150         |

4. **HAVING**: filter groups with `SUM(amount) > 100`

| country | total_sales |
|---------|-------------|
| USA     | 300         |
| UK      | 150         |

5. **SELECT + DISTINCT**: select columns, remove duplicates (redundant here because `GROUP BY` ensures uniqueness)
6. **ORDER BY**: sort by `total_sales DESC`
7. **LIMIT / OFFSET**: return first 2 rows starting from offset 0

### Result:

| country | total_sales |
|---------|-------------|
| USA     | 300         |
| UK      | 150         |

---

## 3️⃣ Notes

- `JOIN`s happen **first**, before `WHERE` filtering
- `DISTINCT` is often **redundant** if `GROUP BY` is used
- `WHERE` filters **rows before grouping**, `HAVING` filters **groups after aggregation**
- `ORDER BY` and `LIMIT/OFFSET` are applied **last**

---

### Quick Reference Table with Joins

| Step | Clause | Purpose |
|------|--------|---------|
| 1    | FROM / JOIN | Select tables and join them |
| 2    | WHERE  | Filter rows before aggregation |
| 3    | GROUP BY | Group rows by column(s) |
| 4    | HAVING | Filter groups after aggregation |
| 5    | SELECT (+ DISTINCT) | Choose columns and remove duplicates |
| 6    | ORDER BY | Sort the result set |
| 7    | LIMIT / OFFSET | Restrict number of rows returned |

---

Understanding the **execution order with joins** helps in writing accurate, optimized SQL queries for **aggregation, filtering, and pagination**.

