# ACID Database Transactions: A Comprehensive Guide

## Introduction to ACID
ACID is an acronym representing the four essential properties that guarantee reliable database transactions:
- **A**tomicity
- **C**onsistency
- **I**solation
- **D**urability

## 1. Atomicity

### Definition
Atomicity guarantees that each transaction is treated as a single, indivisible unit that either completely succeeds or completely fails. There is no partial completion.

### Examples
```sql
-- Example of an atomic transaction
BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 100 WHERE account_id = 'A';
    UPDATE accounts SET balance = balance + 100 WHERE account_id = 'B';
COMMIT;

-- If any step fails, the entire transaction is rolled back
-- Example of rollback
BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 100 WHERE account_id = 'A';
    -- If this fails (e.g., account B doesn't exist)
    UPDATE accounts SET balance = balance + 100 WHERE account_id = 'B';
    -- Transaction is rolled back, account A is restored
ROLLBACK;
```

### Implementation Mechanisms
1. Write-Ahead Logging (WAL)
```
// WAL entries
BEGIN_TXN 1234
UPDATE account_A balance=900 old_value=1000
UPDATE account_B balance=600 old_value=500
COMMIT_TXN 1234
```

2. Shadow Paging
```
Original Page    Shadow Page
[A: 1000]  →    [A: 900]
[B: 500]   →    [B: 600]
```

## 2. Consistency

### Definition
Consistency ensures that a transaction can only bring the database from one valid state to another valid state, maintaining all predefined rules and constraints.

### Types of Consistency
1. **Domain Consistency**
```sql
-- Example: Ensuring age is positive
CREATE TABLE users (
    id INT PRIMARY KEY,
    age INT CHECK (age >= 0)
);
```

2. **Referential Integrity**
```sql
-- Example: Ensuring orders reference valid customers
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);
```

3. **Application-Level Consistency**
```sql
-- Example: Ensuring total balance remains constant
BEGIN TRANSACTION;
    -- Transfer $100
    UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
    UPDATE accounts SET balance = balance + 100 WHERE id = 'B';
    
    -- Verify consistency
    IF (SELECT SUM(balance) FROM accounts) != @original_total
        ROLLBACK;
    ELSE
        COMMIT;
END TRANSACTION;
```

## 3. Isolation

### Definition
Isolation ensures that concurrent execution of transactions leaves the database in the same state as if the transactions were executed sequentially.

### Isolation Levels

1. **Read Uncommitted**
```sql
-- Transaction 1
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100;
-- Transaction 2 can read this uncommitted change
ROLLBACK;
```

2. **Read Committed**
```sql
-- Default in many databases
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT balance FROM accounts WHERE id = 'A';
-- Will only see committed changes
```

3. **Repeatable Read**
```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- Same SELECT will return same results within transaction
```

4. **Serializable**
```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- Transactions execute as if they were serial
```

### Common Isolation Problems

1. **Dirty Reads**
```sql
-- Transaction 1
BEGIN;
UPDATE accounts SET balance = 1000;
-- Transaction 2 reads balance=1000 (dirty read)
ROLLBACK;
-- Transaction 2's read was invalid
```

2. **Non-repeatable Reads**
```sql
-- Transaction 1
BEGIN;
SELECT balance FROM accounts; -- returns 1000
-- Transaction 2 updates balance to 2000 and commits
SELECT balance FROM accounts; -- returns 2000
COMMIT;
```

3. **Phantom Reads**
```sql
-- Transaction 1
BEGIN;
SELECT COUNT(*) FROM accounts WHERE balance > 1000;
-- Transaction 2 inserts new account with balance > 1000
SELECT COUNT(*) FROM accounts WHERE balance > 1000;
-- Different count (phantom read)
COMMIT;
```

## 4. Durability

### Definition
Durability guarantees that once a transaction is committed, it remains committed even in the case of system failure.

### Implementation Mechanisms

1. **Write-Ahead Logging**
```
// Durability sequence
1. Write to WAL
2. Flush WAL to disk
3. Acknowledge commit
4. Update actual data
```

2. **Replication**
```python
def commit_transaction(transaction):
    # Write to primary
    primary.commit(transaction)
    
    # Replicate to secondaries
    for secondary in secondaries:
        secondary.replicate(transaction)
    
    # Acknowledge commit after majority consensus
    return acknowledge_commit()
```

## Practical Implementation Examples

### 1. Bank Transfer
```sql
BEGIN TRANSACTION;
    -- Check sufficient funds
    SELECT balance FROM accounts WHERE id = 'A' FOR UPDATE;
    
    -- Perform transfer
    UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
    UPDATE accounts SET balance = balance + 100 WHERE id = 'B';
    
    -- Verify consistency
    IF (SELECT balance FROM accounts WHERE id = 'A') >= 0
        COMMIT;
    ELSE
        ROLLBACK;
END TRANSACTION;
```

### 2. Order Processing
```sql
BEGIN TRANSACTION;
    -- Create order
    INSERT INTO orders (customer_id, total) VALUES (123, 99.99);
    
    -- Update inventory
    UPDATE products 
    SET stock = stock - 1 
    WHERE product_id = 456 AND stock > 0;
    
    -- If inventory update successful, commit
    IF @@ROWCOUNT > 0
        COMMIT;
    ELSE
        ROLLBACK;
END TRANSACTION;
```

## Choosing Isolation Levels

| Isolation Level   | Dirty Read | Non-repeatable Read | Phantom Read | Use When |
|------------------|------------|---------------------|--------------|-----------|
| Read Uncommitted | Possible   | Possible           | Possible     | Highest performance needed, accuracy less critical |
| Read Committed   | Prevented  | Possible           | Possible     | General purpose operations |
| Repeatable Read  | Prevented  | Prevented          | Possible     | Balance needed between consistency and performance |
| Serializable     | Prevented  | Prevented          | Prevented    | Highest accuracy needed, performance less critical |

## Common Use Cases

1. **Financial Transactions**
   - Requires: Serializable isolation
   - Example: Bank transfers, stock trades

2. **Inventory Management**
   - Requires: Repeatable Read
   - Example: E-commerce order processing

3. **Analytics Queries**
   - Requires: Read Committed
   - Example: Reporting systems

4. **User Activity Logs**
   - Requires: Read Uncommitted
   - Example: Activity monitoring

## Best Practices

1. **Keep Transactions Short**
```sql
-- Good
BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 100;
    UPDATE accounts SET balance = balance + 100;
COMMIT;

-- Bad: Long-running transaction
BEGIN TRANSACTION;
    -- Process thousands of records
    -- Generate reports
    -- Send emails
    -- Much more likely to cause conflicts
COMMIT;
```

2. **Handle Deadlocks**
```sql
BEGIN TRY
    BEGIN TRANSACTION;
        -- Transaction logic here
    COMMIT;
END TRY
BEGIN CATCH
    IF ERROR_NUMBER() = 1205 -- Deadlock error
        -- Retry logic here
    ROLLBACK;
END CATCH;
```

3. **Use Appropriate Isolation Levels**
```sql
-- For read-heavy analytics
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- For financial transactions
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```