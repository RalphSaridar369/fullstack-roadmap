# Clustered vs Non-Clustered Indexes: Complete Comparison

Indexes are database structures that improve query performance by providing fast access to data. Understanding the difference between **clustered** and **non-clustered** indexes is fundamental to database design and optimization.

---

## What Is an Index?

An index is like a book's index—it helps you find information quickly without reading every page. In databases:
- **Without index**: Database scans entire table (slow)
- **With index**: Database jumps directly to relevant rows (fast)

---

## Clustered Index

### Definition
A **clustered index** determines the **physical order** of data in a table. The table data is sorted and stored according to the clustered index key.

**Key concept:** The clustered index IS the table itself—there's no separate structure.

### How It Works

```
Clustered Index B-Tree Structure:

        Root Node
        [1-5000]
           |
    ┌──────┴──────┐
    |             |
  [1-2500]    [2501-5000]
    |             |
┌───┴───┐     ┌───┴────┐
|       |     |        |
Leaf Nodes (Actual Data Pages):

┌──────┬──────────┬───────────┬──────┐
│  ID  │   Name   │   Email   │ Age  │
├──────┼──────────┼───────────┼──────┤
│  1   │  Alice   │ a@ex.com  │  25  │
│  2   │  Bob     │ b@ex.com  │  30  │
│  3   │  Carol   │ c@ex.com  │  28  │
└──────┴──────────┴───────────┴──────┘
     (Data physically sorted by ID)
```

**Important characteristics:**
- Only **ONE** per table (data can only be sorted one way)
- Leaf nodes contain the actual data rows
- Data is physically reorganized when index is created

### Syntax Examples

#### SQL Server
```sql
-- Create table with clustered index on primary key
CREATE TABLE Employees (
    EmployeeID INT PRIMARY KEY CLUSTERED,
    FirstName VARCHAR(50),
    LastName VARCHAR(50),
    Salary DECIMAL(10,2)
);

-- Create clustered index on existing table
CREATE CLUSTERED INDEX IX_Employees_EmployeeID 
ON Employees(EmployeeID);

-- Create clustered index on different column
CREATE CLUSTERED INDEX IX_Employees_LastName 
ON Employees(LastName, FirstName);
```

#### MySQL (InnoDB)
```sql
-- Primary key is automatically clustered in InnoDB
CREATE TABLE Employees (
    EmployeeID INT PRIMARY KEY,  -- Automatically clustered
    FirstName VARCHAR(50),
    LastName VARCHAR(50),
    Salary DECIMAL(10,2)
);

-- You CANNOT create clustered index on other columns
-- The primary key is always the clustered index
```

#### PostgreSQL
```sql
-- PostgreSQL doesn't have true clustered indexes
-- But you can physically reorder a table by an index

-- Create index
CREATE INDEX IX_Employees_EmployeeID 
ON Employees(EmployeeID);

-- One-time clustering (not maintained automatically)
CLUSTER Employees USING IX_Employees_EmployeeID;
```

---

## Non-Clustered Index

### Definition
A **non-clustered index** is a **separate structure** that contains a sorted copy of selected columns plus a pointer back to the actual data row.

**Key concept:** The index is stored separately from the table data.

### How It Works

```
Non-Clustered Index B-Tree Structure:

        Root Node
      [A-M] [N-Z]
           |
    ┌──────┴──────┐
    |             |
  [A-G]         [H-M]
    |             |
┌───┴───┐     ┌───┴────┐
|       |     |        |
Leaf Nodes (Index Pages):

┌───────────┬──────────────────┐
│   Email   │  → Row Pointer   │
├───────────┼──────────────────┤
│ a@ex.com  │  → Row #1        │
│ b@ex.com  │  → Row #2        │
│ c@ex.com  │  → Row #3        │
└───────────┴──────────────────┘
               ↓
        (Lookup to data pages)

┌──────┬──────────┬───────────┬──────┐
│  ID  │   Name   │   Email   │ Age  │
├──────┼──────────┼───────────┼──────┤
│  2   │  Bob     │ b@ex.com  │  30  │  ← Found!
└──────┴──────────┴───────────┴──────┘
```

**Important characteristics:**
- **Multiple** allowed per table
- Leaf nodes contain index key + pointer/reference
- Requires additional storage space
- Lookups may need two steps: index → data

### Syntax Examples

#### SQL Server
```sql
-- Create non-clustered index
CREATE NONCLUSTERED INDEX IX_Employees_LastName 
ON Employees(LastName);

-- Create composite non-clustered index
CREATE NONCLUSTERED INDEX IX_Employees_Name 
ON Employees(LastName, FirstName);

-- Create covering index (includes extra columns)
CREATE NONCLUSTERED INDEX IX_Employees_LastName_Covering 
ON Employees(LastName)
INCLUDE (FirstName, Salary);

-- Create filtered index
CREATE NONCLUSTERED INDEX IX_Active_Employees 
ON Employees(EmployeeID)
WHERE IsActive = 1;

-- Create unique non-clustered index
CREATE UNIQUE NONCLUSTERED INDEX IX_Employees_Email 
ON Employees(Email);
```

#### MySQL
```sql
-- Create non-clustered index (called secondary index in MySQL)
CREATE INDEX IX_Employees_LastName 
ON Employees(LastName);

-- Create composite index
CREATE INDEX IX_Employees_Name 
ON Employees(LastName, FirstName);

-- Create unique index
CREATE UNIQUE INDEX IX_Employees_Email 
ON Employees(Email);

-- Add index to existing table
ALTER TABLE Employees 
ADD INDEX IX_Employees_Salary (Salary);
```

#### PostgreSQL
```sql
-- Create non-clustered index (standard B-tree)
CREATE INDEX IX_Employees_LastName 
ON Employees(LastName);

-- Create partial index (like filtered index)
CREATE INDEX IX_Active_Employees 
ON Employees(EmployeeID)
WHERE IsActive = true;

-- Create index with included columns
CREATE INDEX IX_Employees_LastName_Covering 
ON Employees(LastName)
INCLUDE (FirstName, Salary);

-- Create different index types
CREATE INDEX IX_Employees_Email_Hash 
ON Employees USING hash(Email);  -- Hash index

CREATE INDEX IX_Employees_Name_GIN 
ON Employees USING gin(to_tsvector('english', FirstName || ' ' || LastName));  -- Full-text
```

---

## Side-by-Side Comparison

| Aspect | Clustered Index | Non-Clustered Index |
|--------|-----------------|---------------------|
| **Physical ordering** | Dictates physical order of data | Separate structure, doesn't affect data order |
| **Number per table** | Only 1 | Multiple (typically 10-20 recommended) |
| **Storage** | No additional space (IS the table) | Requires additional disk space |
| **Leaf nodes contain** | Actual data rows (all columns) | Index key + pointer to data row |
| **Data access** | Direct (1 lookup) | Indirect (2 lookups: index → data) |
| **Insert performance** | Can be slow if key not sequential | Generally faster |
| **Update performance** | Slow if clustered key updated | Fast if indexed column not updated |
| **Range queries** | Extremely fast (sequential I/O) | Good, but may need multiple random lookups |
| **Covering queries** | Always covers (has all columns) | Only covers if all needed columns in index |
| **Fragmentation** | More prone (page splits) | Less prone |
| **Best for** | Primary access pattern, range scans | Secondary lookups, covering queries |

---

## How Each Works: Detailed Examples

### Clustered Index Lookup

```sql
-- Table with clustered index on EmployeeID
SELECT * FROM Employees WHERE EmployeeID = 12345;
```

**Steps:**
1. Navigate B-tree: Root → Intermediate → Leaf
2. Find row with EmployeeID = 12345
3. **Done!** All data is right there (1 lookup)

**I/O operations:** 3-4 page reads (tree depth)

---

### Non-Clustered Index Lookup

```sql
-- Non-clustered index on Email
SELECT * FROM Employees WHERE Email = 'john@example.com';
```

**Steps:**
1. Navigate non-clustered index B-tree
2. Find 'john@example.com' in index
3. Get pointer/key to data row
4. **Second lookup:** Navigate to actual data page
5. Retrieve full row

**I/O operations:** 6-8 page reads (index tree + data lookup)

---

### Covering Index (Non-Clustered)

```sql
-- Covering index includes all needed columns
CREATE INDEX IX_Email_Covering 
ON Employees(Email) 
INCLUDE (FirstName, LastName);

-- This query is "covered" by the index
SELECT FirstName, LastName 
FROM Employees 
WHERE Email = 'john@example.com';
```

**Steps:**
1. Navigate non-clustered index B-tree
2. Find 'john@example.com' in index
3. **Done!** FirstName and LastName are in the index (1 lookup)

**I/O operations:** 3-4 page reads (index tree only)

---

## Performance Comparison

### Scenario 1: Exact Match Lookup

```sql
SELECT * FROM Employees WHERE EmployeeID = 100;
```

| Index Type | I/O Operations | Speed |
|------------|----------------|-------|
| **Clustered** on EmployeeID | 3-4 reads | ⚡ Very Fast |
| **Non-Clustered** on EmployeeID | 6-8 reads | ⚡ Fast |
| **No Index** | 10,000+ reads (full scan) | 🐌 Very Slow |

---

### Scenario 2: Range Query

```sql
SELECT * FROM Employees 
WHERE EmployeeID BETWEEN 1000 AND 2000;
```

| Index Type | I/O Operations | Speed |
|------------|----------------|-------|
| **Clustered** on EmployeeID | 100-200 reads | ⚡⚡ Extremely Fast (sequential) |
| **Non-Clustered** on EmployeeID | 2000-4000 reads | 🐌 Slow (1000 random lookups) |
| **No Index** | 10,000+ reads | 🐌🐌 Very Slow |

**Winner:** Clustered index (data is contiguous)

---

### Scenario 3: Sorted Results

```sql
SELECT * FROM Employees ORDER BY EmployeeID;
```

| Index Type | Additional Work | Speed |
|------------|-----------------|-------|
| **Clustered** on EmployeeID | None (already sorted) | ⚡⚡ Instant |
| **Non-Clustered** on EmployeeID | Full table scan + sort | 🐌 Slow |
| **No Index** | Full scan + sort | 🐌 Slow |

**Winner:** Clustered index (no sorting needed)

---

### Scenario 4: Covering Query

```sql
SELECT Email, FirstName FROM Employees WHERE Email LIKE 'john%';
```

| Index Type | I/O Operations | Speed |
|------------|----------------|-------|
| **Covering non-clustered** on Email INCLUDE (FirstName) | 50-100 reads | ⚡⚡ Very Fast |
| **Non-clustered** on Email only | 1000-2000 reads | 🐌 Slower |
| **Clustered** on Email | 50-100 reads | ⚡⚡ Very Fast |

**Winner:** Tie (covering index vs clustered index)

---

### Scenario 5: Insert Performance

```sql
INSERT INTO Employees VALUES (NewRandomID, 'John', 'Doe', 50000);
```

| Clustered Key Type | Page Splits | Speed |
|--------------------|-------------|-------|
| **Sequential** (auto-increment) | Rare | ⚡ Fast |
| **Random** (UUID/GUID) | Frequent | 🐌 Slow |
| **Non-clustered** indexes | Not affected by data order | ⚡ Fast |

**Winner:** Non-clustered indexes (not affected by insert order)

---

## When to Use Each

### Use Clustered Index When:

✅ **Primary access pattern** for the table
```sql
-- Orders table: most queries by OrderID
CREATE TABLE Orders (
    OrderID BIGINT PRIMARY KEY CLUSTERED,  -- Clustered
    CustomerID BIGINT,
    OrderDate DATETIME
);
```

✅ **Range queries** are common
```sql
-- Logs table: queries by date range
CREATE CLUSTERED INDEX IX_Logs_Date ON Logs(LogDate);

SELECT * FROM Logs 
WHERE LogDate BETWEEN '2024-01-01' AND '2024-01-31';
```

✅ **Ordered results** frequently needed
```sql
-- Leaderboard: always ordered by score
CREATE CLUSTERED INDEX IX_Score ON Leaderboard(Score DESC);

SELECT * FROM Leaderboard ORDER BY Score DESC LIMIT 10;
```

✅ **Time-series data** with chronological access
```sql
CREATE CLUSTERED INDEX IX_Timestamp ON SensorData(Timestamp);
```

✅ **Small, stable keys** (INT, BIGINT, DATE)
```sql
-- Good: Small integer
EmployeeID INT PRIMARY KEY CLUSTERED

-- Bad: Wide, random key
SomeGUID UNIQUEIDENTIFIER PRIMARY KEY CLUSTERED  -- Avoid!
```

---

### Use Non-Clustered Index When:

✅ **Secondary search columns**
```sql
-- Search by email (not primary access pattern)
CREATE INDEX IX_Email ON Users(Email);

SELECT * FROM Users WHERE Email = 'user@example.com';
```

✅ **Multiple different access patterns**
```sql
-- Orders table with multiple search needs
CREATE INDEX IX_CustomerID ON Orders(CustomerID);
CREATE INDEX IX_OrderDate ON Orders(OrderDate);
CREATE INDEX IX_Status ON Orders(Status);
```

✅ **Covering specific queries**
```sql
-- Frequently run report query
CREATE INDEX IX_Covering 
ON Employees(Department) 
INCLUDE (Salary, FirstName, LastName);

-- This query is fully covered
SELECT FirstName, LastName, Salary 
FROM Employees 
WHERE Department = 'Engineering';
```

✅ **Unique constraints** on non-primary columns
```sql
CREATE UNIQUE INDEX IX_Email ON Users(Email);
CREATE UNIQUE INDEX IX_SSN ON Employees(SSN);
```

✅ **Filtering specific subsets**
```sql
-- Only index active employees
CREATE INDEX IX_Active 
ON Employees(LastName) 
WHERE Status = 'Active';
```

✅ **Foreign key columns** for joins
```sql
CREATE INDEX IX_CustomerID ON Orders(CustomerID);

-- Fast join
SELECT o.*, c.CustomerName
FROM Orders o
JOIN Customers c ON c.CustomerID = o.CustomerID;
```

---

## Common Mistakes to Avoid

### ❌ Mistake 1: Too Many Indexes
```sql
-- BAD: 15+ indexes on one table
CREATE INDEX IX1 ON Users(Email);
CREATE INDEX IX2 ON Users(FirstName);
CREATE INDEX IX3 ON Users(LastName);
CREATE INDEX IX4 ON Users(CreatedDate);
-- ... 11 more indexes ...
```
**Problem:** Slows down writes, wastes space, query optimizer confusion.
**Solution:** 3-5 indexes per table for OLTP, 5-10 for OLAP.

---

### ❌ Mistake 2: Wide Clustered Index Keys (MySQL)
```sql
-- BAD in MySQL: Wide clustered key
CREATE TABLE Users (
    UserID VARCHAR(255) PRIMARY KEY,  -- Avoid!
    Email VARCHAR(255),
    Name VARCHAR(100)
);
```
**Problem:** Every non-clustered index includes the full primary key.
**Solution:** Use small INT/BIGINT for primary keys.

---

### ❌ Mistake 3: Random Clustered Keys
```sql
-- BAD: Random UUIDs cause fragmentation
CREATE TABLE Sessions (
    SessionID UNIQUEIDENTIFIER PRIMARY KEY CLUSTERED,  -- Avoid!
    UserID INT,
    CreatedAt DATETIME
);
```
**Problem:** Page splits, fragmentation, slow inserts.
**Solution:** Use sequential keys or cluster by a different column.

---

### ❌ Mistake 4: Not Using Covering Indexes
```sql
-- BAD: Index doesn't cover common query
CREATE INDEX IX_Status ON Orders(Status);

-- This query needs to lookup data pages
SELECT OrderID, CustomerName, Total 
FROM Orders 
WHERE Status = 'Pending';
```

```sql
-- GOOD: Covering index
CREATE INDEX IX_Status_Covering 
ON Orders(Status) 
INCLUDE (OrderID, CustomerName, Total);

-- Now fully covered!
```

---

### ❌ Mistake 5: Indexing Low-Cardinality Columns
```sql
-- BAD: Only 2-3 distinct values
CREATE INDEX IX_Gender ON Users(Gender);  -- M, F, Other
CREATE INDEX IX_IsActive ON Users(IsActive);  -- True, False
```
**Problem:** Index not selective enough, optimizer may ignore it.
**Solution:** Use filtered indexes or composite indexes.

```sql
-- BETTER: Filtered index
CREATE INDEX IX_Active_Users 
ON Users(LastName) 
WHERE IsActive = 1;
```

---

## Index Maintenance

### Check Index Usage (SQL Server)
```sql
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    s.user_seeks,
    s.user_scans,
    s.user_lookups,
    s.user_updates
FROM sys.dm_db_index_usage_stats s
JOIN sys.indexes i ON s.object_id = i.object_id 
    AND s.index_id = i.index_id
WHERE OBJECT_NAME(s.object_id) = 'Employees'
ORDER BY s.user_updates DESC;
```

### Find Missing Indexes (SQL Server)
```sql
SELECT 
    OBJECT_NAME(d.object_id) AS TableName,
    d.equality_columns,
    d.inequality_columns,
    d.included_columns,
    s.avg_user_impact,
    s.user_seeks
FROM sys.dm_db_missing_index_details d
JOIN sys.dm_db_missing_index_groups g ON d.index_handle = g.index_handle
JOIN sys.dm_db_missing_index_group_stats s ON g.index_group_handle = s.group_handle
ORDER BY s.avg_user_impact * s.user_seeks DESC;
```

### Check Fragmentation
```sql
-- SQL Server
SELECT 
    OBJECT_NAME(ps.object_id) AS TableName,
    i.name AS IndexName,
    ps.avg_fragmentation_in_percent,
    ps.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ps
JOIN sys.indexes i ON ps.object_id = i.object_id AND ps.index_id = i.index_id
WHERE ps.avg_fragmentation_in_percent > 30
ORDER BY ps.avg_fragmentation_in_percent DESC;
```

### Rebuild Indexes
```sql
-- SQL Server
ALTER INDEX IX_Employees_LastName ON Employees REBUILD;
ALTER INDEX ALL ON Employees REBUILD;

-- MySQL
ALTER TABLE Employees ENGINE=InnoDB;

-- PostgreSQL
REINDEX TABLE Employees;
```

---

## Best Practices Summary

### Clustered Index Best Practices
1. ✅ Use small, stable, unique keys (INT, BIGINT)
2. ✅ Use ever-increasing values (auto-increment, timestamp)
3. ✅ Choose based on most common access pattern
4. ✅ Avoid UUIDs/GUIDs unless absolutely necessary
5. ✅ Never update the clustered key column

### Non-Clustered Index Best Practices
1. ✅ Index foreign key columns used in joins
2. ✅ Create covering indexes for critical queries
3. ✅ Use composite indexes (multi-column) wisely
4. ✅ Limit to 5-10 indexes per table for OLTP
5. ✅ Use filtered indexes for common subsets
6. ✅ Monitor index usage and remove unused indexes
7. ✅ Consider index maintenance overhead

---

## Quick Decision Guide

```
START: Need to speed up a query?
   |
   ├─> Querying by primary key?
   │   └─> Ensure primary key is clustered ✓
   │
   ├─> Range queries or ORDER BY on column?
   │   └─> Consider clustered index on that column
   │
   ├─> Exact match on non-primary column?
   │   └─> Create non-clustered index
   │
   ├─> Joining on foreign key?
   │   └─> Create non-clustered index on FK
   │
   ├─> Query always selects same few columns?
   │   └─> Create covering non-clustered index
   │
   └─> Low cardinality column (few distinct values)?
       └─> Consider filtered non-clustered index
```

---

## Conclusion

**Clustered indexes** determine physical data order and excel at range queries and sorted results. Use for your primary access pattern.

**Non-clustered indexes** are separate structures that enable fast lookups on secondary columns. Use for additional search paths and covering queries.

**Golden rule:** Start with a good clustered index (usually auto-increment primary key), then add non-clustered indexes based on actual query patterns. Monitor, measure, and refine.