# Row-Level Functions in SQL

Row-level functions, also known as scalar functions, operate on individual rows of data and return a single value for each row. Unlike aggregate functions that work across multiple rows, row-level functions process data one row at a time.

## Categories of Row-Level Functions

### String Functions

String functions manipulate text data:

**CONCAT / ||** - Combines strings together
```sql
SELECT CONCAT(first_name, ' ', last_name) AS full_name
FROM employees;
```

**SUBSTRING / SUBSTR** - Extracts part of a string
```sql
SELECT SUBSTRING(email, 1, 5) AS email_prefix
FROM users;
```

**UPPER / LOWER** - Changes case
```sql
SELECT UPPER(city) AS city_upper, LOWER(country) AS country_lower
FROM locations;
```

**TRIM / LTRIM / RTRIM** - Removes whitespace
```sql
SELECT TRIM(product_name) AS clean_name
FROM products;
```

**LENGTH / LEN** - Returns string length
```sql
SELECT name, LENGTH(name) AS name_length
FROM customers;
```

**REPLACE** - Substitutes characters
```sql
SELECT REPLACE(phone, '-', '') AS clean_phone
FROM contacts;
```

### Numeric Functions

Numeric functions perform mathematical operations:

**ROUND** - Rounds to specified decimal places
```sql
SELECT price, ROUND(price, 2) AS rounded_price
FROM products;
```

**CEIL / CEILING** - Rounds up to nearest integer
```sql
SELECT CEIL(average_score) AS rounded_up
FROM test_results;
```

**FLOOR** - Rounds down to nearest integer
```sql
SELECT FLOOR(discount_rate) AS rounded_down
FROM promotions;
```

**ABS** - Returns absolute value
```sql
SELECT ABS(temperature_change) AS abs_change
FROM weather_data;
```

**POWER / POW** - Raises to a power
```sql
SELECT quantity, POWER(quantity, 2) AS quantity_squared
FROM inventory;
```

**MOD** - Returns remainder of division
```sql
SELECT order_id, MOD(order_id, 2) AS is_even
FROM orders;
```

### Date and Time Functions

Date/time functions work with temporal data:

**CURRENT_DATE / CURRENT_TIMESTAMP** - Returns current date/time
```sql
SELECT order_date, CURRENT_DATE AS today
FROM orders;
```

**DATEADD / DATE_ADD** - Adds time interval
```sql
SELECT hire_date, DATEADD(year, 1, hire_date) AS anniversary
FROM employees;
```

**DATEDIFF / DATE_DIFF** - Calculates difference between dates
```sql
SELECT DATEDIFF(day, order_date, ship_date) AS days_to_ship
FROM orders;
```

**EXTRACT / DATE_PART** - Extracts component from date
```sql
SELECT EXTRACT(YEAR FROM birth_date) AS birth_year
FROM customers;
```

**DATE_TRUNC** - Truncates to specified precision
```sql
SELECT DATE_TRUNC('month', created_at) AS month_start
FROM transactions;
```

### Conditional Functions

Conditional functions implement logic:

**CASE** - Implements if-then-else logic
```sql
SELECT product_name,
  CASE 
    WHEN price < 10 THEN 'Budget'
    WHEN price < 50 THEN 'Standard'
    ELSE 'Premium'
  END AS price_category
FROM products;
```

**COALESCE** - Returns first non-null value
```sql
SELECT COALESCE(phone_mobile, phone_home, 'No phone') AS contact_phone
FROM customers;
```

**NULLIF** - Returns NULL if two values are equal
```sql
SELECT NULLIF(discount, 0) AS actual_discount
FROM promotions;
```

**IIF / IF** (some databases) - Simple conditional
```sql
SELECT IIF(quantity > 0, 'In Stock', 'Out of Stock') AS status
FROM inventory;
```

### Conversion Functions

Conversion functions change data types:

**CAST** - Converts between data types
```sql
SELECT CAST(order_id AS VARCHAR) AS order_id_text
FROM orders;
```

**CONVERT** - Alternative conversion syntax (SQL Server, MySQL)
```sql
SELECT CONVERT(VARCHAR, order_date, 101) AS formatted_date
FROM orders;
```

**TO_CHAR / TO_DATE / TO_NUMBER** (Oracle, PostgreSQL)
```sql
SELECT TO_CHAR(salary, '$999,999.99') AS formatted_salary
FROM employees;
```

### Window Functions (Row-Level Context)

While technically analytical functions, these operate in a row-level context:

**ROW_NUMBER** - Assigns unique sequential number
```sql
SELECT product_name, price,
  ROW_NUMBER() OVER (ORDER BY price DESC) AS price_rank
FROM products;
```

**RANK / DENSE_RANK** - Assigns rank with gaps/without gaps
```sql
SELECT employee_name, salary,
  RANK() OVER (ORDER BY salary DESC) AS salary_rank
FROM employees;
```

**LAG / LEAD** - Accesses previous/next row value
```sql
SELECT order_date, revenue,
  LAG(revenue) OVER (ORDER BY order_date) AS prev_revenue
FROM daily_sales;
```

## Key Characteristics

**Per-Row Processing**: Each function evaluates once per row independently.

**Single Value Return**: Returns exactly one value for each input row.

**No Row Reduction**: The result set has the same number of rows as the input.

**Nestable**: Functions can be nested within each other for complex transformations.

**Indexable**: Results can sometimes be indexed or used in WHERE clauses for optimization.

## Common Use Cases

- Data cleaning and standardization
- Calculated columns in SELECT statements
- Conditional logic for categorization
- Type conversions for reporting
- String parsing and manipulation
- Date arithmetic and formatting
- Creating derived metrics within rows

## Best Practices

Keep functions out of WHERE clauses when possible for performance, as they can prevent index usage. Use CASE statements for readable conditional logic rather than nested IIFs. Consider creating computed columns or materialized views for frequently used function results. Test date functions carefully across time zones and daylight saving time boundaries. Always handle NULL values explicitly using COALESCE or IS NULL checks.