# How to Monitor Database Indexes: A Complete Guide

## Why Monitor Indexes?

Database indexes are critical for performance, but they come with costs:

- **Storage overhead**: Indexes consume disk space
- **Write performance impact**: Every INSERT, UPDATE, DELETE maintains all indexes
- **Maintenance overhead**: Indexes need periodic rebuilding or reorganization
- **Memory consumption**: Frequently used indexes occupy buffer cache

Without proper monitoring, you might have:
- **Unused indexes** wasting resources
- **Missing indexes** causing slow queries
- **Fragmented indexes** degrading performance
- **Duplicate indexes** providing no additional value

Effective monitoring helps you maintain optimal database performance.

## Key Metrics to Monitor

### 1. Index Usage Statistics
- **Scan count**: How many times the index was used
- **Seek count**: Number of index seeks (point lookups)
- **Lookup count**: Rows retrieved from the table after index access
- **Update count**: Number of write operations affecting the index

### 2. Index Physical Characteristics
- **Size**: Disk space consumed
- **Fragmentation**: Percentage of logical disorder
- **Fill factor**: How full index pages are
- **Page count**: Number of pages used
- **Depth**: B-tree levels (affects lookup time)

### 3. Index Effectiveness
- **Selectivity**: How well the index narrows results
- **Cache hit ratio**: Percentage of reads from memory vs disk
- **Read/write ratio**: Balance between reads helped vs writes slowed
- **Query coverage**: Queries that can use this index

### 4. Maintenance Metrics
- **Last rebuild/reorganize date**: When maintenance was last performed
- **Statistics update date**: Freshness of query optimizer statistics
- **Lock contention**: Blocking caused by index maintenance

## Monitoring by Database Platform

### PostgreSQL

#### Check Index Usage

```sql
-- Overview of all indexes with usage statistics
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan AS index_scans,
    idx_tup_read AS tuples_read,
    idx_tup_fetch AS tuples_fetched,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
```

#### Find Unused Indexes

```sql
-- Indexes that have never been used
SELECT 
    schemaname || '.' || tablename AS table,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size,
    idx_scan AS scans
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelid NOT IN (
      SELECT conindid 
      FROM pg_constraint 
      WHERE contype IN ('p', 'u')  -- Exclude PK and unique constraints
  )
ORDER BY pg_relation_size(indexrelid) DESC;
```

#### Check Index Bloat/Fragmentation

```sql
-- Estimate index bloat
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    ROUND(100 * (pg_relation_size(indexrelid) - 
          (CASE WHEN relpages > 0 THEN relpages * 8192 ELSE 0 END)::numeric) / 
          NULLIF(pg_relation_size(indexrelid), 0), 2) AS bloat_pct
FROM pg_stat_user_indexes
JOIN pg_class ON pg_class.oid = indexrelid
WHERE pg_relation_size(indexrelid) > 100000  -- > 100KB
ORDER BY pg_relation_size(indexrelid) DESC;
```

#### Identify Duplicate Indexes

```sql
-- Find indexes on the same columns
SELECT 
    pg_size_pretty(SUM(pg_relation_size(idx))::BIGINT) AS total_size,
    array_agg(indexrelname) AS indexes,
    indrelid::regclass AS table
FROM (
    SELECT 
        indexrelid AS idx,
        indrelid,
        indexrelname,
        indkey::text AS columns
    FROM pg_index
    JOIN pg_class ON pg_class.oid = pg_index.indexrelid
    WHERE indisprimary = false
) sub
GROUP BY indrelid, columns
HAVING COUNT(*) > 1
ORDER BY SUM(pg_relation_size(idx)) DESC;
```

#### Monitor Index Cache Hit Ratio

```sql
-- Index cache effectiveness
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_blks_hit AS cache_hits,
    idx_blks_read AS disk_reads,
    ROUND(100.0 * idx_blks_hit / NULLIF(idx_blks_hit + idx_blks_read, 0), 2) AS cache_hit_ratio
FROM pg_statio_user_indexes
WHERE idx_blks_hit + idx_blks_read > 0
ORDER BY cache_hit_ratio ASC
LIMIT 20;
```

#### Check Missing Indexes (via slow queries)

```sql
-- Install pg_stat_statements extension first
-- CREATE EXTENSION pg_stat_statements;

-- Find slow queries without index usage
SELECT 
    LEFT(query, 100) AS query_snippet,
    calls,
    ROUND(total_exec_time::numeric, 2) AS total_time_ms,
    ROUND(mean_exec_time::numeric, 2) AS mean_time_ms,
    rows
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_%'
  AND mean_exec_time > 100  -- Queries taking > 100ms
ORDER BY mean_exec_time DESC
LIMIT 20;
```

### MySQL/MariaDB

#### Check Index Usage

```sql
-- View index statistics (MySQL 5.7+)
SELECT 
    table_schema,
    table_name,
    index_name,
    stat_value AS rows_read
FROM mysql.innodb_index_stats
WHERE stat_name = 'n_rows_read'
  AND table_schema NOT IN ('mysql', 'information_schema', 'performance_schema')
ORDER BY stat_value DESC;
```

#### Find Unused Indexes

```sql
-- Using Performance Schema (MySQL 5.6+)
SELECT 
    object_schema AS database_name,
    object_name AS table_name,
    index_name,
    COUNT_STAR AS index_usage_count
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE index_name IS NOT NULL
  AND index_name != 'PRIMARY'
  AND COUNT_STAR = 0
ORDER BY object_schema, object_name;
```

#### Check Index Cardinality

```sql
-- Low cardinality indexes may be ineffective
SELECT 
    table_schema,
    table_name,
    index_name,
    column_name,
    cardinality,
    ROUND(cardinality / NULLIF((
        SELECT table_rows 
        FROM information_schema.tables t
        WHERE t.table_schema = s.table_schema 
          AND t.table_name = s.table_name
    ), 0) * 100, 2) AS selectivity_pct
FROM information_schema.statistics s
WHERE table_schema NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
  AND cardinality IS NOT NULL
ORDER BY selectivity_pct ASC;
```

#### Monitor Index Size

```sql
-- Index sizes by table
SELECT 
    table_schema,
    table_name,
    ROUND(data_length / 1024 / 1024, 2) AS data_mb,
    ROUND(index_length / 1024 / 1024, 2) AS index_mb,
    ROUND(index_length / data_length * 100, 2) AS index_ratio_pct
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
  AND index_length > 0
ORDER BY index_length DESC;
```

#### Find Duplicate Indexes

```sql
-- Detect redundant indexes
SELECT 
    table_schema,
    table_name,
    GROUP_CONCAT(index_name) AS duplicate_indexes,
    GROUP_CONCAT(column_name ORDER BY seq_in_index) AS columns
FROM information_schema.statistics
WHERE table_schema NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
GROUP BY table_schema, table_name, index_name
HAVING COUNT(*) > 1;
```

### Microsoft SQL Server

#### Check Index Usage

```sql
-- Detailed index usage statistics
SELECT 
    OBJECT_NAME(s.object_id) AS table_name,
    i.name AS index_name,
    i.type_desc AS index_type,
    s.user_seeks,
    s.user_scans,
    s.user_lookups,
    s.user_updates,
    s.user_seeks + s.user_scans + s.user_lookups AS total_reads,
    s.user_updates AS total_writes,
    CASE 
        WHEN s.user_updates > 0 
        THEN (s.user_seeks + s.user_scans + s.user_lookups) / s.user_updates
        ELSE 0 
    END AS read_write_ratio
FROM sys.dm_db_index_usage_stats s
INNER JOIN sys.indexes i ON s.object_id = i.object_id 
    AND s.index_id = i.index_id
WHERE OBJECTPROPERTY(s.object_id, 'IsUserTable') = 1
ORDER BY total_reads DESC;
```

#### Find Unused Indexes

```sql
-- Indexes with no reads but write overhead
SELECT 
    OBJECT_NAME(i.object_id) AS table_name,
    i.name AS index_name,
    i.type_desc AS index_type,
    s.user_updates AS writes,
    (8 * SUM(a.used_pages)) / 1024 AS index_size_mb
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats s 
    ON i.object_id = s.object_id 
    AND i.index_id = s.index_id
LEFT JOIN sys.partitions p ON i.object_id = p.object_id 
    AND i.index_id = p.index_id
LEFT JOIN sys.allocation_units a ON p.partition_id = a.container_id
WHERE OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
  AND i.index_id > 1  -- Exclude clustered index
  AND (s.user_seeks + s.user_scans + s.user_lookups = 0 
       OR s.user_seeks IS NULL)
  AND s.user_updates > 0
GROUP BY i.object_id, i.name, i.type_desc, s.user_updates
ORDER BY s.user_updates DESC, index_size_mb DESC;
```

#### Check Index Fragmentation

```sql
-- Index fragmentation levels
SELECT 
    OBJECT_NAME(ips.object_id) AS table_name,
    i.name AS index_name,
    ips.index_type_desc,
    ips.avg_fragmentation_in_percent,
    ips.page_count,
    CASE 
        WHEN ips.avg_fragmentation_in_percent > 30 THEN 'REBUILD'
        WHEN ips.avg_fragmentation_in_percent > 10 THEN 'REORGANIZE'
        ELSE 'OK'
    END AS recommendation
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
INNER JOIN sys.indexes i ON ips.object_id = i.object_id 
    AND ips.index_id = i.index_id
WHERE ips.page_count > 100  -- Only indexes with > 100 pages
  AND OBJECTPROPERTY(ips.object_id, 'IsUserTable') = 1
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

#### Identify Missing Indexes

```sql
-- SQL Server's missing index recommendations
SELECT 
    OBJECT_NAME(mid.object_id) AS table_name,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns,
    migs.unique_compiles,
    migs.user_seeks,
    migs.avg_total_user_cost,
    migs.avg_user_impact,
    ROUND(migs.avg_total_user_cost * migs.avg_user_impact * 
          (migs.user_seeks + migs.user_scans), 0) AS improvement_measure,
    'CREATE INDEX idx_' + OBJECT_NAME(mid.object_id) + '_missing ON ' +
    mid.statement + ' (' + ISNULL(mid.equality_columns, '') +
    CASE WHEN mid.inequality_columns IS NOT NULL 
         THEN CASE WHEN mid.equality_columns IS NOT NULL THEN ',' ELSE '' END + 
              mid.inequality_columns 
         ELSE '' END + ')' +
    CASE WHEN mid.included_columns IS NOT NULL 
         THEN ' INCLUDE (' + mid.included_columns + ')' 
         ELSE '' END AS create_statement
FROM sys.dm_db_missing_index_details mid
INNER JOIN sys.dm_db_missing_index_groups mig 
    ON mid.index_handle = mig.index_handle
INNER JOIN sys.dm_db_missing_index_group_stats migs 
    ON mig.index_group_handle = migs.group_handle
WHERE migs.avg_user_impact > 30  -- Focus on high-impact indexes
ORDER BY improvement_measure DESC;
```

#### Find Duplicate Indexes

```sql
-- Detect overlapping indexes
WITH IndexColumns AS (
    SELECT 
        OBJECT_NAME(ic.object_id) AS table_name,
        i.name AS index_name,
        i.is_unique,
        i.type_desc,
        STRING_AGG(c.name, ',') WITHIN GROUP (ORDER BY ic.key_ordinal) AS columns
    FROM sys.indexes i
    INNER JOIN sys.index_columns ic ON i.object_id = ic.object_id 
        AND i.index_id = ic.index_id
    INNER JOIN sys.columns c ON ic.object_id = c.object_id 
        AND ic.column_id = c.column_id
    WHERE OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
      AND ic.is_included_column = 0
    GROUP BY ic.object_id, i.name, i.is_unique, i.type_desc
)
SELECT 
    a.table_name,
    a.index_name AS index1,
    b.index_name AS index2,
    a.columns AS shared_columns
FROM IndexColumns a
INNER JOIN IndexColumns b ON a.table_name = b.table_name
    AND a.columns = b.columns
    AND a.index_name < b.index_name
ORDER BY a.table_name, a.index_name;
```

### Oracle Database

#### Check Index Usage

```sql
-- Index usage monitoring (requires monitoring to be enabled)
-- First enable: ALTER INDEX index_name MONITORING USAGE;

SELECT 
    i.owner,
    i.index_name,
    i.table_name,
    u.monitoring,
    u.used,
    u.start_monitoring,
    u.end_monitoring
FROM dba_indexes i
LEFT JOIN v$object_usage u ON i.index_name = u.index_name
    AND i.owner = u.owner
WHERE i.owner NOT IN ('SYS', 'SYSTEM', 'OUTLN', 'DBSNMP')
ORDER BY i.owner, i.table_name;
```

#### Index Statistics

```sql
-- Detailed index statistics
SELECT 
    i.owner,
    i.table_name,
    i.index_name,
    i.index_type,
    i.uniqueness,
    i.blevel AS tree_depth,
    i.leaf_blocks,
    i.distinct_keys,
    i.clustering_factor,
    ROUND(s.bytes / 1024 / 1024, 2) AS size_mb,
    i.num_rows,
    ROUND((i.num_rows / NULLIF(i.distinct_keys, 0)), 2) AS avg_rows_per_key
FROM dba_indexes i
LEFT JOIN dba_segments s ON i.owner = s.owner 
    AND i.index_name = s.segment_name
WHERE i.owner NOT IN ('SYS', 'SYSTEM', 'OUTLN', 'DBSNMP')
ORDER BY s.bytes DESC NULLS LAST;
```

#### Check Index Health

```sql
-- Analyze index efficiency
SELECT 
    index_name,
    table_name,
    blevel,
    leaf_blocks,
    clustering_factor,
    num_rows,
    CASE 
        WHEN blevel > 4 THEN 'High B-tree depth'
        WHEN clustering_factor > num_rows THEN 'Poor clustering'
        WHEN distinct_keys < num_rows * 0.01 THEN 'Low cardinality'
        ELSE 'OK'
    END AS health_status
FROM user_indexes
WHERE num_rows > 0
ORDER BY 
    CASE 
        WHEN blevel > 4 THEN 1
        WHEN clustering_factor > num_rows THEN 2
        WHEN distinct_keys < num_rows * 0.01 THEN 3
        ELSE 4
    END;
```

## Monitoring Strategies

### 1. Automated Daily Checks

Create a monitoring routine that runs daily:

**PostgreSQL Example:**
```sql
-- Create monitoring table
CREATE TABLE index_monitoring (
    check_date DATE,
    schema_name TEXT,
    table_name TEXT,
    index_name TEXT,
    index_scans BIGINT,
    index_size BIGINT,
    cache_hit_ratio NUMERIC
);

-- Daily monitoring job
INSERT INTO index_monitoring
SELECT 
    CURRENT_DATE,
    schemaname,
    tablename,
    indexname,
    idx_scan,
    pg_relation_size(indexrelid),
    ROUND(100.0 * idx_blks_hit / NULLIF(idx_blks_hit + idx_blks_read, 0), 2)
FROM pg_stat_user_indexes
JOIN pg_statio_user_indexes USING (indexrelid);
```

### 2. Alert Thresholds

Set up alerts for critical conditions:

- **Unused indexes** (0 scans after 30 days)
- **High fragmentation** (> 30%)
- **Low cache hit ratio** (< 90%)
- **Excessive index size** (> 10x table size)
- **Missing indexes** (frequent slow queries)

### 3. Weekly Index Health Report

**Sample Report Query (PostgreSQL):**
```sql
SELECT 
    'Unused Indexes' AS category,
    COUNT(*) AS count,
    pg_size_pretty(SUM(pg_relation_size(indexrelid))) AS total_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelid NOT IN (
      SELECT conindid FROM pg_constraint WHERE contype IN ('p', 'u')
  )

UNION ALL

SELECT 
    'Large Indexes' AS category,
    COUNT(*) AS count,
    pg_size_pretty(SUM(pg_relation_size(indexrelid))) AS total_size
FROM pg_stat_user_indexes
WHERE pg_relation_size(indexrelid) > 1073741824  -- > 1GB

UNION ALL

SELECT 
    'Low Cache Hit' AS category,
    COUNT(*) AS count,
    NULL AS total_size
FROM pg_statio_user_indexes
WHERE idx_blks_hit + idx_blks_read > 1000
  AND (idx_blks_hit::float / NULLIF(idx_blks_hit + idx_blks_read, 0)) < 0.9;
```

### 4. Performance Baseline

Track metrics over time to identify trends:

```sql
-- Track index growth over time
WITH monthly_stats AS (
    SELECT 
        DATE_TRUNC('month', check_date) AS month,
        index_name,
        AVG(index_size) AS avg_size,
        AVG(index_scans) AS avg_scans
    FROM index_monitoring
    GROUP BY 1, 2
)
SELECT 
    index_name,
    month,
    avg_size,
    avg_scans,
    avg_size - LAG(avg_size) OVER (PARTITION BY index_name ORDER BY month) AS size_growth,
    avg_scans - LAG(avg_scans) OVER (PARTITION BY index_name ORDER BY month) AS scan_growth
FROM monthly_stats
ORDER BY index_name, month;
```

## Index Maintenance Actions

### When to Rebuild

Rebuild indexes when:
- Fragmentation > 30%
- Significant data changes (> 20% of rows)
- B-tree depth increased
- Performance degradation observed

**PostgreSQL:**
```sql
REINDEX INDEX CONCURRENTLY index_name;
```

**SQL Server:**
```sql
ALTER INDEX index_name ON table_name REBUILD 
WITH (ONLINE = ON, MAXDOP = 4);
```

**MySQL:**
```sql
ALTER TABLE table_name ENGINE=InnoDB;  -- Rebuilds all indexes
-- Or for specific index:
DROP INDEX index_name ON table_name;
CREATE INDEX index_name ON table_name(column);
```

### When to Reorganize

Reorganize indexes when:
- Fragmentation 10-30%
- Less disruptive than rebuild
- Online operation needed

**SQL Server:**
```sql
ALTER INDEX index_name ON table_name REORGANIZE;
```

### When to Update Statistics

Update statistics when:
- Significant data changes
- Query plans seem suboptimal
- After bulk operations

**PostgreSQL:**
```sql
ANALYZE table_name;
```

**SQL Server:**
```sql
UPDATE STATISTICS table_name index_name WITH FULLSCAN;
```

**MySQL:**
```sql
ANALYZE TABLE table_name;
```

## Best Practices

### 1. Regular Monitoring Schedule

- **Daily**: Check for missing indexes causing slow queries
- **Weekly**: Review unused indexes and cache hit ratios
- **Monthly**: Analyze fragmentation and index growth
- **Quarterly**: Comprehensive index audit and optimization

### 2. Document Index Decisions

Keep a log of:
- Why each index was created
- Expected query patterns
- Maintenance history
- Performance impact measurements

### 3. Test Before Dropping

Before dropping an unused index:
1. Verify it's truly unused (check multiple time periods)
2. Confirm it's not a primary key or unique constraint
3. Test in a staging environment first
4. Have a rollback plan ready

### 4. Monitor Application Impact

After index changes, monitor:
- Query response times
- Application error rates
- Database CPU/IO utilization
- Lock contention metrics

### 5. Use Monitoring Tools

Consider specialized tools:
- **pgAdmin** (PostgreSQL)
- **MySQL Workbench** (MySQL)
- **SQL Server Management Studio** (SQL Server)
- **Oracle Enterprise Manager** (Oracle)
- **Third-party**: Datadog, New Relic, SolarWinds, etc.

## Troubleshooting Common Issues

### Issue: Slow Queries Despite Indexes

**Diagnosis:**
```sql
-- Check if indexes are being used
EXPLAIN ANALYZE SELECT ...;
```

**Possible Causes:**
- Statistics out of date
- Wrong column order in composite index
- Query not using leftmost prefix
- Data distribution issues

### Issue: High Write Latency

**Diagnosis:**
```sql
-- Check number of indexes on table
SELECT COUNT(*) 
FROM information_schema.statistics 
WHERE table_name = 'your_table';
```

**Solution:**
- Remove unused indexes
- Consider batch operations
- Schedule maintenance during off-peak hours

### Issue: Index Bloat

**Diagnosis:**
```sql
-- Check index size vs. table size
SELECT 
    index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    pg_size_pretty(pg_relation_size(tablename::regclass)) AS table_size
FROM pg_stat_user_indexes;
```

**Solution:**
- REINDEX or REBUILD
- Adjust FILLFACTOR
- Increase maintenance frequency

## Monitoring Checklist

### Daily
- [ ] Check slow query log for missing indexes
- [ ] Monitor cache hit ratios
- [ ] Review any index-related errors

### Weekly
- [ ] Identify unused indexes
- [ ] Check index fragmentation levels
- [ ] Review index size growth

### Monthly
- [ ] Comprehensive index usage analysis
- [ ] Duplicate index detection
- [ ] Performance baseline comparison
- [ ] Maintenance window planning

### Quarterly
- [ ] Full index audit
- [ ] Capacity planning review
- [ ] Index strategy assessment
- [ ] Documentation update

## Conclusion

Effective index monitoring is essential for maintaining optimal database performance. Key takeaways:

1. **Regular monitoring prevents problems** before they impact users
2. **Unused indexes waste resources** and should be removed
3. **Fragmentation degrades performance** and requires maintenance
4. **Missing indexes cause slow queries** and need to be identified
5. **Automate monitoring** to catch issues early

Remember: indexes are not "set and forget." They require ongoing attention as your data and query patterns evolve. Establish a monitoring routine, set up alerts, and regularly review your index strategy to ensure your database performs at its best.

---

*Proactive index monitoring is the difference between a well-tuned database and one that gradually degrades over time.*