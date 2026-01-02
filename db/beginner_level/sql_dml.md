# DML (Data Manipulation Language) – SQL

Data Manipulation Language (DML) is a subset of SQL used to **retrieve, insert, update, and delete data** in database tables.

---

## What is DML?
DML commands work on the **data itself**, not the structure of the database. These commands are often used in applications and are usually **transactional** (can be committed or rolled back).

---

## Core DML Commands

### 1. SELECT
Used to retrieve data from one or more tables.

```sql
SELECT column1, column2
FROM table_name
WHERE condition;
```

**Example:**
```sql
SELECT id, name, email
FROM users
WHERE is_active = true;
```

---

### 2. INSERT
Used to add new records to a table.

```sql
INSERT INTO table_name (column1, column2)
VALUES (value1, value2);
```

**Example:**
```sql
INSERT INTO users (name, email)
VALUES ('John Doe', 'john@example.com');
```

---

### 3. UPDATE
Used to modify existing records.

```sql
UPDATE table_name
SET column1 = value1
WHERE condition;
```

**Example:**
```sql
UPDATE users
SET is_active = false
WHERE last_login < '2024-01-01';
```

⚠️ **Always use `WHERE` unless you want to update all rows.**

---

### 4. DELETE
Used to remove records from a table.

```sql
DELETE FROM table_name
WHERE condition;
```

**Example:**
```sql
DELETE FROM users
WHERE id = 10;
```

⚠️ Without `WHERE`, all rows will be deleted.

---

## Transactions in DML
DML statements can be wrapped in transactions.

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1;

UPDATE accounts
SET balance = balance + 100
WHERE id = 2;

COMMIT;
-- or ROLLBACK;
```

---

## DML vs Other SQL Categories

| Category | Purpose |
|--------|--------|
| DML | Manipulate data (SELECT, INSERT, UPDATE, DELETE) |
| DDL | Define structure (CREATE, ALTER, DROP) |
| DCL | Control access (GRANT, REVOKE) |
| TCL | Transaction control (COMMIT, ROLLBACK) |

---

## Best Practices
- Always use `WHERE` with `UPDATE` and `DELETE`
- Test with `SELECT` before destructive operations
- Use transactions for critical updates
- Limit results with `LIMIT` when exploring data

---

## Summary
DML is essential for everyday database operations. Mastery of these commands is critical for backend development, data analysis, and database administration.

---

*File format: Markdown (.md)*

