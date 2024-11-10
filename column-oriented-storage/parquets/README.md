# Understanding Parquet Files

## 1. What is Parquet?
Think of Parquet as a highly organized filing system that combines the best of both row and column storage. Like a library that not only separates books by genre (columns) but also maintains detailed indexes and summaries for quick searches.

## 2. Key Features Overview

| Feature | Traditional Files | Parquet Files |
|---------|------------------|---------------|
| Storage Method | Like a continuous book | Like a book with detailed table of contents and index |
| Data Access | Must read entire file | Can skip to relevant sections |
| Compression | Compresses everything together | Compresses similar data together |
| Query Speed | Reads all data to answer questions | Can answer some questions without reading data |
| Memory Usage | Loads more data than needed | Loads only necessary data |

## 3. Pushdown Predicate: Smart Data Reading

### What is Pushdown?
Think of pushdown like using a book's index instead of reading the whole book to find information.

#### Traditional File Reading
```
Question: "Find all orders over $100"

Process (Inefficient):
1. Open file
2. Read ALL order records
3. Check each price
4. Keep orders > $100
5. Return results

Like reading every page of a book to find mentions of a specific topic.
```

#### Parquet Pushdown
```
Question: "Find all orders over $100"

Process (Efficient):
1. Check metadata (like consulting index)
2. See price range for each chunk:
   Chunk 1: $50-$75    → Skip entirely
   Chunk 2: $90-$150   → Read this chunk
   Chunk 3: $25-$80    → Skip entirely

Like using an index to jump directly to relevant pages.
```

### Real-World Example: E-commerce Data
```
Order Data:
- 1 million orders
- Columns: order_id, customer_id, price, date, status

Traditional Query:
- Must read all 1 million rows
- Process: ~1 million × record size bytes

Parquet Query with Pushdown:
- Metadata shows: Only 100,000 orders > $100
- Process: ~100,000 × record size bytes
- 90% less data read!
```

## 4. Encoding: Smart Data Storage

### Types of Encoding

#### 1. Dictionary Encoding
Perfect for columns with repeated values (like status, country, category)

```
Original Data (Status Column):
"Shipped"
"Pending"
"Shipped"
"Cancelled"
"Shipped"
"Pending"

Dictionary Encoded:
Dictionary: {
  0: "Shipped"
  1: "Pending"
  2: "Cancelled"
}

Stored As: [0,1,0,2,0,1]
Savings: Instead of storing ~7 bytes per status, stores 1 byte
```

#### 2. Run Length Encoding (RLE)
Ideal for sequences of repeated values

```
Original Data (Category Column):
"Electronics"
"Electronics"
"Electronics"
"Clothing"
"Clothing"
"Books"

RLE Encoded:
(Electronics, 3)
(Clothing, 2)
(Books, 1)

Savings: Instead of storing the word multiple times, stores count
```

#### 3. Delta Encoding
Perfect for incrementing numbers or timestamps

```
Original Data (Timestamps):
2024-01-01 10:00:00
2024-01-01 10:01:00
2024-01-01 10:02:00
2024-01-01 10:03:00

Delta Encoded:
Base: 2024-01-01 10:00:00
Deltas: [0, +60, +60, +60]

Savings: Store one full timestamp + small increments
```

## 5. Why Parquet is Game-Changing: Real Scenarios

### Scenario 1: Daily Sales Report
```
Task: Calculate total sales by category for the day

Traditional File:
1. Read entire dataset
2. Extract category and price columns
3. Group and sum

Parquet Approach:
1. Read only category and price columns
2. Use dictionary encoding for categories
3. Skip chunks outside date range (pushdown)
4. Process much less data
```

### Scenario 2: Customer Analysis
```
Task: Find customers who spent over $1000

Traditional File:
1. Read all customer records
2. Sum all orders
3. Filter results

Parquet Advantages:
1. Column-based: Read only customer_id and amount
2. Pushdown: Skip chunks where max amount < $1000
3. Delta encoding: Efficiently store customer_ids
Result: Processes fraction of data volume
```

## 6. Performance Impact

### Space Efficiency
```
Example Dataset: E-commerce Orders
Size: 1TB uncompressed

Traditional Storage:
- 1TB raw data
- Basic compression: ~700GB

Parquet Storage:
- Dictionary encoding for categories/status: ~40% savings
- Delta encoding for timestamps: ~60% savings
- Column-based compression: ~70% savings
Final size: ~200GB
```

### Query Performance
```
Example: Find high-value orders from last month

Traditional Query Time:
1. Read 1TB of data
2. Filter dates
3. Filter amounts
Time: 100 seconds

Parquet Query Time:
1. Use pushdown to read only relevant chunks
2. Read only needed columns
3. Benefit from encoding
Time: 10 seconds
```