# Redis — Caching & Beyond

Best practices for using Redis in Node.js/TypeScript applications.
Redis serves many roles: cache, session store, rate limiter, queue, pub/sub, and more.

---

## Table of Contents

1. [Connection Management](#1-connection-management)
2. [Data Modeling & Key Design](#2-data-modeling--key-design)
3. [Caching Patterns](#3-caching-patterns)
4. [Session Management](#4-session-management)
5. [Rate Limiting](#5-rate-limiting)
6. [Queues & Job Processing](#6-queues--job-processing)
7. [Pub/Sub & Real-Time](#7-pubsub--real-time)
8. [Transactions & Atomicity](#8-transactions--atomicity)
9. [Security](#9-security)
10. [Error Handling & Resilience](#10-error-handling--resilience)
11. [Performance](#11-performance)
12. [Testing](#12-testing)

---

## 1. Connection Management

### ioredis (Recommended)

```typescript
import Redis from 'ioredis';

const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: Number(process.env.REDIS_PORT) || 6379,
  password: process.env.REDIS_PASSWORD,
  db: Number(process.env.REDIS_DB) || 0,

  // Connection resilience
  retryStrategy: (times) => {
    if (times > 10) return null; // Stop retrying after 10 attempts
    return Math.min(times * 200, 5000); // Exponential backoff, max 5s
  },
  maxRetriesPerRequest: 3,
  enableReadyCheck: true,

  lazyConnect: true, // Don't connect until first command

  // Timeouts
  connectTimeout: 5_000,
  commandTimeout: 5_000,
});

// Event handlers
redis.on('connect', () => logger.info('Redis connecting...'));
redis.on('ready', () => logger.info('Redis ready'));
redis.on('error', (err) => logger.error('Redis error', { error: err }));
redis.on('close', () => logger.warn('Redis connection closed'));
redis.on('reconnecting', (delay: number) => {
  logger.info(`Redis reconnecting in ${delay}ms`);
});

// Explicit connect (if lazyConnect)
await redis.connect();

// Graceful shutdown
async function closeRedis(): Promise<void> {
  await redis.quit(); // Graceful — waits for pending commands
  logger.info('Redis disconnected');
}
```

**Why ioredis over node-redis:** ioredis has better TypeScript support, built-in cluster
support, Sentinel support, transparent reconnection, and a more intuitive API. It also
supports pipelining and Lua scripting out of the box.

### Redis Cluster

```typescript
const cluster = new Redis.Cluster([
  { host: 'redis-node-1', port: 6379 },
  { host: 'redis-node-2', port: 6379 },
  { host: 'redis-node-3', port: 6379 },
], {
  redisOptions: {
    password: process.env.REDIS_PASSWORD,
  },
  scaleReads: 'slave', // Read from replicas for read-heavy workloads
});
```

### Redis Sentinel (High Availability)

```typescript
const redis = new Redis({
  sentinels: [
    { host: 'sentinel-1', port: 26379 },
    { host: 'sentinel-2', port: 26379 },
    { host: 'sentinel-3', port: 26379 },
  ],
  name: 'mymaster', // Sentinel master name
  password: process.env.REDIS_PASSWORD,
  sentinelPassword: process.env.REDIS_SENTINEL_PASSWORD,
});
```

**Why Sentinel:** Sentinel provides automatic failover. If the master goes down, Sentinel
promotes a replica to master and reconfigures all clients automatically. Use Sentinel for
high availability without the complexity of Redis Cluster.

---

## 2. Data Modeling & Key Design

### Key Naming Conventions

```typescript
// Use colon-separated namespaces
// Pattern: {app}:{entity}:{id}:{sub-entity}

const KEYS = {
  // User data
  user: (id: string) => `app:user:${id}`,
  userProfile: (id: string) => `app:user:${id}:profile`,
  userSessions: (id: string) => `app:user:${id}:sessions`,

  // Cache entries
  cache: (entity: string, id: string) => `cache:${entity}:${id}`,
  cacheList: (entity: string, page: number) => `cache:${entity}:list:${page}`,

  // Rate limiting
  rateLimit: (ip: string, endpoint: string) => `ratelimit:${ip}:${endpoint}`,

  // Queues
  queue: (name: string) => `queue:${name}`,
  queueProcessing: (name: string) => `queue:${name}:processing`,
  queueFailed: (name: string) => `queue:${name}:failed`,

  // Locks
  lock: (resource: string) => `lock:${resource}`,

  // Sessions
  session: (id: string) => `session:${id}`,
} as const;
```

**Why structured keys:**
- Colons are the Redis convention for namespacing (tools like Redis Commander display them as trees)
- Predictable patterns make it easy to find and manage keys
- Prefixes enable bulk operations with `SCAN` patterns
- Separate cache keys from application data so you can flush caches without losing state

### Choose the Right Data Type

| Need | Redis Type | Example |
|------|-----------|---------|
| Simple value, cache entry | String | `SET cache:user:123 '{"name":"Alice"}'` |
| Object with fields | Hash | `HSET user:123 name Alice email alice@example.com` |
| Ordered list, recent items | List | `LPUSH recent:posts post-id-456` |
| Unique collection, tags | Set | `SADD post:456:tags javascript typescript` |
| Ranked items, leaderboard | Sorted Set | `ZADD leaderboard 1500 user-123` |
| Temporary data, sessions | String + TTL | `SET session:abc123 '...' EX 3600` |
| Boolean flags | String | `SET feature:dark-mode 1` |
| Counting unique items | HyperLogLog | `PFADD daily:visitors user-123` |
| Geospatial queries | Geo | `GEOADD locations 13.361 38.116 "Palermo"` |

### Hash vs JSON String

```typescript
// Use Hash (HSET/HGET) when you read/update individual fields frequently
await redis.hset('user:123', {
  name: 'Alice',
  email: 'alice@example.com',
  role: 'admin',
  loginCount: '42',
});

const name = await redis.hget('user:123', 'name'); // Read single field
await redis.hincrby('user:123', 'loginCount', 1);  // Atomic increment

// Use String with JSON when you always read/write the full object
await redis.set(
  'cache:user:123',
  JSON.stringify({ name: 'Alice', email: 'alice@example.com', role: 'admin' }),
  'EX', 3600
);
const user = JSON.parse(await redis.get('cache:user:123') ?? 'null');
```

**Trade-off:** Hashes are more memory-efficient for small objects (Redis optimizes with
ziplist encoding) and allow partial reads/writes. JSON strings are simpler when you always
need the full object and want to deserialize to a typed object.

---

## 3. Caching Patterns

### Cache-Aside (Lazy Loading)

```typescript
// Most common pattern — check cache first, fall back to database
async function getUser(id: string): Promise<User> {
  const cacheKey = KEYS.cache('user', id);

  // 1. Try cache
  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached) as User;
  }

  // 2. Cache miss — query database
  const user = await userRepo.findById(id);
  if (!user) throw new NotFoundError('User', id);

  // 3. Store in cache for next time
  await redis.set(cacheKey, JSON.stringify(user), 'EX', 3600); // 1 hour TTL

  return user;
}
```

**Why TTL is essential:** Without TTL, cached data becomes stale forever. TTL provides
automatic cache invalidation — even if your explicit invalidation logic has bugs, the
worst case is serving data that's at most TTL seconds old.

### Write-Through

```typescript
// Update cache immediately on write — cache is always fresh
async function updateUser(id: string, data: Partial<User>): Promise<User> {
  // 1. Update database (source of truth)
  const user = await userRepo.update(id, data);

  // 2. Update cache
  await redis.set(KEYS.cache('user', id), JSON.stringify(user), 'EX', 3600);

  return user;
}
```

### Write-Behind (Async Write)

```typescript
// Update cache immediately, write to database asynchronously
// Use when write latency is critical and eventual consistency is acceptable

async function incrementViewCount(postId: string): Promise<number> {
  // Update Redis immediately (fast)
  const count = await redis.hincrby(`post:${postId}`, 'viewCount', 1);

  // Flush to database periodically, not on every write
  // (See queue pattern below for the flush implementation)
  return count;
}

// Periodic flush to database
async function flushViewCounts(): Promise<void> {
  const keys = await scanKeys('post:*');

  for (const key of keys) {
    const postId = key.split(':')[1];
    const viewCount = await redis.hget(key, 'viewCount');

    if (viewCount) {
      await db.query(
        'UPDATE posts SET view_count = $1 WHERE id = $2',
        [Number(viewCount), postId]
      );
    }
  }
}
```

### Cache Invalidation

```typescript
// Delete cache when data changes
async function deleteUser(id: string): Promise<void> {
  await userRepo.delete(id);

  // Invalidate specific key
  await redis.del(KEYS.cache('user', id));

  // Invalidate related caches (user list pages, etc.)
  await invalidatePattern('cache:user:list:*');
}

// Pattern-based invalidation (use SCAN, NEVER KEYS in production)
async function invalidatePattern(pattern: string): Promise<number> {
  let cursor = '0';
  let deletedCount = 0;

  do {
    const [nextCursor, keys] = await redis.scan(
      cursor, 'MATCH', pattern, 'COUNT', 100
    );
    cursor = nextCursor;

    if (keys.length > 0) {
      await redis.del(...keys);
      deletedCount += keys.length;
    }
  } while (cursor !== '0');

  return deletedCount;
}
```

**Why SCAN not KEYS:** `KEYS pattern` blocks the Redis server while it scans ALL keys.
On a Redis instance with millions of keys, this blocks all other clients for seconds.
`SCAN` is cursor-based and non-blocking — it returns results incrementally without
stopping the world.

### Cache Stampede Prevention

```typescript
// Problem: Popular cache key expires → hundreds of requests simultaneously
// hit the database to rebuild it → database overload.

// Solution 1: Distributed lock — only one process rebuilds
async function getWithLock<T>(
  key: string,
  fetchFn: () => Promise<T>,
  ttl: number = 3600
): Promise<T> {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached) as T;

  const lockKey = `lock:${key}`;
  const lockId = crypto.randomUUID();
  const lockAcquired = await redis.set(lockKey, lockId, 'EX', 30, 'NX');

  if (lockAcquired) {
    try {
      const data = await fetchFn();
      await redis.set(key, JSON.stringify(data), 'EX', ttl);
      return data;
    } finally {
      // Release lock only if we still own it (Lua for atomicity)
      await redis.eval(
        `if redis.call("get", KEYS[1]) == ARGV[1] then return redis.call("del", KEYS[1]) else return 0 end`,
        1, lockKey, lockId
      );
    }
  }

  // Another process is rebuilding — wait and retry
  await new Promise(resolve => setTimeout(resolve, 100));
  const retryData = await redis.get(key);
  if (retryData) return JSON.parse(retryData) as T;

  // Still no data — fall through to database (rare edge case)
  return fetchFn();
}

// Solution 2: Stale-while-revalidate — return stale data, refresh in background
async function getWithStaleRefresh<T>(
  key: string,
  fetchFn: () => Promise<T>,
  ttl: number = 3600,
  staleThreshold: number = 300 // refresh when < 5 min remaining
): Promise<T> {
  const data = await redis.get(key);

  if (data) {
    const remainingTtl = await redis.ttl(key);

    if (remainingTtl > 0 && remainingTtl < staleThreshold) {
      // Near expiry — refresh in background, return stale data now
      fetchFn()
        .then(fresh => redis.set(key, JSON.stringify(fresh), 'EX', ttl))
        .catch(err => logger.error('Background cache refresh failed', { key, err }));
    }

    return JSON.parse(data) as T;
  }

  // Cache empty — must wait for fresh data
  const fresh = await fetchFn();
  await redis.set(key, JSON.stringify(fresh), 'EX', ttl);
  return fresh;
}
```

---

## 4. Session Management

```typescript
import { randomBytes } from 'node:crypto';

const SESSION_TTL = 60 * 60 * 24; // 24 hours
const SESSION_REFRESH_THRESHOLD = 60 * 60; // Refresh if < 1 hour remaining

interface SessionData {
  userId: string;
  role: string;
  createdAt: string;
  lastActivity: string;
  metadata?: Record<string, unknown>;
}

async function createSession(userId: string, role: string): Promise<string> {
  const sessionId = randomBytes(32).toString('hex');
  const data: SessionData = {
    userId,
    role,
    createdAt: new Date().toISOString(),
    lastActivity: new Date().toISOString(),
  };

  await redis.set(
    KEYS.session(sessionId),
    JSON.stringify(data),
    'EX', SESSION_TTL
  );

  // Track active sessions per user (for "logout all devices")
  await redis.sadd(KEYS.userSessions(userId), sessionId);

  return sessionId;
}

async function getSession(sessionId: string): Promise<SessionData | null> {
  const data = await redis.get(KEYS.session(sessionId));
  if (!data) return null;

  const session = JSON.parse(data) as SessionData;

  // Sliding expiration — refresh TTL on activity
  const ttl = await redis.ttl(KEYS.session(sessionId));
  if (ttl > 0 && ttl < SESSION_REFRESH_THRESHOLD) {
    await redis.expire(KEYS.session(sessionId), SESSION_TTL);
  }

  // Update last activity
  session.lastActivity = new Date().toISOString();
  await redis.set(KEYS.session(sessionId), JSON.stringify(session), 'KEEPTTL');

  return session;
}

async function destroySession(sessionId: string): Promise<void> {
  const data = await redis.get(KEYS.session(sessionId));
  if (data) {
    const session = JSON.parse(data) as SessionData;
    await redis.srem(KEYS.userSessions(session.userId), sessionId);
  }
  await redis.del(KEYS.session(sessionId));
}

// Logout from all devices
async function destroyAllUserSessions(userId: string): Promise<void> {
  const sessionIds = await redis.smembers(KEYS.userSessions(userId));
  if (sessionIds.length > 0) {
    await redis.del(...sessionIds.map(id => KEYS.session(id)));
  }
  await redis.del(KEYS.userSessions(userId));
}
```

---

## 5. Rate Limiting

### Sliding Window Rate Limiter

```typescript
// More accurate than fixed window — prevents burst at window boundaries

async function checkRateLimit(
  identifier: string,    // IP address, user ID, API key
  endpoint: string,
  limit: number,         // max requests
  windowMs: number       // time window in ms
): Promise<{ allowed: boolean; remaining: number; resetMs: number }> {
  const key = KEYS.rateLimit(identifier, endpoint);
  const now = Date.now();
  const windowStart = now - windowMs;

  // Atomic pipeline: remove expired, add new, count, set expiry
  const multi = redis.multi();
  multi.zremrangebyscore(key, '-inf', windowStart);
  multi.zadd(key, now, `${now}:${Math.random()}`);
  multi.zcard(key);
  multi.pexpire(key, windowMs);

  const results = await multi.exec();
  const requestCount = results?.[2]?.[1] as number;

  return {
    allowed: requestCount <= limit,
    remaining: Math.max(0, limit - requestCount),
    resetMs: windowMs,
  };
}

// Express/Fastify middleware
async function rateLimitMiddleware(req: Request, res: Response, next: NextFunction) {
  const result = await checkRateLimit(req.ip ?? 'unknown', req.path, 100, 60_000);

  res.set({
    'X-RateLimit-Limit': '100',
    'X-RateLimit-Remaining': String(result.remaining),
    'X-RateLimit-Reset': String(Math.ceil(result.resetMs / 1000)),
  });

  if (!result.allowed) {
    return res.status(429).json({
      error: 'RATE_LIMITED',
      message: 'Too many requests, please try again later',
      retryAfter: Math.ceil(result.resetMs / 1000),
    });
  }

  next();
}
```

### Token Bucket (for API rate limiting)

```typescript
// Allows bursts up to bucket size, refills at steady rate.
// Uses Lua script for atomicity — entire operation runs on Redis server.

const TOKEN_BUCKET_SCRIPT = `
  local key = KEYS[1]
  local max = tonumber(ARGV[1])
  local rate = tonumber(ARGV[2])
  local required = tonumber(ARGV[3])
  local now = tonumber(ARGV[4])

  local data = redis.call('HMGET', key, 'tokens', 'last_refill')
  local tokens = tonumber(data[1]) or max
  local lastRefill = tonumber(data[2]) or now

  -- Refill tokens based on elapsed time
  local elapsed = now - lastRefill
  tokens = math.min(max, tokens + (elapsed * rate / 1000))

  if tokens >= required then
    tokens = tokens - required
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    redis.call('PEXPIRE', key, math.ceil(max / rate * 1000) + 1000)
    return 1
  end

  redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
  redis.call('PEXPIRE', key, math.ceil(max / rate * 1000) + 1000)
  return 0
`;

async function tokenBucket(
  key: string,
  maxTokens: number,
  refillRate: number,       // tokens per second
  tokensRequired: number = 1
): Promise<boolean> {
  const result = await redis.eval(
    TOKEN_BUCKET_SCRIPT, 1, key,
    maxTokens, refillRate, tokensRequired, Date.now()
  );
  return result === 1;
}

// Usage
const allowed = await tokenBucket('api:user:123', 60, 1); // 60 burst, 1/sec refill
```

**Why Lua scripts for rate limiting:** Rate limiting involves read-modify-write operations
(check count, increment, compare to limit). Without atomicity, two concurrent requests can
both read count=99, both increment to 100, and both pass a limit of 100 — allowing 101
requests. Lua scripts execute atomically on the Redis server, preventing this race condition.

---

## 6. Queues & Job Processing

### Simple Reliable Queue

```typescript
// Producer — add jobs to queue
interface Job<T = unknown> {
  id: string;
  data: T;
  createdAt: number;
  attempts: number;
}

async function enqueueJob<T>(queueName: string, data: T): Promise<string> {
  const job: Job<T> = {
    id: crypto.randomUUID(),
    data,
    createdAt: Date.now(),
    attempts: 0,
  };

  await redis.lpush(KEYS.queue(queueName), JSON.stringify(job));
  return job.id;
}

// Consumer — reliable processing with BRPOPLPUSH
async function processQueue<T>(
  queueName: string,
  handler: (data: T) => Promise<void>,
  options: { concurrency?: number; maxRetries?: number } = {}
): Promise<void> {
  const { maxRetries = 3 } = options;
  const sourceKey = KEYS.queue(queueName);
  const processingKey = KEYS.queueProcessing(queueName);
  const failedKey = KEYS.queueFailed(queueName);

  while (true) {
    // Block until a job is available (5s timeout, then re-check)
    const jobStr = await redis.brpoplpush(sourceKey, processingKey, 5);
    if (!jobStr) continue;

    const job = JSON.parse(jobStr) as Job<T>;
    job.attempts++;

    try {
      await handler(job.data);
      // Success — remove from processing list
      await redis.lrem(processingKey, 1, jobStr);
    } catch (error) {
      logger.error('Job processing failed', {
        jobId: job.id,
        attempt: job.attempts,
        error,
      });

      // Remove from processing
      await redis.lrem(processingKey, 1, jobStr);

      if (job.attempts < maxRetries) {
        // Re-enqueue with updated attempt count
        await redis.lpush(sourceKey, JSON.stringify(job));
      } else {
        // Max retries exceeded — move to dead letter queue
        await redis.lpush(failedKey, JSON.stringify({
          ...job,
          failedAt: Date.now(),
          lastError: error instanceof Error ? error.message : String(error),
        }));
        logger.error('Job moved to dead letter queue', { jobId: job.id });
      }
    }
  }
}
```

**Why BRPOPLPUSH:** Without it, if your worker crashes after popping a job but before
completing it, the job is lost. BRPOPLPUSH atomically moves the job to a "processing"
list. If the worker crashes, the job stays in the processing list and can be recovered
by a cleanup process.

### Stalled Job Recovery

```typescript
// Periodically check for jobs stuck in processing
async function recoverStalledJobs(queueName: string, stallTimeoutMs: number = 60_000) {
  const processingKey = KEYS.queueProcessing(queueName);
  const sourceKey = KEYS.queue(queueName);

  const jobs = await redis.lrange(processingKey, 0, -1);

  for (const jobStr of jobs) {
    const job = JSON.parse(jobStr) as Job;
    const age = Date.now() - job.createdAt;

    if (age > stallTimeoutMs) {
      logger.warn('Recovering stalled job', { jobId: job.id, ageMs: age });
      await redis.lrem(processingKey, 1, jobStr);
      await redis.rpush(sourceKey, jobStr); // Re-enqueue
    }
  }
}

// Run recovery periodically
setInterval(() => recoverStalledJobs('emails'), 30_000);
```

### For Production: Use BullMQ

```typescript
// For serious job processing, use BullMQ instead of rolling your own
import { Queue, Worker } from 'bullmq';

const emailQueue = new Queue('emails', {
  connection: { host: 'localhost', port: 6379 },
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 1000 },
    removeOnComplete: 1000, // Keep last 1000 completed jobs
    removeOnFail: 5000,     // Keep last 5000 failed jobs
  },
});

// Add jobs
await emailQueue.add('welcome', { userId: '123', template: 'welcome' });
await emailQueue.add('digest', { userId: '123' }, {
  delay: 60_000,              // Delay 1 minute
  priority: 1,                // Higher priority
});

// Repeatable jobs (cron)
await emailQueue.add('daily-digest', {}, {
  repeat: { pattern: '0 9 * * *' }, // Every day at 9 AM
});

// Process jobs
const worker = new Worker('emails', async (job) => {
  switch (job.name) {
    case 'welcome':
      await sendWelcomeEmail(job.data.userId, job.data.template);
      break;
    case 'digest':
      await sendDigest(job.data.userId);
      break;
  }
}, {
  connection: { host: 'localhost', port: 6379 },
  concurrency: 5,
  limiter: { max: 10, duration: 1000 }, // Max 10 jobs per second
});

worker.on('completed', (job) => logger.info(`Job ${job.id} completed`));
worker.on('failed', (job, err) => logger.error(`Job ${job?.id} failed`, { err }));
```

**Why BullMQ over custom:** BullMQ handles everything you'd need to build yourself:
retries with backoff, delayed jobs, repeatable jobs, rate limiting, concurrency control,
stalled job recovery, priority queues, job progress tracking, and a dashboard (Bull Board).
For anything beyond simple fire-and-forget, use BullMQ.

---

## 7. Pub/Sub & Real-Time

### Basic Pub/Sub

```typescript
// Subscriber — dedicated connection (pub/sub blocks the connection)
const subscriber = new Redis({
  host: process.env.REDIS_HOST,
  port: Number(process.env.REDIS_PORT),
});

// Publisher — use the main connection
const publisher = redis;

// Subscribe to channels
await subscriber.subscribe('notifications', 'chat');

subscriber.on('message', (channel, message) => {
  const data = JSON.parse(message);

  switch (channel) {
    case 'notifications':
      handleNotification(data);
      break;
    case 'chat':
      handleChatMessage(data);
      break;
  }
});

// Pattern subscribe (wildcard)
await subscriber.psubscribe('events:*');

subscriber.on('pmessage', (pattern, channel, message) => {
  // pattern: 'events:*'
  // channel: 'events:user:created'
  const data = JSON.parse(message);
  handleEvent(channel, data);
});

// Publish
await publisher.publish('notifications', JSON.stringify({
  type: 'new_message',
  userId: '123',
  content: 'Hello!',
}));
```

**Important:** A Redis connection in subscribe mode can ONLY receive messages — it cannot
run other commands. Always use a separate connection for subscribers. Also, Redis pub/sub
is fire-and-forget — if no subscriber is listening when a message is published, it's lost.
For guaranteed delivery, use Redis Streams instead.

### Redis Streams (Reliable Messaging)

```typescript
// Streams provide persistent, consumer-group-based messaging.
// Unlike pub/sub, messages are stored and can be replayed.

// Producer
async function publishEvent(streamKey: string, event: Record<string, string>) {
  await redis.xadd(streamKey, '*', ...Object.entries(event).flat());
  // '*' = auto-generate ID (timestamp-based)
}

await publishEvent('stream:orders', {
  type: 'order_created',
  orderId: '456',
  userId: '123',
  total: '99.99',
});

// Consumer group setup (run once)
try {
  await redis.xgroup('CREATE', 'stream:orders', 'order-processors', '0', 'MKSTREAM');
} catch (err) {
  // Group already exists — that's fine
}

// Consumer — read from consumer group
async function consumeStream(
  streamKey: string,
  groupName: string,
  consumerName: string,
  handler: (id: string, fields: Record<string, string>) => Promise<void>
): Promise<void> {
  while (true) {
    const results = await redis.xreadgroup(
      'GROUP', groupName, consumerName,
      'COUNT', 10,
      'BLOCK', 5000, // Block for 5s waiting for messages
      'STREAMS', streamKey, '>' // '>' means only new messages
    );

    if (!results) continue;

    for (const [, messages] of results) {
      for (const [id, fields] of messages) {
        try {
          // Convert flat array to object
          const data: Record<string, string> = {};
          for (let i = 0; i < fields.length; i += 2) {
            data[fields[i]] = fields[i + 1];
          }

          await handler(id, data);

          // Acknowledge — message processed successfully
          await redis.xack(streamKey, groupName, id);
        } catch (error) {
          logger.error('Stream message processing failed', { id, error });
          // Don't ACK — message will be re-delivered to another consumer
        }
      }
    }
  }
}
```

**Why Streams over Pub/Sub:** Streams persist messages, support consumer groups (multiple
consumers sharing the workload), track acknowledgments (unacknowledged messages are
re-delivered), and allow replaying from any point in history. Use Pub/Sub for ephemeral
notifications; use Streams for anything that needs reliability.

---

## 8. Transactions & Atomicity

### MULTI/EXEC Transactions

```typescript
// MULTI/EXEC groups commands into an atomic batch
// All commands execute together — no other client can interleave

async function transferCredits(fromId: string, toId: string, amount: number) {
  const fromKey = `user:${fromId}:credits`;
  const toKey = `user:${toId}:credits`;

  // WATCH for optimistic locking — abort if key changes before EXEC
  await redis.watch(fromKey);

  const balance = Number(await redis.get(fromKey)) || 0;
  if (balance < amount) {
    await redis.unwatch();
    throw new InsufficientFundsError(fromId, amount);
  }

  const multi = redis.multi();
  multi.decrby(fromKey, amount);
  multi.incrby(toKey, amount);

  const results = await multi.exec();

  if (!results) {
    // Transaction aborted — another client modified the watched key
    throw new ConflictError('Credits were modified concurrently, please retry');
  }
}
```

### Lua Scripts for Complex Atomicity

```typescript
// Lua scripts run atomically on the Redis server — more powerful than MULTI/EXEC

// Example: Atomic "get if exists, set if not" with custom logic
const CONDITIONAL_SET_SCRIPT = `
  local key = KEYS[1]
  local newValue = ARGV[1]
  local maxAge = tonumber(ARGV[2])

  local existing = redis.call('GET', key)
  if existing then
    local data = cjson.decode(existing)
    local age = tonumber(ARGV[3]) - data.timestamp
    if age < maxAge then
      return existing  -- Still fresh, return existing
    end
  end

  -- Set new value
  redis.call('SET', key, newValue, 'EX', maxAge)
  return newValue
`;

async function getOrSetFresh(
  key: string,
  value: unknown,
  maxAgeSeconds: number
): Promise<unknown> {
  const serialized = JSON.stringify({ data: value, timestamp: Date.now() / 1000 });
  const result = await redis.eval(
    CONDITIONAL_SET_SCRIPT, 1, key,
    serialized, maxAgeSeconds, Date.now() / 1000
  );
  return JSON.parse(result as string).data;
}
```

### Distributed Locks

```typescript
// Redlock algorithm for distributed locking across Redis instances

async function acquireLock(
  resource: string,
  ttlMs: number = 10_000
): Promise<{ lockId: string; release: () => Promise<void> } | null> {
  const lockKey = KEYS.lock(resource);
  const lockId = crypto.randomUUID();

  // SET NX (only if not exists) with TTL
  const acquired = await redis.set(lockKey, lockId, 'PX', ttlMs, 'NX');

  if (!acquired) return null;

  // Return lock with release function
  return {
    lockId,
    release: async () => {
      // Only release if we still own the lock (Lua for atomicity)
      await redis.eval(
        `if redis.call("get", KEYS[1]) == ARGV[1] then
           return redis.call("del", KEYS[1])
         else
           return 0
         end`,
        1, lockKey, lockId
      );
    },
  };
}

// Usage
async function processExclusively(resourceId: string) {
  const lock = await acquireLock(`process:${resourceId}`, 30_000);

  if (!lock) {
    throw new ConflictError('Resource is being processed by another worker');
  }

  try {
    await doExpensiveWork(resourceId);
  } finally {
    await lock.release(); // Always release, even on error
  }
}
```

**Why the Lua release script:** Without it, a slow worker could: (1) acquire lock, (2) take
longer than TTL, (3) lock expires, (4) another worker acquires the lock, (5) first worker
finishes and deletes the NEW lock. The Lua script checks that we still own the lock before
deleting it, preventing this race condition.

---

## 9. Security

### Authentication & Access Control

```typescript
// Always use authentication in production
// redis.conf: requirepass your_strong_password_here

// Use ACLs (Redis 6+) for fine-grained access control
// redis.conf:
// user app_read on >password ~cache:* +get +mget +scan
// user app_write on >password ~* +@all -@dangerous

const readOnlyRedis = new Redis({
  host: process.env.REDIS_HOST,
  password: process.env.REDIS_READ_PASSWORD,
  username: 'app_read', // ACL username
});
```

### Network Security

```typescript
// Enable TLS for connections
const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: 6380, // TLS port
  tls: {
    rejectUnauthorized: true,
    ca: process.env.REDIS_CA_CERT,
  },
  password: process.env.REDIS_PASSWORD,
});

// In production:
// - Bind Redis to private network interfaces only (not 0.0.0.0)
// - Use VPC/private subnets, not public internet
// - Use security groups / firewall rules to restrict access
// - Disable dangerous commands: CONFIG, FLUSHALL, FLUSHDB, DEBUG, KEYS
```

### Data Sensitivity

```typescript
// Never store sensitive data without encryption or TTL
// ❌ Bad
await redis.set('password-reset:user123', 'token-value'); // No expiry!

// ✅ Good — short TTL for sensitive tokens
await redis.set('password-reset:user123', hashedToken, 'EX', 900); // 15 min

// ❌ Bad — storing PII in plain text
await redis.set('user:123:ssn', '123-45-6789');

// ✅ If you must cache PII, encrypt it and use short TTLs
await redis.set('user:123:ssn', encrypt('123-45-6789'), 'EX', 300);
```

---

## 10. Error Handling & Resilience

### Graceful Degradation

```typescript
// If Redis is down, the app should still work (just slower)

class CacheService {
  constructor(private redis: Redis) {}

  async get<T>(key: string): Promise<T | null> {
    try {
      const data = await this.redis.get(key);
      return data ? JSON.parse(data) as T : null;
    } catch (error) {
      // Redis is down — log and return cache miss
      logger.warn('Redis get failed, treating as cache miss', { key, error });
      return null;
    }
  }

  async set(key: string, value: unknown, ttl: number): Promise<void> {
    try {
      await this.redis.set(key, JSON.stringify(value), 'EX', ttl);
    } catch (error) {
      // Redis is down — log but don't fail the request
      logger.warn('Redis set failed, continuing without cache', { key, error });
    }
  }
}

// Usage — application works with or without Redis
async function getUser(id: string): Promise<User> {
  const cached = await cache.get<User>(KEYS.cache('user', id));
  if (cached) return cached;

  const user = await userRepo.findById(id);
  if (!user) throw new NotFoundError('User', id);

  await cache.set(KEYS.cache('user', id), user, 3600); // Best-effort cache
  return user;
}
```

**Why graceful degradation:** Redis should accelerate your application, not be a single
point of failure. If caching is down, the app should fall through to the database — slower,
but functional. Critical state (sessions, rate limits) may need a different strategy,
but cache misses should never crash the application.

### Circuit Breaker Pattern

```typescript
// Stop hammering Redis if it's consistently failing

class RedisCircuitBreaker {
  private failures = 0;
  private lastFailure = 0;
  private state: 'closed' | 'open' | 'half-open' = 'closed';

  constructor(
    private redis: Redis,
    private threshold: number = 5,        // failures before opening
    private resetTimeoutMs: number = 30_000 // time before trying again
  ) {}

  async execute<T>(fn: () => Promise<T>, fallback: T): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastFailure > this.resetTimeoutMs) {
        this.state = 'half-open'; // Try one request
      } else {
        return fallback; // Circuit open, skip Redis
      }
    }

    try {
      const result = await fn();
      if (this.state === 'half-open') {
        this.state = 'closed'; // Success — close circuit
        this.failures = 0;
      }
      return result;
    } catch (error) {
      this.failures++;
      this.lastFailure = Date.now();

      if (this.failures >= this.threshold) {
        this.state = 'open';
        logger.error('Redis circuit breaker opened', { failures: this.failures });
      }

      return fallback;
    }
  }
}
```

### Health Check

```typescript
async function redisHealthCheck(): Promise<{
  status: 'healthy' | 'unhealthy';
  latencyMs: number;
  info?: Record<string, string>;
}> {
  const start = Date.now();

  try {
    const pong = await redis.ping();
    const latency = Date.now() - start;

    if (pong !== 'PONG') {
      return { status: 'unhealthy', latencyMs: latency };
    }

    // Get basic info
    const info = await redis.info('memory');
    const usedMemory = info.match(/used_memory_human:(.+)/)?.[1]?.trim();

    return {
      status: latency < 100 ? 'healthy' : 'unhealthy',
      latencyMs: latency,
      info: { usedMemory: usedMemory ?? 'unknown' },
    };
  } catch (error) {
    return { status: 'unhealthy', latencyMs: Date.now() - start };
  }
}
```

---

## 11. Performance

### Pipelining — Batch Commands

```typescript
// ❌ Slow — each command is a separate round trip
for (const userId of userIds) {
  await redis.get(`cache:user:${userId}`); // N round trips
}

// ✅ Fast — pipeline sends all commands in one round trip
const pipeline = redis.pipeline();
for (const userId of userIds) {
  pipeline.get(`cache:user:${userId}`);
}
const results = await pipeline.exec();
// results: [[null, 'value1'], [null, 'value2'], ...]
// Each entry: [error, result]

const users = results
  ?.filter(([err, val]) => !err && val)
  .map(([, val]) => JSON.parse(val as string) as User) ?? [];
```

**Why pipelining:** Without pipelining, each command requires a full network round trip
(send command → wait → receive response). With 100 commands at 1ms latency, that's 100ms.
Pipelining batches all commands into a single write, then reads all responses together —
100 commands at ~2ms total. Use pipelining whenever you have 3+ independent commands.

### Memory Optimization

```typescript
// 1. Use appropriate data types
// Hashes with < 128 fields and < 64 byte values use ziplist (very compact)
// Set hash-max-ziplist-entries 128
// Set hash-max-ziplist-value 64

// 2. Use short key names in high-volume scenarios
// Instead of 'application:user:profile:cache:' use 'u:p:'
// (Only worth it at millions of keys)

// 3. Set TTLs on everything — prevent unbounded growth
await redis.set(key, value, 'EX', ttl); // Always set expiry

// 4. Use SCAN to find and clean up keyspace
async function analyzeKeyspace(): Promise<Record<string, number>> {
  const prefixCounts: Record<string, number> = {};
  let cursor = '0';

  do {
    const [nextCursor, keys] = await redis.scan(cursor, 'COUNT', 1000);
    cursor = nextCursor;

    for (const key of keys) {
      const prefix = key.split(':').slice(0, 2).join(':');
      prefixCounts[prefix] = (prefixCounts[prefix] || 0) + 1;
    }
  } while (cursor !== '0');

  return prefixCounts;
}

// 5. Monitor memory usage
// redis-cli INFO memory
// redis-cli MEMORY USAGE key_name
```

### Connection Pooling Notes

ioredis uses a single connection by default, which is sufficient for most applications
because Redis commands are pipelined. You only need multiple connections for:

- **Pub/Sub subscribers** (dedicated connection required)
- **Blocking commands** (BRPOP, BLPOP — blocks the connection)
- **Very high throughput** (>50k commands/sec — use cluster mode instead)

---

## 12. Testing

### In-Memory Redis for Tests

```typescript
// Option 1: Use ioredis-mock for unit tests (no Redis needed)
import RedisMock from 'ioredis-mock';

function createTestRedis(): Redis {
  return new RedisMock() as unknown as Redis;
}

// Option 2: Use a real Redis instance for integration tests
// Start Redis in Docker for CI:
// docker run -p 6380:6379 redis:7-alpine

function createTestRedis(): Redis {
  return new Redis({
    host: 'localhost',
    port: 6380,
    db: 15, // Use a separate DB for tests
  });
}
```

### Test Setup/Teardown

```typescript
let redis: Redis;

beforeAll(async () => {
  redis = createTestRedis();
});

afterEach(async () => {
  // Flush test database between tests
  await redis.flushdb();
});

afterAll(async () => {
  await redis.quit();
});
```

### Testing Cache Behavior

```typescript
describe('CacheService', () => {
  const cache = new CacheService(redis);

  it('should return null on cache miss', async () => {
    const result = await cache.get('nonexistent');
    expect(result).toBeNull();
  });

  it('should return cached value on hit', async () => {
    const user = { id: '1', name: 'Alice' };
    await cache.set('user:1', user, 3600);

    const result = await cache.get<typeof user>('user:1');
    expect(result).toEqual(user);
  });

  it('should expire after TTL', async () => {
    await cache.set('temp', 'value', 1); // 1 second TTL

    const before = await cache.get('temp');
    expect(before).toBe('value');

    // Wait for expiry
    await new Promise(resolve => setTimeout(resolve, 1100));

    const after = await cache.get('temp');
    expect(after).toBeNull();
  });

  it('should handle Redis failure gracefully', async () => {
    // Simulate Redis being down
    await redis.quit();

    const result = await cache.get('any-key');
    expect(result).toBeNull(); // Graceful degradation, not an error

    // Reconnect for other tests
    redis = createTestRedis();
  });
});
```

### Testing Rate Limiting

```typescript
describe('Rate Limiter', () => {
  it('should allow requests within limit', async () => {
    for (let i = 0; i < 5; i++) {
      const result = await checkRateLimit('test-ip', '/api', 5, 60_000);
      expect(result.allowed).toBe(true);
    }
  });

  it('should block requests exceeding limit', async () => {
    // Fill up the limit
    for (let i = 0; i < 5; i++) {
      await checkRateLimit('test-ip-2', '/api', 5, 60_000);
    }

    // Next request should be blocked
    const result = await checkRateLimit('test-ip-2', '/api', 5, 60_000);
    expect(result.allowed).toBe(false);
    expect(result.remaining).toBe(0);
  });
});
```

---

## Quick Reference Card

**Always do:**
- Use ioredis with proper retry strategy
- Set TTL on every key (prevent unbounded memory growth)
- Use SCAN instead of KEYS in production
- Use pipeline for batch operations (3+ commands)
- Use Lua scripts for atomic read-modify-write operations
- Use separate connections for pub/sub subscribers
- Implement graceful degradation (app works without Redis)
- Use structured key naming with colon separators
- Encrypt sensitive data before storing

**Never do:**
- Use `KEYS *` in production (blocks Redis server)
- Store sensitive data without TTL
- Let caches grow unboundedly (no TTL, no eviction policy)
- Use Redis as the only data store (it's volatile by default)
- Block the main connection with BRPOP in a web server
- Ignore connection errors (always handle `error` events)
- Store large values (>1MB) — use blob storage instead
- Use `FLUSHALL` in production scripts (too easy to accidentally run)
