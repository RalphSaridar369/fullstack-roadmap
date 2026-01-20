# Why Use Database Views: Views vs Tables

## Table of Contents
1. [Introduction](#introduction)
2. [What is a Table?](#what-is-a-table)
3. [What is a View?](#what-is-a-view)
4. [Key Differences Between Tables and Views](#key-differences-between-tables-and-views)
5. [Why Use Views?](#why-use-views)
6. [When to Use Tables vs Views](#when-to-use-tables-vs-views)
7. [Practical Examples](#practical-examples)
8. [Common Misconceptions](#common-misconceptions)
9. [Decision Framework](#decision-framework)

## Introduction

Understanding when to use views versus tables is fundamental to effective database design. While both present data in a tabular format, they serve different purposes and have distinct characteristics that make them suitable for different scenarios.

## What is a Table?

A **table** is a physical database object that stores data persistently on disk. It is the fundamental structure for data storage in relational databases.

### Table Characteristics

- **Physical Storage**: Data is actually stored on disk
- **Persistent**: Data remains until explicitly deleted
- **Indexable**: Can have primary keys, foreign keys, and indexes
- **Direct Manipulation**: Supports INSERT, UPDATE, DELETE operations
- **Performance**: Direct access to stored data
- **Size**: Consumes disk space proportional to data volume

### Table Example

```sql
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100),
    salary DECIMAL(10,2),
    department_id INT,
    hire_date DATE,
    ssn VARCHAR(11),
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);

-- Insert actual data
INSERT INTO employees VALUES 
(1, 'John', 'Doe', 'john@example.com', 75000, 10, '2020-01-15', '123-45-6789');
```

## What is a View?

A **view** is a virtual table defined by a SQL query. It doesn't store data itself but provides a way to look at data from one or more tables.

### View Characteristics

- **Virtual**: No physical data storage (except materialized views)
- **Dynamic**: Results reflect current state of underlying tables
- **Query-based**: Defined by a SELECT statement
- **Limited Manipulation**: Many views are read-only
- **Abstraction Layer**: Hides complexity and provides security
- **Minimal Storage**: Only stores the query definition

### View Example

```sql
CREATE VIEW employee_public_info AS
SELECT 
    employee_id,
    first_name,
    last_name,
    email,
    department_id,
    hire_date
FROM employees;
-- Note: salary and SSN are excluded for privacy
```

## Key Differences Between Tables and Views

### 1. Data Storage

**Table:**
```sql
-- Creates physical storage
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    price DECIMAL(10,2),
    stock_quantity INT
);

-- Data is stored on disk
INSERT INTO products VALUES (1, 'Laptop', 999.99, 50);
-- This data physically exists and consumes disk space
```

**View:**
```sql
-- Creates only a saved query
CREATE VIEW expensive_products AS
SELECT product_id, product_name, price
FROM products
WHERE price > 500;

-- No data is stored, query executes when view is accessed
SELECT * FROM expensive_products;
-- This runs the underlying query each time
```

### 2. Performance Implications

**Table:**
- Fast read operations (data already stored)
- Indexed for quick lookups
- No computation needed for retrieval
- Consumes memory and disk space

**View:**
- Query executed on each access
- Performance depends on underlying query complexity
- Can be slower for complex joins and aggregations
- Uses minimal storage (just the query definition)

### 3. Data Manipulation

**Table:**
```sql
-- Full DML support
INSERT INTO employees VALUES (...);
UPDATE employees SET salary = 80000 WHERE employee_id = 1;
DELETE FROM employees WHERE employee_id = 1;
```

**View:**
```sql
-- Limited DML support, many restrictions
-- This might work (simple view):
UPDATE employee_public_info 
SET email = 'newemail@example.com' 
WHERE employee_id = 1;

-- This won't work (view with JOIN):
UPDATE employee_department_view 
SET department_name = 'IT' 
WHERE employee_id = 1;
-- Error: Cannot modify more than one base table through a view
```

### 4. Indexes

**Table:**
```sql
-- Can create indexes for performance
CREATE INDEX idx_employee_email ON employees(email);
CREATE INDEX idx_employee_dept ON employees(department_id);
```

**View:**
```sql
-- Standard views cannot be indexed
-- (Exception: Materialized views and indexed views in some databases)
CREATE VIEW employee_view AS SELECT * FROM employees;

-- This is NOT possible for regular views:
-- CREATE INDEX idx_view ON employee_view(email); -- Error!
```

### 5. Dependencies

**Table:**
- Independent structure
- Other objects can depend on it (views, stored procedures)
- Changes may break dependent objects

**View:**
- Depends on underlying tables
- Breaking changes to base tables break the view
- Must be maintained when table structure changes

```sql
-- Table change affects view
ALTER TABLE employees ADD COLUMN phone VARCHAR(20);

-- View using SELECT * automatically includes new column
-- View with explicit columns needs updating
```

## Why Use Views?

### 1. Security and Access Control

Views allow you to expose only the data users need to see, hiding sensitive information.

**Problem: Exposing Sensitive Data**
```sql
-- You don't want everyone to see salaries and SSN
SELECT * FROM employees;
```

**Solution: Create a Restricted View**
```sql
CREATE VIEW employee_directory AS
SELECT 
    employee_id,
    first_name,
    last_name,
    email,
    department_id
FROM employees;

-- Grant access to view instead of table
GRANT SELECT ON employee_directory TO public_users;
-- Deny direct table access
REVOKE SELECT ON employees FROM public_users;
```

**Real-World Example:**
```sql
-- HR sees everything
CREATE VIEW hr_employee_view AS
SELECT * FROM employees;

-- Managers see their department only
CREATE VIEW manager_employee_view AS
SELECT 
    employee_id,
    first_name,
    last_name,
    email,
    department_id,
    hire_date
FROM employees
WHERE department_id IN (
    SELECT department_id 
    FROM departments 
    WHERE manager_id = CURRENT_USER_ID()
);

-- Public directory excludes sensitive data
CREATE VIEW public_employee_directory AS
SELECT 
    first_name,
    last_name,
    email,
    department_id
FROM employees
WHERE status = 'active';
```

### 2. Simplifying Complex Queries

Views encapsulate complex logic, making it reusable and easier to understand.

**Without View: Repetitive Complex Query**
```sql
-- Every time you need this data, write this complex query:
SELECT 
    c.customer_id,
    c.name,
    c.email,
    COUNT(DISTINCT o.order_id) AS total_orders,
    SUM(oi.quantity * oi.unit_price) AS total_spent,
    MAX(o.order_date) AS last_order_date,
    AVG(oi.quantity * oi.unit_price) AS avg_order_value
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
LEFT JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY c.customer_id, c.name, c.email;
```

**With View: Simple and Reusable**
```sql
-- Create once
CREATE VIEW customer_summary AS
SELECT 
    c.customer_id,
    c.name,
    c.email,
    COUNT(DISTINCT o.order_id) AS total_orders,
    SUM(oi.quantity * oi.unit_price) AS total_spent,
    MAX(o.order_date) AS last_order_date,
    AVG(oi.quantity * oi.unit_price) AS avg_order_value
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
LEFT JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY c.customer_id, c.name, c.email;

-- Use anywhere with simple query
SELECT * FROM customer_summary WHERE total_spent > 1000;
SELECT * FROM customer_summary ORDER BY last_order_date DESC LIMIT 10;
SELECT name, total_orders FROM customer_summary WHERE total_orders > 5;
```

### 3. Data Abstraction and Independence

Views provide a layer between applications and tables, allowing schema changes without breaking applications.

**Scenario: Table Structure Changes**
```sql
-- Original table
CREATE TABLE users (
    user_id INT,
    username VARCHAR(50),
    full_name VARCHAR(100)
);

-- Application code uses this
SELECT user_id, username, full_name FROM users;
```

**Problem: Need to Split Name Column**
```sql
-- New table structure
ALTER TABLE users 
ADD COLUMN first_name VARCHAR(50),
ADD COLUMN last_name VARCHAR(50);

UPDATE users SET 
    first_name = SUBSTRING(full_name, 1, POSITION(' ' IN full_name) - 1),
    last_name = SUBSTRING(full_name, POSITION(' ' IN full_name) + 1);

ALTER TABLE users DROP COLUMN full_name;

-- Old application code breaks!
-- SELECT user_id, username, full_name FROM users; -- Error!
```

**Solution: Use a View**
```sql
-- Create view with old structure
CREATE VIEW users_legacy AS
SELECT 
    user_id,
    username,
    CONCAT(first_name, ' ', last_name) AS full_name
FROM users;

-- Old application continues to work
SELECT user_id, username, full_name FROM users_legacy;
```

### 4. Presenting Data in Different Formats

Different users may need the same data presented differently.

```sql
-- Raw sales data table
CREATE TABLE sales (
    sale_id INT,
    sale_date DATE,
    product_id INT,
    quantity INT,
    unit_price DECIMAL(10,2),
    salesperson_id INT,
    region VARCHAR(50)
);

-- Executive view: High-level summary
CREATE VIEW executive_sales_dashboard AS
SELECT 
    DATE_TRUNC('month', sale_date) AS month,
    region,
    SUM(quantity * unit_price) AS revenue,
    COUNT(DISTINCT sale_id) AS transaction_count,
    COUNT(DISTINCT salesperson_id) AS active_salespeople
FROM sales
GROUP BY DATE_TRUNC('month', sale_date), region;

-- Sales team view: Detailed performance
CREATE VIEW salesperson_performance AS
SELECT 
    s.salesperson_id,
    sp.name,
    COUNT(s.sale_id) AS sales_count,
    SUM(s.quantity * s.unit_price) AS total_revenue,
    AVG(s.quantity * s.unit_price) AS avg_sale_value,
    s.region
FROM sales s
JOIN salespeople sp ON s.salesperson_id = sp.salesperson_id
GROUP BY s.salesperson_id, sp.name, s.region;

-- Product team view: Product performance
CREATE VIEW product_sales_analysis AS
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    SUM(s.quantity) AS units_sold,
    SUM(s.quantity * s.unit_price) AS revenue,
    COUNT(DISTINCT s.sale_id) AS number_of_sales
FROM sales s
JOIN products p ON s.product_id = p.product_id
GROUP BY p.product_id, p.product_name, p.category;
```

### 5. Enforcing Business Rules

Views can automatically apply business logic and filters.

```sql
-- Table with all data
CREATE TABLE orders (
    order_id INT,
    customer_id INT,
    order_date DATE,
    status VARCHAR(20),
    total_amount DECIMAL(10,2),
    is_deleted BOOLEAN DEFAULT FALSE
);

-- View that automatically filters out deleted records
CREATE VIEW active_orders AS
SELECT 
    order_id,
    customer_id,
    order_date,
    status,
    total_amount
FROM orders
WHERE is_deleted = FALSE;

-- View that shows only completed orders
CREATE VIEW completed_orders AS
SELECT 
    order_id,
    customer_id,
    order_date,
    total_amount
FROM orders
WHERE status = 'completed' AND is_deleted = FALSE;

-- View with calculated fields based on business rules
CREATE VIEW order_priority AS
SELECT 
    order_id,
    customer_id,
    order_date,
    total_amount,
    CASE 
        WHEN total_amount > 10000 THEN 'HIGH'
        WHEN total_amount > 1000 THEN 'MEDIUM'
        ELSE 'LOW'
    END AS priority,
    CASE 
        WHEN CURRENT_DATE - order_date > 7 THEN 'OVERDUE'
        WHEN CURRENT_DATE - order_date > 3 THEN 'AT_RISK'
        ELSE 'ON_TRACK'
    END AS delivery_status
FROM orders
WHERE is_deleted = FALSE;
```

### 6. Joining Tables for Convenience

Views can pre-join commonly used tables.

**Without View:**
```sql
-- Every time you need customer with order info:
SELECT 
    c.customer_id,
    c.name,
    c.email,
    o.order_id,
    o.order_date,
    o.total_amount,
    o.status
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date > '2024-01-01';
```

**With View:**
```sql
-- Create once
CREATE VIEW customer_orders AS
SELECT 
    c.customer_id,
    c.name AS customer_name,
    c.email,
    o.order_id,
    o.order_date,
    o.total_amount,
    o.status
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id;

-- Use anywhere
SELECT * FROM customer_orders WHERE order_date > '2024-01-01';
SELECT * FROM customer_orders WHERE customer_name LIKE 'John%';
SELECT customer_name, SUM(total_amount) 
FROM customer_orders 
GROUP BY customer_name;
```

## When to Use Tables vs Views

### Use Tables When:

1. **Storing Actual Data**
   - Transactional data (orders, customers, products)
   - Master data (employees, inventory)
   - Historical records
   - Any data that needs to persist

2. **Performance is Critical**
   - Frequently accessed data
   - Data requiring fast INSERT/UPDATE/DELETE operations
   - Data that benefits from indexing

3. **Complex Data Manipulation Required**
   - Need to perform multiple types of operations
   - Require foreign key constraints
   - Need triggers or stored procedures

4. **Data Independence**
   - Core business data
   - Data that doesn't depend on other tables for its existence

**Example:**
```sql
-- Use tables for core business entities
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    total_amount DECIMAL(10,2),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

### Use Views When:

1. **Simplifying Complex Queries**
   - Queries with multiple joins
   - Aggregations used repeatedly
   - Complex calculations

2. **Security Requirements**
   - Need to hide sensitive columns
   - Row-level security based on user context
   - Different access levels for different users

3. **Presenting Different Perspectives**
   - Same data needed in different formats
   - Different user roles need different views
   - Reporting and analytics

4. **Maintaining Legacy Compatibility**
   - Table structure changed but old applications exist
   - Gradual migration strategies
   - API backward compatibility

**Example:**
```sql
-- Use views for computed and filtered data
CREATE VIEW high_value_customers AS
SELECT 
    c.customer_id,
    c.name,
    c.email,
    COUNT(o.order_id) AS order_count,
    SUM(o.total_amount) AS lifetime_value
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name, c.email
HAVING SUM(o.total_amount) > 10000;

-- Use views for security
CREATE VIEW public_customer_info AS
SELECT customer_id, name, email
FROM customers;
-- Hide created_at and other sensitive info
```

## Practical Examples

### Example 1: E-Commerce System

**Tables (Core Data):**
```sql
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(100),
    description TEXT,
    cost_price DECIMAL(10,2),
    selling_price DECIMAL(10,2),
    stock_quantity INT,
    category_id INT
);

CREATE TABLE order_items (
    order_item_id INT PRIMARY KEY,
    order_id INT,
    product_id INT,
    quantity INT,
    price_at_purchase DECIMAL(10,2)
);
```

**Views (Derived Information):**
```sql
-- Product profitability analysis
CREATE VIEW product_profitability AS
SELECT 
    p.product_id,
    p.name,
    p.selling_price,
    p.cost_price,
    (p.selling_price - p.cost_price) AS profit_per_unit,
    ((p.selling_price - p.cost_price) / p.cost_price * 100) AS profit_margin_pct,
    p.stock_quantity,
    (p.selling_price - p.cost_price) * p.stock_quantity AS potential_profit
FROM products p;

-- Best selling products
CREATE VIEW best_selling_products AS
SELECT 
    p.product_id,
    p.name,
    p.category_id,
    SUM(oi.quantity) AS total_units_sold,
    SUM(oi.quantity * oi.price_at_purchase) AS total_revenue,
    COUNT(DISTINCT oi.order_id) AS number_of_orders
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.name, p.category_id
ORDER BY total_units_sold DESC;
```

### Example 2: Employee Management

**Tables:**
```sql
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100),
    salary DECIMAL(10,2),
    department_id INT,
    manager_id INT,
    hire_date DATE,
    ssn VARCHAR(11),
    performance_rating DECIMAL(3,2)
);
```

**Views:**
```sql
-- For general employees (no sensitive data)
CREATE VIEW employee_directory AS
SELECT 
    employee_id,
    CONCAT(first_name, ' ', last_name) AS full_name,
    email,
    department_id
FROM employees;

-- For HR (all data)
CREATE VIEW hr_employee_details AS
SELECT * FROM employees;

-- For managers (their team only)
CREATE VIEW manager_team_view AS
SELECT 
    employee_id,
    first_name,
    last_name,
    email,
    department_id,
    performance_rating,
    hire_date
FROM employees
WHERE manager_id = CURRENT_USER_ID();

-- For payroll (salary info only)
CREATE VIEW payroll_view AS
SELECT 
    employee_id,
    CONCAT(first_name, ' ', last_name) AS full_name,
    salary,
    department_id
FROM employees;
```

## Common Misconceptions

### Misconception 1: "Views Store Data"

**Wrong:**
```sql
CREATE VIEW my_view AS SELECT * FROM large_table;
-- This does NOT create a copy of the data
```

**Right:**
Views are just saved queries. Each time you query a view, it runs the underlying SELECT statement against the current data.

### Misconception 2: "Views are Always Slower"

**Not Necessarily:**
- Simple views on indexed tables can be as fast as direct table access
- Views can actually improve performance by encouraging query reuse and optimization
- Materialized views can be faster than tables for complex aggregations

### Misconception 3: "You Can't Update Through Views"

**Partially Wrong:**
Simple views CAN be updated:
```sql
CREATE VIEW simple_view AS
SELECT employee_id, first_name, last_name, email
FROM employees;

-- This works:
UPDATE simple_view SET email = 'new@email.com' WHERE employee_id = 1;
```

Complex views with joins, aggregations, or DISTINCT generally cannot be updated.

### Misconception 4: "Views Replace Tables"

**Wrong:**
Views complement tables, they don't replace them. You need tables to store actual data, and views to present that data in useful ways.

## Decision Framework

### Quick Decision Tree

```
Do you need to store actual data persistently?
├─ YES → Use a TABLE
└─ NO → Do you need to present existing data in a specific way?
   ├─ YES → Consider a VIEW
   │   ├─ Is it a complex query used repeatedly? → VIEW
   │   ├─ Do you need to hide sensitive columns? → VIEW
   │   ├─ Do you need different presentations of same data? → VIEW
   │   └─ Is query performance critical and data rarely changes? → MATERIALIZED VIEW
   └─ NO → Use a TABLE
```

### Checklist

**Use a Table if you need:**
- [ ] Permanent data storage
- [ ] Fast INSERT, UPDATE, DELETE operations
- [ ] Primary and foreign key constraints
- [ ] Indexes for query optimization
- [ ] Triggers
- [ ] Full control over data structure

**Use a View if you need:**
- [ ] To simplify complex queries
- [ ] To hide sensitive data
- [ ] To present data differently for different users
- [ ] To maintain backward compatibility
- [ ] To automatically apply filters or business rules
- [ ] To avoid data duplication

## Conclusion

Tables and views serve different but complementary purposes in database design:

- **Tables** are for storing data
- **Views** are for presenting data

Use tables as your foundation for data storage and views as your presentation layer. This separation provides flexibility, security, and maintainability in your database architecture.

The key is understanding that views don't compete with tables—they enhance them by providing controlled, customized access to table data without duplicating it.

---

*Last Updated: January 2025*