# 🔹 SQL DDL (Data Definition Language)

**DDL (Data Definition Language)** is a subset of SQL used to **define, modify, and manage database structures** such as tables, indexes, and schemas. Unlike DML (Data Manipulation Language), DDL **does not manipulate data directly**, but changes the structure of the database itself.

---

## 1️⃣ Common DDL Statements

| Statement | Purpose | Example |
|-----------|---------|---------|
| `CREATE`  | Create a new database, table, index, or view | `CREATE TABLE customers (id INT PRIMARY KEY, name VARCHAR(50));` |
| `ALTER`   | Modify an existing database object | `ALTER TABLE customers ADD COLUMN email VARCHAR(100);` |
| `DROP`    | Delete a database object | `DROP TABLE customers;` |
| `TRUNCATE` | Remove all rows from a table (resets storage) | `TRUNCATE TABLE orders;` |
| `RENAME`  | Change the name of a database object | `ALTER TABLE customers RENAME TO clients;` |

---

## 2️⃣ CREATE TABLE Example

```sql
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) DEFAULT 0.0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

- `PRIMARY KEY` → unique identifier for rows
- `DEFAULT` → default value if none is provided
- `NOT NULL` → column must have a value

---

## 3️⃣ ALTER TABLE Example

```sql
-- Add a new column
ALTER TABLE products ADD COLUMN stock INT DEFAULT 0;

-- Modify column type
ALTER TABLE products ALTER COLUMN price TYPE DECIMAL(12,2);

-- Drop a column
ALTER TABLE products DROP COLUMN stock;
```

---

## 4️⃣ DROP TABLE / DATABASE Example

```sql
-- Drop a table
DROP TABLE products;

-- Drop a database
DROP DATABASE shop;
```

> ⚠️ Be careful: `DROP` is **destructive** and cannot be undone.

---

## 5️⃣ TRUNCATE Example

```sql
TRUNCATE TABLE orders;
```
- Removes all rows **quickly** without logging individual deletions
- Resets auto-increment counters in many databases

---

## 6️⃣ Notes / Best Practices

- DDL statements often require **higher privileges** than DML
- Some DDL changes (like `ALTER TABLE`) can **lock tables**, so plan accordingly
- Always **backup** critical tables before destructive operations like `DROP` or `TRUNCATE`
- DDL changes can trigger **cascading effects** (e.g., dropping a table with foreign keys)

---

DDL is fundamental for **creating and maintaining the structure of a database**, forming the foundation for all data manipulation and querying operations.

