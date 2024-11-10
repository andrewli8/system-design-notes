# Two-Phase Locking (2PL)

## 1. Core Concept
Two-Phase Locking is a concurrency control protocol that ensures serializability by dividing transaction locking operations into two phases:
- Growing/Expansion Phase: Only acquire locks
- Shrinking/Contraction Phase: Only release locks

Think of it like boarding and deboarding a train - passengers can only enter during boarding phase and only exit during deboarding phase, never both simultaneously.

### Fundamental Rule
Once a transaction releases a lock, it cannot obtain any new locks.

## 2. How It Works

### Lock Types
1. **Shared Lock (S-Lock)**
   - Used for reading data
   - Multiple transactions can hold shared locks simultaneously
   - Also called "Read Lock"

2. **Exclusive Lock (X-Lock)**
   - Used for writing data
   - Only one transaction can hold an exclusive lock
   - Also called "Write Lock"

### Lock Compatibility Matrix
```
         S-Lock | X-Lock
S-Lock    ✓    |    ✗
X-Lock    ✗    |    ✗

✓ = Compatible
✗ = Incompatible
```

### Two Phases

#### Phase 1: Growing/Expansion
```
Transaction Timeline (Growing):
T1: Acquire S-Lock(A) → Acquire S-Lock(B) → Acquire X-Lock(C)
[Can still acquire more locks]
```

#### Phase 2: Shrinking/Contraction
```
Transaction Timeline (Shrinking):
T1: Release S-Lock(A) → Release S-Lock(B) → Release X-Lock(C)
[Cannot acquire new locks]
```

## 3. Types of 2PL

### 1. Basic 2PL
```
Transaction Example:
BEGIN TRANSACTION
   1. Acquire all necessary locks
   2. Perform operations
   3. Release locks
   4. Commit/Rollback
END TRANSACTION
```

### 2. Conservative 2PL (Static)
```
Transaction Example:
BEGIN TRANSACTION
   1. Acquire ALL needed locks before starting
   2. Perform operations
   3. Release locks
   4. Commit/Rollback
END TRANSACTION
```

### 3. Strict 2PL (S2PL)
```
Transaction Example:
BEGIN TRANSACTION
   1. Acquire locks as needed
   2. Perform operations
   3. Commit/Rollback
   4. Release ALL locks
END TRANSACTION
```

### 4. Strong Strict 2PL (SS2PL)
```
Transaction Example:
BEGIN TRANSACTION
   1. Acquire locks as needed
   2. Perform operations
   3. Commit/Rollback
   4. Release ALL locks atomically
END TRANSACTION
```

## 4. Real-World Examples

### Example 1: Bank Transfer
```
Initial State:
- Account A: $1000
- Account B: $500

Transaction T1: Transfer $200 from A to B
1. Growing Phase:
   - Acquire X-Lock(A)
   - Acquire X-Lock(B)
2. Execute:
   - Read A = 1000
   - A = A - 200
   - Read B = 500
   - B = B + 200
3. Shrinking Phase:
   - Release X-Lock(A)
   - Release X-Lock(B)

Final State:
- Account A: $800
- Account B: $700
```

### Example 2: Concurrent Read/Write
```
Transaction T1 (Read Balance)    | Transaction T2 (Update Balance)
1. Acquire S-Lock(A)            | 1. Wait for X-Lock(A)
2. Read Balance                 | 
3. Release S-Lock(A)            | 2. Acquire X-Lock(A)
                               | 3. Update Balance
                               | 4. Release X-Lock(A)
```

## 5. Implementation Example

### Basic 2PL Implementation
```python
class LockManager:
    def __init__(self):
        self.locks = {}
        self.waiting = {}
    
    def acquire_lock(self, transaction_id, item, lock_type):
        if self.is_compatible(item, transaction_id, lock_type):
            self.locks[item] = {
                'holder': transaction_id,
                'type': lock_type
            }
            return True
        else:
            # Add to waiting queue
            self.waiting[item] = self.waiting.get(item, [])
            self.waiting[item].append({
                'transaction': transaction_id,
                'type': lock_type
            })
            return False
    
    def release_lock(self, transaction_id, item):
        if item in self.locks and self.locks[item]['holder'] == transaction_id:
            del self.locks[item]
            # Process waiting queue
            self.process_waiting_queue(item)
            
    def is_compatible(self, item, transaction_id, lock_type):
        if item not in self.locks:
            return True
            
        current_lock = self.locks[item]
        if current_lock['holder'] == transaction_id:
            return True
            
        if lock_type == 'S' and current_lock['type'] == 'S':
            return True
            
        return False
```

### Transaction Example
```python
class Transaction:
    def __init__(self, id, lock_manager):
        self.id = id
        self.lock_manager = lock_manager
        self.locked_items = set()
        self.growing_phase = True
    
    def read_item(self, item):
        if not self.growing_phase:
            raise Exception("Cannot acquire locks in shrinking phase")
            
        if self.lock_manager.acquire_lock(self.id, item, 'S'):
            self.locked_items.add(item)
            return True
        return False
    
    def write_item(self, item):
        if not self.growing_phase:
            raise Exception("Cannot acquire locks in shrinking phase")
            
        if self.lock_manager.acquire_lock(self.id, item, 'X'):
            self.locked_items.add(item)
            return True
        return False
    
    def begin_shrinking_phase(self):
        self.growing_phase = False
    
    def release_locks(self):
        for item in self.locked_items:
            self.lock_manager.release_lock(self.id, item)
        self.locked_items.clear()
```

## 6. Common Problems and Solutions

### 1. Deadlocks
```
Problem Scenario:
T1: Has lock on A, wants lock on B
T2: Has lock on B, wants lock on A

Solution 1: Deadlock Prevention
- Assign timestamps to transactions
- Use wound-wait or wait-die schemes

Solution 2: Deadlock Detection
```
```python
def detect_deadlocks():
    graph = build_wait_for_graph()
    return has_cycle(graph)

def handle_deadlock():
    victim = select_victim()
    abort_transaction(victim)
```

### 2. Starvation
```
Problem: Transaction repeatedly denied locks
Solution: Queue-based lock management
```
```python
class FairLockManager:
    def __init__(self):
        self.lock_queues = {}
    
    def request_lock(self, transaction_id, item):
        queue = self.lock_queues.get(item, [])
        queue.append(transaction_id)
        if len(queue) == 1:
            return True  # Lock granted
        return False    # Must wait
```

## 7. Performance Considerations

### 1. Lock Granularity
```
Coarse-grained:
- Table-level locks
- Less overhead
- More contention

Fine-grained:
- Row-level locks
- More overhead
- Less contention
```

### 2. Lock Duration
```sql
-- Bad: Long lock duration
BEGIN TRANSACTION;
    SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
    -- Long processing
    EXEC some_long_process;
    UPDATE accounts SET balance = new_balance WHERE id = 1;
COMMIT;

-- Better: Minimal lock duration
BEGIN TRANSACTION;
    -- Do processing first
    SET @new_balance = EXEC calculate_new_balance;
    -- Then acquire locks
    UPDATE accounts SET balance = @new_balance WHERE id = 1;
COMMIT;
```

## 8. Best Practices

### 1. Lock Ordering
```python
def process_transfer(from_account, to_account, amount):
    # Always acquire locks in the same order
    first_account = min(from_account, to_account)
    second_account = max(from_account, to_account)
    
    acquire_lock(first_account)
    acquire_lock(second_account)
```

### 2. Lock Timeouts
```python
def acquire_lock_with_timeout(item, timeout):
    start_time = current_time()
    while not lock_acquired:
        if current_time() - start_time > timeout:
            raise LockTimeoutException()
        try_acquire_lock(item)
```

### 3. Minimal Locking
```sql
-- Bad: Excessive locking
BEGIN TRANSACTION;
    LOCK TABLE accounts IN EXCLUSIVE MODE;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;

-- Good: Minimal locking
BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

## 9. Monitoring and Troubleshooting

### Key Metrics to Monitor
1. Lock Wait Time
2. Lock Contention Rate
3. Deadlock Frequency
4. Transaction Abort Rate

```sql
-- Example monitoring query
SELECT 
    wait_type,
    waiting_tasks_count,
    wait_time_ms / 1000.0 as wait_time_seconds
FROM sys.dm_os_wait_stats
WHERE wait_type LIKE 'LCK%'
ORDER BY wait_time_ms DESC;
```

### Common Issues
```
1. High Lock Contention:
   - Monitor lock wait times
   - Identify hot spots
   - Consider denormalization

2. Frequent Deadlocks:
   - Review lock ordering
   - Implement deadlock detection
   - Adjust transaction isolation levels
```