# Stored Procedures

## 1. What Are Stored Procedures?

A **stored procedure** is a precompiled set of SQL statements stored in
the database and executed as a single unit.

Key characteristics:

-   Stored inside the database.
-   Can accept parameters and return results.
-   Can contain control flow (IF, LOOP, WHILE).
-   Precompiled and optimized by the database engine.
-   Reusable and centralized business logic.

------------------------------------------------------------------------

## 2. Why Stored Procedures Exist

Stored procedures are used to:

-   Encapsulate complex logic in the database.
-   Improve performance by reducing network calls.
-   Enforce business rules consistently.
-   Improve security by limiting direct table access.

------------------------------------------------------------------------

## 3. Basic Syntax by Database Engine

### 3.1 PostgreSQL (PL/pgSQL)

``` sql
CREATE OR REPLACE PROCEDURE get_active_users()
LANGUAGE plpgsql
AS $$
BEGIN
    SELECT * FROM users WHERE active = true;
END;
$$;
```

Procedure with parameters:

``` sql
CREATE OR REPLACE PROCEDURE get_user_by_id(p_id INT)
LANGUAGE plpgsql
AS $$
BEGIN
    SELECT * FROM users WHERE id = p_id;
END;
$$;
```

Calling a procedure:

``` sql
CALL get_user_by_id(1);
```

------------------------------------------------------------------------

### 3.2 MySQL

``` sql
DELIMITER $$

CREATE PROCEDURE get_active_users()
BEGIN
    SELECT * FROM users WHERE active = 1;
END $$

DELIMITER ;
```

Procedure with parameters:

``` sql
CREATE PROCEDURE get_user_by_id(IN p_id INT)
BEGIN
    SELECT * FROM users WHERE id = p_id;
END;
```

Call:

``` sql
CALL get_user_by_id(1);
```

------------------------------------------------------------------------

### 3.3 SQL Server (T-SQL)

``` sql
CREATE PROCEDURE get_active_users
AS
BEGIN
    SELECT * FROM users WHERE active = 1;
END;
GO
```

With parameters:

``` sql
CREATE PROCEDURE get_user_by_id @id INT
AS
BEGIN
    SELECT * FROM users WHERE id = @id;
END;
GO
```

Execute:

``` sql
EXEC get_user_by_id 1;
```

------------------------------------------------------------------------

### 3.4 Oracle (PL/SQL)

``` sql
CREATE OR REPLACE PROCEDURE get_active_users AS
BEGIN
    SELECT * FROM users WHERE active = 1;
END;
/
```

------------------------------------------------------------------------

## 4. Execution Model (How Stored Procedures Work)

### 4.1 Compilation Phase

When a stored procedure is created:

1.  SQL code is parsed.
2.  Execution plan is generated.
3.  Procedure is stored in the database catalog.
4.  Plan may be cached for reuse.

------------------------------------------------------------------------

### 4.2 Execution Phase

When called:

1.  Parameters are passed.
2.  Execution plan is reused or recompiled.
3.  Statements run inside the database engine.
4.  Results are returned to the client.

Key difference from normal queries:

-   Less network overhead.
-   More work done inside the database.

------------------------------------------------------------------------

### 4.3 Transaction Behavior

Stored procedures can:

-   Start transactions
-   Commit or rollback
-   Handle exceptions

Example (PostgreSQL):

``` sql
BEGIN;
CALL update_user_status(1);
COMMIT;
```

------------------------------------------------------------------------

## 5. Stored Procedures vs Normal Queries

### 5.1 Conceptual Difference

  Aspect            Stored Procedure           Normal Query
  ----------------- -------------------------- ---------------------------
  Location          Stored in DB               Sent from application
  Compilation       Precompiled                Parsed every time
  Reusability       High                       Low
  Complexity        High                       Low
  Performance       Better for complex logic   Better for simple queries
  Maintainability   Centralized                Distributed in code
  Security          Stronger                   Weaker
  Network Calls     Fewer                      More
  Flexibility       Lower                      Higher

------------------------------------------------------------------------

### 5.2 Performance Comparison

Stored procedures are faster when:

-   Multiple queries are executed together.
-   Heavy business logic is involved.
-   Network latency is high.
-   Execution plans are reused.

Normal queries are better when:

-   Queries are simple and dynamic.
-   Logic changes frequently.
-   Application-layer control is preferred.

------------------------------------------------------------------------

## 6. Stored Procedures vs Functions vs Queries

  Feature                        Stored Procedure   Function          Query
  ------------------------------ ------------------ ----------------- ----------------
  Returns value                  Optional           Required          Yes
  Side effects (INSERT/UPDATE)   Yes                Usually limited   Yes
  Called from SQL                Limited            Yes               Yes
  Control flow                   Yes                Yes               No
  Use case                       Business logic     Computation       Data retrieval

------------------------------------------------------------------------

## 7. When to Use Stored Procedures

### 7.1 Complex Business Logic

Use when:

-   Multiple steps are required.
-   Logic must be atomic.

Example:

-   Order processing
-   Payment workflows
-   Inventory updates

------------------------------------------------------------------------

### 7.2 Performance Optimization

Use when:

-   Multiple queries must run together.
-   Large datasets are processed.
-   You want to reduce round trips.

------------------------------------------------------------------------

### 7.3 Security & Access Control

Use when:

-   Users should not access tables directly.
-   You want controlled data access.

Example:

-   Grant permission only to execute procedures.

------------------------------------------------------------------------

### 7.4 Data Integrity & Transactions

Use when:

-   Multiple operations must succeed or fail together.

Example:

-   Bank transfers
-   Account updates

------------------------------------------------------------------------

### 7.5 Reusable Database Logic

Use when:

-   Same logic is used by multiple applications.

------------------------------------------------------------------------

## 8. When NOT to Use Stored Procedures

Avoid when:

-   Logic changes frequently.
-   Queries are simple CRUD operations.
-   You need high flexibility in application code.
-   You follow microservices with thin databases.

------------------------------------------------------------------------

## 9. Real-World Use Cases

### 9.1 Banking Systems

-   Transfers
-   Fraud checks
-   Account reconciliation

### 9.2 Enterprise Systems

-   ERP workflows
-   Payroll processing
-   Reporting pipelines

### 9.3 Data Engineering

-   ETL pipelines
-   Batch processing

### 9.4 Legacy Systems

-   Heavy database-driven logic

------------------------------------------------------------------------

## 10. Architectural Perspective

### 10.1 Database-Centric Architecture

Logic in DB (Stored Procedures):

    Application → Stored Procedures → Database Tables

Pros:

-   High performance
-   Strong consistency

Cons:

-   Harder to maintain
-   DB vendor lock-in

------------------------------------------------------------------------

### 10.2 Application-Centric Architecture

Logic in application code:

    Application → Queries → Database Tables

Pros:

-   Flexibility
-   Easier testing

Cons:

-   More network calls
-   Less centralized logic

------------------------------------------------------------------------

## 11. Decision Guide

  Scenario                          Recommended
  --------------------------------- ------------------
  Complex transactional logic       Stored Procedure
  Simple CRUD operations            Queries
  Performance-critical batch jobs   Stored Procedure
  Rapidly changing business logic   Queries
  Shared logic across systems       Stored Procedure
  Microservices architecture        Queries

------------------------------------------------------------------------

## 12. Key Takeaways

-   Stored procedures are powerful but should be used strategically.
-   They excel in performance, security, and complex logic.
-   Overusing them can reduce flexibility and maintainability.

------------------------------------------------------------------------

## 13. Mental Model

Think of stored procedures as:

> "Backend functions that live inside the database instead of your
> application."
