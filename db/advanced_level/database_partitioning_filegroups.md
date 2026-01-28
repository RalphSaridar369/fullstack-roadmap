# Database Partitioning, Files & Filegroups: A Complete Guide

## Table of Contents
1. [Database Partitioning Strategies](#database-partitioning-strategies)
2. [File and Filegroup Management](#file-and-filegroup-management)
3. [Partitioning Implementation](#partitioning-implementation)
4. [Filegroup Assignment & Optimization](#filegroup-assignment--optimization)
5. [Maintenance & Monitoring](#maintenance--monitoring)

---

## 1. Database Partitioning Strategies

### 1.1 What is Partitioning?

Partitioning is the process of dividing large tables and indexes into smaller, more manageable pieces called partitions, while maintaining the logical view as a single object.

**Key Benefits:**
- Improved query performance on large tables
- Faster data loading and archival
- Better maintenance operations (rebuild individual partitions)
- Enhanced availability (partition-level operations)
- Optimized storage management

**Types of Partitioning:**
1. **Horizontal Partitioning** - Splitting rows across partitions
2. **Vertical Partitioning** - Splitting columns (less common)

### 1.2 Partitioning Strategies

#### 1.2.1 Range Partitioning

Most common strategy - data divided by ranges of values.

**Use Cases:**
- Time-series data (logs, events, transactions)
- Historical data with clear boundaries
- Sequential numeric data

**SQL Server Example:**
```sql
-- Step 1: Create partition function
CREATE PARTITION FUNCTION pf_OrderDate (DATE)
AS RANGE RIGHT FOR VALUES 
    ('2023-01-01', '2023-04-01', '2023-07-01', '2023-10-01', '2024-01-01');
-- Creates 6 partitions:
-- P1: < 2023-01-01
-- P2: 2023-01-01 to < 2023-04-01
-- P3: 2023-04-01 to < 2023-07-01
-- P4: 2023-07-01 to < 2023-10-01
-- P5: 2023-10-01 to < 2024-01-01
-- P6: >= 2024-01-01

-- Step 2: Create partition scheme (maps to filegroups)
CREATE PARTITION SCHEME ps_OrderDate
AS PARTITION pf_OrderDate
TO (FG_2022, FG_Q1_2023, FG_Q2_2023, FG_Q3_2023, FG_Q4_2023, FG_2024);

-- Step 3: Create partitioned table
CREATE TABLE Orders (
    OrderID BIGINT NOT NULL,
    OrderDate DATE NOT NULL,
    CustomerID INT,
    TotalAmount DECIMAL(10,2),
    Status VARCHAR(20),
    CONSTRAINT PK_Orders PRIMARY KEY (OrderID, OrderDate)
) ON ps_OrderDate(OrderDate);
```

**PostgreSQL Example:**
```sql
-- Create parent table
CREATE TABLE orders (
    order_id BIGINT NOT NULL,
    order_date DATE NOT NULL,
    customer_id INT,
    total_amount DECIMAL(10,2),
    status VARCHAR(20),
    PRIMARY KEY (order_id, order_date)
) PARTITION BY RANGE (order_date);

-- Create partitions
CREATE TABLE orders_2023_q1 PARTITION OF orders
    FOR VALUES FROM ('2023-01-01') TO ('2023-04-01')
    TABLESPACE ts_2023_q1;

CREATE TABLE orders_2023_q2 PARTITION OF orders
    FOR VALUES FROM ('2023-04-01') TO ('2023-07-01')
    TABLESPACE ts_2023_q2;

CREATE TABLE orders_2023_q3 PARTITION OF orders
    FOR VALUES FROM ('2023-07-01') TO ('2023-10-01')
    TABLESPACE ts_2023_q3;

CREATE TABLE orders_2023_q4 PARTITION OF orders
    FOR VALUES FROM ('2023-10-01') TO ('2024-01-01')
    TABLESPACE ts_2023_q4;

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO (MAXVALUE)
    TABLESPACE ts_2024;

-- Create default partition for out-of-range values
CREATE TABLE orders_default PARTITION OF orders DEFAULT;
```

**MySQL Example:**
```sql
CREATE TABLE orders (
    order_id BIGINT NOT NULL,
    order_date DATE NOT NULL,
    customer_id INT,
    total_amount DECIMAL(10,2),
    status VARCHAR(20),
    PRIMARY KEY (order_id, order_date)
)
PARTITION BY RANGE (YEAR(order_date) * 100 + MONTH(order_date)) (
    PARTITION p_2023_q1 VALUES LESS THAN (202304),
    PARTITION p_2023_q2 VALUES LESS THAN (202307),
    PARTITION p_2023_q3 VALUES LESS THAN (202310),
    PARTITION p_2023_q4 VALUES LESS THAN (202401),
    PARTITION p_2024 VALUES LESS THAN MAXVALUE
);
```

#### 1.2.2 List Partitioning

Data divided by discrete list of values.

**Use Cases:**
- Geographic regions
- Product categories
- Status codes
- Department codes

**SQL Server Example:**
```sql
-- Partition by region
CREATE PARTITION FUNCTION pf_Region (VARCHAR(10))
AS RANGE LEFT FOR VALUES 
    ('EAST', 'WEST', 'NORTH', 'SOUTH');

CREATE PARTITION SCHEME ps_Region
AS PARTITION pf_Region
TO (FG_East, FG_West, FG_North, FG_South, FG_Other);

CREATE TABLE Customers (
    CustomerID INT NOT NULL,
    Region VARCHAR(10) NOT NULL,
    Name VARCHAR(100),
    CONSTRAINT PK_Customers PRIMARY KEY (CustomerID, Region)
) ON ps_Region(Region);
```

**PostgreSQL Example:**
```sql
CREATE TABLE customers (
    customer_id INT NOT NULL,
    region VARCHAR(10) NOT NULL,
    name VARCHAR(100),
    PRIMARY KEY (customer_id, region)
) PARTITION BY LIST (region);

CREATE TABLE customers_east PARTITION OF customers
    FOR VALUES IN ('EAST', 'NORTHEAST', 'SOUTHEAST')
    TABLESPACE ts_east;

CREATE TABLE customers_west PARTITION OF customers
    FOR VALUES IN ('WEST', 'NORTHWEST', 'SOUTHWEST')
    TABLESPACE ts_west;

CREATE TABLE customers_central PARTITION OF customers
    FOR VALUES IN ('CENTRAL', 'MIDWEST')
    TABLESPACE ts_central;

CREATE TABLE customers_other PARTITION OF customers DEFAULT
    TABLESPACE ts_other;
```

#### 1.2.3 Hash Partitioning

Data distributed evenly across partitions using hash function.

**Use Cases:**
- Even data distribution needed
- No natural partitioning key
- Parallel processing optimization

**PostgreSQL Example:**
```sql
CREATE TABLE events (
    event_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    event_type VARCHAR(50),
    event_data JSONB,
    created_at TIMESTAMP,
    PRIMARY KEY (event_id, user_id)
) PARTITION BY HASH (user_id);

-- Create 8 partitions for even distribution
CREATE TABLE events_p0 PARTITION OF events
    FOR VALUES WITH (MODULUS 8, REMAINDER 0)
    TABLESPACE ts_events_0;

CREATE TABLE events_p1 PARTITION OF events
    FOR VALUES WITH (MODULUS 8, REMAINDER 1)
    TABLESPACE ts_events_1;

CREATE TABLE events_p2 PARTITION OF events
    FOR VALUES WITH (MODULUS 8, REMAINDER 2)
    TABLESPACE ts_events_2;

-- ... continue for all 8 partitions
```

**MySQL Example:**
```sql
CREATE TABLE events (
    event_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    event_type VARCHAR(50),
    event_data JSON,
    created_at TIMESTAMP,
    PRIMARY KEY (event_id, user_id)
)
PARTITION BY HASH(user_id)
PARTITIONS 8;
```

#### 1.2.4 Multi-Level (Composite) Partitioning

Partitioning on multiple levels for complex scenarios.

**PostgreSQL Example:**
```sql
-- Partition by year, then by month
CREATE TABLE sales (
    sale_id BIGINT,
    sale_date DATE NOT NULL,
    region VARCHAR(10) NOT NULL,
    amount DECIMAL(10,2),
    PRIMARY KEY (sale_id, sale_date, region)
) PARTITION BY RANGE (EXTRACT(YEAR FROM sale_date));

-- Create year partitions
CREATE TABLE sales_2023 PARTITION OF sales
    FOR VALUES FROM (2023) TO (2024)
    PARTITION BY LIST (region);

-- Create region sub-partitions within 2023
CREATE TABLE sales_2023_east PARTITION OF sales_2023
    FOR VALUES IN ('EAST')
    TABLESPACE ts_2023_east;

CREATE TABLE sales_2023_west PARTITION OF sales_2023
    FOR VALUES IN ('WEST')
    TABLESPACE ts_2023_west;

-- Repeat for 2024
CREATE TABLE sales_2024 PARTITION OF sales
    FOR VALUES FROM (2024) TO (2025)
    PARTITION BY LIST (region);

CREATE TABLE sales_2024_east PARTITION OF sales_2024
    FOR VALUES IN ('EAST')
    TABLESPACE ts_2024_east;

CREATE TABLE sales_2024_west PARTITION OF sales_2024
    FOR VALUES IN ('WEST')
    TABLESPACE ts_2024_west;
```

### 1.3 Choosing a Partitioning Strategy

**Decision Matrix:**

| Scenario | Strategy | Reason |
|----------|----------|--------|
| Time-series data | Range | Natural boundaries, easy archival |
| Geographic data | List | Discrete regions |
| User data (millions) | Hash | Even distribution |
| Multi-tenant SaaS | List (tenant_id) | Isolation, easy migration |
| Audit logs | Range (date) | Historical archival |
| IoT sensor data | Range (timestamp) + Hash (device_id) | Time + even distribution |
| E-commerce orders | Range (order_date) | Common query pattern |

**Key Considerations:**
1. **Query Patterns** - How is data typically queried?
2. **Data Distribution** - Is data evenly distributed?
3. **Growth Pattern** - Steady or seasonal growth?
4. **Maintenance Requirements** - Need to archive old data?
5. **Performance Goals** - Query speed vs. maintenance speed?

### 1.4 Partitioning Best Practices

#### ✅ Do's:

1. **Include partition key in PRIMARY KEY**
```sql
-- GOOD
PRIMARY KEY (order_id, order_date)

-- BAD - order_date not in PK
PRIMARY KEY (order_id)
```

2. **Align indexes with partitions**
```sql
-- SQL Server: Create aligned index
CREATE INDEX IX_Orders_Customer 
ON Orders(CustomerID, OrderDate)
ON ps_OrderDate(OrderDate);
```

3. **Plan for future partitions**
```sql
-- PostgreSQL: Create template for new partitions
-- Automate partition creation
CREATE OR REPLACE FUNCTION create_monthly_partition(p_date DATE)
RETURNS VOID AS $$
DECLARE
    partition_name TEXT;
    start_date DATE;
    end_date DATE;
BEGIN
    partition_name := 'orders_' || TO_CHAR(p_date, 'YYYY_MM');
    start_date := DATE_TRUNC('month', p_date);
    end_date := start_date + INTERVAL '1 month';
    
    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS %I PARTITION OF orders 
         FOR VALUES FROM (%L) TO (%L)
         TABLESPACE ts_%s',
        partition_name, start_date, end_date, TO_CHAR(p_date, 'YYYY_MM')
    );
END;
$$ LANGUAGE plpgsql;

-- Create next 12 months of partitions
SELECT create_monthly_partition(CURRENT_DATE + (n || ' months')::INTERVAL)
FROM generate_series(0, 11) n;
```

4. **Use partition elimination in queries**
```sql
-- GOOD - Partition key in WHERE clause
SELECT * FROM orders 
WHERE order_date BETWEEN '2024-01-01' AND '2024-01-31'
  AND customer_id = 12345;

-- BAD - No partition key, scans all partitions
SELECT * FROM orders WHERE customer_id = 12345;
```

#### ❌ Don'ts:

1. **Don't over-partition** - Too many small partitions increase overhead
2. **Don't partition small tables** - < 100GB usually don't need partitioning
3. **Don't forget to maintain** - Old partitions need archival/deletion
4. **Don't ignore partition key in queries** - Defeats the purpose

---

## 2. File and Filegroup Management

### 2.1 Understanding Database Files

**File Types:**

1. **Primary Data File (.mdf)** - Contains startup info and data
2. **Secondary Data Files (.ndf)** - Additional data files
3. **Log Files (.ldf)** - Transaction log

### 2.2 Filegroup Basics (SQL Server)

A filegroup is a logical grouping of database files.

**Default Filegroups:**
- **PRIMARY** - Contains system tables and objects not assigned elsewhere
- **User-defined** - Custom filegroups for organization

**Benefits:**
- Performance through I/O distribution
- Simplified backup/restore (filegroup-level)
- Storage management (place on different disks)
- Administrative flexibility

### 2.3 Creating Filegroups and Files

#### Basic Filegroup Setup

```sql
-- Create database with multiple filegroups
CREATE DATABASE SalesDB
ON PRIMARY
(
    NAME = 'SalesDB_Primary',
    FILENAME = 'D:\SQLData\SalesDB_Primary.mdf',
    SIZE = 100MB,
    MAXSIZE = 1GB,
    FILEGROWTH = 50MB
),
FILEGROUP FG_Current
(
    NAME = 'SalesDB_Current_Data1',
    FILENAME = 'E:\SQLData\SalesDB_Current_Data1.ndf',
    SIZE = 500MB,
    MAXSIZE = 10GB,
    FILEGROWTH = 100MB
),
(
    NAME = 'SalesDB_Current_Data2',
    FILENAME = 'F:\SQLData\SalesDB_Current_Data2.ndf',
    SIZE = 500MB,
    MAXSIZE = 10GB,
    FILEGROWTH = 100MB
),
FILEGROUP FG_Historical
(
    NAME = 'SalesDB_Historical_Data',
    FILENAME = 'G:\SQLData\SalesDB_Historical_Data.ndf',
    SIZE = 2GB,
    MAXSIZE = 50GB,
    FILEGROWTH = 500MB
),
FILEGROUP FG_Archive
(
    NAME = 'SalesDB_Archive_Data',
    FILENAME = 'H:\SQLArchive\SalesDB_Archive_Data.ndf',
    SIZE = 5GB,
    MAXSIZE = 100GB,
    FILEGROWTH = 1GB
)
LOG ON
(
    NAME = 'SalesDB_Log',
    FILENAME = 'L:\SQLLog\SalesDB_Log.ldf',
    SIZE = 1GB,
    MAXSIZE = 20GB,
    FILEGROWTH = 500MB
);
```

#### Adding Filegroups to Existing Database

```sql
-- Add new filegroup
ALTER DATABASE SalesDB
ADD FILEGROUP FG_2024_Q1;

-- Add files to filegroup
ALTER DATABASE SalesDB
ADD FILE
(
    NAME = 'SalesDB_2024_Q1_Data1',
    FILENAME = 'E:\SQLData\2024\SalesDB_2024_Q1_Data1.ndf',
    SIZE = 1GB,
    MAXSIZE = 20GB,
    FILEGROWTH = 500MB
) TO FILEGROUP FG_2024_Q1;

-- Add second file to same filegroup for I/O distribution
ALTER DATABASE SalesDB
ADD FILE
(
    NAME = 'SalesDB_2024_Q1_Data2',
    FILENAME = 'F:\SQLData\2024\SalesDB_2024_Q1_Data2.ndf',
    SIZE = 1GB,
    MAXSIZE = 20GB,
    FILEGROWTH = 500MB
) TO FILEGROUP FG_2024_Q1;
```

### 2.4 Filegroup Organization Strategies

#### Strategy 1: Time-Based Organization

```sql
-- Create quarterly filegroups
CREATE DATABASE TimeSeriesDB;
GO

ALTER DATABASE TimeSeriesDB ADD FILEGROUP FG_2023_Q1;
ALTER DATABASE TimeSeriesDB ADD FILEGROUP FG_2023_Q2;
ALTER DATABASE TimeSeriesDB ADD FILEGROUP FG_2023_Q3;
ALTER DATABASE TimeSeriesDB ADD FILEGROUP FG_2023_Q4;
ALTER DATABASE TimeSeriesDB ADD FILEGROUP FG_2024_Q1;
ALTER DATABASE TimeSeriesDB ADD FILEGROUP FG_2024_Q2;

-- Add files to each filegroup
ALTER DATABASE TimeSeriesDB ADD FILE
(
    NAME = 'TS_2023_Q1',
    FILENAME = 'D:\SQLData\2023\Q1\TimeSeriesDB_2023_Q1.ndf',
    SIZE = 2GB,
    FILEGROWTH = 500MB
) TO FILEGROUP FG_2023_Q1;

-- Repeat for each quarter...

-- Create partition function and scheme
CREATE PARTITION FUNCTION pf_Quarterly (DATE)
AS RANGE RIGHT FOR VALUES 
(
    '2023-01-01', '2023-04-01', '2023-07-01', '2023-10-01',
    '2024-01-01', '2024-04-01'
);

CREATE PARTITION SCHEME ps_Quarterly
AS PARTITION pf_Quarterly
TO (
    FG_2023_Q1, FG_2023_Q1, FG_2023_Q2, 
    FG_2023_Q3, FG_2023_Q4, 
    FG_2024_Q1, FG_2024_Q2
);
```

#### Strategy 2: Hot/Warm/Cold Data

```sql
-- Create filegroups based on access patterns
ALTER DATABASE DataWarehouse ADD FILEGROUP FG_Hot;    -- Frequently accessed
ALTER DATABASE DataWarehouse ADD FILEGROUP FG_Warm;   -- Moderately accessed
ALTER DATABASE DataWarehouse ADD FILEGROUP FG_Cold;   -- Rarely accessed

-- Hot data: Fast SSD storage
ALTER DATABASE DataWarehouse ADD FILE
(
    NAME = 'DW_Hot_Data',
    FILENAME = 'S:\FastSSD\DW_Hot_Data.ndf',  -- SSD
    SIZE = 10GB,
    FILEGROWTH = 1GB
) TO FILEGROUP FG_Hot;

-- Warm data: Standard SSD
ALTER DATABASE DataWarehouse ADD FILE
(
    NAME = 'DW_Warm_Data',
    FILENAME = 'D:\StandardSSD\DW_Warm_Data.ndf',
    SIZE = 50GB,
    FILEGROWTH = 5GB
) TO FILEGROUP FG_Warm;

-- Cold data: Cheaper HDD storage
ALTER DATABASE DataWarehouse ADD FILE
(
    NAME = 'DW_Cold_Data',
    FILENAME = 'H:\SlowHDD\DW_Cold_Data.ndf',
    SIZE = 200GB,
    FILEGROWTH = 20GB
) TO FILEGROUP FG_Cold;
```

#### Strategy 3: Functional Separation

```sql
-- Separate filegroups by object type/function
ALTER DATABASE AppDB ADD FILEGROUP FG_Tables;
ALTER DATABASE AppDB ADD FILEGROUP FG_Indexes;
ALTER DATABASE AppDB ADD FILEGROUP FG_LOB;  -- Large objects (BLOB, CLOB)
ALTER DATABASE AppDB ADD FILEGROUP FG_FullText;

-- Place files on appropriate storage
ALTER DATABASE AppDB ADD FILE
(
    NAME = 'AppDB_Tables',
    FILENAME = 'D:\SQLData\Tables\AppDB_Tables.ndf',
    SIZE = 10GB
) TO FILEGROUP FG_Tables;

ALTER DATABASE AppDB ADD FILE
(
    NAME = 'AppDB_Indexes',
    FILENAME = 'E:\SQLData\Indexes\AppDB_Indexes.ndf',
    SIZE = 5GB
) TO FILEGROUP FG_Indexes;

ALTER DATABASE AppDB ADD FILE
(
    NAME = 'AppDB_LOB',
    FILENAME = 'F:\SQLData\LOB\AppDB_LOB.ndf',
    SIZE = 50GB
) TO FILEGROUP FG_LOB;

-- Set default filegroup for tables
ALTER DATABASE AppDB MODIFY FILEGROUP FG_Tables DEFAULT;
```

### 2.5 File Placement Best Practices

#### Disk Layout Strategy

```
Disk Configuration Example:
├── C:\ (OS Drive - 100GB SSD)
│   └── SQL Server Binaries
│
├── D:\ (Data Drive 1 - 500GB SSD RAID 10)
│   ├── Primary Data Files (.mdf)
│   └── Current Year Data (.ndf)
│
├── E:\ (Data Drive 2 - 500GB SSD RAID 10)
│   ├── Recent Data (.ndf)
│   └── Indexes for current data
│
├── F:\ (Data Drive 3 - 1TB SSD RAID 10)
│   └── Historical Data (.ndf)
│
├── G:\ (Archive Drive - 2TB HDD RAID 5)
│   └── Archived/Cold Data (.ndf)
│
└── L:\ (Log Drive - 200GB SSD RAID 10)
    └── Transaction Logs (.ldf)
```

**Key Principles:**
1. **Separate logs from data** - Different I/O patterns
2. **Use RAID 10 for performance** - Data and log files
3. **Use RAID 5/6 for archive** - Cost-effective for cold data
4. **Multiple files per filegroup** - Parallel I/O (one per CPU core)
5. **Pre-size files appropriately** - Avoid autogrowth overhead

### 2.6 Monitoring File and Filegroup Usage

```sql
-- Check file usage
SELECT 
    DB_NAME() AS database_name,
    f.name AS file_logical_name,
    f.physical_name,
    fg.name AS filegroup_name,
    f.type_desc,
    CAST(f.size * 8.0 / 1024 AS DECIMAL(10,2)) AS size_mb,
    CAST(FILEPROPERTY(f.name, 'SpaceUsed') * 8.0 / 1024 AS DECIMAL(10,2)) AS used_mb,
    CAST((f.size - FILEPROPERTY(f.name, 'SpaceUsed')) * 8.0 / 1024 AS DECIMAL(10,2)) AS free_mb,
    CAST(100.0 * FILEPROPERTY(f.name, 'SpaceUsed') / f.size AS DECIMAL(5,2)) AS pct_used
FROM sys.database_files f
LEFT JOIN sys.filegroups fg ON f.data_space_id = fg.data_space_id
ORDER BY fg.name, f.name;

-- Check filegroup usage
SELECT 
    fg.name AS filegroup_name,
    COUNT(f.file_id) AS file_count,
    SUM(CAST(f.size * 8.0 / 1024 AS DECIMAL(10,2))) AS total_size_mb,
    SUM(CAST(FILEPROPERTY(f.name, 'SpaceUsed') * 8.0 / 1024 AS DECIMAL(10,2))) AS used_mb,
    SUM(CAST((f.size - FILEPROPERTY(f.name, 'SpaceUsed')) * 8.0 / 1024 AS DECIMAL(10,2))) AS free_mb
FROM sys.filegroups fg
JOIN sys.database_files f ON fg.data_space_id = f.data_space_id
GROUP BY fg.name
ORDER BY total_size_mb DESC;

-- Find which objects are in which filegroups
SELECT 
    OBJECT_SCHEMA_NAME(p.object_id) AS schema_name,
    OBJECT_NAME(p.object_id) AS object_name,
    i.name AS index_name,
    i.type_desc AS index_type,
    fg.name AS filegroup_name,
    p.rows AS row_count,
    au.total_pages * 8 / 1024 AS size_mb
FROM sys.partitions p
JOIN sys.allocation_units au ON p.partition_id = au.container_id
JOIN sys.indexes i ON p.object_id = i.object_id AND p.index_id = i.index_id
JOIN sys.filegroups fg ON au.data_space_id = fg.data_space_id
WHERE p.object_id > 100 -- Exclude system objects
ORDER BY au.total_pages DESC;
```

---

## 3. Partitioning Implementation

### 3.1 Complete Partitioning Example: E-Commerce Order System

#### Step 1: Design the Partitioning Strategy

**Requirements:**
- 100 million orders
- Most queries access last 3 months
- Need to archive data older than 2 years
- Monthly partition for easy management

#### Step 2: Create Filegroup Structure

```sql
USE master;
GO

-- Create database
CREATE DATABASE ECommerceDB
ON PRIMARY
(
    NAME = 'ECommerce_Primary',
    FILENAME = 'D:\SQLData\ECommerce_Primary.mdf',
    SIZE = 100MB,
    MAXSIZE = 1GB,
    FILEGROWTH = 50MB
)
LOG ON
(
    NAME = 'ECommerce_Log',
    FILENAME = 'L:\SQLLog\ECommerce_Log.ldf',
    SIZE = 500MB,
    MAXSIZE = 10GB,
    FILEGROWTH = 250MB
);
GO

USE ECommerceDB;
GO

-- Create filegroups for each month (example: 2023-2024)
DECLARE @year INT = 2023;
DECLARE @month INT;
DECLARE @sql NVARCHAR(MAX);
DECLARE @fg_name NVARCHAR(50);
DECLARE @file_name NVARCHAR(100);
DECLARE @file_path NVARCHAR(200);

WHILE @year <= 2024
BEGIN
    SET @month = 1;
    WHILE @month <= 12
    BEGIN
        SET @fg_name = 'FG_' + CAST(@year AS VARCHAR(4)) + '_' + RIGHT('0' + CAST(@month AS VARCHAR(2)), 2);
        SET @file_name = 'Orders_' + CAST(@year AS VARCHAR(4)) + '_' + RIGHT('0' + CAST(@month AS VARCHAR(2)), 2);
        SET @file_path = 'E:\SQLData\' + CAST(@year AS VARCHAR(4)) + '\' + @file_name + '.ndf';
        
        -- Create filegroup
        SET @sql = 'ALTER DATABASE ECommerceDB ADD FILEGROUP ' + @fg_name;
        EXEC sp_executesql @sql;
        
        -- Add file to filegroup
        SET @sql = '
        ALTER DATABASE ECommerceDB ADD FILE
        (
            NAME = ''' + @file_name + ''',
            FILENAME = ''' + @file_path + ''',
            SIZE = 500MB,
            MAXSIZE = 10GB,
            FILEGROWTH = 250MB
        ) TO FILEGROUP ' + @fg_name;
        EXEC sp_executesql @sql;
        
        SET @month = @month + 1;
    END
    SET @year = @year + 1;
END
GO

-- Create archive filegroup for old data
ALTER DATABASE ECommerceDB ADD FILEGROUP FG_Archive;
ALTER DATABASE ECommerceDB ADD FILE
(
    NAME = 'Orders_Archive',
    FILENAME = 'G:\SQLArchive\Orders_Archive.ndf',
    SIZE = 5GB,
    MAXSIZE = 100GB,
    FILEGROWTH = 1GB
) TO FILEGROUP FG_Archive;
GO

-- Create future filegroup for unbounded data
ALTER DATABASE ECommerceDB ADD FILEGROUP FG_Future;
ALTER DATABASE ECommerceDB ADD FILE
(
    NAME = 'Orders_Future',
    FILENAME = 'E:\SQLData\Future\Orders_Future.ndf',
    SIZE = 1GB,
    MAXSIZE = 20GB,
    FILEGROWTH = 500MB
) TO FILEGROUP FG_Future;
GO
```

#### Step 3: Create Partition Function

```sql
-- Create partition function for monthly partitions
CREATE PARTITION FUNCTION pf_OrderDate (DATE)
AS RANGE RIGHT FOR VALUES 
(
    -- 2023
    '2023-01-01', '2023-02-01', '2023-03-01', '2023-04-01',
    '2023-05-01', '2023-06-01', '2023-07-01', '2023-08-01',
    '2023-09-01', '2023-10-01', '2023-11-01', '2023-12-01',
    -- 2024
    '2024-01-01', '2024-02-01', '2024-03-01', '2024-04-01',
    '2024-05-01', '2024-06-01', '2024-07-01', '2024-08-01',
    '2024-09-01', '2024-10-01', '2024-11-01', '2024-12-01',
    -- 2025
    '2025-01-01'
);
GO

-- Verify partition function
SELECT 
    pf.name AS partition_function,
    prv.value AS boundary_value,
    prv.boundary_id
FROM sys.partition_functions pf
JOIN sys.partition_range_values prv ON pf.function_id = prv.function_id
WHERE pf.name = 'pf_OrderDate'
ORDER BY prv.boundary_id;
```

#### Step 4: Create Partition Scheme

```sql
-- Map partitions to filegroups
CREATE PARTITION SCHEME ps_OrderDate
AS PARTITION pf_OrderDate
TO (
    FG_Archive,        -- < 2023-01-01
    FG_2023_01, FG_2023_02, FG_2023_03, FG_2023_04,
    FG_2023_05, FG_2023_06, FG_2023_07, FG_2023_08,
    FG_2023_09, FG_2023_10, FG_2023_11, FG_2023_12,
    FG_2024_01, FG_2024_02, FG_2024_03, FG_2024_04,
    FG_2024_05, FG_2024_06, FG_2024_07, FG_2024_08,
    FG_2024_09, FG_2024_10, FG_2024_11, FG_2024_12,
    FG_Future          -- >= 2025-01-01
);
GO

-- Verify partition scheme
SELECT 
    ps.name AS partition_scheme,
    pf.name AS partition_function,
    ds.name AS filegroup_name,
    prv.value AS boundary_value
FROM sys.partition_schemes ps
JOIN sys.partition_functions pf ON ps.function_id = pf.function_id
LEFT JOIN sys.destination_data_spaces dds ON ps.data_space_id = dds.partition_scheme_id
LEFT JOIN sys.data_spaces ds ON dds.data_space_id = ds.data_space_id
LEFT JOIN sys.partition_range_values prv ON pf.function_id = prv.function_id 
    AND dds.destination_id = prv.boundary_id + 1
WHERE ps.name = 'ps_OrderDate'
ORDER BY dds.destination_id;
```

#### Step 5: Create Partitioned Table

```sql
CREATE TABLE Orders (
    OrderID BIGINT NOT NULL,
    OrderDate DATE NOT NULL,
    CustomerID INT NOT NULL,
    OrderTotal DECIMAL(10,2),
    Status VARCHAR(20),
    ShippingAddress NVARCHAR(500),
    CreatedAt DATETIME2 DEFAULT SYSDATETIME(),
    ModifiedAt DATETIME2,
    CONSTRAINT PK_Orders PRIMARY KEY CLUSTERED (OrderDate, OrderID)
) ON ps_OrderDate(OrderDate);
GO

-- Create partitioned non-clustered indexes
CREATE NONCLUSTERED INDEX IX_Orders_Customer
ON Orders(CustomerID, OrderDate)
INCLUDE (OrderTotal, Status)
ON ps_OrderDate(OrderDate);

CREATE NONCLUSTERED INDEX IX_Orders_Status
ON Orders(Status, OrderDate)
INCLUDE (CustomerID, OrderTotal)
ON ps_OrderDate(OrderDate);
GO
```

#### Step 6: Create Related Tables

```sql
-- Order items table (also partitioned)
CREATE TABLE OrderItems (
    OrderItemID BIGINT IDENTITY(1,1) NOT NULL,
    OrderID BIGINT NOT NULL,
    OrderDate DATE NOT NULL,  -- Redundant but needed for partitioning
    ProductID INT NOT NULL,
    Quantity INT,
    UnitPrice DECIMAL(10,2),
    CONSTRAINT PK_OrderItems PRIMARY KEY CLUSTERED (OrderDate, OrderID, OrderItemID),
    CONSTRAINT FK_OrderItems_Orders FOREIGN KEY (OrderDate, OrderID)
        REFERENCES Orders(OrderDate, OrderID)
) ON ps_OrderDate(OrderDate);
GO

CREATE NONCLUSTERED INDEX IX_OrderItems_Product
ON OrderItems(ProductID, OrderDate)
ON ps_OrderDate(OrderDate);
GO
```

### 3.2 Sliding Window Pattern for Partitioning

Efficient pattern for managing time-based partitions - add new, archive old.

#### Automated Partition Management

```sql
-- Create stored procedure to add new partition
CREATE PROCEDURE sp_AddMonthlyPartition
    @NextMonth DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @Year VARCHAR(4) = CAST(YEAR(@NextMonth) AS VARCHAR(4));
    DECLARE @Month VARCHAR(2) = RIGHT('0' + CAST(MONTH(@NextMonth) AS VARCHAR(2)), 2);
    DECLARE @FG_Name NVARCHAR(50) = 'FG_' + @Year + '_' + @Month;
    DECLARE @File_Name NVARCHAR(100) = 'Orders_' + @Year + '_' + @Month;
    DECLARE @File_Path NVARCHAR(200) = 'E:\SQLData\' + @Year + '\' + @File_Name + '.ndf';
    DECLARE @sql NVARCHAR(MAX);
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Create filegroup if not exists
        IF NOT EXISTS (SELECT 1 FROM sys.filegroups WHERE name = @FG_Name)
        BEGIN
            SET @sql = 'ALTER DATABASE ' + DB_NAME() + ' ADD FILEGROUP ' + @FG_Name;
            EXEC sp_executesql @sql;
            
            -- Add file to filegroup
            SET @sql = '
            ALTER DATABASE ' + DB_NAME() + ' ADD FILE
            (
                NAME = ''' + @File_Name + ''',
                FILENAME = ''' + @File_Path + ''',
                SIZE = 500MB,
                MAXSIZE = 10GB,
                FILEGROWTH = 250MB
            ) TO FILEGROUP ' + @FG_Name;
            EXEC sp_executesql @sql;
        END
        
        -- Split partition to create new partition
        ALTER PARTITION SCHEME ps_OrderDate
        NEXT USED @FG_Name;
        
        ALTER PARTITION FUNCTION pf_OrderDate()
        SPLIT RANGE (@NextMonth);
        
        COMMIT TRANSACTION;
        
        PRINT 'Successfully created partition for ' + CAST(@NextMonth AS VARCHAR(10));
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END
GO

-- Create procedure to archive old partition
CREATE PROCEDURE sp_ArchiveOldPartition
    @ArchiveDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @PartitionNumber INT;
    DECLARE @RowCount BIGINT;
    
    -- Find partition number for the date
    SELECT @PartitionNumber = $PARTITION.pf_OrderDate(@ArchiveDate);
    
    -- Check row count
    SELECT @RowCount = SUM(p.rows)
    FROM sys.partitions p
    JOIN sys.indexes i ON p.object_id = i.object_id AND p.index_id = i.index_id
    WHERE p.object_id = OBJECT_ID('Orders')
        AND i.type <= 1 -- Clustered or heap
        AND p.partition_number = @PartitionNumber;
    
    PRINT 'Partition ' + CAST(@PartitionNumber AS VARCHAR(10)) + 
          ' contains ' + CAST(@RowCount AS VARCHAR(20)) + ' rows';
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Switch partition to staging table
        CREATE TABLE Orders_Archive_Staging (
            OrderID BIGINT NOT NULL,
            OrderDate DATE NOT NULL,
            CustomerID INT NOT NULL,
            OrderTotal DECIMAL(10,2),
            Status VARCHAR(20),
            ShippingAddress NVARCHAR(500),
            CreatedAt DATETIME2,
            ModifiedAt DATETIME2,
            CONSTRAINT PK_Orders_Archive_Staging PRIMARY KEY CLUSTERED (OrderDate, OrderID)
        ) ON FG_Archive;
        
        -- Switch partition out
        ALTER TABLE Orders
        SWITCH PARTITION @PartitionNumber TO Orders_Archive_Staging;
        
        -- Optional: Compress staging table
        ALTER TABLE Orders_Archive_Staging REBUILD WITH (DATA_COMPRESSION = PAGE);
        
        -- Rename to permanent archive table
        DECLARE @ArchiveTableName NVARCHAR(128) = 'Orders_Archive_' + 
            REPLACE(CAST(@ArchiveDate AS VARCHAR(10)), '-', '_');
        
        EXEC sp_rename 'Orders_Archive_Staging', @ArchiveTableName;
        
        -- Merge the now-empty partition
        ALTER PARTITION FUNCTION pf_OrderDate()
        MERGE RANGE (@ArchiveDate);
        
        COMMIT TRANSACTION;
        
        PRINT 'Successfully archived partition to ' + @ArchiveTableName;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        IF OBJECT_ID('Orders_Archive_Staging') IS NOT NULL
            DROP TABLE Orders_Archive_Staging;
        THROW;
    END CATCH
END
GO

-- Schedule monthly partition maintenance
CREATE PROCEDURE sp_MonthlyPartitionMaintenance
AS
BEGIN
    -- Add partition for next month
    DECLARE @NextMonth DATE = DATEADD(MONTH, 1, DATEFROMPARTS(YEAR(GETDATE()), MONTH(GETDATE()), 1));
    EXEC sp_AddMonthlyPartition @NextMonth;
    
    -- Archive data older than 24 months
    DECLARE @ArchiveDate DATE = DATEADD(MONTH, -24, DATEFROMPARTS(YEAR(GETDATE()), MONTH(GETDATE()), 1));
    EXEC sp_ArchiveOldPartition @ArchiveDate;
END
GO
```

### 3.3 Loading Data into Partitioned Tables

#### Bulk Load Best Practices

```sql
-- Method 1: Direct INSERT (partition-aware)
INSERT INTO Orders (OrderID, OrderDate, CustomerID, OrderTotal, Status)
SELECT OrderID, OrderDate, CustomerID, OrderTotal, Status
FROM StagingOrders
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2024-02-01';

-- Method 2: Partition switch (fastest for large data)
-- Step 1: Create staging table on same filegroup
CREATE TABLE Orders_Staging_2024_01 (
    OrderID BIGINT NOT NULL,
    OrderDate DATE NOT NULL,
    CustomerID INT NOT NULL,
    OrderTotal DECIMAL(10,2),
    Status VARCHAR(20),
    ShippingAddress NVARCHAR(500),
    CreatedAt DATETIME2,
    ModifiedAt DATETIME2,
    CONSTRAINT PK_Orders_Staging_2024_01 PRIMARY KEY CLUSTERED (OrderDate, OrderID),
    CONSTRAINT CK_Orders_Staging_2024_01_Date CHECK (
        OrderDate >= '2024-01-01' AND OrderDate < '2024-02-01'
    )
) ON FG_2024_01;

-- Step 2: Load data into staging
INSERT INTO Orders_Staging_2024_01
SELECT * FROM ExternalDataSource
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2024-02-01';

-- Step 3: Create matching indexes
CREATE NONCLUSTERED INDEX IX_Staging_Customer
ON Orders_Staging_2024_01(CustomerID, OrderDate)
INCLUDE (OrderTotal, Status);

-- Step 4: Switch partition (metadata-only operation, instant!)
DECLARE @PartitionNumber INT = $PARTITION.pf_OrderDate('2024-01-01');

ALTER TABLE Orders_Staging_2024_01
SWITCH TO Orders PARTITION @PartitionNumber;

-- Step 5: Drop staging table
DROP TABLE Orders_Staging_2024_01;
```

---

## 4. Filegroup Assignment & Optimization

### 4.1 Strategic Filegroup Assignment

#### Assign Tables to Filegroups

```sql
-- Create table on specific filegroup
CREATE TABLE Products (
    ProductID INT PRIMARY KEY,
    ProductName NVARCHAR(200),
    CategoryID INT,
    Price DECIMAL(10,2)
) ON FG_Tables;

-- Move existing table to different filegroup
CREATE CLUSTERED INDEX PK_Products_New ON Products(ProductID)
WITH (DROP_EXISTING = ON) ON FG_Hot;
```

#### Assign Indexes to Filegroups

```sql
-- Create index on specific filegroup
CREATE NONCLUSTERED INDEX IX_Products_Category
ON Products(CategoryID)
ON FG_Indexes;

-- Separate table and index filegroups
CREATE TABLE Orders (
    OrderID INT PRIMARY KEY CLUSTERED,
    OrderDate DATE,
    CustomerID INT
) ON FG_Tables;

CREATE NONCLUSTERED INDEX IX_Orders_Customer
ON Orders(CustomerID)
ON FG_Indexes;

CREATE NONCLUSTERED INDEX IX_Orders_Date
ON Orders(OrderDate)
ON FG_Indexes;
```

#### LOB Data Separation

```sql
-- Store large objects separately
CREATE TABLE Documents (
    DocumentID INT PRIMARY KEY,
    DocumentName NVARCHAR(200),
    DocumentContent VARBINARY(MAX),
    CreatedDate DATETIME
) ON FG_Tables
TEXTIMAGE_ON FG_LOB; -- Separate filegroup for LOB data
```

### 4.2 Performance Optimization Through Filegroups

#### Multi-File Filegroups for Parallel I/O

```sql
-- Create filegroup with multiple files for parallel I/O
ALTER DATABASE SalesDB ADD FILEGROUP FG_Sales_Hot;

-- Add multiple files (one per CPU core recommended)
ALTER DATABASE SalesDB ADD FILE
(NAME = 'Sales_Hot_01', FILENAME = 'D:\SQLData\Sales_Hot_01.ndf', SIZE = 1GB)
TO FILEGROUP FG_Sales_Hot;

ALTER DATABASE SalesDB ADD FILE
(NAME = 'Sales_Hot_02', FILENAME = 'E:\SQLData\Sales_Hot_02.ndf', SIZE = 1GB)
TO FILEGROUP FG_Sales_Hot;

ALTER DATABASE SalesDB ADD FILE
(NAME = 'Sales_Hot_03', FILENAME = 'F:\SQLData\Sales_Hot_03.ndf', SIZE = 1GB)
TO FILEGROUP FG_Sales_Hot;

ALTER DATABASE SalesDB ADD FILE
(NAME = 'Sales_Hot_04', FILENAME = 'G:\SQLData\Sales_Hot_04.ndf', SIZE = 1GB)
TO FILEGROUP FG_Sales_Hot;

-- SQL Server will automatically distribute data across files using proportional fill
```

#### Read-Only Filegroups for Historical Data

```sql
-- Mark filegroup as read-only (prevents modifications)
ALTER DATABASE SalesDB MODIFY FILEGROUP FG_2022 READ_ONLY;

-- Benefits:
-- 1. No need to backup read-only filegroups repeatedly
-- 2. Can be stored on cheaper, slower storage
-- 3. Prevents accidental modifications
-- 4. Can exclude from regular backup schedule

-- To make writable again:
ALTER DATABASE SalesDB MODIFY FILEGROUP FG_2022 READWRITE;
```

#### Piecemeal Restore Strategy

```sql
-- Backup strategy for large databases
-- Full backup of PRIMARY filegroup
BACKUP DATABASE SalesDB
FILEGROUP = 'PRIMARY'
TO DISK = 'D:\Backups\SalesDB_PRIMARY_Full.bak'
WITH INIT;

-- Differential backup of active filegroups only
BACKUP DATABASE SalesDB
FILEGROUP = 'FG_2024_01', FILEGROUP = 'FG_2024_02'
TO DISK = 'D:\Backups\SalesDB_Current_Diff.bak'
WITH DIFFERENTIAL;

-- Read-only filegroups: backup once, never again
BACKUP DATABASE SalesDB
FILEGROUP = 'FG_2022'
TO DISK = 'D:\Backups\SalesDB_2022_Full.bak'
WITH INIT;

-- Mark as read-only
ALTER DATABASE SalesDB MODIFY FILEGROUP FG_2022 READ_ONLY;

-- Piecemeal restore (restore only what's needed)
-- Step 1: Restore PRIMARY filegroup first
RESTORE DATABASE SalesDB
FILEGROUP = 'PRIMARY'
FROM DISK = 'D:\Backups\SalesDB_PRIMARY_Full.bak'
WITH PARTIAL, NORECOVERY;

-- Step 2: Restore recent filegroups
RESTORE DATABASE SalesDB
FILEGROUP = 'FG_2024_01'
FROM DISK = 'D:\Backups\SalesDB_Current_Diff.bak'
WITH NORECOVERY;

-- Step 3: Recover database (older filegroups offline)
RESTORE DATABASE SalesDB WITH RECOVERY;

-- Step 4: Bring older filegroups online as needed
RESTORE DATABASE SalesDB
FILEGROUP = 'FG_2022'
FROM DISK = 'D:\Backups\SalesDB_2022_Full.bak'
WITH RECOVERY;
```

### 4.3 Compression Strategies by Filegroup

```sql
-- Apply different compression levels based on access patterns

-- Hot data: No compression (maximum performance)
CREATE TABLE CurrentOrders (
    OrderID INT,
    OrderDate DATE,
    CustomerID INT
) ON FG_Hot;

-- Warm data: ROW compression (good balance)
CREATE TABLE RecentOrders (
    OrderID INT,
    OrderDate DATE,
    CustomerID INT
) ON FG_Warm
WITH (DATA_COMPRESSION = ROW);

-- Cold data: PAGE compression (maximum compression)
CREATE TABLE ArchivedOrders (
    OrderID INT,
    OrderDate DATE,
    CustomerID INT
) ON FG_Cold
WITH (DATA_COMPRESSION = PAGE);

-- Compress existing partitions differently
ALTER TABLE Orders REBUILD PARTITION = 1 WITH (DATA_COMPRESSION = PAGE);    -- Old data
ALTER TABLE Orders REBUILD PARTITION = 12 WITH (DATA_COMPRESSION = ROW);    -- Recent data
ALTER TABLE Orders REBUILD PARTITION = 24 WITH (DATA_COMPRESSION = NONE);   -- Current data
```

---

## 5. Maintenance & Monitoring

### 5.1 Partition Monitoring

#### View Partition Information

```sql
-- Comprehensive partition view
SELECT 
    OBJECT_SCHEMA_NAME(p.object_id) AS schema_name,
    OBJECT_NAME(p.object_id) AS table_name,
    i.name AS index_name,
    p.partition_number,
    fg.name AS filegroup_name,
    prv.value AS boundary_value,
    p.rows AS row_count,
    au.total_pages * 8 / 1024 AS size_mb,
    au.used_pages * 8 / 1024 AS used_mb,
    au.data_pages * 8 / 1024 AS data_mb,
    ps.data_compression_desc AS compression
FROM sys.partitions p
JOIN sys.indexes i ON p.object_id = i.object_id AND p.index_id = i.index_id
JOIN sys.allocation_units au ON p.partition_id = au.container_id
JOIN sys.partition_schemes psch ON i.data_space_id = psch.data_space_id
JOIN sys.destination_data_spaces dds ON psch.data_space_id = dds.partition_scheme_id 
    AND p.partition_number = dds.destination_id
JOIN sys.filegroups fg ON dds.data_space_id = fg.data_space_id
JOIN sys.partition_functions pf ON psch.function_id = pf.function_id
LEFT JOIN sys.partition_range_values prv ON pf.function_id = prv.function_id 
    AND p.partition_number = prv.boundary_id + 1
WHERE p.object_id = OBJECT_ID('Orders')
    AND i.index_id <= 1  -- Clustered index or heap only
ORDER BY p.partition_number;

-- Partition summary by month
SELECT 
    p.partition_number,
    ISNULL(CAST(prv.value AS DATE), 'Unbounded') AS partition_boundary,
    fg.name AS filegroup,
    FORMAT(SUM(p.rows), 'N0') AS total_rows,
    CAST(SUM(au.total_pages) * 8.0 / 1024 / 1024 AS DECIMAL(10,2)) AS size_gb,
    ps.data_compression_desc AS compression
FROM sys.partitions p
JOIN sys.indexes i ON p.object_id = i.object_id AND p.index_id = i.index_id
JOIN sys.allocation_units au ON p.partition_id = au.container_id
JOIN sys.partition_schemes psch ON i.data_space_id = psch.data_space_id
JOIN sys.destination_data_spaces dds ON psch.data_space_id = dds.partition_scheme_id 
    AND p.partition_number = dds.destination_id
JOIN sys.filegroups fg ON dds.data_space_id = fg.data_space_id
JOIN sys.partition_functions pf ON psch.function_id = pf.function_id
LEFT JOIN sys.partition_range_values prv ON pf.function_id = prv.function_id 
    AND p.partition_number = CASE 
        WHEN pf.boundary_value_on_right = 1 THEN prv.boundary_id + 1
        ELSE prv.boundary_id
    END
WHERE p.object_id = OBJECT_ID('Orders')
    AND i.index_id <= 1
GROUP BY p.partition_number, prv.value, fg.name, ps.data_compression_desc
ORDER BY p.partition_number;
```

#### Identify Partition for Specific Date

```sql
-- Find which partition contains a specific date
DECLARE @SearchDate DATE = '2024-03-15';

SELECT 
    $PARTITION.pf_OrderDate(@SearchDate) AS partition_number,
    fg.name AS filegroup_name,
    df.physical_name AS file_path
FROM sys.partition_schemes ps
JOIN sys.destination_data_spaces dds 
    ON ps.data_space_id = dds.partition_scheme_id
    AND dds.destination_id = $PARTITION.pf_OrderDate(@SearchDate)
JOIN sys.filegroups fg ON dds.data_space_id = fg.data_space_id
JOIN sys.database_files df ON fg.data_space_id = df.data_space_id
WHERE ps.name = 'ps_OrderDate';
```

### 5.2 Filegroup Health Monitoring

```sql
-- Filegroup space usage
SELECT 
    fg.name AS filegroup_name,
    df.name AS file_name,
    df.physical_name,
    CAST(df.size * 8.0 / 1024 AS DECIMAL(10,2)) AS size_mb,
    CAST(df.size * 8.0 / 1024 / 1024 AS DECIMAL(10,2)) AS size_gb,
    CAST(FILEPROPERTY(df.name, 'SpaceUsed') * 8.0 / 1024 AS DECIMAL(10,2)) AS used_mb,
    CAST((df.size - FILEPROPERTY(df.name, 'SpaceUsed')) * 8.0 / 1024 AS DECIMAL(10,2)) AS free_mb,
    CAST(100.0 * FILEPROPERTY(df.name, 'SpaceUsed') / df.size AS DECIMAL(5,2)) AS pct_used,
    CASE 
        WHEN df.max_size = -1 THEN 'Unlimited'
        WHEN df.max_size = 268435456 THEN 'Unlimited'
        ELSE CAST(df.max_size * 8.0 / 1024 / 1024 AS VARCHAR(20)) + ' GB'
    END AS max_size,
    CASE df.is_read_only WHEN 1 THEN 'READ_ONLY' ELSE 'READWRITE' END AS status
FROM sys.database_files df
LEFT JOIN sys.filegroups fg ON df.data_space_id = fg.data_space_id
WHERE df.type = 0  -- Data files only
ORDER BY fg.name, df.name;

-- Alert: Files nearing capacity
SELECT 
    fg.name AS filegroup_name,
    df.name AS file_name,
    CAST(100.0 * FILEPROPERTY(df.name, 'SpaceUsed') / df.size AS DECIMAL(5,2)) AS pct_used,
    CAST((df.size - FILEPROPERTY(df.name, 'SpaceUsed')) * 8.0 / 1024 AS DECIMAL(10,2)) AS free_mb
FROM sys.database_files df
LEFT JOIN sys.filegroups fg ON df.data_space_id = fg.data_space_id
WHERE df.type = 0 
    AND CAST(100.0 * FILEPROPERTY(df.name, 'SpaceUsed') / df.size AS DECIMAL(5,2)) > 80
ORDER BY pct_used DESC;
```

### 5.3 Maintenance Scripts

#### Rebuild Partition Indexes

```sql
-- Rebuild specific partition
ALTER INDEX ALL ON Orders
REBUILD PARTITION = 12 
WITH (DATA_COMPRESSION = PAGE, ONLINE = ON);

-- Rebuild all partitions in a filegroup
DECLARE @partition_number INT;
DECLARE partition_cursor CURSOR FOR
SELECT DISTINCT p.partition_number
FROM sys.partitions p
JOIN sys.indexes i ON p.object_id = i.object_id AND p.index_id = i.index_id
JOIN sys.partition_schemes ps ON i.data_space_id = ps.data_space_id
JOIN sys.destination_data_spaces dds ON ps.data_space_id = dds.partition_scheme_id 
    AND p.partition_number = dds.destination_id
JOIN sys.filegroups fg ON dds.data_space_id = fg.data_space_id
WHERE p.object_id = OBJECT_ID('Orders')
    AND fg.name = 'FG_2023_01';

OPEN partition_cursor;
FETCH NEXT FROM partition_cursor INTO @partition_number;

WHILE @@FETCH_STATUS = 0
BEGIN
    PRINT 'Rebuilding partition ' + CAST(@partition_number AS VARCHAR(10));
    
    DECLARE @sql NVARCHAR(MAX) = 
        'ALTER INDEX ALL ON Orders REBUILD PARTITION = ' + 
        CAST(@partition_number AS VARCHAR(10)) + 
        ' WITH (DATA_COMPRESSION = PAGE, ONLINE = ON)';
    
    EXEC sp_executesql @sql;
    
    FETCH NEXT FROM partition_cursor INTO @partition_number;
END

CLOSE partition_cursor;
DEALLOCATE partition_cursor;
```

#### Update Statistics on Partitions

```sql
-- Update statistics on specific partition
UPDATE STATISTICS Orders WITH FULLSCAN ON PARTITIONS(12);

-- Update statistics on all partitions
DECLARE @partition INT = 1;
DECLARE @max_partition INT;

SELECT @max_partition = MAX(partition_number)
FROM sys.partitions
WHERE object_id = OBJECT_ID('Orders')
    AND index_id <= 1;

WHILE @partition <= @max_partition
BEGIN
    PRINT 'Updating statistics for partition ' + CAST(@partition AS VARCHAR(10));
    UPDATE STATISTICS Orders WITH FULLSCAN ON PARTITIONS(@partition);
    SET @partition = @partition + 1;
END
```

### 5.4 Automated Maintenance Jobs

#### SQL Agent Job: Monthly Partition Maintenance

```sql
-- Create job to run monthly
USE msdb;
GO

EXEC sp_add_job
    @job_name = 'Monthly_Partition_Maintenance',
    @enabled = 1,
    @description = 'Add new partition and archive old data';

EXEC sp_add_jobstep
    @job_name = 'Monthly_Partition_Maintenance',
    @step_name = 'Execute_Partition_Maintenance',
    @subsystem = 'TSQL',
    @database_name = 'ECommerceDB',
    @command = 'EXEC sp_MonthlyPartitionMaintenance;';

-- Schedule: First day of each month at 2 AM
EXEC sp_add_schedule
    @schedule_name = 'Monthly_Schedule',
    @freq_type = 16,  -- Monthly
    @freq_interval = 1,  -- First day
    @active_start_time = 020000;  -- 2:00 AM

EXEC sp_attach_schedule
    @job_name = 'Monthly_Partition_Maintenance',
    @schedule_name = 'Monthly_Schedule';

EXEC sp_add_jobserver
    @job_name = 'Monthly_Partition_Maintenance';
GO
```

### 5.5 Performance Monitoring

```sql
-- Identify queries that aren't using partition elimination
SELECT 
    qs.execution_count,
    qs.total_logical_reads,
    qs.total_elapsed_time / 1000000.0 AS total_elapsed_seconds,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS query_text,
    qp.query_plan
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
WHERE st.text LIKE '%Orders%'
    AND qp.query_plan.exist('//RelOp[@PhysicalOp="Table Scan"]') = 1
    AND OBJECT_NAME(st.objectid) = 'Orders'
ORDER BY qs.total_logical_reads DESC;

-- Check if partition elimination is occurring
SET STATISTICS XML ON;

SELECT * FROM Orders
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2024-02-01';

SET STATISTICS XML OFF;
-- Look for "Actual Partition Count" in execution plan
```

---

## Conclusion

### Key Takeaways

**Partitioning:**
- Use for tables > 100GB
- Choose strategy based on query patterns
- Include partition key in all queries for elimination
- Automate partition maintenance

**Filegroups:**
- Separate hot/warm/cold data
- Use multiple files for parallel I/O
- Place on appropriate storage tiers
- Implement read-only for historical data

**Best Practices:**
1. **Plan ahead** - Design partition strategy before implementation
2. **Monitor continuously** - Track partition sizes and access patterns
3. **Automate maintenance** - Use sliding window for time-based partitions
4. **Test thoroughly** - Validate performance improvements
5. **Document everything** - Maintenance procedures and partition boundaries

**Common Pitfalls to Avoid:**
- ❌ Over-partitioning (too many small partitions)
- ❌ Forgetting partition key in WHERE clauses
- ❌ Not automating partition management
- ❌ Placing all files on same physical disk
- ❌ Ignoring alignment between partition and filegroup strategy

**Performance Checklist:**
- ✅ Partition key in WHERE clause
- ✅ Statistics up to date
- ✅ Indexes aligned with partitions
- ✅ Appropriate compression per partition
- ✅ Files distributed across multiple disks
- ✅ Regular maintenance scheduled
- ✅ Monitoring in place

This comprehensive approach to partitioning and filegroup management will help you build scalable, maintainable, and high-performance database solutions.