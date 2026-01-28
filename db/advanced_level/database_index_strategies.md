# Database Index Strategies: A Practical Guide

## Table of Contents
1. [Initial Indexing Strategy](#initial-indexing-strategy)
2. [Usage Patterns Indexing](#usage-patterns-indexing)
3. [Scenario-Based Indexing](#scenario-based-indexing)
4. [Monitoring & Maintenance](#monitoring--maintenance)

---

## 1. Initial Indexing Strategy

### 1.1 Foundation: Start with the Essentials

When setting up a new database or table, begin with these fundamental indexes:

#### Primary Key Index
```sql
-- Always define a primary key - it creates a clustered index
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    username VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Why it matters:**
- Ensures data uniqueness
- Provides fastest access path for single-record lookups
- In most databases (MySQL InnoDB, SQL Server), becomes the clustered index
- All other indexes reference the primary key

**Best Practices:**
- Use auto-incrementing integers or UUIDs
- Keep primary keys small (other indexes store this value)
- Never use business data as primary key (emails, SSNs change)
- Consider surrogate keys for composite natural keys

#### Foreign Key Indexes
```sql
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    total_amount DECIMAL(10,2),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- CRITICAL: Index the foreign key
CREATE INDEX idx_orders_customer ON orders(customer_id);
```

**Why it matters:**
- JOIN operations heavily rely on foreign keys
- Unindexed foreign keys are a top performance killer
- Improves referential integrity checks

**Rule of thumb:** Index ALL foreign key columns unless you have a specific reason not to.

#### Unique Constraints
```sql
-- Enforce uniqueness on business keys
CREATE UNIQUE INDEX idx_users_email ON users(email);
CREATE UNIQUE INDEX idx_users_username ON users(username);
```

**Why it matters:**
- Prevents duplicate data
- Automatically creates an index
- Faster than non-unique indexes for equality searches

### 1.2 Initial Analysis Checklist

Before creating any additional indexes, gather this information:

**Data Profile:**
- [ ] Table row count (current and projected)
- [ ] Table growth rate (rows/day or rows/month)
- [ ] Column cardinality (distinct values per column)
- [ ] Data distribution (uniform vs skewed)

**Workload Profile:**
- [ ] Read:Write ratio
- [ ] Most frequent queries
- [ ] Critical queries (business-critical paths)
- [ ] Peak usage times

**Infrastructure:**
- [ ] Available storage space
- [ ] Memory allocation for indexes
- [ ] Backup window constraints
- [ ] Maintenance window availability

### 1.3 Initial Index Priority Framework

**Priority 1 - Must Have (Create Immediately):**
1. Primary key
2. Foreign keys
3. Unique constraints on business keys
4. Columns in frequent WHERE clauses (>100 queries/day)

**Priority 2 - Should Have (Create in First Week):**
1. Columns used in JOINs (beyond foreign keys)
2. Columns in ORDER BY clauses
3. Columns in GROUP BY clauses
4. Frequently filtered date/timestamp columns

**Priority 3 - Nice to Have (Create Based on Monitoring):**
1. Covering indexes for specific critical queries
2. Composite indexes for multi-column filters
3. Partial indexes for common subsets
4. Full-text indexes for search features

### 1.4 Initial Index Templates by Table Type

#### Transactional Tables (OLTP)
```sql
CREATE TABLE transactions (
    transaction_id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    transaction_date TIMESTAMP NOT NULL,
    amount DECIMAL(10,2),
    status VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Initial indexes
CREATE INDEX idx_trans_user ON transactions(user_id);
CREATE INDEX idx_trans_date ON transactions(transaction_date);
CREATE INDEX idx_trans_status ON transactions(status) 
    WHERE status IN ('pending', 'processing'); -- Partial index
```

#### Lookup/Reference Tables
```sql
CREATE TABLE products (
    product_id BIGINT PRIMARY KEY,
    sku VARCHAR(50) NOT NULL,
    name VARCHAR(255),
    category_id INT,
    price DECIMAL(10,2)
);

-- Initial indexes
CREATE UNIQUE INDEX idx_products_sku ON products(sku);
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_name ON products(name) WHERE name IS NOT NULL;
```

#### Audit/Log Tables
```sql
CREATE TABLE audit_logs (
    log_id BIGINT PRIMARY KEY,
    user_id BIGINT,
    action VARCHAR(50),
    table_name VARCHAR(100),
    record_id BIGINT,
    timestamp TIMESTAMP NOT NULL
);

-- Initial indexes - optimize for time-based queries
CREATE INDEX idx_audit_timestamp ON audit_logs(timestamp DESC);
CREATE INDEX idx_audit_user_time ON audit_logs(user_id, timestamp);
```

### 1.5 Index Naming Conventions

Establish clear naming conventions from the start:

```sql
-- Format: idx_{table}_{columns}[_{condition}]

-- Single column
CREATE INDEX idx_users_email ON users(email);

-- Multiple columns
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);

-- Partial/filtered
CREATE INDEX idx_orders_pending ON orders(created_at) WHERE status = 'pending';

-- Covering
CREATE INDEX idx_orders_covering ON orders(customer_id) INCLUDE (order_date, total);

-- Unique
CREATE UNIQUE INDEX unq_users_email ON users(email);
```

**Benefits:**
- Easy to understand index purpose
- Simplifies maintenance
- Helps identify redundant indexes

---

## 2. Usage Patterns Indexing

### 2.1 Query Pattern Analysis

#### Identifying Query Patterns

**Step 1: Capture Query Logs**
```sql
-- PostgreSQL: Enable query logging
ALTER SYSTEM SET log_min_duration_statement = 100; -- Log queries > 100ms
SELECT pg_reload_conf();

-- MySQL: Enable slow query log
SET GLOBAL slow_query_log = 1;
SET GLOBAL long_query_time = 0.1; -- 100ms
```

**Step 2: Analyze Top Queries**
```sql
-- PostgreSQL: Query statistics
SELECT 
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- MySQL: Slow query analysis
SELECT 
    DIGEST_TEXT,
    COUNT_STAR,
    AVG_TIMER_WAIT/1000000000 as avg_ms,
    SUM_ROWS_EXAMINED
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

### 2.2 Index Patterns by Query Type

#### Pattern 1: Equality Filters
```sql
-- Query pattern
SELECT * FROM users WHERE email = 'user@example.com';
SELECT * FROM users WHERE status = 'active' AND role = 'admin';

-- Indexing strategy
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_status_role ON users(status, role);
```

**Key Points:**
- Single-column index for simple equality
- Multi-column index with most selective column first
- Consider unique indexes for guaranteed single results

#### Pattern 2: Range Queries
```sql
-- Query pattern
SELECT * FROM orders 
WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31';

SELECT * FROM products 
WHERE price >= 100 AND price <= 500;

-- Indexing strategy
CREATE INDEX idx_orders_date ON orders(order_date);
CREATE INDEX idx_products_price ON products(price);
```

**Key Points:**
- B-tree indexes work well for ranges
- Consider DESC for "most recent" queries
- Partition large tables by date ranges

#### Pattern 3: Prefix Matching (LIKE)
```sql
-- Query pattern
SELECT * FROM products WHERE name LIKE 'iPhone%';
SELECT * FROM users WHERE email LIKE 'admin%';

-- Indexing strategy
CREATE INDEX idx_products_name ON products(name);
-- Or for case-insensitive
CREATE INDEX idx_products_name_lower ON products(LOWER(name));
```

**Key Points:**
- Only works for prefix matching (LIKE 'abc%')
- Does NOT work for %abc% or %abc
- Use full-text indexes for non-prefix matching

#### Pattern 4: Sorting (ORDER BY)
```sql
-- Query pattern
SELECT * FROM posts 
WHERE author_id = 123 
ORDER BY created_at DESC 
LIMIT 10;

-- Indexing strategy - composite with sort column
CREATE INDEX idx_posts_author_created ON posts(author_id, created_at DESC);
```

**Key Points:**
- Include ORDER BY columns in index
- Order matters: filter columns first, then sort columns
- DESC indexes for descending sorts

#### Pattern 5: Grouping and Aggregation
```sql
-- Query pattern
SELECT category_id, COUNT(*), AVG(price)
FROM products
GROUP BY category_id;

-- Indexing strategy
CREATE INDEX idx_products_category ON products(category_id);
```

**Key Points:**
- Index GROUP BY columns
- Consider covering index with aggregated columns
- Helps with DISTINCT operations too

#### Pattern 6: JOIN Operations
```sql
-- Query pattern
SELECT o.*, c.name, c.email
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date > '2024-01-01';

-- Indexing strategy
CREATE INDEX idx_orders_date ON orders(order_date);
CREATE INDEX idx_orders_customer ON orders(customer_id); -- Already exists as FK
-- customers.customer_id already indexed (primary key)
```

**Key Points:**
- Index both sides of JOIN condition
- Primary keys are already indexed
- Consider covering indexes for frequently accessed columns

### 2.3 Read-Heavy vs Write-Heavy Patterns

#### Read-Heavy Applications (>80% Reads)

**Strategy: Aggressive Indexing**
```sql
-- E-commerce product catalog
CREATE TABLE products (
    product_id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    category_id INT,
    brand_id INT,
    price DECIMAL(10,2),
    in_stock BOOLEAN,
    rating DECIMAL(3,2)
);

-- Multiple indexes for different search patterns
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_brand ON products(brand_id);
CREATE INDEX idx_products_price ON products(price);
CREATE INDEX idx_products_rating ON products(rating DESC);
CREATE INDEX idx_products_stock ON products(in_stock) WHERE in_stock = true;

-- Covering indexes for common queries
CREATE INDEX idx_products_search 
ON products(category_id, in_stock) 
INCLUDE (name, price, rating);
```

**Characteristics:**
- More indexes = acceptable
- Covering indexes to avoid table lookups
- Partial indexes for filtered views
- Write overhead is tolerable

#### Write-Heavy Applications (>50% Writes)

**Strategy: Minimal, High-Impact Indexing**
```sql
-- Event logging system
CREATE TABLE events (
    event_id BIGINT PRIMARY KEY,
    user_id BIGINT,
    event_type VARCHAR(50),
    event_data JSONB,
    timestamp TIMESTAMP NOT NULL
);

-- Only essential indexes
CREATE INDEX idx_events_user ON events(user_id);
CREATE INDEX idx_events_timestamp ON events(timestamp DESC);
-- Avoid indexing event_type (low selectivity, high write cost)
```

**Characteristics:**
- Fewer indexes
- Focus on critical queries only
- Avoid indexes on frequently updated columns
- Consider batch indexing during off-hours

### 2.4 Time-Based Access Patterns

#### Recent Data Access Pattern
```sql
-- Query pattern: Most queries access recent data
SELECT * FROM logs WHERE created_at > NOW() - INTERVAL '7 days';

-- Strategy 1: DESC index
CREATE INDEX idx_logs_created_desc ON logs(created_at DESC);

-- Strategy 2: Partial index on recent data
CREATE INDEX idx_logs_recent 
ON logs(created_at) 
WHERE created_at > NOW() - INTERVAL '30 days';

-- Strategy 3: Partitioning (PostgreSQL)
CREATE TABLE logs (
    log_id BIGINT,
    message TEXT,
    created_at TIMESTAMP
) PARTITION BY RANGE (created_at);

CREATE TABLE logs_2024_01 PARTITION OF logs
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

#### Historical Data Access Pattern
```sql
-- Query pattern: Analytical queries on all data
SELECT DATE_TRUNC('month', order_date) as month,
       SUM(total_amount) as revenue
FROM orders
WHERE order_date >= '2020-01-01'
GROUP BY month;

-- Strategy: Composite indexes for time-based aggregations
CREATE INDEX idx_orders_date_amount ON orders(order_date, total_amount);
```

### 2.5 Search Patterns

#### Exact Match Search
```sql
-- User login
SELECT * FROM users WHERE email = 'user@example.com' AND status = 'active';

-- Index strategy
CREATE INDEX idx_users_email_status ON users(email, status);
-- Or unique if email is unique
CREATE UNIQUE INDEX unq_users_email ON users(email);
```

#### Multi-Criteria Search
```sql
-- Product search with filters
SELECT * FROM products
WHERE category_id = 5
  AND price BETWEEN 100 AND 500
  AND in_stock = true
ORDER BY rating DESC;

-- Index strategy
CREATE INDEX idx_products_search 
ON products(category_id, in_stock, price, rating DESC);
```

#### Full-Text Search
```sql
-- Article search
SELECT * FROM articles 
WHERE MATCH(title, body) AGAINST('database optimization' IN NATURAL LANGUAGE MODE);

-- Index strategy
CREATE FULLTEXT INDEX idx_articles_fulltext ON articles(title, body);
```

### 2.6 Pagination Patterns

#### Offset-Based Pagination (Avoid for large offsets)
```sql
-- Query pattern
SELECT * FROM posts 
ORDER BY created_at DESC 
LIMIT 20 OFFSET 10000; -- Slow!

-- Index strategy (helps but still not optimal)
CREATE INDEX idx_posts_created ON posts(created_at DESC);
```

#### Cursor-Based Pagination (Recommended)
```sql
-- Query pattern
SELECT * FROM posts 
WHERE created_at < '2024-01-15 10:30:00'
ORDER BY created_at DESC 
LIMIT 20;

-- Index strategy
CREATE INDEX idx_posts_created ON posts(created_at DESC);
-- Much faster: uses index range scan instead of scanning all rows
```

---

## 3. Scenario-Based Indexing

### 3.1 E-Commerce Platform

#### Product Catalog
```sql
CREATE TABLE products (
    product_id BIGINT PRIMARY KEY,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(255),
    description TEXT,
    category_id INT,
    brand_id INT,
    price DECIMAL(10,2),
    sale_price DECIMAL(10,2),
    in_stock BOOLEAN,
    stock_quantity INT,
    rating DECIMAL(3,2),
    review_count INT,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- Index strategy
CREATE UNIQUE INDEX unq_products_sku ON products(sku);

-- Browse by category
CREATE INDEX idx_products_category ON products(category_id, in_stock);

-- Search with filters
CREATE INDEX idx_products_category_price 
ON products(category_id, price) 
WHERE in_stock = true;

-- Sort by popularity
CREATE INDEX idx_products_rating 
ON products(category_id, rating DESC, review_count DESC);

-- Admin: Find low stock
CREATE INDEX idx_products_stock 
ON products(stock_quantity) 
WHERE stock_quantity < 10;

-- Full-text search
CREATE FULLTEXT INDEX idx_products_search ON products(name, description);
```

#### Shopping Cart
```sql
CREATE TABLE cart_items (
    cart_item_id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT,
    added_at TIMESTAMP,
    session_id VARCHAR(100)
);

-- Index strategy
CREATE INDEX idx_cart_user ON cart_items(user_id);
CREATE INDEX idx_cart_session ON cart_items(session_id);

-- Prevent duplicate items
CREATE UNIQUE INDEX unq_cart_user_product 
ON cart_items(user_id, product_id);

-- Cleanup old abandoned carts
CREATE INDEX idx_cart_cleanup 
ON cart_items(added_at) 
WHERE user_id IS NULL;
```

#### Order Management
```sql
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    order_date TIMESTAMP NOT NULL,
    status VARCHAR(20),
    total_amount DECIMAL(10,2),
    shipping_address_id BIGINT,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- Index strategy
-- Customer order history
CREATE INDEX idx_orders_customer_date 
ON orders(customer_id, order_date DESC);

-- Admin: Orders by status
CREATE INDEX idx_orders_status_date 
ON orders(status, order_date DESC);

-- Covering index for order list view
CREATE INDEX idx_orders_list 
ON orders(customer_id, order_date DESC) 
INCLUDE (order_id, status, total_amount);

-- Pending orders requiring action
CREATE INDEX idx_orders_pending 
ON orders(status, created_at) 
WHERE status IN ('pending', 'processing');
```

### 3.2 Social Media Platform

#### User Posts
```sql
CREATE TABLE posts (
    post_id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    content TEXT,
    likes_count INT DEFAULT 0,
    comments_count INT DEFAULT 0,
    created_at TIMESTAMP NOT NULL,
    visibility VARCHAR(20) -- public, friends, private
);

-- Index strategy
-- User timeline
CREATE INDEX idx_posts_user_created 
ON posts(user_id, created_at DESC);

-- News feed (public posts)
CREATE INDEX idx_posts_feed 
ON posts(created_at DESC) 
WHERE visibility = 'public';

-- Trending posts
CREATE INDEX idx_posts_trending 
ON posts(likes_count DESC, created_at DESC) 
WHERE created_at > NOW() - INTERVAL '7 days';

-- Full-text search
CREATE FULLTEXT INDEX idx_posts_content ON posts(content);
```

#### Followers/Following
```sql
CREATE TABLE follows (
    follow_id BIGINT PRIMARY KEY,
    follower_id BIGINT NOT NULL,
    following_id BIGINT NOT NULL,
    created_at TIMESTAMP,
    UNIQUE(follower_id, following_id)
);

-- Index strategy
-- Who does user X follow?
CREATE INDEX idx_follows_follower ON follows(follower_id);

-- Who follows user X?
CREATE INDEX idx_follows_following ON follows(following_id);

-- Mutual followers query
CREATE INDEX idx_follows_mutual ON follows(follower_id, following_id);
```

#### Notifications
```sql
CREATE TABLE notifications (
    notification_id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    type VARCHAR(50),
    is_read BOOLEAN DEFAULT false,
    created_at TIMESTAMP NOT NULL
);

-- Index strategy
-- User's unread notifications
CREATE INDEX idx_notif_user_unread 
ON notifications(user_id, created_at DESC) 
WHERE is_read = false;

-- Cleanup old read notifications
CREATE INDEX idx_notif_cleanup 
ON notifications(created_at, is_read) 
WHERE is_read = true;
```

### 3.3 SaaS Application

#### Multi-Tenant Data
```sql
CREATE TABLE documents (
    document_id BIGINT PRIMARY KEY,
    tenant_id INT NOT NULL,
    user_id BIGINT NOT NULL,
    title VARCHAR(255),
    content TEXT,
    folder_id BIGINT,
    is_deleted BOOLEAN DEFAULT false,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- Index strategy
-- CRITICAL: All queries must filter by tenant_id first
CREATE INDEX idx_docs_tenant_user 
ON documents(tenant_id, user_id, created_at DESC);

CREATE INDEX idx_docs_tenant_folder 
ON documents(tenant_id, folder_id) 
WHERE is_deleted = false;

-- Search within tenant
CREATE FULLTEXT INDEX idx_docs_content ON documents(content);
-- Note: Include tenant_id in WHERE clause for security

-- Soft-deleted items
CREATE INDEX idx_docs_deleted 
ON documents(tenant_id, updated_at) 
WHERE is_deleted = true;
```

#### Subscription Management
```sql
CREATE TABLE subscriptions (
    subscription_id BIGINT PRIMARY KEY,
    tenant_id INT NOT NULL,
    plan_id INT NOT NULL,
    status VARCHAR(20), -- active, cancelled, expired
    start_date DATE,
    end_date DATE,
    auto_renew BOOLEAN
);

-- Index strategy
CREATE INDEX idx_subs_tenant ON subscriptions(tenant_id);

-- Find expiring subscriptions
CREATE INDEX idx_subs_expiring 
ON subscriptions(end_date) 
WHERE status = 'active' AND auto_renew = false;

-- Active subscriptions by plan
CREATE INDEX idx_subs_active 
ON subscriptions(status, plan_id) 
WHERE status = 'active';
```

### 3.4 Analytics/Reporting System

#### Event Tracking
```sql
CREATE TABLE events (
    event_id BIGINT PRIMARY KEY,
    user_id BIGINT,
    event_type VARCHAR(50),
    properties JSONB,
    timestamp TIMESTAMP NOT NULL,
    date DATE GENERATED ALWAYS AS (timestamp::DATE) STORED
);

-- Index strategy
-- Time-series queries
CREATE INDEX idx_events_date ON events(date DESC, event_type);

-- User activity timeline
CREATE INDEX idx_events_user_time ON events(user_id, timestamp DESC);

-- JSONB property queries (PostgreSQL)
CREATE INDEX idx_events_props ON events USING GIN (properties);

-- Specific property lookup
CREATE INDEX idx_events_source 
ON events((properties->>'source')) 
WHERE properties->>'source' IS NOT NULL;
```

#### Aggregated Reports (Pre-calculated)
```sql
CREATE TABLE daily_stats (
    stat_id BIGINT PRIMARY KEY,
    date DATE NOT NULL,
    metric_name VARCHAR(50),
    dimension VARCHAR(50),
    dimension_value VARCHAR(255),
    value NUMERIC,
    UNIQUE(date, metric_name, dimension, dimension_value)
);

-- Index strategy
CREATE INDEX idx_stats_date_metric 
ON daily_stats(date DESC, metric_name);

CREATE INDEX idx_stats_dimension 
ON daily_stats(metric_name, dimension, dimension_value);
```

### 3.5 Real-Time Chat Application

#### Messages
```sql
CREATE TABLE messages (
    message_id BIGINT PRIMARY KEY,
    conversation_id BIGINT NOT NULL,
    sender_id BIGINT NOT NULL,
    content TEXT,
    created_at TIMESTAMP NOT NULL,
    is_deleted BOOLEAN DEFAULT false
);

-- Index strategy
-- Load conversation history (most recent first)
CREATE INDEX idx_messages_conv_time 
ON messages(conversation_id, created_at DESC) 
WHERE is_deleted = false;

-- User's recent messages across all conversations
CREATE INDEX idx_messages_sender_time 
ON messages(sender_id, created_at DESC);

-- Covering index for chat list
CREATE INDEX idx_messages_conv_recent 
ON messages(conversation_id, created_at DESC) 
INCLUDE (sender_id, content);
```

#### Conversations
```sql
CREATE TABLE conversations (
    conversation_id BIGINT PRIMARY KEY,
    type VARCHAR(20), -- direct, group
    last_message_at TIMESTAMP,
    created_at TIMESTAMP
);

CREATE TABLE conversation_participants (
    participant_id BIGINT PRIMARY KEY,
    conversation_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    joined_at TIMESTAMP,
    last_read_at TIMESTAMP,
    UNIQUE(conversation_id, user_id)
);

-- Index strategy
-- User's conversations ordered by activity
CREATE INDEX idx_participants_user 
ON conversation_participants(user_id, last_read_at DESC);

-- Unread message count
CREATE INDEX idx_participants_unread 
ON conversation_participants(user_id) 
WHERE last_read_at < (SELECT last_message_at FROM conversations WHERE conversation_id = conversation_participants.conversation_id);
```

### 3.6 Job Queue System

#### Job Queue
```sql
CREATE TABLE jobs (
    job_id BIGINT PRIMARY KEY,
    queue_name VARCHAR(50),
    status VARCHAR(20), -- pending, processing, completed, failed
    priority INT DEFAULT 0,
    payload JSONB,
    scheduled_at TIMESTAMP,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    attempts INT DEFAULT 0,
    max_attempts INT DEFAULT 3,
    created_at TIMESTAMP NOT NULL
);

-- Index strategy
-- Workers pull next job
CREATE INDEX idx_jobs_next 
ON jobs(queue_name, priority DESC, scheduled_at) 
WHERE status = 'pending' AND scheduled_at <= NOW();

-- Failed jobs for retry
CREATE INDEX idx_jobs_failed 
ON jobs(queue_name, created_at) 
WHERE status = 'failed' AND attempts < max_attempts;

-- Job monitoring
CREATE INDEX idx_jobs_monitoring 
ON jobs(status, queue_name, created_at DESC);

-- Cleanup old completed jobs
CREATE INDEX idx_jobs_cleanup 
ON jobs(completed_at) 
WHERE status = 'completed';
```

---

## 4. Monitoring & Maintenance

### 4.1 Index Health Monitoring

#### 4.1.1 Index Usage Statistics

**PostgreSQL:**
```sql
-- Identify unused indexes
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan AS index_scans,
    idx_tup_read AS tuples_read,
    idx_tup_fetch AS tuples_fetched,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
    AND indexrelname NOT LIKE '%_pkey' -- Exclude primary keys
ORDER BY pg_relation_size(indexrelid) DESC;

-- Index usage rates
SELECT 
    schemaname || '.' || tablename AS table_name,
    indexrelname AS index_name,
    idx_scan AS scans,
    idx_tup_read AS reads,
    idx_tup_fetch AS fetches,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size,
    ROUND(100.0 * idx_scan / NULLIF(seq_scan + idx_scan, 0), 2) AS index_usage_pct
FROM pg_stat_user_indexes
    JOIN pg_stat_user_tables USING (schemaname, tablename)
ORDER BY idx_scan DESC;
```

**MySQL:**
```sql
-- Unused indexes
SELECT 
    object_schema AS database_name,
    object_name AS table_name,
    index_name,
    COUNT_STAR AS queries_used
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE index_name IS NOT NULL
    AND index_name != 'PRIMARY'
    AND COUNT_STAR = 0
ORDER BY object_schema, object_name;

-- Index usage statistics
SELECT 
    object_schema,
    object_name,
    index_name,
    COUNT_STAR AS total_access,
    SUM_TIMER_WAIT/1000000000 AS total_latency_ms,
    COUNT_READ AS reads,
    COUNT_WRITE AS writes
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema NOT IN ('mysql', 'performance_schema', 'sys')
ORDER BY total_access DESC;
```

**SQL Server:**
```sql
-- Unused indexes
SELECT 
    OBJECT_NAME(i.object_id) AS table_name,
    i.name AS index_name,
    i.type_desc,
    s.user_seeks,
    s.user_scans,
    s.user_lookups,
    s.user_updates,
    (s.user_seeks + s.user_scans + s.user_lookups) AS total_reads
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats s 
    ON i.object_id = s.object_id AND i.index_id = s.index_id
WHERE OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
    AND i.index_id > 0 -- Exclude heaps
    AND (s.user_seeks + s.user_scans + s.user_lookups) = 0
ORDER BY s.user_updates DESC;
```

#### 4.1.2 Index Fragmentation

**PostgreSQL:**
```sql
-- Check bloat
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) AS index_size,
    ROUND(100 * (pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename))::numeric / 
        NULLIF(pg_total_relation_size(schemaname||'.'||tablename), 0), 2) AS index_ratio_pct
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

**MySQL:**
```sql
-- Check table fragmentation (approximation)
SELECT 
    TABLE_SCHEMA,
    TABLE_NAME,
    ENGINE,
    ROUND(DATA_LENGTH/1024/1024, 2) AS data_mb,
    ROUND(INDEX_LENGTH/1024/1024, 2) AS index_mb,
    ROUND(DATA_FREE/1024/1024, 2) AS free_mb,
    ROUND(DATA_FREE/(DATA_LENGTH + INDEX_LENGTH + DATA_FREE) * 100, 2) AS frag_pct
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys')
    AND DATA_FREE > 0
ORDER BY frag_pct DESC;
```

**SQL Server:**
```sql
-- Index fragmentation
SELECT 
    OBJECT_NAME(ips.object_id) AS table_name,
    i.name AS index_name,
    ips.index_type_desc,
    ips.avg_fragmentation_in_percent,
    ips.page_count,
    ips.avg_page_space_used_in_percent
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 10
    AND ips.page_count > 1000 -- Only consider indexes with significant pages
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

#### 4.1.3 Missing Index Identification

**PostgreSQL:**
```sql
-- Tables with high sequential scans (potential missing indexes)
SELECT 
    schemaname,
    tablename,
    seq_scan,
    seq_tup_read,
    idx_scan,
    seq_tup_read / NULLIF(seq_scan, 0) AS avg_seq_tup_read,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS table_size
FROM pg_stat_user_tables
WHERE seq_scan > 0
    AND seq_tup_read / NULLIF(seq_scan, 0) > 10000
ORDER BY seq_tup_read DESC;
```

**MySQL:**
```sql
-- Tables with many full table scans
SELECT 
    object_schema,
    object_name,
    COUNT_READ AS full_scans,
    COUNT_WRITE,
    ROUND((COUNT_READ / (COUNT_READ + COUNT_WRITE)) * 100, 2) AS read_pct
FROM performance_schema.table_io_waits_summary_by_table
WHERE object_schema NOT IN ('mysql', 'performance_schema', 'sys')
    AND COUNT_READ > 1000
ORDER BY COUNT_READ DESC;
```

**SQL Server:**
```sql
-- Missing index recommendations
SELECT 
    migs.avg_user_impact AS impact,
    migs.avg_total_user_cost AS cost,
    migs.user_seeks + migs.user_scans AS total_reads,
    mid.statement AS table_name,
    'CREATE INDEX idx_' + 
        REPLACE(REPLACE(mid.equality_columns, '[', ''), ']', '') +
        ISNULL('_' + REPLACE(REPLACE(mid.inequality_columns, '[', ''), ']', ''), '') +
        ' ON ' + mid.statement +
        ' (' + ISNULL(mid.equality_columns, '') + 
        ISNULL(CASE WHEN mid.inequality_columns IS NOT NULL 
            THEN ',' + mid.inequality_columns ELSE '' END, '') + ')' +
        ISNULL(' INCLUDE (' + mid.included_columns + ')', '') AS create_statement
FROM sys.dm_db_missing_index_details mid
JOIN sys.dm_db_missing_index_groups mig ON mid.index_handle = mig.index_handle
JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
WHERE migs.avg_user_impact > 50
ORDER BY migs.avg_user_impact * migs.avg_total_user_cost * (migs.user_seeks + migs.user_scans) DESC;
```

### 4.2 Index Maintenance Tasks

#### 4.2.1 Rebuild vs Reorganize

**Decision Matrix:**

| Fragmentation Level | Action | When |
|---------------------|--------|------|
| < 10% | None | Index is healthy |
| 10% - 30% | REORGANIZE | Weekly maintenance |
| > 30% | REBUILD | Monthly or as needed |

**PostgreSQL:**
```sql
-- Rebuild (requires lock)
REINDEX INDEX idx_name;
REINDEX TABLE table_name;

-- Concurrent rebuild (no lock, but slower)
CREATE INDEX CONCURRENTLY idx_name_new ON table_name(column);
DROP INDEX CONCURRENTLY idx_name;
ALTER INDEX idx_name_new RENAME TO idx_name;

-- Vacuum to reclaim space
VACUUM ANALYZE table_name;
```

**MySQL:**
```sql
-- Rebuild table and indexes
OPTIMIZE TABLE table_name;

-- Or alter table (rebuilds indexes)
ALTER TABLE table_name ENGINE=InnoDB;
```

**SQL Server:**
```sql
-- Reorganize (online, less resource intensive)
ALTER INDEX idx_name ON table_name REORGANIZE;

-- Rebuild (offline by default)
ALTER INDEX idx_name ON table_name REBUILD;

-- Rebuild online (Enterprise Edition)
ALTER INDEX idx_name ON table_name REBUILD WITH (ONLINE = ON);

-- Rebuild all indexes on a table
ALTER INDEX ALL ON table_name REBUILD;
```

#### 4.2.2 Update Statistics

**PostgreSQL:**
```sql
-- Analyze specific table
ANALYZE table_name;

-- Analyze all tables
ANALYZE;

-- Verbose output
ANALYZE VERBOSE table_name;

-- Set autovacuum parameters
ALTER TABLE table_name SET (
    autovacuum_vacuum_scale_factor = 0.05,
    autovacuum_analyze_scale_factor = 0.02
);
```

**MySQL:**
```sql
-- Update statistics
ANALYZE TABLE table_name;

-- Update multiple tables
ANALYZE TABLE table1, table2, table3;

-- Persistent statistics
ALTER TABLE table_name STATS_PERSISTENT=1;
```

**SQL Server:**
```sql
-- Update statistics for index
UPDATE STATISTICS table_name idx_name;

-- Update all statistics for table
UPDATE STATISTICS table_name;

-- Update with full scan
UPDATE STATISTICS table_name WITH FULLSCAN;

-- Auto update statistics configuration
ALTER DATABASE dbname SET AUTO_UPDATE_STATISTICS ON;
```

#### 4.2.3 Maintenance Schedule Template

**Daily Tasks:**
```sql
-- 1. Check for long-running queries
-- 2. Monitor index usage for anomalies
-- 3. Review error logs

-- PostgreSQL daily check
SELECT now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active' AND now() - query_start > interval '5 minutes'
ORDER BY duration DESC;
```

**Weekly Tasks:**
```sql
-- 1. Review slow query log
-- 2. Analyze new query patterns
-- 3. Update statistics on active tables
-- 4. Reorganize fragmented indexes (10-30%)

-- Weekly stats update script (PostgreSQL)
DO $$
DECLARE
    tbl RECORD;
BEGIN
    FOR tbl IN 
        SELECT schemaname, tablename 
        FROM pg_stat_user_tables 
        WHERE n_tup_ins + n_tup_upd + n_tup_del > 10000
    LOOP
        EXECUTE 'ANALYZE ' || tbl.schemaname || '.' || tbl.tablename;
    END LOOP;
END $$;
```

**Monthly Tasks:**
```sql
-- 1. Full index usage audit
-- 2. Remove unused indexes
-- 3. Rebuild heavily fragmented indexes (>30%)
-- 4. Review and optimize new indexes
-- 5. Capacity planning review

-- Monthly cleanup script (PostgreSQL example)
-- Identify and optionally drop unused indexes
SELECT 
    'DROP INDEX CONCURRENTLY ' || schemaname || '.' || indexrelname || ';' AS drop_statement
FROM pg_stat_user_indexes
WHERE idx_scan = 0
    AND indexrelname NOT LIKE '%_pkey'
    AND schemaname NOT IN ('pg_catalog', 'information_schema')
    AND pg_relation_size(indexrelid) > 1024*1024*10; -- > 10MB
```

**Quarterly Tasks:**
```sql
-- 1. Comprehensive performance review
-- 2. Index strategy reassessment
-- 3. Archival of old data
-- 4. Database growth analysis
-- 5. Hardware capacity evaluation
```

### 4.3 Performance Monitoring

#### 4.3.1 Query Performance Tracking

**PostgreSQL - pg_stat_statements:**
```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top 20 slowest queries
SELECT 
    ROUND(total_exec_time::numeric, 2) AS total_time_ms,
    calls,
    ROUND(mean_exec_time::numeric, 2) AS mean_time_ms,
    ROUND((100 * total_exec_time / SUM(total_exec_time) OVER ())::numeric, 2) AS pct_total,
    LEFT(query, 100) AS query_preview
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Queries with highest I/O
SELECT 
    query,
    calls,
    shared_blks_read + shared_blks_written AS total_io,
    ROUND((shared_blks_read + shared_blks_written)::numeric / calls, 2) AS avg_io_per_call
FROM pg_stat_statements
WHERE shared_blks_read + shared_blks_written > 0
ORDER BY total_io DESC
LIMIT 20;

-- Reset statistics
SELECT pg_stat_statements_reset();
```

**MySQL - Performance Schema:**
```sql
-- Enable performance schema
UPDATE performance_schema.setup_instruments 
SET ENABLED = 'YES', TIMED = 'YES' 
WHERE NAME LIKE '%statement%';

-- Top queries by execution time
SELECT 
    DIGEST_TEXT,
    COUNT_STAR AS executions,
    ROUND(AVG_TIMER_WAIT/1000000000, 2) AS avg_ms,
    ROUND(SUM_TIMER_WAIT/1000000000, 2) AS total_ms,
    ROUND(SUM_ROWS_EXAMINED/COUNT_STAR, 0) AS avg_rows_examined
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;

-- Queries not using indexes
SELECT 
    DIGEST_TEXT,
    COUNT_STAR,
    SUM_NO_INDEX_USED,
    SUM_NO_GOOD_INDEX_USED
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_NO_INDEX_USED > 0 OR SUM_NO_GOOD_INDEX_USED > 0
ORDER BY SUM_NO_INDEX_USED + SUM_NO_GOOD_INDEX_USED DESC;
```

**SQL Server - Query Store:**
```sql
-- Enable Query Store
ALTER DATABASE dbname SET QUERY_STORE = ON;

-- Top resource-consuming queries
SELECT TOP 20
    q.query_id,
    qt.query_sql_text,
    rs.count_executions,
    rs.avg_duration/1000 AS avg_duration_ms,
    rs.avg_logical_io_reads,
    rs.avg_physical_io_reads
FROM sys.query_store_query q
JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
ORDER BY rs.avg_duration DESC;

-- Queries with multiple plans
SELECT 
    q.query_id,
    COUNT(DISTINCT p.plan_id) AS plan_count,
    qt.query_sql_text
FROM sys.query_store_query q
JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan p ON q.query_id = p.query_id
GROUP BY q.query_id, qt.query_sql_text
HAVING COUNT(DISTINCT p.plan_id) > 1
ORDER BY plan_count DESC;
```

#### 4.3.2 Index Effectiveness Metrics

**Key Metrics to Track:**

1. **Index Scan Ratio**
```sql
-- PostgreSQL
SELECT 
    tablename,
    ROUND(100.0 * idx_scan / (seq_scan + idx_scan), 2) AS index_scan_pct
FROM pg_stat_user_tables
WHERE seq_scan + idx_scan > 0
ORDER BY index_scan_pct;
```

2. **Cache Hit Ratio**
```sql
-- PostgreSQL
SELECT 
    'Index Hit Rate' AS metric,
    ROUND(100.0 * sum(idx_blks_hit) / NULLIF(sum(idx_blks_hit + idx_blks_read), 0), 2) AS hit_rate_pct
FROM pg_statio_user_indexes;
```

3. **Index Size vs Usage**
```sql
-- PostgreSQL
SELECT 
    schemaname || '.' || tablename AS table_name,
    indexrelname AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan AS scans,
    CASE 
        WHEN idx_scan = 0 THEN 'UNUSED'
        WHEN idx_scan < 100 THEN 'LOW'
        WHEN idx_scan < 1000 THEN 'MEDIUM'
        ELSE 'HIGH'
    END AS usage_category
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;
```

### 4.4 Alerting and Thresholds

#### 4.4.1 Critical Alerts

**Set up monitoring for:**

1. **Unused Large Indexes** (> 100MB with 0 scans)
2. **High Fragmentation** (> 40%)
3. **Missing Indexes** (Optimizer suggestions with high impact)
4. **Slow Query Threshold** (> 1 second)
5. **Lock Waits** (> 30 seconds)
6. **Index Bloat** (> 50% wasted space)

**Example Alert Query (PostgreSQL):**
```sql
-- Alert: Large unused indexes
SELECT 
    schemaname,
    tablename,
    indexrelname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size,
    idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
    AND pg_relation_size(indexrelid) > 100 * 1024 * 1024 -- 100MB
    AND indexrelname NOT LIKE '%_pkey';
```

#### 4.4.2 Monitoring Dashboard Metrics

**Essential Dashboard Components:**

1. **Index Usage Overview**
   - Total indexes
   - Unused indexes count
   - Average scans per index
   - Total index size

2. **Query Performance**
   - Avg query time
   - Slow query count (> 1s)
   - Queries not using indexes
   - Top 10 slowest queries

3. **Index Health**
   - Fragmentation levels
   - Bloat percentage
   - Cache hit ratio
   - Index maintenance schedule compliance

4. **Growth Trends**
   - Table growth rate
   - Index growth rate
   - Storage capacity forecast

### 4.5 Best Practices Summary

#### Do's:
✅ Monitor index usage regularly (weekly minimum)
✅ Update statistics frequently on active tables
✅ Remove unused indexes after thorough verification
✅ Test index changes in staging first
✅ Document index purpose and queries they serve
✅ Use EXPLAIN ANALYZE before and after index changes
✅ Schedule maintenance during low-traffic periods
✅ Keep index naming conventions consistent
✅ Track index changes in version control
✅ Set up automated monitoring and alerts

#### Don'ts:
❌ Don't create indexes blindly without analysis
❌ Don't ignore fragmentation warnings
❌ Don't drop indexes without checking dependencies
❌ Don't over-index write-heavy tables
❌ Don't forget to update statistics after bulk operations
❌ Don't rebuild indexes during peak hours
❌ Don't assume all recommended indexes are beneficial
❌ Don't neglect to test performance impact
❌ Don't skip documentation of index strategy
❌ Don't ignore slow query logs

### 4.6 Troubleshooting Guide

#### Issue: Query not using expected index

**Diagnostic Steps:**
```sql
-- 1. Check if index exists
-- PostgreSQL
\d table_name

-- 2. Verify statistics are up to date
ANALYZE table_name;

-- 3. Check query plan
EXPLAIN ANALYZE SELECT ...;

-- 4. Look for function wrapping
-- Bad: WHERE LOWER(email) = 'user@example.com'
-- Good: WHERE email = 'user@example.com'

-- 5. Check data types match
SELECT column_name, data_type 
FROM information_schema.columns 
WHERE table_name = 'table_name';
```

**Common Causes:**
- Outdated statistics
- Function wrapping column
- Type mismatch in WHERE clause
- Index selectivity too low
- Table too small (full scan faster)

#### Issue: Slow index creation

**Solutions:**
```sql
-- PostgreSQL: Create index concurrently
CREATE INDEX CONCURRENTLY idx_name ON table_name(column);

-- Increase maintenance_work_mem temporarily
SET maintenance_work_mem = '2GB';
CREATE INDEX idx_name ON table_name(column);
RESET maintenance_work_mem;

-- SQL Server: Online index creation
CREATE INDEX idx_name ON table_name(column) WITH (ONLINE = ON);
```

#### Issue: Index bloat

**Diagnosis and Fix:**
```sql
-- PostgreSQL: Check bloat
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - 
        pg_relation_size(schemaname||'.'||tablename)) AS index_size
FROM pg_tables
WHERE schemaname = 'public';

-- Fix: Rebuild indexes
REINDEX TABLE table_name;

-- Or vacuum
VACUUM FULL table_name; -- Requires exclusive lock
```

---

## Conclusion

Effective database indexing is an ongoing process that requires:

1. **Strategic Planning**: Start with essential indexes and build based on actual usage
2. **Continuous Monitoring**: Regularly review index usage and performance metrics
3. **Adaptive Maintenance**: Adjust strategy as data volumes and query patterns evolve
4. **Performance Testing**: Always validate index changes with real-world queries

Remember:
- More indexes ≠ better performance
- Context matters: OLTP vs OLAP, read-heavy vs write-heavy
- Monitor, measure, and iterate
- Document your indexing decisions and rationale

The goal is to find the optimal balance between query performance, storage costs, and write overhead for your specific workload.