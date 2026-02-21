# Database Optimization

## Overview
Best practices for designing efficient database schemas, writing optimized queries, and scaling database operations for high-performance applications.

## When to Use This Skill
- Writing complex SQL queries
- Optimizing slow queries
- Designing database schemas
- Implementing indexing strategies
- Scaling database operations
- Troubleshooting N+1 query problems
- Working with ORMs efficiently

## Indexing Strategies

### When to Add Indexes
- Columns used in WHERE clauses
- Foreign keys for JOIN operations
- Columns used in ORDER BY, GROUP BY
- Columns frequently searched

### Index Types

**B-Tree Index (Default)**
```sql
-- Single column index
CREATE INDEX idx_users_email ON users(email);

-- Composite index (order matters!)
CREATE INDEX idx_users_status_created ON users(status, created_at);

-- Use composite index for queries like:
SELECT * FROM users WHERE status = 'active' ORDER BY created_at;
```

**Unique Index**
```sql
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);
-- Enforces uniqueness and improves lookup performance
```

**Partial/Filtered Index**
```sql
-- PostgreSQL
CREATE INDEX idx_active_users ON users(created_at) WHERE status = 'active';

-- Only indexes active users, smaller and faster
```

**Full-Text Search Index**
```sql
-- PostgreSQL
CREATE INDEX idx_products_search ON products USING GIN(to_tsvector('english', name || ' ' || description));

-- Search query
SELECT * FROM products 
WHERE to_tsvector('english', name || ' ' || description) @@ to_tsquery('laptop');
```

### Index Trade-offs
**Pros:**
- Faster SELECT queries
- Faster JOIN operations
- Enforces uniqueness

**Cons:**
- Slower INSERT/UPDATE/DELETE
- Takes up disk space
- Maintenance overhead

**Best Practice:** Index frequently read columns, be selective for write-heavy tables

## Query Optimization

### 1. Use EXPLAIN to Analyze Queries
```sql
EXPLAIN ANALYZE
SELECT u.name, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.status = 'active'
AND o.created_at > '2026-01-01';

-- Look for:
-- - Seq Scan (bad) vs Index Scan (good)
-- - High cost values
-- - Nested loops on large datasets
```

### 2. Avoid SELECT *
```sql
-- ❌ Bad - Retrieves unnecessary data
SELECT * FROM users WHERE id = 123;

-- ✅ Good - Only fetch needed columns
SELECT id, name, email FROM users WHERE id = 123;
```

### 3. Use Appropriate JOIN Types
```sql
-- INNER JOIN - Only matching records
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN - All from left table, matching from right
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;

-- ⚠️ Avoid unnecessary JOINs when possible
```

### 4. Optimize WHERE Clauses
```sql
-- ✅ Good - Uses index on status
SELECT * FROM users WHERE status = 'active';

-- ❌ Bad - Function prevents index usage
SELECT * FROM users WHERE UPPER(status) = 'ACTIVE';

-- ✅ Good - Store normalized data
SELECT * FROM users WHERE status = 'active'; -- Ensure data is lowercase

-- ❌ Bad - Leading wildcard prevents index
SELECT * FROM users WHERE email LIKE '%@example.com';

-- ✅ Good - Can use index
SELECT * FROM users WHERE email LIKE 'john%';
```

### 5. Use LIMIT for Large Result Sets
```sql
-- ❌ Bad - Returns all results
SELECT * FROM orders ORDER BY created_at DESC;

-- ✅ Good - Pagination
SELECT * FROM orders 
ORDER BY created_at DESC 
LIMIT 20 OFFSET 40; -- Page 3

-- ✅ Better - Cursor-based pagination (for large offsets)
SELECT * FROM orders 
WHERE created_at < '2026-02-01'
ORDER BY created_at DESC 
LIMIT 20;
```

## N+1 Query Problem

### The Problem
```javascript
// ❌ Bad - N+1 queries
const users = await User.findAll(); // 1 query

for (const user of users) {
  const orders = await Order.findAll({ where: { userId: user.id } }); // N queries
  user.orders = orders;
}
// Total: 1 + N queries
```

### Solutions

**1. Eager Loading (ORM)**
```javascript
// ✅ Good - 2 queries total
const users = await User.findAll({
  include: [{
    model: Order,
    as: 'orders'
  }]
});

// Or with Prisma
const users = await prisma.user.findMany({
  include: {
    orders: true
  }
});
```

**2. Manual JOIN**
```sql
-- ✅ Good - Single query
SELECT 
  u.id, u.name, u.email,
  o.id as order_id, o.total, o.created_at as order_created_at
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
```

**3. Batch Loading (DataLoader)**
```javascript
const DataLoader = require('dataloader');

const orderLoader = new DataLoader(async (userIds) => {
  const orders = await Order.findAll({
    where: { userId: userIds }
  });
  
  // Group orders by userId
  const ordersByUser = userIds.map(id => 
    orders.filter(o => o.userId === id)
  );
  
  return ordersByUser;
});

// Usage - automatically batches requests
const user1Orders = await orderLoader.load(user1.id);
const user2Orders = await orderLoader.load(user2.id);
// Only 1 query executed
```

## Database Schema Design

### Normalization

**1NF (First Normal Form)**
- Atomic values (no arrays in cells)
- Each row is unique

**2NF (Second Normal Form)**
- 1NF + No partial dependencies
- All non-key attributes depend on entire primary key

**3NF (Third Normal Form)**
- 2NF + No transitive dependencies
- Non-key attributes don't depend on other non-key attributes

### When to Denormalize
```sql
-- Normalized (3NF)
CREATE TABLE orders (
  id INT PRIMARY KEY,
  user_id INT,
  created_at TIMESTAMP
);

CREATE TABLE order_items (
  id INT PRIMARY KEY,
  order_id INT,
  product_id INT,
  quantity INT,
  price DECIMAL
);

-- Calculating total requires aggregation
SELECT o.id, SUM(oi.quantity * oi.price) as total
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
GROUP BY o.id;

-- Denormalized - add total column
CREATE TABLE orders (
  id INT PRIMARY KEY,
  user_id INT,
  total DECIMAL, -- Denormalized for performance
  created_at TIMESTAMP
);

-- Update total when items change (use triggers or application logic)
```

**When to Denormalize:**
- Read-heavy operations
- Expensive aggregations
- Reporting/analytics tables
- Caching computed values

**When NOT to Denormalize:**
- Write-heavy tables
- Data frequently changes
- Storage is a concern

### Appropriate Data Types
```sql
-- ✅ Good - Right-sized types
CREATE TABLE users (
  id BIGINT PRIMARY KEY,           -- For large tables
  email VARCHAR(255) NOT NULL,     -- Reasonable max length
  age SMALLINT,                    -- 0-255 is enough
  balance DECIMAL(10,2),           -- Precise for money
  is_active BOOLEAN,               -- Not INT
  created_at TIMESTAMP DEFAULT NOW()
);

-- ❌ Bad - Oversized types
CREATE TABLE users (
  id VARCHAR(1000),                -- Waste of space
  email TEXT,                      -- No max length validation
  age INT,                         -- Unnecessarily large
  balance FLOAT,                   -- Precision issues for money
  is_active VARCHAR(10),           -- Use BOOLEAN
  created_at VARCHAR(50)           -- Use TIMESTAMP
);
```

## Transactions

### ACID Properties
- **Atomicity**: All or nothing
- **Consistency**: Valid state transitions
- **Isolation**: Concurrent transactions don't interfere
- **Durability**: Committed data persists

### Transaction Usage
```javascript
// ✅ Good - Use transactions for related operations
await db.transaction(async (trx) => {
  const order = await trx('orders').insert({
    user_id: userId,
    total: 100
  }).returning('*');
  
  await trx('order_items').insert([
    { order_id: order.id, product_id: 1, quantity: 2 },
    { order_id: order.id, product_id: 2, quantity: 1 }
  ]);
  
  await trx('users').where({ id: userId }).decrement('balance', 100);
  
  // All succeed or all fail together
});

// ❌ Bad - No transaction
const order = await Order.create({ userId, total: 100 });
await OrderItem.insert([...]); // Could fail, leaving orphaned order
await User.update({ balance: balance - 100 }); // Could fail, inconsistent state
```

### Isolation Levels
```sql
-- Read Uncommitted (lowest isolation, highest performance)
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- Read Committed (default in PostgreSQL)
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Repeatable Read
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Serializable (highest isolation, lowest performance)
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

## Connection Pooling

### Configuration
```javascript
const { Pool } = require('pg');

const pool = new Pool({
  host: 'localhost',
  port: 5432,
  database: 'mydb',
  user: 'user',
  password: 'password',
  max: 20,          // Maximum connections
  min: 5,           // Minimum connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// ✅ Good - Use pool
app.get('/api/users', async (req, res) => {
  const client = await pool.connect();
  try {
    const result = await client.query('SELECT * FROM users');
    res.json(result.rows);
  } finally {
    client.release(); // Important!
  }
});

// ❌ Bad - Creating new connection each time
app.get('/api/users', async (req, res) => {
  const client = new Client({ ... });
  await client.connect();
  const result = await client.query('SELECT * FROM users');
  await client.end();
  res.json(result.rows);
});
```

## Caching Strategies

### Application-Level Caching
```javascript
const Redis = require('ioredis');
const redis = new Redis();

async function getUserById(id) {
  // Try cache first
  const cached = await redis.get(`user:${id}`);
  if (cached) {
    return JSON.parse(cached);
  }
  
  // Cache miss - query database
  const user = await User.findById(id);
  
  // Store in cache (TTL: 1 hour)
  await redis.setex(`user:${id}`, 3600, JSON.stringify(user));
  
  return user;
}

// Invalidate cache on update
async function updateUser(id, data) {
  const user = await User.update(id, data);
  await redis.del(`user:${id}`); // Invalidate cache
  return user;
}
```

### Query Result Caching
```sql
-- PostgreSQL - Materialized views
CREATE MATERIALIZED VIEW popular_products AS
SELECT p.id, p.name, COUNT(oi.id) as order_count
FROM products p
JOIN order_items oi ON p.id = oi.product_id
GROUP BY p.id, p.name
ORDER BY order_count DESC;

-- Refresh periodically
REFRESH MATERIALIZED VIEW popular_products;

-- Query the cached view
SELECT * FROM popular_products LIMIT 10;
```

## Database Partitioning

### Range Partitioning
```sql
-- PostgreSQL - Partition by date
CREATE TABLE orders (
  id BIGINT,
  user_id INT,
  total DECIMAL,
  created_at TIMESTAMP
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2025 PARTITION OF orders
FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

CREATE TABLE orders_2026 PARTITION OF orders
FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');

-- Queries automatically use correct partition
SELECT * FROM orders WHERE created_at > '2026-01-01';
```

### List Partitioning
```sql
-- Partition by region
CREATE TABLE users (
  id BIGINT,
  name VARCHAR(255),
  region VARCHAR(10)
) PARTITION BY LIST (region);

CREATE TABLE users_us PARTITION OF users FOR VALUES IN ('US');
CREATE TABLE users_eu PARTITION OF users FOR VALUES IN ('EU', 'UK');
CREATE TABLE users_asia PARTITION OF users FOR VALUES IN ('CN', 'JP', 'IN');
```

## Database Migrations

### Best Practices
```javascript
// Migration: add_email_index.js

exports.up = async function(knex) {
  // Check if index exists before creating
  const hasIndex = await knex.schema.hasColumn('users', 'email');
  if (hasIndex) {
    await knex.schema.alterTable('users', (table) => {
      table.index('email', 'idx_users_email');
    });
  }
};

exports.down = async function(knex) {
  await knex.schema.alterTable('users', (table) => {
    table.dropIndex('email', 'idx_users_email');
  });
};

// ✅ Always provide rollback (down)
// ✅ Make migrations idempotent
// ✅ Test migrations on staging first
// ⚠️ Be careful with data migrations on large tables
```

### Safe Schema Changes
```sql
-- ✅ Safe - Add nullable column
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- ✅ Safe - Add column with default (PostgreSQL 11+)
ALTER TABLE users ADD COLUMN status VARCHAR(20) DEFAULT 'active';

-- ⚠️ Risky - Add NOT NULL column without default
ALTER TABLE users ADD COLUMN phone VARCHAR(20) NOT NULL;
-- Better: Add nullable first, populate, then add constraint

-- Step 1: Add nullable column
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Step 2: Populate with default value
UPDATE users SET phone = '' WHERE phone IS NULL;

-- Step 3: Add NOT NULL constraint
ALTER TABLE users ALTER COLUMN phone SET NOT NULL;
```

## ORM Best Practices

### Efficient Queries with Prisma
```javascript
// ✅ Good - Select only needed fields
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    email: true
  }
});

// ✅ Good - Use findUnique for single records
const user = await prisma.user.findUnique({
  where: { id: 123 }
});

// ✅ Good - Batch operations
await prisma.order.createMany({
  data: orders, // Array of orders
  skipDuplicates: true
});

// ❌ Bad - Multiple individual creates
for (const order of orders) {
  await prisma.order.create({ data: order });
}
```

### TypeORM Optimization
```typescript
// ✅ Good - Use QueryBuilder for complex queries
const users = await userRepository
  .createQueryBuilder('user')
  .leftJoinAndSelect('user.orders', 'order')
  .where('user.status = :status', { status: 'active' })
  .andWhere('order.total > :minTotal', { minTotal: 100 })
  .getMany();

// ❌ Bad - Loading all entities
const users = await userRepository.find({
  relations: ['orders'] // Loads ALL orders
});
```

## Monitoring & Maintenance

### Slow Query Log
```sql
-- PostgreSQL - Enable slow query logging
ALTER SYSTEM SET log_min_duration_statement = 1000; -- Log queries > 1s
SELECT pg_reload_conf();

-- MySQL
SET GLOBAL slow_query_log = 1;
SET GLOBAL long_query_time = 1;
```

### Database Statistics
```sql
-- PostgreSQL - Check table sizes
SELECT 
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;

-- Check index usage
SELECT 
  schemaname,
  tablename,
  indexname,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0 -- Unused indexes
ORDER BY pg_relation_size(indexrelid) DESC;
```

### VACUUM and ANALYZE
```sql
-- PostgreSQL - Reclaim space and update statistics
VACUUM ANALYZE users;

-- Full vacuum (locks table)
VACUUM FULL users;

-- Auto-vacuum configuration
ALTER TABLE users SET (autovacuum_vacuum_scale_factor = 0.1);
```

## Database Optimization Checklist

- [ ] Indexes on frequently queried columns
- [ ] Composite indexes for multi-column queries
- [ ] No unused indexes (check pg_stat_user_indexes)
- [ ] Proper data types (no oversized columns)
- [ ] Foreign key constraints with indexes
- [ ] Normalized schema (unless denormalization justified)
- [ ] Connection pooling configured
- [ ] Query optimization (use EXPLAIN ANALYZE)
- [ ] No N+1 query problems
- [ ] Batch operations for bulk inserts/updates
- [ ] Transactions for related operations
- [ ] Caching for expensive queries
- [ ] Slow query logging enabled
- [ ] Regular VACUUM and ANALYZE
- [ ] Monitoring query performance
- [ ] Prepared statements (SQL injection prevention)
- [ ] Database backups automated
- [ ] Partitioning for large tables
- [ ] Read replicas for scaling reads
- [ ] Regular security updates
