# NoSQL Databases — MongoDB & DynamoDB

Best practices for working with document and key-value databases in Node.js/TypeScript.
Covers both ODM/ORM (Mongoose, Dynamoose) and raw driver (mongodb, @aws-sdk/client-dynamodb) patterns.

---

## Table of Contents

1. [Connection Management](#1-connection-management)
2. [Schema Design & Data Modeling](#2-schema-design--data-modeling)
3. [Query Patterns & Optimization](#3-query-patterns--optimization)
4. [Indexing Strategies](#4-indexing-strategies)
5. [Transactions & Consistency](#5-transactions--consistency)
6. [Migrations & Schema Evolution](#6-migrations--schema-evolution)
7. [Security](#7-security)
8. [Error Handling & Retry Logic](#8-error-handling--retry-logic)
9. [Testing](#9-testing)

---

## 1. Connection Management

### MongoDB — Native Driver

```typescript
import { MongoClient, Db } from 'mongodb';

let client: MongoClient | null = null;
let db: Db | null = null;

async function connectMongo(): Promise<Db> {
  if (db) return db;

  client = new MongoClient(process.env.MONGODB_URI!, {
    maxPoolSize: 20,           // Max connections in pool
    minPoolSize: 2,            // Keep warm connections
    maxIdleTimeMS: 30_000,     // Close idle connections after 30s
    connectTimeoutMS: 5_000,   // Fail fast on connection
    serverSelectionTimeoutMS: 5_000,
    socketTimeoutMS: 30_000,   // Query timeout
    retryWrites: true,         // Auto-retry transient write failures
    retryReads: true,
    w: 'majority',             // Write concern — wait for majority
    readPreference: 'primaryPreferred', // Read from primary, fallback to secondary
  });

  await client.connect();

  // Verify connection
  await client.db('admin').command({ ping: 1 });
  logger.info('MongoDB connected');

  db = client.db(process.env.MONGODB_DB_NAME);
  return db;
}

async function closeMongo(): Promise<void> {
  if (client) {
    await client.close();
    client = null;
    db = null;
    logger.info('MongoDB disconnected');
  }
}
```

**Why `w: 'majority'`:** Without it, MongoDB acknowledges writes after writing to the
primary only. If the primary crashes before replicating, data is lost. `majority` waits
until a majority of replica set members confirm the write, ensuring durability.
`retryWrites` handles transient network blips automatically.

### MongoDB — Mongoose (ODM)

```typescript
import mongoose from 'mongoose';

async function connectMongoose(): Promise<void> {
  mongoose.set('strictQuery', true); // Only allow querying schema-defined fields

  await mongoose.connect(process.env.MONGODB_URI!, {
    maxPoolSize: 20,
    minPoolSize: 2,
    serverSelectionTimeoutMS: 5_000,
    socketTimeoutMS: 30_000,
  });

  mongoose.connection.on('error', (err) => {
    logger.error('Mongoose connection error', { error: err });
  });

  mongoose.connection.on('disconnected', () => {
    logger.warn('Mongoose disconnected');
  });

  logger.info('Mongoose connected');
}

async function closeMongoose(): Promise<void> {
  await mongoose.disconnect();
}
```

### DynamoDB — AWS SDK v3

```typescript
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb';

// Document client wraps the base client with marshalling/unmarshalling
const baseClient = new DynamoDBClient({
  region: process.env.AWS_REGION || 'us-east-1',

  // Local development with DynamoDB Local
  ...(process.env.NODE_ENV === 'development' && {
    endpoint: 'http://localhost:8000',
    credentials: {
      accessKeyId: 'local',
      secretAccessKey: 'local',
    },
  }),

  // Retry configuration
  maxAttempts: 3,
});

const dynamodb = DynamoDBDocumentClient.from(baseClient, {
  marshallOptions: {
    removeUndefinedValues: true, // Don't store undefined as null
    convertClassInstanceToMap: true,
  },
  unmarshallOptions: {
    wrapNumbers: false, // Return numbers as JS numbers (not strings)
  },
});

export { dynamodb };
```

**Why DynamoDB Document Client:** The base DynamoDB client requires you to manually
specify type descriptors (`{ S: 'string' }`, `{ N: '123' }`). The Document Client
handles marshalling/unmarshalling automatically, letting you work with plain JavaScript
objects. Always use the Document Client.

---

## 2. Schema Design & Data Modeling

### MongoDB — Design for Access Patterns

```typescript
// MongoDB encourages embedding related data when:
// 1. Data is always read together
// 2. The embedded array is bounded (won't grow infinitely)
// 3. Updates are infrequent

// ✅ Good — address is always read with user, bounded to a few entries
interface User {
  _id: ObjectId;
  name: string;
  email: string;
  addresses: Array<{
    label: string;      // 'home', 'work'
    street: string;
    city: string;
    country: string;
  }>;
  createdAt: Date;
  updatedAt: Date;
}

// ✅ Good — comments are separate when they can grow unboundedly
interface Post {
  _id: ObjectId;
  title: string;
  content: string;
  authorId: ObjectId;   // Reference to users collection
  tags: string[];       // Bounded, queried frequently
  likeCount: number;    // Denormalized counter (updated atomically)
  createdAt: Date;
}

// Comments in their own collection — can grow unboundedly
interface Comment {
  _id: ObjectId;
  postId: ObjectId;     // Reference to posts
  authorId: ObjectId;   // Reference to users
  content: string;
  createdAt: Date;
}
```

### Mongoose Schemas

```typescript
import { Schema, model, Types } from 'mongoose';

const userSchema = new Schema({
  name: {
    type: String,
    required: [true, 'Name is required'],
    trim: true,
    maxlength: [200, 'Name cannot exceed 200 characters'],
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true,
    validate: {
      validator: (v: string) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v),
      message: 'Invalid email format',
    },
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'moderator'],
    default: 'user',
  },
  addresses: [{
    label: { type: String, required: true },
    street: { type: String, required: true },
    city: { type: String, required: true },
    country: { type: String, required: true },
  }],
}, {
  timestamps: true,       // Auto-manage createdAt/updatedAt
  toJSON: {
    transform: (_doc, ret) => {
      ret.id = ret._id.toString();
      delete ret._id;
      delete ret.__v;
      return ret;
    },
  },
});

// Indexes
userSchema.index({ email: 1 }, { unique: true });
userSchema.index({ role: 1 });
userSchema.index({ createdAt: -1 });

export const UserModel = model('User', userSchema);
```

### DynamoDB — Single Table Design

```typescript
// DynamoDB best practice: store multiple entity types in one table
// Design access patterns FIRST, then model the data

// Access patterns:
// 1. Get user by ID
// 2. Get all posts by user
// 3. Get post by ID
// 4. Get comments on a post

// Single table with composite keys:
// PK (Partition Key) | SK (Sort Key) | Type | Data...
// USER#123          | PROFILE       | User | { name, email }
// USER#123          | POST#456      | Post | { title, content }
// POST#456          | COMMENT#789   | Comment | { content, authorId }
// POST#456          | META          | Post | { title, likeCount }

interface DynamoDBItem {
  PK: string;
  SK: string;
  Type: string;
  GSI1PK?: string;  // Global Secondary Index
  GSI1SK?: string;
  [key: string]: unknown;
}

// Helper to build keys
const keys = {
  user: {
    profile: (userId: string) => ({ PK: `USER#${userId}`, SK: 'PROFILE' }),
    posts: (userId: string) => ({ PK: `USER#${userId}`, SK_prefix: 'POST#' }),
  },
  post: {
    meta: (postId: string) => ({ PK: `POST#${postId}`, SK: 'META' }),
    comments: (postId: string) => ({ PK: `POST#${postId}`, SK_prefix: 'COMMENT#' }),
  },
};
```

**Why single table design:** DynamoDB doesn't support JOINs. If related data is in
separate tables, fetching it requires multiple requests. Single table design co-locates
related data on the same partition key, enabling a single `Query` to fetch a user and
all their posts. The trade-off is that the data model is harder to understand — but
the performance and cost benefits are substantial.

### DynamoDB Operations

```typescript
import { PutCommand, GetCommand, QueryCommand, UpdateCommand } from '@aws-sdk/lib-dynamodb';

const TABLE_NAME = process.env.DYNAMODB_TABLE!;

// Create
async function createUser(user: User): Promise<void> {
  await dynamodb.send(new PutCommand({
    TableName: TABLE_NAME,
    Item: {
      PK: `USER#${user.id}`,
      SK: 'PROFILE',
      Type: 'User',
      ...user,
      createdAt: new Date().toISOString(),
    },
    ConditionExpression: 'attribute_not_exists(PK)', // Prevent overwrite
  }));
}

// Read
async function getUser(userId: string): Promise<User | null> {
  const result = await dynamodb.send(new GetCommand({
    TableName: TABLE_NAME,
    Key: { PK: `USER#${userId}`, SK: 'PROFILE' },
  }));
  return (result.Item as User) ?? null;
}

// Query — get all posts by a user
async function getUserPosts(userId: string): Promise<Post[]> {
  const result = await dynamodb.send(new QueryCommand({
    TableName: TABLE_NAME,
    KeyConditionExpression: 'PK = :pk AND begins_with(SK, :sk)',
    ExpressionAttributeValues: {
      ':pk': `USER#${userId}`,
      ':sk': 'POST#',
    },
    ScanIndexForward: false, // newest first
  }));
  return (result.Items as Post[]) ?? [];
}

// Update — atomic counter increment
async function incrementLikeCount(postId: string): Promise<number> {
  const result = await dynamodb.send(new UpdateCommand({
    TableName: TABLE_NAME,
    Key: { PK: `POST#${postId}`, SK: 'META' },
    UpdateExpression: 'SET likeCount = if_not_exists(likeCount, :zero) + :inc',
    ExpressionAttributeValues: { ':inc': 1, ':zero': 0 },
    ReturnValues: 'UPDATED_NEW',
  }));
  return result.Attributes?.likeCount as number;
}
```

---

## 3. Query Patterns & Optimization

### MongoDB — Aggregation Pipeline

```typescript
// The aggregation pipeline is MongoDB's most powerful query tool.
// Use it instead of fetching data and processing in Node.js.

// Get top 10 authors by post count with their latest post
const topAuthors = await db.collection('posts').aggregate([
  // Stage 1: Only published posts
  { $match: { published: true } },

  // Stage 2: Sort by date (for $first in group)
  { $sort: { createdAt: -1 } },

  // Stage 3: Group by author
  { $group: {
    _id: '$authorId',
    postCount: { $sum: 1 },
    latestPostTitle: { $first: '$title' },
    latestPostDate: { $first: '$createdAt' },
  }},

  // Stage 4: Sort by post count
  { $sort: { postCount: -1 } },

  // Stage 5: Top 10
  { $limit: 10 },

  // Stage 6: Join with users collection
  { $lookup: {
    from: 'users',
    localField: '_id',
    foreignField: '_id',
    as: 'author',
  }},

  // Stage 7: Flatten the author array
  { $unwind: '$author' },

  // Stage 8: Shape the output
  { $project: {
    _id: 0,
    authorName: '$author.name',
    authorEmail: '$author.email',
    postCount: 1,
    latestPostTitle: 1,
    latestPostDate: 1,
  }},
]).toArray();
```

### MongoDB — Lean Queries (Mongoose)

```typescript
// ❌ Slow — Mongoose hydrates full document instances with methods
const users = await UserModel.find({ role: 'admin' });
// Each user is a full Mongoose document with save(), validate(), etc.

// ✅ Fast — lean() returns plain JavaScript objects
const users = await UserModel.find({ role: 'admin' })
  .select('name email')
  .lean();
// Returns plain objects — 2-5x faster for read-only operations
```

**Why lean:** Mongoose documents are heavy objects with change tracking, validation methods,
and prototype chains. For read-only queries (API responses, reports, exports), this overhead
is wasted. `.lean()` returns plain objects, significantly reducing memory and CPU usage.

### DynamoDB — Query vs Scan

```typescript
// ✅ Query — efficient, uses index, reads only matching items
const result = await dynamodb.send(new QueryCommand({
  TableName: TABLE_NAME,
  KeyConditionExpression: 'PK = :pk',
  ExpressionAttributeValues: { ':pk': `USER#${userId}` },
}));

// ❌ Scan — reads EVERY item in the table, then filters
// Only use for admin tools, data migrations, or very small tables
const result = await dynamodb.send(new ScanCommand({
  TableName: TABLE_NAME,
  FilterExpression: '#type = :type',
  ExpressionAttributeNames: { '#type': 'Type' },
  ExpressionAttributeValues: { ':type': 'User' },
}));
```

**Why never Scan in production:** DynamoDB charges per read capacity unit consumed. A Scan
reads every item in the table (consuming RCU for each), then filters client-side. For a
table with 1M items, a Scan reads all 1M even if only 10 match your filter. A Query with
the right key structure reads exactly the items you need.

### DynamoDB — Pagination

```typescript
// DynamoDB returns max 1MB per request — you must paginate
async function getAllUserPosts(userId: string): Promise<Post[]> {
  const allItems: Post[] = [];
  let lastKey: Record<string, unknown> | undefined;

  do {
    const result = await dynamodb.send(new QueryCommand({
      TableName: TABLE_NAME,
      KeyConditionExpression: 'PK = :pk AND begins_with(SK, :sk)',
      ExpressionAttributeValues: {
        ':pk': `USER#${userId}`,
        ':sk': 'POST#',
      },
      ExclusiveStartKey: lastKey,
      Limit: 100, // items per page
    }));

    allItems.push(...(result.Items as Post[]) ?? []);
    lastKey = result.LastEvaluatedKey;
  } while (lastKey);

  return allItems;
}
```

---

## 4. Indexing Strategies

### MongoDB Indexes

```typescript
// Create indexes in your schema or migration scripts

// Single field — for exact match and range queries
await db.collection('users').createIndex({ email: 1 }, { unique: true });

// Compound index — order matters!
// Supports: { status: X }, { status: X, createdAt: Y }, but NOT { createdAt: Y } alone
await db.collection('posts').createIndex({ status: 1, createdAt: -1 });

// Text index — for full-text search
await db.collection('posts').createIndex(
  { title: 'text', content: 'text' },
  { weights: { title: 10, content: 1 } } // title matches ranked higher
);

// TTL index — auto-delete expired documents
await db.collection('sessions').createIndex(
  { expiresAt: 1 },
  { expireAfterSeconds: 0 } // delete when expiresAt is in the past
);

// Partial index — only index documents matching a filter
await db.collection('posts').createIndex(
  { authorId: 1, createdAt: -1 },
  { partialFilterExpression: { published: true } }
);
// Smaller index, only covers published posts
```

### MongoDB — Explain Queries

```typescript
// Check if a query uses an index
const explanation = await db.collection('posts')
  .find({ authorId: userId, published: true })
  .explain('executionStats');

// Look for:
// - winningPlan.stage should be "IXSCAN" (not "COLLSCAN")
// - executionStats.totalDocsExamined should be close to nReturned
// - If totalDocsExamined >> nReturned, you need a better index
```

### DynamoDB — GSI (Global Secondary Index)

```typescript
// DynamoDB: you can only Query by partition key + sort key.
// GSI lets you query by different attributes.

// Example: Query users by email (email isn't the partition key)
// GSI1: PK=email, SK=Type

// Create via CloudFormation / CDK / Console, then query:
async function getUserByEmail(email: string): Promise<User | null> {
  const result = await dynamodb.send(new QueryCommand({
    TableName: TABLE_NAME,
    IndexName: 'GSI1',
    KeyConditionExpression: 'GSI1PK = :email',
    ExpressionAttributeValues: { ':email': email },
  }));
  return (result.Items?.[0] as User) ?? null;
}

// When creating items, include GSI keys:
await dynamodb.send(new PutCommand({
  TableName: TABLE_NAME,
  Item: {
    PK: `USER#${user.id}`,
    SK: 'PROFILE',
    GSI1PK: user.email,     // Enable email lookups
    GSI1SK: 'User',
    Type: 'User',
    ...user,
  },
}));
```

**DynamoDB indexing rules:**
- Design GSIs for your access patterns BEFORE writing code
- Max 20 GSIs per table (soft limit)
- GSI writes consume additional WCU — don't create GSIs you don't need
- GSI is eventually consistent by default (can't do strongly consistent reads)

---

## 5. Transactions & Consistency

### MongoDB Transactions

```typescript
// MongoDB supports multi-document transactions (since 4.0)
async function transferCredits(fromId: string, toId: string, amount: number) {
  const session = client.startSession();

  try {
    await session.withTransaction(async () => {
      const sender = await db.collection('accounts').findOne(
        { _id: new ObjectId(fromId) },
        { session }
      );

      if (!sender || sender.balance < amount) {
        throw new InsufficientFundsError(fromId, amount);
      }

      await db.collection('accounts').updateOne(
        { _id: new ObjectId(fromId) },
        { $inc: { balance: -amount } },
        { session }
      );

      await db.collection('accounts').updateOne(
        { _id: new ObjectId(toId) },
        { $inc: { balance: amount } },
        { session }
      );

      await db.collection('transactions').insertOne({
        fromId, toId, amount, createdAt: new Date(),
      }, { session });
    }, {
      readConcern: { level: 'snapshot' },
      writeConcern: { w: 'majority' },
    });
  } finally {
    await session.endSession();
  }
}
```

**When NOT to use MongoDB transactions:** If you can model the operation as a single
document update, do that instead. MongoDB atomic operations on a single document
(`$inc`, `$push`, `$set`) are faster and simpler than transactions. Transactions add
latency and lock overhead. Use them only when you truly need multi-document atomicity.

### DynamoDB Transactions

```typescript
import { TransactWriteCommand, TransactGetCommand } from '@aws-sdk/lib-dynamodb';

// DynamoDB supports up to 100 items per transaction
async function transferCredits(fromId: string, toId: string, amount: number) {
  await dynamodb.send(new TransactWriteCommand({
    TransactItems: [
      {
        Update: {
          TableName: TABLE_NAME,
          Key: { PK: `USER#${fromId}`, SK: 'PROFILE' },
          UpdateExpression: 'SET balance = balance - :amount',
          ConditionExpression: 'balance >= :amount', // Prevent negative balance
          ExpressionAttributeValues: { ':amount': amount },
        },
      },
      {
        Update: {
          TableName: TABLE_NAME,
          Key: { PK: `USER#${toId}`, SK: 'PROFILE' },
          UpdateExpression: 'SET balance = balance + :amount',
          ExpressionAttributeValues: { ':amount': amount },
        },
      },
      {
        Put: {
          TableName: TABLE_NAME,
          Item: {
            PK: `TXN#${Date.now()}`,
            SK: `${fromId}#${toId}`,
            Type: 'Transaction',
            amount,
            createdAt: new Date().toISOString(),
          },
        },
      },
    ],
  }));
}
```

### DynamoDB — Optimistic Locking

```typescript
// Use a version number to prevent lost updates
async function updateUserProfile(userId: string, updates: Partial<User>) {
  const current = await getUser(userId);
  if (!current) throw new NotFoundError('User', userId);

  await dynamodb.send(new UpdateCommand({
    TableName: TABLE_NAME,
    Key: { PK: `USER#${userId}`, SK: 'PROFILE' },
    UpdateExpression: 'SET #name = :name, version = :newVersion',
    ConditionExpression: 'version = :currentVersion', // Fails if someone else updated
    ExpressionAttributeNames: { '#name': 'name' },
    ExpressionAttributeValues: {
      ':name': updates.name,
      ':currentVersion': current.version,
      ':newVersion': current.version + 1,
    },
  }));
  // If another update happened, this throws ConditionalCheckFailedException
  // → retry with fresh data
}
```

---

## 6. Migrations & Schema Evolution

### MongoDB — Schema Versioning

```typescript
// MongoDB is schemaless, but you still need to manage schema evolution

// Strategy 1: Version field + lazy migration
interface UserV1 {
  _id: ObjectId;
  name: string;       // Full name as single field
  schemaVersion: 1;
}

interface UserV2 {
  _id: ObjectId;
  firstName: string;  // Split into first/last
  lastName: string;
  schemaVersion: 2;
}

type User = UserV1 | UserV2;

// Read function handles both versions
function normalizeUser(doc: User): NormalizedUser {
  if (doc.schemaVersion === 1) {
    const [firstName, ...rest] = doc.name.split(' ');
    return { firstName, lastName: rest.join(' ') || '' };
  }
  return { firstName: doc.firstName, lastName: doc.lastName };
}

// Lazy migration: update documents when they're read
async function getUser(id: string): Promise<NormalizedUser | null> {
  const doc = await db.collection('users').findOne({ _id: new ObjectId(id) });
  if (!doc) return null;

  if (doc.schemaVersion < 2) {
    const normalized = normalizeUser(doc as User);
    await db.collection('users').updateOne(
      { _id: doc._id },
      { $set: { ...normalized, schemaVersion: 2 }, $unset: { name: '' } }
    );
  }

  return normalizeUser(doc as User);
}

// Strategy 2: Batch migration script
async function migrateUsersToV2(): Promise<void> {
  const cursor = db.collection('users').find({ schemaVersion: { $lt: 2 } });

  let batch: AnyBulkWriteOperation[] = [];

  for await (const doc of cursor) {
    const [firstName, ...rest] = doc.name.split(' ');
    batch.push({
      updateOne: {
        filter: { _id: doc._id },
        update: {
          $set: { firstName, lastName: rest.join(' ') || '', schemaVersion: 2 },
          $unset: { name: '' },
        },
      },
    });

    if (batch.length >= 500) {
      await db.collection('users').bulkWrite(batch);
      batch = [];
    }
  }

  if (batch.length > 0) {
    await db.collection('users').bulkWrite(batch);
  }
}
```

### DynamoDB — Migration Patterns

DynamoDB doesn't have ALTER TABLE. Schema changes happen at the application level:

- **Adding fields:** Just start writing them — existing items won't have them, handle `undefined`
- **Renaming fields:** Write both old and new field, then stop writing old field after all readers updated
- **Adding a GSI:** Can be done live, but takes time to backfill — plan for it
- **Changing key structure:** Requires creating a new table and migrating data

---

## 7. Security

### MongoDB Security

```typescript
// Always enable authentication in production
// mongodb://user:password@host:27017/dbname?authSource=admin

// Use specific database users with minimal permissions
// db.createUser({
//   user: "app_user",
//   pwd: "secure_password",
//   roles: [{ role: "readWrite", db: "myapp" }]
// });

// ❌ Never pass user input to MongoDB operators
const query = { $where: `this.name == '${userInput}'` }; // Code injection!

// ❌ NoSQL injection via object
// If userInput is { "$gt": "" } instead of a string:
const query = { password: userInput }; // Matches ALL documents!

// ✅ Always validate and sanitize
import { z } from 'zod';

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

// ✅ Use type-safe query builders or explicit operators
const user = await db.collection('users').findOne({
  email: { $eq: validatedEmail }, // Explicit $eq prevents operator injection
});
```

### DynamoDB Security

```typescript
// Use IAM roles (not access keys) in production
// Principle of least privilege — only the permissions the app needs

// IAM policy example (CloudFormation):
// {
//   "Effect": "Allow",
//   "Action": [
//     "dynamodb:GetItem",
//     "dynamodb:PutItem",
//     "dynamodb:UpdateItem",
//     "dynamodb:Query"
//   ],
//   "Resource": "arn:aws:dynamodb:us-east-1:123456789:table/MyTable"
// }

// Never allow dynamodb:Scan or dynamodb:DeleteTable in app IAM role

// Enable encryption at rest (default since 2018, but verify)
// Enable Point-in-Time Recovery for disaster recovery
```

---

## 8. Error Handling & Retry Logic

### MongoDB Error Handling

```typescript
import { MongoServerError, MongoNetworkError } from 'mongodb';

async function createUser(data: CreateUserInput): Promise<User> {
  try {
    const result = await db.collection('users').insertOne(data);
    return { ...data, _id: result.insertedId };
  } catch (error) {
    if (error instanceof MongoServerError) {
      if (error.code === 11000) { // Duplicate key
        const field = Object.keys(error.keyPattern || {})[0];
        throw new ConflictError(`User with this ${field} already exists`);
      }
    }
    if (error instanceof MongoNetworkError) {
      throw new DatabaseConnectionError('MongoDB connection failed', error);
    }
    throw error;
  }
}
```

### DynamoDB Error Handling

```typescript
import { ConditionalCheckFailedException, ProvisionedThroughputExceededException } from '@aws-sdk/client-dynamodb';

async function updateUser(userId: string, updates: Partial<User>): Promise<void> {
  try {
    await dynamodb.send(new UpdateCommand({ /* ... */ }));
  } catch (error) {
    if (error instanceof ConditionalCheckFailedException) {
      throw new ConflictError('User was modified by another process');
    }
    if (error instanceof ProvisionedThroughputExceededException) {
      // Retry with backoff — or let AWS SDK retry config handle it
      throw new RateLimitError('DynamoDB throughput exceeded');
    }
    throw error;
  }
}
```

### DynamoDB — Handling Throttling

```typescript
// DynamoDB throttles when you exceed provisioned capacity (or burst on on-demand)
// The SDK retries automatically, but you can customize:

const client = new DynamoDBClient({
  maxAttempts: 5, // default is 3
  retryMode: 'adaptive', // Adapts delay based on retry metadata
});

// For batch operations, handle UnprocessedItems
async function batchWriteWithRetry(items: Record<string, WriteRequest[]>) {
  let unprocessed = items;
  let attempt = 0;

  while (Object.keys(unprocessed).length > 0 && attempt < 5) {
    const result = await dynamodb.send(new BatchWriteCommand({
      RequestItems: unprocessed,
    }));

    unprocessed = result.UnprocessedItems ?? {};

    if (Object.keys(unprocessed).length > 0) {
      attempt++;
      const delay = Math.min(100 * 2 ** attempt, 5000);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }

  if (Object.keys(unprocessed).length > 0) {
    throw new Error(`Failed to write ${Object.keys(unprocessed).length} items after retries`);
  }
}
```

---

## 9. Testing

### MongoDB — In-Memory Server for Tests

```typescript
import { MongoMemoryServer } from 'mongodb-memory-server';
import { MongoClient, Db } from 'mongodb';

let mongod: MongoMemoryServer;
let client: MongoClient;
let db: Db;

beforeAll(async () => {
  mongod = await MongoMemoryServer.create();
  client = await MongoClient.connect(mongod.getUri());
  db = client.db('test');
});

afterEach(async () => {
  const collections = await db.listCollections().toArray();
  for (const { name } of collections) {
    await db.collection(name).deleteMany({});
  }
});

afterAll(async () => {
  await client.close();
  await mongod.stop();
});
```

### DynamoDB — Local Testing

```typescript
// Use DynamoDB Local (Docker) for integration tests
// docker run -p 8000:8000 amazon/dynamodb-local

import { CreateTableCommand, DeleteTableCommand } from '@aws-sdk/client-dynamodb';

const testClient = new DynamoDBClient({
  endpoint: 'http://localhost:8000',
  region: 'local',
  credentials: { accessKeyId: 'test', secretAccessKey: 'test' },
});

beforeAll(async () => {
  await testClient.send(new CreateTableCommand({
    TableName: 'test-table',
    KeySchema: [
      { AttributeName: 'PK', KeyType: 'HASH' },
      { AttributeName: 'SK', KeyType: 'RANGE' },
    ],
    AttributeDefinitions: [
      { AttributeName: 'PK', AttributeType: 'S' },
      { AttributeName: 'SK', AttributeType: 'S' },
    ],
    BillingMode: 'PAY_PER_REQUEST',
  }));
});

afterAll(async () => {
  await testClient.send(new DeleteTableCommand({ TableName: 'test-table' }));
});
```
