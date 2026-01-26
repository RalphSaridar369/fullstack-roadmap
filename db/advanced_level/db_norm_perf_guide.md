# Database Normalization vs Denormalization Performance

Whether it is more performant to use fewer columns in more tables (**normalization**) or more columns in fewer tables (**denormalization**) is a classic database tradeoff that depends heavily on your workload (read-heavy vs write-heavy) and data size.

Generally:
- **Fewer columns + more tables (normalized)** → better when queries access only a small subset of columns.
- **Fewer tables + more columns (denormalized)** → better for read-heavy analytical workloads that need all data at once.

---

## When Fewer Columns + More Tables (Normalized) Is Faster

### ✅ Reduced I/O
If a table has 100 columns but 90% of queries only need 5, splitting those columns into a lean table reduces disk reads significantly.

### ✅ Better Caching
Smaller, narrow tables allow more rows to fit in memory (RAM), increasing cache hit rates.

### ✅ Faster Updates
Because data is not duplicated, writes, updates, and deletes are generally faster and safer, reducing locking issues.

### ✅ Smaller Indexes
Indexes on narrow tables are smaller and faster to scan.

---

## When Fewer Tables + More Columns (Denormalized) Is Faster

### ✅ Eliminating Joins
Joins consume CPU and memory. A single wide table can retrieve all required data without complex joins.

### ✅ Analytical Reads
For reporting (OLAP), pre-joined wide tables often provide much faster read performance because joins are avoided.

### ✅ Modern Columnar Storage
In columnar databases (e.g., Snowflake, BigQuery), wide tables are highly performant because only required columns are read, making table width less relevant.

---

## Key Performance Considerations

- **Joins are not inherently slow**  
  With proper indexing (especially foreign keys), joins can be very fast. However, the cost increases as result sets grow to millions of rows.

- **`SELECT *` is the enemy**  
  Avoid `SELECT *` on wide tables. Selecting only needed columns can make wide tables nearly as fast as narrow ones.

- **Data Size Matters**  
  If the dataset fits in memory, performance differences are negligible. When data exceeds RAM, normalization scales better due to lower memory usage.

- **Optimal Strategy**  
  Start with a normalized design (3NF).  
  Denormalize only when profiling shows specific joins are bottlenecks in read-heavy workloads.

---

## Summary Table: Performance Tradeoffs

| Aspect        | Normalized (More Tables / Less Columns) | Denormalized (Fewer Tables / More Columns) |
|--------------|----------------------------------------|-------------------------------------------|
| Read Speed   | Fast (if selective), Slow (if many joins) | Very fast (few/no joins)                 |
| Write Speed  | Fast (lower overhead)                  | Slower (due to duplicated updates)        |
| I/O Overhead | Low (narrow columns)                   | High (many columns)                       |
| Best For     | OLTP (Transactional, Web Apps)         | OLAP (Reporting, Data Warehouses)         |
| Risk         | Slow joins, higher complexity          | Data inconsistency, higher storage        |

---

## Conclusion

The choice between normalization and denormalization is not binary. Modern database systems often use hybrid approaches, where core transactional data is normalized while read-heavy analytics leverage denormalized views or materialized tables. Profile your specific workload and optimize accordingly.