# Database Triggers: Complete Guide

## What Are Database Triggers?

A database trigger is a stored procedure that automatically executes (or "fires") in response to specific events on a particular table or view in a database. Triggers are special types of stored procedures that run implicitly when an INSERT, UPDATE, or DELETE operation occurs.

## Why Use Triggers?

### Key Benefits

1. **Automated Data Integrity**: Enforce complex business rules and data validation automatically without relying on application code
2. **Audit Trails**: Automatically log changes to sensitive data for compliance and tracking
3. **Cascading Changes**: Propagate changes across related tables automatically
4. **Derived Data Management**: Calculate and maintain computed or summary data
5. **Centralized Logic**: Keep business rules in the database rather than scattered across multiple applications
6. **Prevention**: Block certain operations based on business rules

### Common Use Cases

- Maintaining audit logs and change history
- Enforcing complex referential integrity
- Validating data before insertion or updates
- Automatic timestamp updates (created_at, updated_at)
- Sending notifications when data changes
- Preventing deletion of critical records
- Calculating and updating aggregate data
- Synchronizing data across tables

## When to Use Triggers

### Good Scenarios

- **Audit Requirements**: When you need to track all changes to sensitive data
- **Complex Validation**: Business rules too complex for simple constraints
- **Data Denormalization**: Maintaining calculated fields or summary tables
- **Multiple Applications**: When many applications access the same database and need consistent behavior
- **Cascading Operations**: When changes need to propagate to related tables

### When to Avoid Triggers

- **Simple Operations**: Use constraints instead (CHECK, FOREIGN KEY, NOT NULL)
- **Complex Business Logic**: Consider stored procedures or application code for better visibility
- **Performance-Critical Operations**: Triggers add overhead to every DML operation
- **Debugging Difficulty**: Hidden logic can make troubleshooting harder
- **Over-complication**: Too many triggers can create maintenance nightmares

## Types of Triggers

### By Timing

1. **BEFORE Triggers**: Execute before the triggering event (INSERT/UPDATE/DELETE)
   - Can modify the new values before they're saved
   - Can prevent the operation from completing

2. **AFTER Triggers**: Execute after the triggering event completes
   - Work with the final committed data
   - Cannot modify the triggering operation's data

3. **INSTEAD OF Triggers**: Replace the triggering event (mainly for views)
   - Common with complex views that aren't directly updatable

### By Event

- **INSERT Triggers**: Fire when new rows are inserted
- **UPDATE Triggers**: Fire when existing rows are modified
- **DELETE Triggers**: Fire when rows are deleted

### By Scope

- **Row-Level Triggers**: Execute once for each affected row
- **Statement-Level Triggers**: Execute once per SQL statement, regardless of rows affected

## Syntax and Examples

### MySQL Syntax

```sql
CREATE TRIGGER trigger_name
{BEFORE | AFTER} {INSERT | UPDATE | DELETE}
ON table_name
FOR EACH ROW
BEGIN
    -- trigger body
END;
```

#### MySQL Example: Audit Trail

```sql
-- Create audit table
CREATE TABLE users_audit (
    audit_id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT,
    old_email VARCHAR(255),
    new_email VARCHAR(255),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    changed_by VARCHAR(100)
);

-- Create trigger
DELIMITER $$
CREATE TRIGGER users_email_audit
AFTER UPDATE ON users
FOR EACH ROW
BEGIN
    IF OLD.email != NEW.email THEN
        INSERT INTO users_audit (user_id, old_email, new_email, changed_by)
        VALUES (NEW.id, OLD.email, NEW.email, USER());
    END IF;
END$$
DELIMITER ;
```

#### MySQL Example: Automatic Timestamp

```sql
DELIMITER $$
CREATE TRIGGER update_timestamp
BEFORE UPDATE ON products
FOR EACH ROW
BEGIN
    SET NEW.updated_at = CURRENT_TIMESTAMP;
END$$
DELIMITER ;
```

### PostgreSQL Syntax

```sql
-- First, create the trigger function
CREATE OR REPLACE FUNCTION function_name()
RETURNS TRIGGER AS $$
BEGIN
    -- trigger logic
    RETURN NEW; -- or OLD or NULL
END;
$$ LANGUAGE plpgsql;

-- Then, create the trigger
CREATE TRIGGER trigger_name
{BEFORE | AFTER | INSTEAD OF} {INSERT | UPDATE | DELETE}
ON table_name
[FOR EACH {ROW | STATEMENT}]
EXECUTE FUNCTION function_name();
```

#### PostgreSQL Example: Audit Trail

```sql
-- Create audit table
CREATE TABLE orders_audit (
    audit_id SERIAL PRIMARY KEY,
    order_id INT,
    operation VARCHAR(10),
    old_status VARCHAR(50),
    new_status VARCHAR(50),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    changed_by VARCHAR(100)
);

-- Create trigger function
CREATE OR REPLACE FUNCTION log_order_changes()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' AND OLD.status != NEW.status THEN
        INSERT INTO orders_audit (order_id, operation, old_status, new_status, changed_by)
        VALUES (NEW.id, TG_OP, OLD.status, NEW.status, current_user);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Create trigger
CREATE TRIGGER order_status_audit
AFTER UPDATE ON orders
FOR EACH ROW
EXECUTE FUNCTION log_order_changes();
```

#### PostgreSQL Example: Maintain Summary Table

```sql
-- Trigger function to update inventory summary
CREATE OR REPLACE FUNCTION update_inventory_summary()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE inventory_summary
        SET total_quantity = total_quantity + NEW.quantity
        WHERE product_id = NEW.product_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE inventory_summary
        SET total_quantity = total_quantity - OLD.quantity
        WHERE product_id = OLD.product_id;
    ELSIF TG_OP = 'UPDATE' THEN
        UPDATE inventory_summary
        SET total_quantity = total_quantity - OLD.quantity + NEW.quantity
        WHERE product_id = NEW.product_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Create trigger
CREATE TRIGGER maintain_inventory_summary
AFTER INSERT OR UPDATE OR DELETE ON inventory_transactions
FOR EACH ROW
EXECUTE FUNCTION update_inventory_summary();
```

### SQL Server Syntax

```sql
CREATE TRIGGER trigger_name
ON table_name
{AFTER | INSTEAD OF} {INSERT | UPDATE | DELETE}
AS
BEGIN
    -- trigger body
END;
```

#### SQL Server Example: Prevent Deletion

```sql
CREATE TRIGGER prevent_admin_delete
ON users
INSTEAD OF DELETE
AS
BEGIN
    IF EXISTS (SELECT 1 FROM deleted WHERE role = 'admin')
    BEGIN
        RAISERROR('Cannot delete admin users', 16, 1);
        ROLLBACK TRANSACTION;
    END
    ELSE
    BEGIN
        DELETE FROM users
        WHERE id IN (SELECT id FROM deleted);
    END
END;
```

### Oracle Syntax

```sql
CREATE OR REPLACE TRIGGER trigger_name
{BEFORE | AFTER} {INSERT | UPDATE | DELETE}
ON table_name
[FOR EACH ROW]
[WHEN (condition)]
DECLARE
    -- variable declarations
BEGIN
    -- trigger body
END;
```

#### Oracle Example: Sequence-based ID

```sql
CREATE OR REPLACE TRIGGER set_employee_id
BEFORE INSERT ON employees
FOR EACH ROW
BEGIN
    IF :NEW.id IS NULL THEN
        SELECT employee_seq.NEXTVAL INTO :NEW.id FROM dual;
    END IF;
END;
```

## Execution Flow

### Row-Level Trigger Flow

1. **Triggering Statement** executed (INSERT/UPDATE/DELETE)
2. **BEFORE Statement Triggers** fire (if any)
3. **For each affected row**:
   - BEFORE Row Triggers fire
   - Row operation performed
   - AFTER Row Triggers fire
4. **AFTER Statement Triggers** fire (if any)
5. **Transaction commits** (unless rolled back)

### Special Variables

Different databases provide access to old and new values:

- **MySQL**: `OLD.column_name`, `NEW.column_name`
- **PostgreSQL**: `OLD.column_name`, `NEW.column_name`, plus `TG_OP`, `TG_TABLE_NAME`, etc.
- **SQL Server**: `inserted` and `deleted` pseudo-tables
- **Oracle**: `:OLD.column_name`, `:NEW.column_name`

## Best Practices

1. **Keep triggers simple**: Complex logic is hard to debug and maintain
2. **Document thoroughly**: Triggers are "hidden" code - document their purpose and behavior
3. **Avoid chaining**: Multiple triggers calling each other can create confusing behavior
4. **Consider performance**: Triggers add overhead to every DML operation
5. **Error handling**: Implement proper error handling and rollback logic
6. **Avoid infinite loops**: Be careful with triggers that modify the same table
7. **Use appropriate timing**: Choose BEFORE vs AFTER based on your needs
8. **Test thoroughly**: Test with bulk operations, not just single-row cases
9. **Limit side effects**: Avoid complex external operations in triggers
10. **Version control**: Keep trigger definitions in version control like other code

## Managing Triggers

### View Existing Triggers

**MySQL:**
```sql
SHOW TRIGGERS FROM database_name;
-- or
SELECT * FROM information_schema.TRIGGERS WHERE TRIGGER_SCHEMA = 'database_name';
```

**PostgreSQL:**
```sql
SELECT * FROM information_schema.triggers WHERE trigger_schema = 'public';
```

**SQL Server:**
```sql
SELECT * FROM sys.triggers;
```

### Drop a Trigger

**MySQL:**
```sql
DROP TRIGGER IF EXISTS trigger_name;
```

**PostgreSQL:**
```sql
DROP TRIGGER IF EXISTS trigger_name ON table_name;
```

**SQL Server:**
```sql
DROP TRIGGER trigger_name;
```

### Disable/Enable Triggers

**MySQL:**
```sql
-- Not directly supported, must DROP and recreate
```

**PostgreSQL:**
```sql
ALTER TABLE table_name DISABLE TRIGGER trigger_name;
ALTER TABLE table_name ENABLE TRIGGER trigger_name;
```

**SQL Server:**
```sql
DISABLE TRIGGER trigger_name ON table_name;
ENABLE TRIGGER trigger_name ON table_name;
```

## Common Pitfalls

1. **Mutating table error**: Querying the same table being modified (Oracle)
2. **Performance degradation**: Triggers on high-volume tables can slow operations
3. **Cascading triggers**: One trigger fires another, creating complex chains
4. **Transaction conflicts**: Triggers participate in the transaction - rollbacks affect them
5. **Debugging difficulty**: Hidden logic that doesn't appear in application code
6. **Maintenance burden**: Forgotten triggers cause mysterious behavior years later

## Conclusion

Database triggers are powerful tools for maintaining data integrity, enforcing business rules, and automating database operations. However, they should be used judiciously. While they provide centralized logic and automatic enforcement, they can also add complexity, reduce performance, and make debugging more difficult. Consider alternatives like application logic, stored procedures, or database constraints before implementing triggers, and always document their purpose and behavior thoroughly.