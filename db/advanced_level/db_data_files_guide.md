# Database Data Files: Structure and Performance

Database data files are the physical storage layer where your database actually persists data on disk. Understanding how databases organize and access these files is crucial for performance optimization and troubleshooting.

---

## What Are Database Data Files?

Database data files are binary files on disk that store:
- **Table data** (rows and columns)
- **Indexes** (B-trees, hash indexes, etc.)
- **Metadata** (schema definitions, constraints)
- **Transaction logs** (WAL, redo logs)

Different database systems organize these files differently, but the core concepts remain similar.

---

## Common Database File Structures

### PostgreSQL
- **Data files**: Stored in `base/` directory, organized by database OID
- **WAL (Write-Ahead Log)**: Stored in `pg_wal/` for crash recovery
- **Each table/index**: Stored in separate files (1GB segments)
- **TOAST**: Large objects stored separately in TOAST tables

### MySQL (InnoDB)
- **System tablespace**: `ibdata1` (stores data dictionary, undo logs)
- **File-per-table**: Each table gets its own `.ibd` file (recommended)
- **Redo logs**: `ib_logfile0`, `ib_logfile1` for crash recovery
- **Binary logs**: For replication and point-in-time recovery

### SQL Server
- **Primary data file**: `.mdf` (contains data and metadata)
- **Secondary data files**: `.ndf` (optional, for splitting data)
- **Transaction log**: `.ldf` (write-ahead logging)
- **Filegroups**: Logical groupings of data files

### SQLite
- **Single file**: Entire database in one `.db` file
- **WAL mode**: Optional `.db-wal` and `.db-shm` files
- **Extremely portable** but limited concurrency

---

## How Databases Read and Write Data

### Pages/Blocks: The Fundamental Unit
Databases don't read individual rows—they read **pages** (also called blocks):
- **PostgreSQL**: 8 KB pages
- **MySQL (InnoDB)**: 16 KB pages
- **SQL Server**: 8 KB pages
- **Oracle**: Configurable, typically 8 KB

**Why pages matter:**
- Even if you need 1 row, the entire page is loaded into memory
- Narrow tables = more rows per page = fewer disk I/O operations
- Wide tables = fewer rows per page = more disk reads

### The Read Process
1. **Query arrives** → Query planner determines which pages to read
2. **Check buffer pool** → If page is in RAM, read from memory (fast)
3. **Disk read** → If not cached, perform disk I/O (slow)
4. **Load into buffer** → Page loaded into memory for future queries
5. **Return result** → Extract needed rows/columns

### The Write Process
1. **Write to WAL first** (Write-Ahead Logging)
   - Changes written to sequential log file (fast)
   - Ensures durability even if crash occurs
2. **Modify buffer pool** → Update in-memory pages
3. **Background checkpoint** → Periodically flush dirty pages to data files
4. **Transaction commit** → WAL is synced to disk

---

## Key Performance Factors

### 1. Buffer Pool / Shared Buffers
The in-memory cache that holds frequently accessed pages.

**Configuration:**
- **PostgreSQL**: `shared_buffers` (default: 128 MB, recommend 25% of RAM)
- **MySQL**: `innodb_buffer_pool_size` (default: 128 MB, recommend 70-80% of RAM)
- **SQL Server**: Automatically managed, but can set max memory

**Impact:**
- Larger buffer pool = more cache hits = fewer disk reads
- Buffer pool too small = constant disk I/O (thrashing)

### 2. I/O Patterns: Sequential vs Random

| Type | Description | Performance |
|------|-------------|-------------|
| **Sequential I/O** | Reading consecutive pages (e.g., full table scan) | ~100-200 MB/s (HDD), ~500+ MB/s (SSD) |
| **Random I/O** | Jumping between pages (e.g., index lookups) | ~100-200 IOPS (HDD), ~10,000+ IOPS (SSD) |

**Why this matters:**
- Full table scans on small tables can be faster than index lookups
- SSDs massively improve random read performance
- Indexes help convert random reads to sequential reads

### 3. Page Size and Row Width

**Example:**
- Page size: 8 KB
- Row width: 200 bytes
- Rows per page: ~40 rows

If you need 1000 rows:
- **Normalized (narrow rows)**: Read 25 pages = 200 KB
- **Denormalized (wide rows, 800 bytes)**: Read 100 pages = 800 KB

**Result:** 4x more I/O for denormalized structure.

### 4. Fillfactor and Page Fragmentation

**Fillfactor** determines how full pages are kept:
- **100%**: Pages completely full (good for read-only data)
- **90%**: 10% free space for updates (reduces page splits)

**Fragmentation occurs when:**
- Frequent updates cause page splits
- Data spreads across non-contiguous pages
- Solution: `VACUUM` (PostgreSQL), `OPTIMIZE TABLE` (MySQL)

---

## Storage Layout Optimization

### Clustered vs Non-Clustered Indexes

**Clustered Index** (e.g., InnoDB primary key):
- Data physically sorted by index key
- Only one per table
- Range queries are extremely fast (sequential I/O)

**Non-Clustered Index**:
- Separate structure pointing to data rows
- Multiple allowed per table
- Requires additional I/O to fetch actual row data

### Partitioning
Splitting large tables across multiple files:

```sql
-- Example: Partition by date range
CREATE TABLE sales (
    sale_id INT,
    sale_date DATE,
    amount DECIMAL
) PARTITION BY RANGE (YEAR(sale_date)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
);
```

**Benefits:**
- **Partition pruning**: Only scan relevant partitions
- **Maintenance**: Drop old partitions instead of deleting rows
- **Parallel I/O**: Different partitions on different disks

### Compression
Many databases support page-level compression:
- **Pros**: Less disk space, fewer I/O operations
- **Cons**: CPU overhead for decompression
- **Best for**: Read-heavy workloads, archival data

---

## Monitoring and Troubleshooting

### Key Metrics to Watch

| Metric | What It Means | Tool |
|--------|---------------|------|
| **Cache Hit Ratio** | % of reads served from memory | `pg_stat_database`, `SHOW STATUS` |
| **Disk I/O Wait** | Time spent waiting for disk | `iostat`, Performance Monitor |
| **Page Splits** | Indicator of fragmentation | Database logs |
| **Checkpoint Frequency** | How often dirty pages are flushed | Database logs |

### Common Issues

**Problem: Low cache hit ratio (<90%)**
- **Solution**: Increase buffer pool size, add RAM

**Problem: High disk I/O wait**
- **Solution**: Add indexes, upgrade to SSD, partition data

**Problem: Slow writes**
- **Solution**: Increase WAL buffers, use faster storage, batch writes

**Problem: Table bloat**
- **Solution**: Run `VACUUM FULL` (PostgreSQL), `OPTIMIZE TABLE` (MySQL)

---

## Best Practices

1. **Size your buffer pool correctly** (25-80% of available RAM depending on database)
2. **Use SSDs** for database storage (massive random I/O improvement)
3. **Monitor cache hit ratios** and disk I/O metrics regularly
4. **Separate data and WAL files** onto different disks if possible
5. **Regular maintenance**: vacuum, analyze, rebuild indexes
6. **Partition large tables** (>100M rows) for better manageability
7. **Use appropriate page fillfactor** (90% for frequently updated tables)
8. **Avoid `SELECT *`**: Reduces pages loaded into memory

---

## Summary

Database data files are organized into fixed-size pages that are the fundamental unit of I/O. Performance depends heavily on:
- How much data fits in memory (buffer pool)
- Whether I/O is sequential or random
- Row width and table structure
- Index design and usage
- Storage hardware (HDD vs SSD)

Understanding the physical storage layer helps you make informed decisions about schema design, indexing strategies, and hardware investments.