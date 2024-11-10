# Understanding Database Durability

## What is Durability?
Durability guarantees that once a transaction is committed, it will remain committed even in the case of system failure (power loss, crashes). Think of it like saving a video game - once you've saved your progress, it should remain saved even if the power goes out.

## Understanding System Failures

### 1. Types of Failures

#### Power Failure
```
Scenario:
1. Transaction commits
2. Power loss occurs
3. System restarts
Expected: All committed transactions should be present
```

#### Process Crash
```
Scenario:
1. Database is processing multiple transactions
2. Database process crashes
3. Process restarts
Expected: All committed transactions intact, in-progress transactions rolled back
```

#### Storage Failure
```
Scenario:
1. Disk sector becomes corrupted
2. System detects corruption
Expected: Ability to recover data from redundant storage
```

## Implementation Mechanisms

### 1. Write-Ahead Logging (WAL)
Think of WAL as a detailed receipt of all transactions.

#### How WAL Works
```
Transaction Timeline:
1. Begin transaction
2. Write changes to WAL
3. Flush WAL to disk
4. Apply changes to database
5. Mark transaction as committed in WAL

Example WAL Entries:
[TX_001][2024-01-01 10:00:00] BEGIN
[TX_001] UPDATE accounts SET balance = balance - 100 WHERE id = 'A'
[TX_001] UPDATE accounts SET balance = balance + 100 WHERE id = 'B'
[TX_001][2024-01-01 10:00:01] COMMIT
```

#### Recovery Process
```sql
-- Pseudocode for recovery
FOR EACH log_entry IN wal_log:
    IF log_entry.has_commit_record:
        -- Transaction was committed
        APPLY log_entry.changes
    ELSE:
        -- Transaction was not committed
        ROLLBACK log_entry.changes
```

### 2. Double-Write Buffer
Prevents partial page writes that could corrupt data.

```
Write Process:
1. Write page to double-write buffer
2. Flush buffer to disk
3. Write page to actual location

Example:
Original Page:  [Account A: 1000][Account B: 500]
Double-Write:   [Account A: 900][Account B: 600]
                      ↓
Final Location: [Account A: 900][Account B: 600]
```

### 3. Checkpoints
Regular snapshots of database state.

```
Checkpoint Process:
1. Pause new transactions
2. Flush dirty pages to disk
3. Record checkpoint in WAL
4. Resume transactions

Example Timeline:
[Checkpoint 1] → [1000 transactions] → [Checkpoint 2]
Recovery only needs to process transactions after last checkpoint
```

## Real-World Durability Scenarios

### 1. Bank Transaction During Power Failure

#### Without Proper Durability
```
Initial State:
- Account A: $1000
- Account B: $500

Timeline:
1. Transfer $200 from A to B
2. Update memory: A = $800, B = $700
3. Power fails before disk write
4. System restarts
5. Data lost, accounts revert to original state
```

#### With Proper Durability
```sql
BEGIN TRANSACTION;
    -- 1. Write to WAL
    -- WAL Entry: UPDATE accounts SET balance = balance - 200 WHERE id = 'A'
    UPDATE accounts SET balance = balance - 200 WHERE id = 'A';
    
    -- WAL Entry: UPDATE accounts SET balance = balance + 200 WHERE id = 'B'
    UPDATE accounts SET balance = balance + 200 WHERE id = 'B';
    
    -- 2. Flush WAL to disk
    CHECKPOINT;
    
    -- 3. Commit transaction
    -- WAL Entry: COMMIT TRANSACTION
    COMMIT;
    
-- If power fails, on restart:
-- 1. Read WAL
-- 2. Find committed transaction
-- 3. Reapply changes if needed
```

### 2. E-commerce Order Processing

#### Durability Challenges
```
Order Process:
1. Create order
2. Update inventory
3. Process payment
4. Send confirmation

Failure Points:
- After order creation
- After inventory update
- After payment processing
- During confirmation send
```

#### Proper Implementation
```sql
BEGIN TRANSACTION;
    -- 1. Write all changes to WAL first
    INSERT INTO orders (customer_id, total) VALUES (123, 499.99);
    UPDATE inventory SET stock = stock - 1 WHERE product_id = 456;
    INSERT INTO payments (order_id, amount) VALUES (@order_id, 499.99);
    
    -- 2. Ensure WAL is flushed to disk
    CHECKPOINT;
    
    -- 3. Commit transaction
    COMMIT;
    
    -- 4. External operations (after commit)
    EXEC send_confirmation_email @order_id;
```

## Durability Implementation Techniques

### 1. Synchronous vs Asynchronous Writes

#### Synchronous (Safer but Slower)
```sql
-- Configure for synchronous writes
SET SYNCHRONOUS_COMMIT = ON;

BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 100;
    -- Wait for disk write confirmation
    COMMIT;
-- Transaction only returns after disk write
```

#### Asynchronous (Faster but Less Safe)
```sql
-- Configure for asynchronous writes
SET SYNCHRONOUS_COMMIT = OFF;

BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 100;
    COMMIT;
-- Returns immediately, writes happen in background
```

### 2. Replication for Durability

#### Synchronous Replication
```
Primary Server Timeline:
1. Receive transaction
2. Process transaction
3. Wait for replica confirmation
4. Confirm to client

Replica Server Timeline:
1. Receive transaction
2. Write to disk
3. Send confirmation to primary
```

#### Asynchronous Replication
```
Primary Server Timeline:
1. Receive transaction
2. Process transaction
3. Confirm to client
4. Send to replica (background)

Replica Server Timeline:
1. Receive transaction
2. Write to disk
3. Send acknowledgment
```

## Common Durability Problems and Solutions

### 1. Partial Writes
```
Problem Scenario:
Page Size: 8KB
Write Size: 16KB
Power fails after 10KB

Solution: Double-Write Buffer
1. Write full 16KB to double-write buffer
2. Verify complete write
3. Write to final location
```

### 2. Delayed Durability
```sql
-- Problem: Slow performance due to waiting for disk
BEGIN TRANSACTION;
    UPDATE large_table SET status = 'processed';
    -- Wait for disk write
    COMMIT;

-- Solution: Batch commits
BEGIN TRANSACTION;
    UPDATE large_table SET status = 'processed';
    -- Group with other transactions
    COMMIT WITH (DELAYED_DURABILITY = ON);
```

## Best Practices

### 1. Configure Proper Storage
```sql
-- Separate WAL and data files
ALTER DATABASE MyDB 
SET LOG ON 
    (NAME = 'MyDB_log', 
     FILENAME = 'D:\logs\MyDB_log.ldf');
     
-- Configure appropriate file sizes
ALTER DATABASE MyDB 
MODIFY FILE 
    (NAME = 'MyDB_log', 
     SIZE = 10GB);
```

### 2. Regular Checkpoints
```sql
-- Automatic checkpoints
ALTER DATABASE MyDB 
SET TARGET_RECOVERY_TIME = 60 SECONDS;

-- Manual checkpoint
CHECKPOINT;
```

### 3. Monitoring and Maintenance
```sql
-- Check recovery time
DBCC CHECKDB WITH ESTIMATEONLY;

-- Monitor log space
SELECT DB_NAME() AS DBName, 
       name AS LogicalFileName, 
       size/128.0 AS CurrentSizeMB,
       size/128.0 - CAST(FILEPROPERTY(name, 'SpaceUsed') AS INT)/128.0 
           AS FreeSpaceMB
FROM sys.database_files
WHERE type_desc = 'LOG';
```