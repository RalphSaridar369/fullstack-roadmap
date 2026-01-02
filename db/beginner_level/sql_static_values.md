# 🔹 SQL Static Values

In SQL, **static values** (also called literals or constants) are values that are **directly written into a query** instead of coming from a table or variable. They are useful for testing, default values, or when you need **fixed criteria** in queries.

---

## 1️⃣ What Are Static Values?

Examples of static values:
- Numbers: `100`, `3.14`
- Strings: `'USA'`, `'Alice'`
- Dates: `'2026-01-02'`
- Booleans: `TRUE`, `FALSE`

These values are **hard-coded** into the SQL statement.

```sql
SELECT *
FROM customers
WHERE country = 'USA';
```
- `'USA'` is a **static value**.

---

## 2️⃣ Why Use Static Values?

### a) Filtering
Use static values to **filter rows** directly:
```sql
SELECT *
FROM orders
WHERE status = 'completed';
```

### b) Default Values
Insert rows with predefined static values:
```sql
INSERT INTO customers (name, country, active)
VALUES ('Alice', 'USA', TRUE);
```

### c) Testing Queries
Static values allow you to **quickly test queries** without relying on dynamic data.

### d) Aggregation / Comparison
You can compare columns with fixed static thresholds:
```sql
SELECT *
FROM orders
WHERE amount > 100;
```

### e) Placeholder for Business Logic
Sometimes static values represent **constants in business logic**, e.g., tax rate, discount, or status codes.

```sql
SELECT *
FROM invoices
WHERE tax_rate = 0.2; -- 20% tax
```

---

## 3️⃣ Notes / Best Practices

- ✅ Use static values for **fixed criteria or defaults**
- ✅ Useful in **WHERE, SELECT, INSERT, UPDATE** clauses
- ❌ Avoid using static values excessively for production data if they should be **dynamic or configurable**
- ❌ For user inputs, prefer **parameters or prepared statements** to prevent SQL injection

---

Static values are simple but powerful for **testing, defaults, and fixed business rules** in SQL queries.

