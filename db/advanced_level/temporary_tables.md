# Temporary Tables (TMP Tables)

## 1. Definition

A **temporary table** is a table that exists only for a limited scope
(session or transaction) and is used to store intermediate data during
query execution or data processing.

Key properties:

-   Created dynamically at runtime.
-   Automatically dropped after scope ends.
-   Isolated from permanent schema.
-   Optimized for intermediate computations.

------------------------------------------------------------------------

## 2. Scope & Lifetime

### 2.1 Session-Level Temporary Tables

-   Visible only within the current database session.
-   Dropped when the session ends.

### 2.2 Transaction-Level Temporary Tables

-   Exist only during a transaction.
-   Dropped or cleared at transaction commit/rollback.

### 2.3 Global Temporary Tables (DB-specific)

-   Table structure persists.
-   Data is temporary and session/transaction scoped.

------------------------------------------------------------------------

## 3. Syntax by Database Engine

### 3.1 PostgreSQL

``` sql
CREATE TEMP TABLE temp_users (
    id INT,
    name TEXT
);
```

With transaction behavior:

``` sql
CREATE TEMP TABLE temp_users (
    id INT,
    name TEXT
) ON COMMIT DROP;
```

Options:

-   `ON COMMIT DROP` → drop table on commit
-   `ON COMMIT DELETE ROWS` → clear rows on commit
-   `ON COMMIT PRESERVE ROWS` → keep rows until session ends

------------------------------------------------------------------------

### 3.2 MySQL

``` sql
CREATE TEMPORARY TABLE temp_users (
    id INT,
    name VARCHAR(255)
);
```

Notes:

-   Dropped when connection closes.
-   Same name as permanent table hides the permanent table in the
    session.

------------------------------------------------------------------------

### 3.3 SQL Server

Local temporary table:

``` sql
CREATE TABLE #temp_users (
    id INT,
    name NVARCHAR(255)
);
```

Global temporary table:

``` sql
CREATE TABLE ##temp_users (
    id INT,
    name NVARCHAR(255)
);
```

------------------------------------------------------------------------

### 3.4 Oracle

``` sql
CREATE GLOBAL TEMPORARY TABLE temp_users (
    id NUMBER,
    name VARCHAR2(255)
) ON COMMIT DELETE ROWS;
```

------------------------------------------------------------------------

## 4. Execution Model (How Temporary Tables Work)

### 4.1 Creation Phase

1.  Metadata allocated in temporary schema.
2.  Storage allocated in memory or disk-based temp space.
3.  Table registered in session context.

### 4.2 Data Population Phase

``` sql
CREATE TEMP TABLE temp_users AS
SELECT * FROM users WHERE active = true;
```

-   Data is materialized.
-   Query planner optimizes access paths.
-   Indexes may be created.

### 4.3 Query Phase

``` sql
SELECT COUNT(*) FROM temp_users;
```

Differences:

-   No cross-session visibility (usually).
-   Reduced locking overhead.
-   Faster repeated access.

### 4.4 Destruction Phase

Temp tables are dropped when:

-   Session ends
-   Transaction ends
-   Explicit `DROP TABLE` is executed

------------------------------------------------------------------------

## 5. Internal Storage & Performance

### 5.1 Storage Location

  Database     Temp Storage
  ------------ ---------------------------
  PostgreSQL   pg_temp schema
  MySQL        /tmp or InnoDB temp space
  SQL Server   tempdb
  Oracle       Temporary tablespaces

### 5.2 Performance Characteristics

Advantages:

-   Avoid repeated computation.
-   Reduce query complexity.
-   Allow indexing of intermediate results.

Costs:

-   Memory usage.
-   Disk I/O if spilled to disk.
-   Creation and deletion overhead.

------------------------------------------------------------------------

## 6. Comparison with Other Constructs

### 6.1 Temporary Tables vs CTE

  Feature           Temporary Table                CTE
  ----------------- ------------------------------ -------------------------
  Scope             Session / Transaction          Single query
  Materialization   Yes                            Usually no
  Reusability       Yes                            No
  Indexing          Yes                            No
  Performance       Better for large/reused data   Better for simple logic

### 6.2 Temporary Tables vs Views

  Feature        Temporary Table     View
  -------------- ------------------- ----------------
  Persistence    Temporary           Permanent
  Data Storage   Yes                 No
  Use Case       Intermediate data   Reusable logic

------------------------------------------------------------------------

## 7. When to Use Temporary Tables

### 7.1 Complex Query Decomposition

``` sql
CREATE TEMP TABLE tmp_orders AS
SELECT * FROM orders WHERE total > 1000;

SELECT customer_id, SUM(total)
FROM tmp_orders
GROUP BY customer_id;
```

Use when:

-   Query readability matters.
-   Intermediate results are reused.

### 7.2 Performance Optimization

-   Heavy joins reused multiple times.
-   Large aggregations.

### 7.3 ETL and Data Pipelines

-   Extract → Transform → Load workflows.

### 7.4 Data Migration & Scripts

``` sql
CREATE TEMP TABLE tmp_users AS
SELECT id FROM users WHERE status = 'inactive';

UPDATE users
SET status = 'active'
WHERE id IN (SELECT id FROM tmp_users);
```

### 7.5 Business Logic Processing

-   Ranking, scoring, filtering.

### 7.6 Analytics & Reporting

-   Precomputing metrics.
-   Cohort analysis.
-   Reporting.

------------------------------------------------------------------------

## 8. Best Practices

-   Use naming conventions: `tmp_`, `temp_`, `stg_`.
-   Drop temp tables explicitly in long sessions.
-   Index only when necessary.
-   Monitor temp space usage.

------------------------------------------------------------------------

## 9. Conceptual Summary

Temporary tables are materialized intermediate datasets used to optimize
performance and structure complex database operations.

------------------------------------------------------------------------

## 10. Quick Decision Matrix

  Use Case                   Recommended Approach
  -------------------------- ----------------------
  Simple transformation      CTE
  One-time logic             Subquery
  Reused heavy dataset       Temporary Table
  Permanent reusable logic   View
  Cached query result        Materialized View
