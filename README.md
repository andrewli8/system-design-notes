# system-design-notes

# Database Index Structures: A Comprehensive Guide

## 1. Hash Indexes

### Definition
A hash index is an in-memory data structure that uses a hash function to map keys to values, providing direct access to data locations. It's essentially a hash table implementation for database indexing.

### How it Works
1. When inserting a key-value pair:
   - Hash function converts the key into a memory address
   - Value is stored at that location
   - Index maintains mapping in RAM

2. For lookups:
   - Same hash function is applied to the search key
   - Directly access the memory location
   - Retrieve the value

### Durability Mechanism
- Write-Ahead Log (WAL):
  ```
  // Example WAL entries
  INSERT key1=value1
  UPDATE key1=value2
  DELETE key1
  ```
  - Each operation is recorded sequentially
  - On crash recovery, replay WAL to rebuild index

### Use Cases
- Session storage systems
- Cache implementations (like Redis)
- Small lookup tables
- Real-time gaming leaderboards

### Example Implementation Scenario
```python
class HashIndex:
    def __init__(self):
        self.index = {}
        self.wal = []
    
    def put(self, key, value):
        self.index[key] = value
        self.wal.append(f"PUT {key}={value}")
    
    def get(self, key):
        return self.index.get(key)
```

## 2. B-trees (Balanced Trees)

### Definition
A B-tree is a self-balancing tree data structure that maintains sorted data and allows for efficient insertion, deletion, and search operations. Unlike binary trees, B-trees can have more than two children per node.

### Structure
```
       [30, 60]
      /    |    \
[10,20] [40,50] [70,80]
```

### Properties
- Each node can contain multiple keys
- Keys within each node are sorted
- All leaf nodes are at the same level
- Nodes are typically sized to match disk block size (e.g., 4KB)

### Use Cases
- Traditional relational databases (MySQL InnoDB)
- File systems
- Geographic information systems
- Any system requiring range-based queries

### Example Operations
```sql
-- B-tree efficiently handles these queries
SELECT * FROM users WHERE age BETWEEN 25 AND 35;
SELECT * FROM orders WHERE order_date >= '2024-01-01';
```

## 3. LSM Trees (Log-Structured Merge Trees) + SSTables

### Definition
LSM trees combine in-memory trees with disk-based sorted string tables (SSTables) to provide efficient write performance while maintaining good read performance.

### Components
1. MemTable (In-memory):
   - Sorted tree structure
   - Accepts writes
   - Eventually flushed to disk

2. SSTables (On-disk):
   - Immutable sorted files
   - Multiple levels
   - Periodically compacted

### Example Structure
```
MemTable (RAM):
  key1 -> value1
  key2 -> value2

SSTable L0 (Disk):
  key3 -> value3
  key4 -> value4

SSTable L1 (Disk):
  key5 -> value5
  key6 -> value6
```

### Optimizations
1. Bloom Filters:
```python
def might_contain(key, bloom_filter):
    return all(hash_i(key) in bloom_filter for hash_i in hash_functions)
```

2. Sparse Indexing:
```
Index:
key1 -> offset 0
key100 -> offset 1000
key200 -> offset 2000
```

### Use Cases
- Write-heavy applications
- Time-series data
- Log data storage
- Systems like Cassandra, RocksDB

## Comparison Summary

### Performance Characteristics

| Operation | Hash Index | B-tree | LSM Tree |
|-----------|------------|---------|-----------|
| Read      | O(1)       | O(log n)| O(log n)  |
| Write     | O(1)       | O(log n)| O(1)*     |
| Range Query| N/A       | Fast    | Moderate  |
| Space     | High       | Moderate| Low       |

*Amortized with background compaction

### When to Choose Each

Choose **Hash Indexes** when:
- Dataset fits in memory
- Need fastest possible point lookups
- Don't need range queries
- Example: User session storage

Choose **B-trees** when:
- Need strong read performance
- Require range queries
- Have a balanced read/write workload
- Example: Traditional RDBMS

Choose **LSM Trees** when:
- Write performance is critical
- Can tolerate slightly slower reads
- Need good space efficiency
- Example: Time-series data storage

### Trade-offs Summary
1. **Hash Indexes**
   - ✅ Fastest possible lookups
   - ❌ Memory limited
   - ❌ No range queries

2. **B-trees**
   - ✅ Balanced performance
   - ✅ Excellent range queries
   - ❌ Higher disk I/O on writes

3. **LSM Trees**
   - ✅ Excellent write performance
   - ✅ Good space efficiency
   - ❌ Complex implementation
   - ❌ Background compaction overhead
