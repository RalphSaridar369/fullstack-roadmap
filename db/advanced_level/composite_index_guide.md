# Composite Indexes: Why and When to Use Them

## What is a Composite Index?

A composite index (also called a multi-column or compound index) is a database index that includes two or more columns from a table. Unlike a single-column index that organizes data by one field, a composite index creates a sorted structure based on multiple columns in a specific order.

**Single-Column Index:**
```sql
CREATE INDEX idx_last_name ON employees(last_name);
```

**Composite Index:**
```sql
CREATE INDEX idx_name ON employees(last_name, first_name);
```

## How Composite Indexes Work

### Internal Structure

A composite index organizes data hierarchically by column order. Think of it like a phone book sorted first by last name, then by first name within each last name.

**Example Table:**
```sql
employees (id, first_name, last_name, department, hire_date, salary)
```

**Composite Index on (last_name, first_name):**
```
Anderson, Alice
Anderson, Bob
Anderson, Carol
Brown, David
Brown, Emma
Chen, Frank
Chen, Grace
...
```

The index first sorts by `last_name`, then within each last name group, it sorts by `first_name`.

### Column Order Matters

The order of columns in a composite index is critical. An index on `(A, B, C)` is **not** the same as an index on `(C, B, A)`.

**Key Principle**: A composite index can be used for queries that filter on:
- The first column alone
- The first and second columns
- The first, second, and third columns
- **But NOT** the second or third column alone (efficiently)

This is called the **leftmost prefix rule**.

## Why Use Composite Indexes?

### 1. Accelerate Multi-Column Queries

**Without Composite Index:**
```sql
SELECT * FROM employees 
WHERE last_name = 'Smith' AND first_name = 'John';
```
- Uses only the `last_name` index
- Must scan all 'Smith' rows to find 'John'
- If there are 1,000 Smiths, scans 1,000 rows

**With Composite Index on (last_name, first_name):**
- Directly locates 'Smith, John'
- Scans only matching rows
- Dramatic performance improvement

### 2. Cover Queries Entirely (Covering Index)

A composite index can include all columns needed by a query, eliminating the need to access the table data.

```sql
-- Index: (department, salary, hire_date)
SELECT salary, hire_date 
FROM employees 
WHERE department = 'Engineering';
```

This query can be satisfied entirely from the index without touching the table (index-only scan).

### 3. Enable Efficient Sorting

Composite indexes provide pre-sorted data for ORDER BY clauses.

```sql
-- Index: (department, hire_date)
SELECT * FROM employees 
WHERE department = 'Sales'
ORDER BY hire_date;
```

The database can read directly from the index in sorted order, avoiding an expensive sort operation.

### 4. Optimize Range Queries

```sql
-- Index: (status, created_date)
SELECT * FROM orders 
WHERE status = 'pending' 
  AND created_date >= '2024-01-01';
```

The index first filters to 'pending' status, then performs a range scan on dates within that subset.

### 5. Support Unique Constraints

```sql
CREATE UNIQUE INDEX idx_email_tenant 
ON users(email, tenant_id);
```

Ensures email is unique within each tenant, but allows the same email across different tenants.

## When to Use Composite Indexes

### Scenario 1: Frequent Multi-Column Filters

**Use Case:** E-commerce order tracking

```sql
-- Common query pattern
SELECT * FROM orders 
WHERE customer_id = ? AND status = ?;

-- Optimal index
CREATE INDEX idx_customer_status ON orders(customer_id, status);
```

**Why It Works:**
- Quickly narrows to specific customer
- Then filters by status within that customer's orders
- Avoids scanning all orders

### Scenario 2: Covering Common Analytical Queries

**Use Case:** Sales reporting

```sql
-- Frequent report query
SELECT region, SUM(amount), COUNT(*) 
FROM sales 
WHERE sale_date >= '2024-01-01'
GROUP BY region;

-- Optimal index
CREATE INDEX idx_date_region_amount ON sales(sale_date, region, amount);
```

**Why It Works:**
- Filters by date range (leftmost column)
- Groups by region (second column)
- Includes amount for aggregation
- Completely avoids table access (covering index)

### Scenario 3: Enforcing Complex Uniqueness

**Use Case:** Multi-tenant SaaS application

```sql
-- Ensure usernames are unique per tenant
CREATE UNIQUE INDEX idx_tenant_username 
ON users(tenant_id, username);
```

**Why It Works:**
- Allows same username across different tenants
- Prevents duplicates within a tenant
- Efficiently enforces business rule at database level

### Scenario 4: Optimizing Sorted Pagination

**Use Case:** Search results with pagination

```sql
-- Paginated query
SELECT * FROM products 
WHERE category = 'Electronics'
ORDER BY price DESC, product_id
LIMIT 20 OFFSET 100;

-- Optimal index
CREATE INDEX idx_category_price_id ON products(category, price DESC, product_id);
```

**Why It Works:**
- Filters by category
- Pre-sorted by price (descending)
- Tie-breaker on product_id for stable sorting
- Efficient offset navigation

### Scenario 5: Range Queries with Equality Filters

**Use Case:** Event logging system

```sql
-- Query pattern
SELECT * FROM logs 
WHERE application = 'web-app' 
  AND severity = 'ERROR'
  AND timestamp BETWEEN '2024-01-01' AND '2024-01-31';

-- Optimal index
CREATE INDEX idx_app_severity_time 
ON logs(application, severity, timestamp);
```

**Why It Works:**
- Equality filters (application, severity) come first
- Range filter (timestamp) comes last
- Maximizes index efficiency

## Column Order Strategy

### The Golden Rule

**Order columns from most selective to least selective, with these exceptions:**

1. **Equality filters before range filters**
2. **Commonly queried columns first**
3. **GROUP BY/ORDER BY columns considered**

### Example: Choosing Column Order

**Table:** `products (id, category, brand, price, stock_quantity, created_date)`

**Query Pattern:**
```sql
SELECT * FROM products
WHERE category = ?
  AND price BETWEEN ? AND ?
  AND stock_quantity > 0
ORDER BY created_date DESC;
```

**Analysis:**
- `category`: Equality filter, moderate selectivity
- `price`: Range filter
- `stock_quantity`: Range filter
- `created_date`: Sort column

**Optimal Index:**
```sql
CREATE INDEX idx_products_optimal 
ON products(category, stock_quantity, price, created_date DESC);
```

**Reasoning:**
1. `category` first (equality filter, most selective)
2. `stock_quantity` second (more selective than price range)
3. `price` third (range filter)
4. `created_date DESC` last (supports sorting)

### Leftmost Prefix Examples

**Index:** `(A, B, C)`

**Queries That Can Use This Index:**
```sql
WHERE A = 1                          -- ✓ Uses index
WHERE A = 1 AND B = 2                -- ✓ Uses index
WHERE A = 1 AND B = 2 AND C = 3      -- ✓ Uses index
WHERE A = 1 AND C = 3                -- ✓ Uses index (A only)
WHERE A = 1 ORDER BY B               -- ✓ Uses index
```

**Queries That CANNOT Use This Index Efficiently:**
```sql
WHERE B = 2                          -- ✗ No leftmost column
WHERE C = 3                          -- ✗ No leftmost column
WHERE B = 2 AND C = 3                -- ✗ No leftmost column
ORDER BY B                           -- ✗ No filter on A
```

## When NOT to Use Composite Indexes

### 1. Low Cardinality Leading Columns

**Bad Example:**
```sql
-- 'is_active' only has 2 values (true/false)
CREATE INDEX idx_bad ON users(is_active, email);
```

**Better Approach:**
```sql
-- Put high-cardinality column first
CREATE INDEX idx_better ON users(email, is_active);

-- Or use a partial index
CREATE INDEX idx_partial ON users(email) WHERE is_active = true;
```

### 2. Columns Rarely Queried Together

**Bad Example:**
```sql
-- These columns are never queried together
CREATE INDEX idx_wasteful ON employees(shoe_size, favorite_color, hire_date);
```

**Better Approach:**
Create separate indexes for columns that are queried independently.

### 3. Highly Volatile Columns

**Bad Example:**
```sql
-- 'last_login' changes constantly
CREATE INDEX idx_volatile ON users(username, last_login);
```

**Problem:** Every login updates the index, causing performance overhead.

**Better Approach:** Index only relatively stable columns, or use a partial index.

### 4. Too Many Columns

**Bad Example:**
```sql
-- 7 columns in one index!
CREATE INDEX idx_kitchen_sink 
ON orders(customer_id, status, region, product_id, quantity, price, date);
```

**Problems:**
- Large index size
- Slow write operations
- Diminishing returns on query performance
- High maintenance overhead

**Better Approach:** Keep composite indexes to 2-4 columns max in most cases.

## Real-World Examples

### Example 1: Social Media Platform

**Scenario:** Fetch user's recent posts

```sql
-- Query
SELECT * FROM posts
WHERE user_id = ?
ORDER BY created_at DESC
LIMIT 20;

-- Index
CREATE INDEX idx_user_posts ON posts(user_id, created_at DESC);
```

**Benefits:**
- Instantly locates user's posts
- Pre-sorted by date
- Efficient pagination

### Example 2: Multi-Tenant SaaS

**Scenario:** Tenant isolation with search

```sql
-- Query
SELECT * FROM documents
WHERE tenant_id = ?
  AND status = 'published'
  AND category = ?
ORDER BY updated_at DESC;

-- Index
CREATE INDEX idx_tenant_docs 
ON documents(tenant_id, status, category, updated_at DESC);
```

**Benefits:**
- Perfect tenant isolation
- Filters efficiently
- Supports sorting
- Could be covering if SELECT columns are added

### Example 3: Time-Series Data

**Scenario:** IoT sensor readings

```sql
-- Query
SELECT AVG(temperature), MAX(temperature)
FROM sensor_readings
WHERE device_id = ?
  AND timestamp >= NOW() - INTERVAL '1 hour';

-- Index
CREATE INDEX idx_device_time_temp 
ON sensor_readings(device_id, timestamp DESC, temperature);
```

**Benefits:**
- Filters to specific device
- Efficiently scans recent data
- Covering index for aggregations
- Optimized for time-series queries

### Example 4: E-Commerce Inventory

**Scenario:** Product availability search

```sql
-- Query
SELECT * FROM products
WHERE category = 'Electronics'
  AND in_stock = true
  AND price <= 1000
ORDER BY popularity DESC;

-- Index
CREATE INDEX idx_products_search 
ON products(category, in_stock, price, popularity DESC);
```

**Benefits:**
- Filters by category (high selectivity)
- Filters available items
- Range scan on price
- Pre-sorted by popularity

## Performance Metrics

### Index Size Impact

**Table:** 10 million rows

| Index Type | Columns | Size | Write Overhead |
|------------|---------|------|----------------|
| Single | (user_id) | 200 MB | Low |
| Composite | (user_id, created_at) | 350 MB | Medium |
| Composite | (user_id, created_at, status) | 500 MB | Medium-High |
| Composite | (user_id, created_at, status, category, type) | 850 MB | High |

### Query Performance Improvement

**Scenario:** Finding specific customer orders

| Approach | Query Time | Rows Scanned |
|----------|------------|--------------|
| No index | 2,500 ms | 10,000,000 |
| Single index (customer_id) | 150 ms | 50,000 |
| Composite (customer_id, status) | 12 ms | 500 |
| Covering composite | 8 ms | 500 |

## Best Practices

### 1. Analyze Query Patterns First

Before creating indexes, analyze your actual query workload:

```sql
-- PostgreSQL: Check slow queries
SELECT query, calls, mean_exec_time
FROM pg_stat_statements
WHERE mean_exec_time > 100
ORDER BY mean_exec_time DESC
LIMIT 20;
```

### 2. Use Index Inclusion (INCLUDE) When Available

Modern databases support INCLUDE to add non-key columns:

```sql
-- PostgreSQL, SQL Server
CREATE INDEX idx_user_email 
ON users(tenant_id, email) 
INCLUDE (first_name, last_name);
```

This creates a covering index without affecting the B-tree structure.

### 3. Consider Partial Indexes

For queries with common filters:

```sql
-- Index only active users
CREATE INDEX idx_active_users 
ON users(last_name, first_name) 
WHERE is_active = true;
```

Smaller index, faster queries, less maintenance overhead.

### 4. Monitor Index Usage

```sql
-- PostgreSQL: Find unused indexes
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexname NOT LIKE '%_pkey';
```

Remove indexes that aren't being used.

### 5. Balance Read vs. Write Performance

- More indexes = faster reads, slower writes
- Monitor write latency on heavily indexed tables
- Consider batch processing during off-peak hours

### 6. Test with Production-Like Data

Index effectiveness depends on:
- Data distribution
- Table size
- Query patterns
- Hardware capabilities

Always test with realistic data volumes.

## Common Mistakes to Avoid

### Mistake 1: Wrong Column Order

```sql
-- Bad: Range column first
CREATE INDEX idx_wrong ON orders(created_date, customer_id);

-- Good: Equality column first
CREATE INDEX idx_right ON orders(customer_id, created_date);
```

### Mistake 2: Duplicate Indexes

```sql
-- Redundant!
CREATE INDEX idx1 ON users(email);
CREATE INDEX idx2 ON users(email, name);

-- The second index can handle queries on 'email' alone
-- Remove idx1
```

### Mistake 3: Ignoring Selectivity

```sql
-- Bad: Low-selectivity column first
CREATE INDEX idx_bad ON orders(status, order_id);
-- If 90% of orders have status='completed', this wastes space

-- Good: High-selectivity column first
CREATE INDEX idx_good ON orders(order_id, status);
```

### Mistake 4: Over-Indexing Small Tables

Don't index tables with < 1,000 rows unless there's a specific need. Sequential scans are often faster.

## Conclusion

Composite indexes are powerful tools for optimizing database performance, but they require careful planning:

**Use composite indexes when:**
- Queries filter on multiple columns together
- You need covering indexes for common queries
- Sorting on multiple columns is required
- Enforcing multi-column uniqueness constraints

**Remember:**
- Column order is critical (leftmost prefix rule)
- Balance index benefits against maintenance overhead
- Monitor and adjust based on actual usage
- Test with realistic data and query patterns

Well-designed composite indexes can transform slow queries into millisecond responses, but poorly designed ones waste resources and slow down writes. Always analyze your specific workload before creating indexes.

---

*Effective indexing is an iterative process. Start with the most common queries, measure performance, and refine as needed.*