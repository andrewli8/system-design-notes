# Understanding Database Consistency

## What is Consistency?
Consistency ensures that a transaction can only bring the database from one valid state to another valid state, maintaining all predefined rules, constraints, and data integrity. Think of it like a chess game - every move must follow the rules, and the board must always be in a valid configuration.

## Types of Consistency Rules

### 1. Entity Integrity (Primary Key Rules)
Ensures every record has a unique identifier.

#### Without Entity Integrity
```
Users Table:
ID  | Name
1   | John
1   | Jane    -- Problem: Duplicate ID
null| Bob     -- Problem: Null ID
```

#### With Entity Integrity
```sql
-- Proper definition with constraints
CREATE TABLE users (
    id INT PRIMARY KEY,  -- Ensures unique, non-null ID
    name VARCHAR(100) NOT NULL
);

-- Example violations:
INSERT INTO users (id, name) VALUES (1, 'John');  -- Succeeds
INSERT INTO users (id, name) VALUES (1, 'Jane');  -- Fails: Duplicate ID
INSERT INTO users (id, name) VALUES (NULL, 'Bob'); -- Fails: Null ID
```

### 2. Referential Integrity (Foreign Key Rules)
Ensures relationships between tables remain valid.

#### Without Referential Integrity
```
Orders Table:
OrderID | CustomerID | Amount
1       | 100       | 50.00
2       | 999       | 75.00  -- Problem: Customer 999 doesn't exist

Customers Table:
CustomerID | Name
100       | John
```

#### With Referential Integrity
```sql
-- Proper table definitions
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    amount DECIMAL(10,2),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- Example scenarios:
-- Scenario 1: Valid insert
INSERT INTO customers VALUES (100, 'John');
INSERT INTO orders VALUES (1, 100, 50.00);  -- Succeeds

-- Scenario 2: Invalid insert
INSERT INTO orders VALUES (2, 999, 75.00);  -- Fails: Customer 999 doesn't exist

-- Scenario 3: Attempted deletion of referenced customer
DELETE FROM customers WHERE customer_id = 100;  -- Fails: Has orders
```

### 3. Domain Consistency
Ensures values fall within specified ranges or sets.

#### Without Domain Consistency
```
Products Table:
ID | Price  | Stock
1  | -10.00 | -5     -- Problems: Negative price and stock
2  | 0      | 1000000 -- Problem: Unrealistic stock level
```

#### With Domain Consistency
```sql
CREATE TABLE products (
    id INT PRIMARY KEY,
    price DECIMAL(10,2) CHECK (price >= 0),
    stock INT CHECK (stock >= 0 AND stock <= 10000),
    status VARCHAR(20) CHECK (status IN ('active', 'discontinued', 'out_of_stock'))
);

-- Example violations:
INSERT INTO products VALUES (1, -10.00, 100);     -- Fails: Negative price
INSERT INTO products VALUES (2, 10.00, -5);       -- Fails: Negative stock
INSERT INTO products VALUES (3, 10.00, 1000000);  -- Fails: Stock too high
INSERT INTO products VALUES (4, 10.00, 100, 'pending');  -- Fails: Invalid status
```

## Real-World Consistency Scenarios

### 1. Banking System Example

#### Account Transfer with Consistency Rules
```
Rule 1: Account balance cannot be negative
Rule 2: Total money in system must remain constant
Rule 3: Transaction amount must be positive
```

```sql
BEGIN TRANSACTION;
    -- Get current balances
    DECLARE @balance_A DECIMAL(10,2);
    DECLARE @balance_B DECIMAL(10,2);
    DECLARE @transfer_amount DECIMAL(10,2) = 200.00;
    
    SELECT @balance_A = balance FROM accounts WHERE account_id = 'A';
    SELECT @balance_B = balance FROM accounts WHERE account_id = 'B';
    
    -- Check consistency rules
    IF @transfer_amount <= 0
        THROW 50001, 'Transfer amount must be positive', 1;
        
    IF @balance_A - @transfer_amount < 0
        THROW 50002, 'Insufficient funds', 1;
    
    -- Perform transfer
    UPDATE accounts SET balance = balance - @transfer_amount 
    WHERE account_id = 'A';
    
    UPDATE accounts SET balance = balance + @transfer_amount 
    WHERE account_id = 'B';
    
    -- Verify system consistency
    IF (SELECT SUM(balance) FROM accounts) != 
       (SELECT @original_total FROM @snapshot)
        THROW 50003, 'System total mismatch', 1;
    
COMMIT;
```

### 2. E-commerce Inventory System

#### Order Processing with Consistency Rules
```
Rule 1: Cannot sell more than available stock
Rule 2: Order total must match sum of line items
Rule 3: Shipping address must be valid
```

```sql
BEGIN TRANSACTION;
    -- Create order
    DECLARE @order_id INT;
    DECLARE @total_amount DECIMAL(10,2);
    
    -- Insert order header
    INSERT INTO orders (customer_id, order_date, status)
    VALUES (@customer_id, GETDATE(), 'pending');
    
    SET @order_id = SCOPE_IDENTITY();
    
    -- Add order items (with consistency checks)
    INSERT INTO order_items (order_id, product_id, quantity, price)
    SELECT 
        @order_id,
        product_id,
        quantity,
        (SELECT price FROM products WHERE id = cart.product_id)
    FROM shopping_cart cart
    WHERE customer_id = @customer_id;
    
    -- Verify stock levels
    IF EXISTS (
        SELECT 1 
        FROM order_items oi
        JOIN products p ON oi.product_id = p.id
        WHERE oi.order_id = @order_id 
        AND oi.quantity > p.stock
    )
        THROW 50001, 'Insufficient stock', 1;
    
    -- Verify order total
    SELECT @total_amount = SUM(quantity * price)
    FROM order_items
    WHERE order_id = @order_id;
    
    IF @total_amount != (SELECT total FROM orders WHERE id = @order_id)
        THROW 50002, 'Order total mismatch', 1;
    
    -- Update inventory
    UPDATE p
    SET stock = p.stock - oi.quantity
    FROM products p
    JOIN order_items oi ON p.id = oi.product_id
    WHERE oi.order_id = @order_id;
    
COMMIT;
```

## Common Consistency Violations and Solutions

### 1. Race Conditions
```sql
-- Problem: Race condition in stock check
SELECT stock FROM products WHERE id = 1;  -- Shows 10 units
-- Meanwhile, another transaction buys 10 units
UPDATE products SET stock = stock - 10 WHERE id = 1;  -- Now 0 units
-- Our transaction tries to buy 5 units
UPDATE products SET stock = stock - 5 WHERE id = 1;  -- Goes negative!

-- Solution: Use proper isolation and locking
BEGIN TRANSACTION;
    SELECT stock FROM products WITH (UPDLOCK) WHERE id = 1;
    -- No other transaction can modify stock until we're done
    IF @stock >= @requested_quantity
        UPDATE products SET stock = stock - @requested_quantity;
COMMIT;
```

### 2. Cascading Effects
```sql
-- Problem: Deleting parent record breaks consistency
DELETE FROM customers WHERE id = 100;
-- Orphans all orders for customer 100

-- Solution 1: Use CASCADE
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    FOREIGN KEY (customer_id) 
    REFERENCES customers(customer_id)
    ON DELETE CASCADE
);

-- Solution 2: Manual handling
BEGIN TRANSACTION;
    -- First delete dependent records
    DELETE FROM order_items WHERE order_id IN 
        (SELECT order_id FROM orders WHERE customer_id = 100);
    DELETE FROM orders WHERE customer_id = 100;
    -- Then delete parent
    DELETE FROM customers WHERE id = 100;
COMMIT;
```

## Best Practices

### 1. Define Clear Constraints
```sql
-- Good: Clear, explicit constraints
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) CHECK (price >= 0),
    stock INT CHECK (stock >= 0),
    category_id INT FOREIGN KEY REFERENCES categories(id)
);

-- Bad: Implicit or missing constraints
CREATE TABLE products (
    id INT,
    name VARCHAR(100),
    price DECIMAL(10,2),  -- Could be negative
    stock INT,            -- Could be negative
    category_id INT       -- No referential integrity
);
```

### 2. Use Transactions Appropriately
```sql
-- Good: Grouping related changes
BEGIN TRANSACTION;
    INSERT INTO orders (customer_id, total) VALUES (1, 100);
    UPDATE inventory SET stock = stock - 1;
    UPDATE customer_balance SET amount = amount - 100;
    -- All or nothing
COMMIT;

-- Bad: Separate operations
INSERT INTO orders (customer_id, total) VALUES (1, 100);
-- System could fail here
UPDATE inventory SET stock = stock - 1;
-- Or here
UPDATE customer_balance SET amount = amount - 100;
```

### 3. Regular Consistency Checks
```sql
-- Example consistency check procedure
CREATE PROCEDURE check_system_consistency
AS
BEGIN
    -- Check for orphaned orders
    IF EXISTS (
        SELECT 1 FROM orders o
        LEFT JOIN customers c ON o.customer_id = c.id
        WHERE c.id IS NULL
    )
        THROW 50001, 'Orphaned orders found', 1;
        
    -- Check for negative balances
    IF EXISTS (
        SELECT 1 FROM accounts
        WHERE balance < 0
    )
        THROW 50002, 'Negative balances found', 1;
        
    -- Check inventory consistency
    IF EXISTS (
        SELECT 1 FROM products
        WHERE stock < 0 OR price < 0
    )
        THROW 50003, 'Invalid product data found', 1;
END;
```

Would you like me to create the detailed Durability guide next?