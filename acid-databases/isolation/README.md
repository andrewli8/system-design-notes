# Understanding Database Isolation Levels

## What is Isolation?
Isolation determines how transaction integrity is visible to other users and systems. It defines when changes made by one transaction can be seen by other transactions.

Think of transactions like people using ATMs to access the same bank account. Isolation levels determine how these operations interact with each other.

## Common Isolation Problems

Let's first understand the three main problems that can occur without proper isolation:

### 1. Dirty Reads
A dirty read occurs when a transaction reads data that has been written by another transaction that hasn't yet been committed.

#### Real-world Example
```
Initial balance: $1000

Timeline:
1. John starts transaction to withdraw $500
   - Account temporarily updated to $500
2. Mary reads balance (sees $500) 
3. John's transaction fails and rolls back to $1000
4. Mary has read incorrect data ($500)
```

```sql
-- Transaction 1 (John's withdrawal)
BEGIN TRANSACTION;
UPDATE account SET balance = balance - 500 WHERE account_id = 'A';
-- At this point, balance is $500
-- Transaction hasn't committed yet
ROLLBACK;  -- Transaction fails
-- Balance returns to $1000

-- Transaction 2 (Mary checking balance) - WITH READ UNCOMMITTED
BEGIN TRANSACTION;
SELECT balance FROM account WHERE account_id = 'A';
-- Returns $500 (incorrect/dirty read)
COMMIT;
```

### 2. Non-repeatable Reads
Occurs when a transaction reads the same row twice and gets different data each time because another transaction modified the row between reads.

#### Real-world Example
```
Timeline:
1. John starts checking account history
   - Reads balance: $1000
2. Mary deposits $500 and commits
3. John reads balance again: $1500
   - Same query, different results!
```

```sql
-- Transaction 1 (John checking balance)
BEGIN TRANSACTION;
SELECT balance FROM account WHERE account_id = 'A';
-- Returns $1000

-- Transaction 2 (Mary's deposit) happens here
UPDATE account SET balance = balance + 500 WHERE account_id = 'A';
COMMIT;

-- Transaction 1 continues
SELECT balance FROM account WHERE account_id = 'A';
-- Returns $1500 (different from first read!)
COMMIT;
```

### 3. Phantom Reads
Occurs when a transaction reads a set of rows twice and gets different rows each time because another transaction added or removed rows that match the search condition.

#### Real-world Example
```
Timeline:
1. Bank manager runs report of accounts with balance > $1000
   - Finds 5 accounts
2. New customer opens account with $5000
3. Manager runs same report again
   - Now finds 6 accounts (phantom row appeared!)
```

```sql
-- Transaction 1 (Manager's report)
BEGIN TRANSACTION;
SELECT COUNT(*) FROM accounts WHERE balance > 1000;
-- Returns 5

-- Transaction 2 (New account creation) happens here
INSERT INTO accounts (account_id, balance) VALUES ('B', 5000);
COMMIT;

-- Transaction 1 continues
SELECT COUNT(*) FROM accounts WHERE balance > 1000;
-- Returns 6 (phantom row!)
COMMIT;
```

## Isolation Levels Explained

Let's understand each isolation level and what problems they prevent:

### 1. READ UNCOMMITTED (Lowest Isolation)
- Allows transactions to read uncommitted changes from other transactions
- Provides no protection against any concurrency problems
- Like reading someone's bank statement while they're in the middle of a transaction

```sql
-- Example showing dirty read
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- Transaction 1
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100;  -- Uncommitted change
-- Transaction fails here
ROLLBACK;

-- Transaction 2 (running simultaneously)
BEGIN TRANSACTION;
SELECT balance FROM accounts;  -- Reads uncommitted balance
COMMIT;
```

### 2. READ COMMITTED
- Only allows reading data that has been committed
- Prevents dirty reads
- Still allows non-repeatable reads and phantom reads
- Like checking your bank balance after each transaction has completed

```sql
-- Example showing prevention of dirty read
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Transaction 1
BEGIN TRANSACTION;
SELECT balance FROM accounts;  -- Only sees committed data
COMMIT;

-- Transaction 2
BEGIN TRANSACTION;
UPDATE accounts SET balance = 500;  -- Uncommitted
-- Transaction 1 won't see this change until it's committed
COMMIT;
```

### 3. REPEATABLE READ
- Ensures that if a transaction reads a row, that row will remain unchanged while the transaction is running
- Prevents dirty reads and non-repeatable reads
- Still allows phantom reads
- Like taking a snapshot of your account data at the start of your bank statement review

```sql
-- Example showing prevention of non-repeatable read
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

BEGIN TRANSACTION;
-- First read
SELECT balance FROM accounts WHERE account_id = 'A';  -- Returns 1000

-- Even if another transaction updates this balance,
-- this transaction will still see 1000 when reading again
SELECT balance FROM accounts WHERE account_id = 'A';  -- Still returns 1000
COMMIT;
```

### 4. SERIALIZABLE (Highest Isolation)
- Transactions are completely isolated from each other
- Prevents all concurrency problems
- Like having everyone's transactions wait in line and execute one at a time

```sql
-- Example showing complete isolation
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Transaction 1
BEGIN TRANSACTION;
SELECT COUNT(*) FROM accounts WHERE balance > 1000;
-- No other transaction can add/remove rows that would affect this count
-- until this transaction commits
COMMIT;
```

## Visual Summary of Protection Levels

```
Protection Level:    | Dirty Read | Non-repeatable Read | Phantom Read |
------------------- | ---------- | ------------------ | ------------ |
READ UNCOMMITTED    |     ❌     |         ❌         |      ❌      |
READ COMMITTED      |     ✅     |         ❌         |      ❌      |
REPEATABLE READ     |     ✅     |         ✅         |      ❌      |
SERIALIZABLE        |     ✅     |         ✅         |      ✅      |
```

## Real-World Analogies

1. **READ UNCOMMITTED**
   - Like reading a draft email while someone is still typing it
   - The writer might still make changes or delete it entirely

2. **READ COMMITTED**
   - Like reading an email only after the sender has hit "send"
   - You might get different results if you read again later

3. **REPEATABLE READ**
   - Like taking a screenshot of your inbox
   - The data you're looking at won't change during your session

4. **SERIALIZABLE**
   - Like having everyone agree to not check their email until you're done reading yours
   - Complete isolation but slower performance

## Common Use Cases

1. **READ UNCOMMITTED**
   - Real-time analytics where approximate numbers are acceptable
   - Activity monitors where absolute accuracy isn't critical

2. **READ COMMITTED**
   - Most applications' default level
   - General purpose OLTP (Online Transaction Processing)

3. **REPEATABLE READ**
   - Financial reports that need consistent data throughout the report generation
   - Balance inquiries and statements

4. **SERIALIZABLE**
   - Financial transactions
   - Critical accounting operations
   - Situations where data consistency is absolutely crucial