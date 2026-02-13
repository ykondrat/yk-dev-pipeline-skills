# SQL Databases — PostgreSQL & MySQL

Best practices for working with relational databases in Node.js/TypeScript.
Covers both ORM (Prisma, Drizzle) and raw driver (pg, mysql2) patterns.

---

## Table of Contents

1. [Connection Management & Pooling](#1-connection-management--pooling)
2. [Schema Design & Data Modeling](#2-schema-design--data-modeling)
3. [Query Patterns & Optimization](#3-query-patterns--optimization)
4. [Indexing Strategies](#4-indexing-strategies)
5. [Transactions & Consistency](#5-transactions--consistency)
6. [Migrations](#6-migrations)
7. [Security](#7-security)
8. [Error Handling & Retry Logic](#8-error-handling--retry-logic)
9. [Testing](#9-testing)

---

## 1. Connection Management & Pooling

### Why Pooling Matters

Every database connection consumes resources — TCP socket, memory for session state,
authentication overhead. Creating a new connection per query is expensive (~20-50ms for
PostgreSQL). A connection pool maintains a set of reusable connections, dramatically
reducing latency and resource consumption.

### Raw Driver — pg (PostgreSQL)

```typescript
import pg from 'pg';

const pool = new pg.Pool({
  host: process.env.DB_HOST,
  port: Number(process.env.DB_PORT) || 5432,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,

  // Pool sizing
  min: 2,                    // Minimum idle connections
  max: 20,                   // Maximum total connections
  idleTimeoutMillis: 30_000, // Close idle connections after 30s
  connectionTimeoutMillis: 5_000, // Fail if can't connect in 5s

  // Statement timeout — prevent runaway queries
  statement_timeout: 30_000, // 30s max per query
});

// Monitor pool health
pool.on('error', (err) => {
  logger.error('Unexpected pool error', { error: err });
});

pool.on('connect', () => {
  logger.debug('New database connection established');
});

// Graceful shutdown
async function closePool(): Promise<void> {
  await pool.end();
  logger.info('Database pool closed');
}

// Usage — always release connections
async function getUser(id: string): Promise<User | null> {
  const client = await pool.connect();
  try {
    const result = await client.query<User>(
      'SELECT * FROM users WHERE id = $1',
      [id]
    );
    return result.rows[0] ?? null;
  } finally {
    client.release(); // ALWAYS release, even on error
  }
}

// Simpler: use pool.query() for single queries (auto-releases)
async function getUserSimple(id: string): Promise<User | null> {
  const result = await pool.query<User>(
    'SELECT * FROM users WHERE id = $1',
    [id]
  );
  return result.rows[0] ?? null;
}
```

**Why these pool settings:** `min: 2` keeps connections warm for instant use. `max: 20`
prevents overwhelming the database (PostgreSQL's default `max_connections` is 100 — if
you have 5 app instances, each with max 20, that's the limit). `idleTimeoutMillis` reclaims
unused connections. `statement_timeout` prevents a single bad query from holding a connection
forever.

### Raw Driver — mysql2 (MySQL)

```typescript
import mysql from 'mysql2/promise';

const pool = mysql.createPool({
  host: process.env.DB_HOST,
  port: Number(process.env.DB_PORT) || 3306,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,

  waitForConnections: true,
  connectionLimit: 20,
  queueLimit: 0,              // Unlimited queue (or set a bound)
  enableKeepAlive: true,
  keepAliveInitialDelay: 10_000,

  // Timezone handling — avoid surprises
  timezone: '+00:00',         // Always store in UTC
  dateStrings: false,         // Return Date objects, not strings
});

// Usage
async function getUser(id: string): Promise<User | null> {
  const [rows] = await pool.execute<mysql.RowDataPacket[]>(
    'SELECT * FROM users WHERE id = ?',
    [id]
  );
  return (rows[0] as User) ?? null;
}
```

**MySQL vs PostgreSQL note:** MySQL uses `?` for parameter placeholders (positional),
PostgreSQL uses `$1, $2, ...` (numbered). MySQL's `execute()` uses prepared statements
(more efficient for repeated queries) while `query()` does string interpolation.
Always prefer `execute()`.

### ORM — Prisma

```typescript
import { PrismaClient } from '@prisma/client';

// Singleton pattern — create once, reuse everywhere
const prisma = new PrismaClient({
  log: [
    { level: 'query', emit: 'event' },   // Log queries in development
    { level: 'error', emit: 'stdout' },
    { level: 'warn', emit: 'stdout' },
  ],
  datasources: {
    db: {
      url: process.env.DATABASE_URL,
    },
  },
});

// Query logging (development only)
if (process.env.NODE_ENV === 'development') {
  prisma.$on('query', (e) => {
    logger.debug('Query', {
      query: e.query,
      params: e.params,
      duration: `${e.duration}ms`,
    });
  });
}

// Connection management
async function connectDatabase(): Promise<void> {
  await prisma.$connect();
  logger.info('Database connected');
}

async function disconnectDatabase(): Promise<void> {
  await prisma.$disconnect();
  logger.info('Database disconnected');
}

// Prisma manages its own connection pool via the connection URL:
// postgresql://user:pass@host:5432/db?connection_limit=20&pool_timeout=30
```

**Why Prisma singleton:** Prisma manages its own connection pool internally. Creating
multiple `PrismaClient` instances creates multiple pools, wasting connections. The
singleton pattern ensures one client (and one pool) per process.

### ORM — Drizzle

```typescript
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import * as schema from './schema.js';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,
});

const db = drizzle(pool, { schema, logger: true });

// Usage — type-safe queries
import { eq } from 'drizzle-orm';

async function getUser(id: string) {
  const result = await db.select()
    .from(schema.users)
    .where(eq(schema.users.id, id))
    .limit(1);

  return result[0] ?? null;
}
```

**Drizzle vs Prisma:** Drizzle is SQL-first — it generates SQL that looks like what you'd
write by hand, with full TypeScript inference from your schema. Prisma is more abstracted —
it has its own query language and generates SQL behind the scenes. Drizzle gives you more
control and generates more efficient queries; Prisma has better tooling (migrations, studio)
and a gentler learning curve.

---

## 2. Schema Design & Data Modeling

### Core Principles

**Normalize first, denormalize for performance.** Start with a properly normalized schema
(3NF). Only denormalize when you have measured performance data showing a join is too slow.
Premature denormalization leads to data inconsistency.

**Every table needs:**
- A primary key (prefer UUID or ULID over auto-increment for distributed systems)
- `created_at` timestamp (when was this row created)
- `updated_at` timestamp (when was it last modified)
- Indexes on foreign keys and frequently queried columns

### Prisma Schema Example

```prisma
// schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql" // or "mysql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String
  role      Role     @default(USER)
  password  String   // hashed, never plain text
  
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")
  
  posts     Post[]
  profile   Profile?
  
  @@map("users") // table name in snake_case
  @@index([email])
  @@index([role])
}

model Post {
  id          String   @id @default(cuid())
  title       String
  content     String?
  published   Boolean  @default(false)
  
  authorId    String   @map("author_id")
  author      User     @relation(fields: [authorId], references: [id], onDelete: Cascade)
  
  categoryId  String?  @map("category_id")
  category    Category? @relation(fields: [categoryId], references: [id])
  
  tags        PostTag[]
  
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")
  
  @@map("posts")
  @@index([authorId])
  @@index([categoryId])
  @@index([published, createdAt]) // composite for common query
}

// Many-to-many via explicit join table (more control than implicit)
model PostTag {
  postId  String @map("post_id")
  tagId   String @map("tag_id")
  
  post    Post   @relation(fields: [postId], references: [id], onDelete: Cascade)
  tag     Tag    @relation(fields: [tagId], references: [id], onDelete: Cascade)
  
  @@id([postId, tagId])
  @@map("post_tags")
}

enum Role {
  USER
  ADMIN
  MODERATOR
}
```

### Drizzle Schema Example

```typescript
// schema.ts
import {
  pgTable, varchar, text, boolean, timestamp,
  primaryKey, index, pgEnum
} from 'drizzle-orm/pg-core';
import { createId } from '@paralleldrive/cuid2';

export const roleEnum = pgEnum('role', ['USER', 'ADMIN', 'MODERATOR']);

export const users = pgTable('users', {
  id: varchar('id', { length: 128 }).$defaultFn(() => createId()).primaryKey(),
  email: varchar('email', { length: 255 }).notNull().unique(),
  name: varchar('name', { length: 255 }).notNull(),
  role: roleEnum('role').default('USER').notNull(),
  password: varchar('password', { length: 255 }).notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
}, (table) => ({
  emailIdx: index('users_email_idx').on(table.email),
  roleIdx: index('users_role_idx').on(table.role),
}));

export const posts = pgTable('posts', {
  id: varchar('id', { length: 128 }).$defaultFn(() => createId()).primaryKey(),
  title: varchar('title', { length: 500 }).notNull(),
  content: text('content'),
  published: boolean('published').default(false).notNull(),
  authorId: varchar('author_id', { length: 128 }).notNull()
    .references(() => users.id, { onDelete: 'cascade' }),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
}, (table) => ({
  authorIdx: index('posts_author_id_idx').on(table.authorId),
  publishedIdx: index('posts_published_idx').on(table.published, table.createdAt),
}));
```

### ID Strategy

```typescript
// ❌ Auto-increment — leaks info, bad for distributed systems
// id SERIAL PRIMARY KEY

// ✅ CUID2 — collision-resistant, URL-safe, sortable
import { createId } from '@paralleldrive/cuid2';
const id = createId(); // "clh3am47c0000qf2t7z3z8z3z"

// ✅ ULID — sortable by time, 128-bit, Crockford base32
import { ulid } from 'ulid';
const id = ulid(); // "01ARZ3NDEKTSV4RRFFQ69G5FAV"

// ✅ UUID v7 — time-ordered UUID (best for PostgreSQL b-tree performance)
import { v7 as uuidv7 } from 'uuid';
const id = uuidv7(); // "018f3e1e-5b3a-7b3a-8b3a-1b3a5b3a7b3a"
```

**Why time-ordered IDs:** Random UUIDs (v4) fragment b-tree indexes because inserts land
at random positions. Time-ordered IDs (ULID, UUID v7, CUID2) insert sequentially,
keeping the index efficient and improving write performance by 2-10x for large tables.

### Soft Deletes

```typescript
// Add to tables where you need recoverability or audit trails
model User {
  // ... other fields
  deletedAt DateTime? @map("deleted_at")
  
  @@index([deletedAt]) // for filtering
}

// Always filter in queries
const activeUsers = await prisma.user.findMany({
  where: { deletedAt: null },
});

// Soft delete
await prisma.user.update({
  where: { id },
  data: { deletedAt: new Date() },
});
```

**Why soft deletes:** Hard deletes are irreversible and break foreign key references.
Soft deletes preserve data for audit trails, legal compliance, and "undo" functionality.
The trade-off is that every query must filter `WHERE deleted_at IS NULL`, which is easy
to forget — consider using database views or Prisma middleware to auto-filter.

---

## 3. Query Patterns & Optimization

### N+1 Problem — The Most Common Performance Killer

```typescript
// ❌ N+1 — 1 query for posts + N queries for authors
const posts = await prisma.post.findMany();
for (const post of posts) {
  const author = await prisma.user.findUnique({
    where: { id: post.authorId },
  });
  // This runs a separate query for EACH post
}

// ✅ Prisma — use include for eager loading
const posts = await prisma.post.findMany({
  include: { author: true }, // Single JOIN query
});

// ✅ Drizzle — explicit join
const posts = await db.select()
  .from(schema.posts)
  .leftJoin(schema.users, eq(schema.posts.authorId, schema.users.id));

// ✅ Raw SQL — explicit JOIN
const result = await pool.query(`
  SELECT p.*, u.name as author_name, u.email as author_email
  FROM posts p
  LEFT JOIN users u ON p.author_id = u.id
  WHERE p.published = true
  ORDER BY p.created_at DESC
  LIMIT $1 OFFSET $2
`, [limit, offset]);
```

**Why this matters:** N+1 queries turn a single page load into 101 database round trips
(1 for the list + 100 for each item's related data). Each round trip has network latency,
query parsing, and execution overhead. A single JOIN does the same work in one round trip.
Always check for N+1 queries by enabling query logging in development.

### Select Only What You Need

```typescript
// ❌ Bad — fetches all columns including password hash
const users = await prisma.user.findMany();

// ✅ Good — only the fields you need
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    email: true,
    // password deliberately excluded
  },
});

// ✅ Drizzle — pick specific columns
const users = await db.select({
  id: schema.users.id,
  name: schema.users.name,
  email: schema.users.email,
}).from(schema.users);

// ✅ Raw SQL
const result = await pool.query(
  'SELECT id, name, email FROM users WHERE role = $1',
  ['ADMIN']
);
```

**Why:** Selecting `*` transfers unnecessary data (especially large text/blob columns),
wastes memory, and can accidentally expose sensitive fields. Explicit column selection
also makes your code self-documenting — you can see exactly what data each query needs.

### Pagination — Cursor-Based

```typescript
// ✅ Cursor-based (efficient, stable)
async function getPosts(cursor?: string, limit = 20) {
  const posts = await prisma.post.findMany({
    take: limit + 1, // fetch one extra to detect "has more"
    ...(cursor && {
      cursor: { id: cursor },
      skip: 1, // skip the cursor item itself
    }),
    orderBy: { createdAt: 'desc' },
    select: { id: true, title: true, createdAt: true },
  });

  const hasMore = posts.length > limit;
  const items = hasMore ? posts.slice(0, limit) : posts;

  return {
    items,
    nextCursor: hasMore ? items[items.length - 1].id : null,
  };
}

// Raw SQL cursor pagination
async function getPostsRaw(cursor?: string, limit = 20) {
  const query = cursor
    ? 'SELECT * FROM posts WHERE created_at < (SELECT created_at FROM posts WHERE id = $1) ORDER BY created_at DESC LIMIT $2'
    : 'SELECT * FROM posts ORDER BY created_at DESC LIMIT $1';

  const params = cursor ? [cursor, limit + 1] : [limit + 1];
  const result = await pool.query(query, params);

  const hasMore = result.rows.length > limit;
  const items = hasMore ? result.rows.slice(0, limit) : result.rows;

  return {
    items,
    nextCursor: hasMore ? items[items.length - 1].id : null,
  };
}
```

### Bulk Operations

```typescript
// ✅ Prisma — bulk create
await prisma.user.createMany({
  data: users, // array of user objects
  skipDuplicates: true,
});

// ✅ Prisma — bulk update with transaction
await prisma.$transaction(
  userIds.map((id) =>
    prisma.user.update({
      where: { id },
      data: { role: 'ADMIN' },
    })
  )
);

// ✅ Raw SQL — bulk insert (much faster for large datasets)
const values = users.map((u, i) =>
  `($${i * 3 + 1}, $${i * 3 + 2}, $${i * 3 + 3})`
).join(', ');

const params = users.flatMap(u => [u.id, u.name, u.email]);

await pool.query(
  `INSERT INTO users (id, name, email) VALUES ${values}
   ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name`,
  params
);
```

**Why raw SQL for bulk:** ORMs typically generate individual INSERT statements, even in
"bulk" mode. For thousands of rows, a single multi-row INSERT is orders of magnitude
faster because it's one round trip and one transaction. Use ORM for convenience on small
batches; drop to raw SQL for large data loads.

---

## 4. Indexing Strategies

### What to Index

```sql
-- Always index:
-- 1. Foreign keys (most ORMs DON'T auto-create these)
CREATE INDEX idx_posts_author_id ON posts(author_id);

-- 2. Columns in WHERE clauses used frequently
CREATE INDEX idx_users_email ON users(email);

-- 3. Columns used in ORDER BY with LIMIT
CREATE INDEX idx_posts_created_at ON posts(created_at DESC);

-- 4. Composite indexes for multi-column queries
-- Column order matters! Put equality columns first, range columns last
CREATE INDEX idx_posts_published_created
  ON posts(published, created_at DESC);
-- Supports: WHERE published = true ORDER BY created_at DESC
-- Also supports: WHERE published = true (prefix)
-- Does NOT efficiently support: ORDER BY created_at DESC alone

-- 5. Partial indexes for filtered subsets (PostgreSQL)
CREATE INDEX idx_active_users ON users(email)
  WHERE deleted_at IS NULL;
-- Smaller index, faster queries for the common case
```

### What NOT to Index

- Columns with very low cardinality (boolean with 50/50 split — full scan is faster)
- Tables with fewer than ~1000 rows (full scan is faster than index lookup)
- Columns that are written frequently but read rarely (index maintenance cost)
- Every column "just in case" (each index slows down writes)

### Monitoring Index Usage (PostgreSQL)

```sql
-- Find unused indexes
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- Find slow queries that might need indexes
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Check if a query uses an index
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
```

### Index Types (PostgreSQL)

```sql
-- B-tree (default) — equality, range, sorting
CREATE INDEX idx_users_name ON users(name);

-- Hash — equality only, slightly faster than B-tree for exact matches
CREATE INDEX idx_users_email_hash ON users USING hash(email);

-- GIN — full-text search, JSONB, arrays
CREATE INDEX idx_posts_content_gin ON posts USING gin(to_tsvector('english', content));
CREATE INDEX idx_users_metadata ON users USING gin(metadata); -- JSONB column

-- GiST — geometric, range types, full-text (alternative to GIN)
CREATE INDEX idx_events_time_range ON events USING gist(time_range);
```

---

## 5. Transactions & Consistency

### When to Use Transactions

Use transactions when multiple operations must succeed or fail together. Classic examples:
transferring money (debit + credit), creating an order (order + order items + inventory
update), or any multi-table update where partial completion would leave data inconsistent.

### Prisma Transactions

```typescript
// Interactive transaction — multiple operations with logic between them
async function transferFunds(fromId: string, toId: string, amount: number) {
  await prisma.$transaction(async (tx) => {
    const sender = await tx.account.findUnique({ where: { id: fromId } });
    if (!sender || sender.balance < amount) {
      throw new InsufficientFundsError(fromId, amount);
    }

    await tx.account.update({
      where: { id: fromId },
      data: { balance: { decrement: amount } },
    });

    await tx.account.update({
      where: { id: toId },
      data: { balance: { increment: amount } },
    });

    await tx.transactionLog.create({
      data: { fromId, toId, amount, type: 'TRANSFER' },
    });
  }, {
    isolationLevel: 'Serializable', // strongest isolation
    timeout: 10_000, // 10s max
  });
}

// Batch transaction — atomic array of operations (simpler, no logic between)
await prisma.$transaction([
  prisma.post.update({ where: { id: postId }, data: { published: true } }),
  prisma.auditLog.create({ data: { action: 'PUBLISH', postId } }),
]);
```

### Raw Driver Transactions

```typescript
// pg — manual transaction control
async function createOrder(userId: string, items: OrderItem[]) {
  const client = await pool.connect();

  try {
    await client.query('BEGIN');

    // Set isolation level if needed
    await client.query('SET TRANSACTION ISOLATION LEVEL READ COMMITTED');

    const orderResult = await client.query<{ id: string }>(
      'INSERT INTO orders (user_id, status) VALUES ($1, $2) RETURNING id',
      [userId, 'pending']
    );
    const orderId = orderResult.rows[0].id;

    for (const item of items) {
      // Check stock with row-level lock
      const stockResult = await client.query(
        'SELECT quantity FROM inventory WHERE product_id = $1 FOR UPDATE',
        [item.productId]
      );

      if (!stockResult.rows[0] || stockResult.rows[0].quantity < item.quantity) {
        throw new OutOfStockError(item.productId);
      }

      await client.query(
        'INSERT INTO order_items (order_id, product_id, quantity, price) VALUES ($1, $2, $3, $4)',
        [orderId, item.productId, item.quantity, item.price]
      );

      await client.query(
        'UPDATE inventory SET quantity = quantity - $1 WHERE product_id = $2',
        [item.quantity, item.productId]
      );
    }

    await client.query('COMMIT');
    return orderId;
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

### Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Use When |
|-------|-----------|-------------------|-------------|----------|
| READ UNCOMMITTED | Yes | Yes | Yes | Never in practice |
| READ COMMITTED | No | Yes | Yes | Default for most queries |
| REPEATABLE READ | No | No | Yes (PG: No) | Reports, analytics |
| SERIALIZABLE | No | No | No | Financial transactions |

**Why this matters:** The default (`READ COMMITTED`) is fine for most operations. Use
`SERIALIZABLE` only when correctness is critical (money transfers, inventory), because
it locks more aggressively and can cause serialization failures that require retrying.

---

## 6. Migrations

### Prisma Migrations

```bash
# Create migration from schema changes
npx prisma migrate dev --name add_user_roles

# Apply migrations in production
npx prisma migrate deploy

# Reset database (development only!)
npx prisma migrate reset
```

### Drizzle Migrations

```bash
# Generate migration
npx drizzle-kit generate

# Apply migration
npx drizzle-kit migrate

# Push schema directly (development only)
npx drizzle-kit push
```

### Migration Best Practices

```sql
-- ✅ Always make migrations backward-compatible when possible

-- Adding a column: safe (nullable or with default)
ALTER TABLE users ADD COLUMN bio TEXT;
ALTER TABLE users ADD COLUMN status VARCHAR(20) DEFAULT 'active';

-- Renaming a column: dangerous! Do it in steps:
-- Step 1: Add new column
ALTER TABLE users ADD COLUMN full_name VARCHAR(255);
-- Step 2: Copy data
UPDATE users SET full_name = name;
-- Step 3: Deploy app reading from both columns
-- Step 4: Remove old column (after all app instances updated)
ALTER TABLE users DROP COLUMN name;

-- Adding an index: use CONCURRENTLY (PostgreSQL) to avoid locking
CREATE INDEX CONCURRENTLY idx_users_status ON users(status);

-- Dropping a table: never do it in the same deploy as the code change
-- Step 1: Deploy code that no longer uses the table
-- Step 2: Drop the table in a separate migration
```

**Why staged migrations:** In production, you typically have multiple app instances
running during a deploy (rolling update). If you rename a column and deploy new code
simultaneously, old instances will fail when they try to query the old column name.
Staged migrations ensure every version of the code can work with the current schema.

---

## 7. Security

### Parameterized Queries — Always

```typescript
// ❌ NEVER — SQL injection
const query = `SELECT * FROM users WHERE name = '${name}'`;

// ✅ pg — numbered parameters
await pool.query('SELECT * FROM users WHERE name = $1 AND role = $2', [name, role]);

// ✅ mysql2 — positional parameters
await pool.execute('SELECT * FROM users WHERE name = ? AND role = ?', [name, role]);

// ✅ ORMs handle this automatically (but beware of raw query methods)
await prisma.$queryRaw`SELECT * FROM users WHERE name = ${name}`;
// Prisma's tagged template literal auto-parameterizes

// ❌ Prisma — NOT safe with $queryRawUnsafe
await prisma.$queryRawUnsafe(`SELECT * FROM users WHERE name = '${name}'`);
```

### Row-Level Security (PostgreSQL)

```sql
-- Enable RLS on a table
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;

-- Users can only see their own posts
CREATE POLICY user_posts ON posts
  FOR ALL
  USING (author_id = current_setting('app.current_user_id'));

-- Set the user context in your application
await client.query("SET app.current_user_id = $1", [userId]);
```

### Encryption at Rest

```typescript
// Encrypt sensitive columns before storing
import { createCipheriv, createDecipheriv, randomBytes } from 'node:crypto';

const ALGORITHM = 'aes-256-gcm';
const KEY = Buffer.from(process.env.ENCRYPTION_KEY!, 'hex'); // 32 bytes

function encrypt(text: string): string {
  const iv = randomBytes(16);
  const cipher = createCipheriv(ALGORITHM, KEY, iv);
  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  const authTag = cipher.getAuthTag().toString('hex');
  return `${iv.toString('hex')}:${authTag}:${encrypted}`;
}

function decrypt(data: string): string {
  const [ivHex, authTagHex, encrypted] = data.split(':');
  const iv = Buffer.from(ivHex, 'hex');
  const authTag = Buffer.from(authTagHex, 'hex');
  const decipher = createDecipheriv(ALGORITHM, KEY, iv);
  decipher.setAuthTag(authTag);
  let decrypted = decipher.update(encrypted, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  return decrypted;
}
```

### Connection Security

```typescript
// Always use SSL in production
const pool = new pg.Pool({
  ssl: process.env.NODE_ENV === 'production'
    ? { rejectUnauthorized: true, ca: process.env.DB_CA_CERT }
    : false,
});
```

---

## 8. Error Handling & Retry Logic

### Database-Specific Error Handling

```typescript
import { Prisma } from '@prisma/client';

async function createUser(data: CreateUserInput): Promise<User> {
  try {
    return await prisma.user.create({ data });
  } catch (error) {
    if (error instanceof Prisma.PrismaClientKnownRequestError) {
      switch (error.code) {
        case 'P2002': // Unique constraint violation
          const field = (error.meta?.target as string[])?.[0];
          throw new ConflictError(`User with this ${field} already exists`);
        case 'P2025': // Record not found
          throw new NotFoundError('User', data.id ?? 'unknown');
        case 'P2003': // Foreign key constraint
          throw new ValidationError('Referenced record does not exist');
        default:
          throw new DatabaseError(`Database error: ${error.code}`, error);
      }
    }
    throw error; // Re-throw unknown errors
  }
}

// pg — error codes
async function createUserRaw(data: CreateUserInput): Promise<User> {
  try {
    const result = await pool.query(/* ... */);
    return result.rows[0];
  } catch (error: unknown) {
    if (error && typeof error === 'object' && 'code' in error) {
      const pgError = error as { code: string; constraint?: string };
      switch (pgError.code) {
        case '23505': // unique_violation
          throw new ConflictError(`Duplicate entry: ${pgError.constraint}`);
        case '23503': // foreign_key_violation
          throw new ValidationError('Referenced record does not exist');
        case '23502': // not_null_violation
          throw new ValidationError('Required field is missing');
        case '57014': // query_canceled (timeout)
          throw new TimeoutError('Query timed out');
      }
    }
    throw error;
  }
}
```

### Retry with Backoff for Transient Failures

```typescript
async function withRetry<T>(
  operation: () => Promise<T>,
  options: {
    maxRetries?: number;
    baseDelayMs?: number;
    retryableCodes?: string[];
  } = {}
): Promise<T> {
  const {
    maxRetries = 3,
    baseDelayMs = 100,
    retryableCodes = [
      '40001', // serialization_failure
      '40P01', // deadlock_detected
      '08006', // connection_failure
      '08001', // unable to connect
      '57P03', // cannot_connect_now
    ],
  } = options;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error: unknown) {
      const isRetryable = error && typeof error === 'object' && 'code' in error
        && retryableCodes.includes((error as { code: string }).code);

      if (!isRetryable || attempt === maxRetries) throw error;

      const delay = baseDelayMs * 2 ** attempt * (0.5 + Math.random() * 0.5);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }

  throw new Error('Unreachable');
}

// Usage
const user = await withRetry(() =>
  prisma.$transaction(async (tx) => {
    // transaction that might hit serialization failure
  })
);
```

### Connection Error Recovery

```typescript
// pg pool automatically handles connection drops and reconnects.
// But you should monitor for persistent failures:

let consecutiveFailures = 0;
const MAX_CONSECUTIVE_FAILURES = 5;

pool.on('error', (err) => {
  consecutiveFailures++;
  logger.error('Pool connection error', {
    error: err,
    consecutiveFailures,
  });

  if (consecutiveFailures >= MAX_CONSECUTIVE_FAILURES) {
    logger.fatal('Too many consecutive database failures, shutting down');
    process.exit(1);
  }
});

// Reset counter on successful query
const originalQuery = pool.query.bind(pool);
pool.query = async (...args: Parameters<typeof pool.query>) => {
  const result = await originalQuery(...args);
  consecutiveFailures = 0;
  return result;
};
```

---

## 9. Testing

### Test Database Setup

```typescript
// Use a separate database for tests
// .env.test
// DATABASE_URL="postgresql://localhost:5432/myapp_test"

// Setup/teardown
import { PrismaClient } from '@prisma/client';
import { execSync } from 'node:child_process';

const prisma = new PrismaClient();

beforeAll(async () => {
  // Apply migrations to test database
  execSync('npx prisma migrate deploy', {
    env: { ...process.env, DATABASE_URL: process.env.TEST_DATABASE_URL },
  });
});

afterEach(async () => {
  // Clean all tables between tests (order matters for FK constraints)
  await prisma.$transaction([
    prisma.postTag.deleteMany(),
    prisma.post.deleteMany(),
    prisma.user.deleteMany(),
  ]);
});

afterAll(async () => {
  await prisma.$disconnect();
});
```

### Test Factories

```typescript
// factories.ts — create realistic test data
import { faker } from '@faker-js/faker';

export function buildUser(overrides: Partial<CreateUserInput> = {}): CreateUserInput {
  return {
    name: faker.person.fullName(),
    email: faker.internet.email(),
    role: 'USER',
    ...overrides,
  };
}

export function buildPost(overrides: Partial<CreatePostInput> = {}): CreatePostInput {
  return {
    title: faker.lorem.sentence(),
    content: faker.lorem.paragraphs(3),
    published: false,
    ...overrides,
  };
}

// Usage in tests
it('should create a user', async () => {
  const input = buildUser({ role: 'ADMIN' });
  const user = await userService.create(input);
  expect(user.role).toBe('ADMIN');
});
```

### Testing Repository Layer

```typescript
describe('UserRepository', () => {
  it('should find user by email', async () => {
    // Arrange
    const input = buildUser({ email: 'test@example.com' });
    await prisma.user.create({ data: input });

    // Act
    const found = await userRepo.findByEmail('test@example.com');

    // Assert
    expect(found).not.toBeNull();
    expect(found!.email).toBe('test@example.com');
  });

  it('should return null for non-existent email', async () => {
    const found = await userRepo.findByEmail('nonexistent@example.com');
    expect(found).toBeNull();
  });

  it('should throw ConflictError on duplicate email', async () => {
    const input = buildUser({ email: 'dupe@example.com' });
    await prisma.user.create({ data: input });

    await expect(userRepo.create(input))
      .rejects.toThrow(ConflictError);
  });
});
```

### Mocking for Unit Tests

```typescript
// When testing services, mock the repository
import { vi } from 'vitest';

const mockUserRepo: UserRepository = {
  findById: vi.fn(),
  findByEmail: vi.fn(),
  create: vi.fn(),
  update: vi.fn(),
  delete: vi.fn(),
};

describe('UserService', () => {
  const service = new UserService(mockUserRepo);

  it('should throw NotFoundError when user does not exist', async () => {
    vi.mocked(mockUserRepo.findById).mockResolvedValue(null);

    await expect(service.getUser('nonexistent'))
      .rejects.toThrow(NotFoundError);
  });
});
```
