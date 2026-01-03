# SQL WHERE Operators – Brief Reference

This document lists the **most common operators used inside a `WHERE` clause** to filter rows in SQL.

---

## Comparison Operators
Used to compare values.

- `=` equal
- `!=` or `<>` not equal
- `>` greater than
- `<` less than
- `>=` greater than or equal
- `<=` less than or equal

```sql
WHERE age >= 18
```

---

## Logical Operators
Used to combine conditions.

- `AND`
- `OR`
- `NOT`

```sql
WHERE age > 18 AND country = 'Lebanon'
```

---

## Set & Range Operators

- `IN`, `NOT IN`
- `BETWEEN`, `NOT BETWEEN`

```sql
WHERE status IN ('ACTIVE', 'PENDING')
```

---

## NULL Operators
Used to check missing values.

- `IS NULL`
- `IS NOT NULL`

```sql
WHERE deleted_at IS NULL
```

---

## Pattern Matching Operators
Used with strings.

- `LIKE`, `NOT LIKE`
- `ILIKE` (PostgreSQL, case-insensitive)

Wildcards:
- `%` any characters
- `_` single character

```sql
WHERE email LIKE '%@gmail.com'
```

---

## Subquery Operators

- `EXISTS`, `NOT EXISTS`
- `ANY`, `ALL`

```sql
WHERE salary > ALL (
  SELECT salary FROM employees WHERE dept = 'HR'
)
```

---

## Operator Precedence

1. `()`
2. `NOT`
3. `AND`
4. `OR`

```sql
WHERE (a = 1 OR b = 2) AND c = 3
```

---

## Summary

Commonly used in `WHERE` clauses:
- Comparison
- Logical
- Set & Range
- NULL checks
- Pattern matching
- Subqueries

