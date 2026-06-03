# Real Notification System Design Document (Stages 1 - 5)

This document contains the core architectural specifications and design patterns for the Full Stack Notification Evaluation System, organized across five primary design stages.

---

## STAGE 1: API Design & Real-Time Strategy

### 1. REST API Design
To handle notifications across the system, we define a structured REST interface.

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `POST` | `/api/v1/notifications` | Publish a new notification |
| `GET` | `/api/v1/notifications` | Retrieve paginated notifications with filters |
| `PATCH` | `/api/v1/notifications/:id/read` | Mark a notification as read/viewed |
| `GET` | `/api/v1/notifications/summary` | Get counts of unread notifications by category |

### 2. Request/Response Schemas & Headers
Standard HTTP headers are required for authorization and content negotiation:
* `Content-Type: application/json`
* `Authorization: Bearer <JWT_TOKEN>`

#### POST /api/v1/notifications (Publish)
* **Request Schema (JSON)**:
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "title": { "type": "string", "minLength": 3, "maxLength": 100 },
    "message": { "type": "string", "minLength": 5, "maxLength": 1000 },
    "type": { "type": "string", "enum": ["placement", "result", "event"] },
    "priority": { "type": "string", "enum": ["high", "medium", "low"] },
    "targetAudience": { "type": "string", "enum": ["all", "students", "faculty"] }
  },
  "required": ["title", "message", "type", "priority"]
}
```

* **Response Schema (201 Created)**:
```json
{
  "success": true,
  "data": {
    "id": "c1a93b4f-8b2c-4ef1-a83d-e6b8a8b1d9c2",
    "title": "Placement Drive - Google India",
    "message": "Registration open for Software Engineering role.",
    "type": "placement",
    "priority": "high",
    "targetAudience": "students",
    "createdAt": "2026-06-03T16:15:00.000Z"
  }
}
```

---

### 3. Real-Time Notification Strategy (WebSocket vs. SSE)

To stream notifications to users in real time, we analyze **WebSockets** vs. **Server-Sent Events (SSE)**:

| Feature | WebSockets | Server-Sent Events (SSE) | Chosen Strategy |
| :--- | :--- | :--- | :--- |
| **Communication** | Bidirectional (Duplex) | Unidirectional (Server-to-Client) | **SSE** (Preferred for Notifications) |
| **Protocol** | Custom ws/wss protocol | Standard HTTP (text/event-stream) | **SSE** runs over standard HTTP/2 |
| **Reconnection** | Requires custom client handling | Built-in auto-reconnect with last-event-ID | **SSE** simplifies client state recovery |
| **Firewall Friendliness** | Can be blocked by aggressive corporate proxies | Standard HTTP, passes through firewalls | **SSE** requires no proxy configuration |

**Conclusion**: Since notifications are strictly unidirectional (server sending alerts to clients), **Server-Sent Events (SSE)** is the selected real-time strategy. It is lightweight, supports native auto-reconnection, and operates over HTTP/2 out-of-the-box.

---

## STAGE 2: Database Design & Scaling

### 1. Database Selection: PostgreSQL
We choose **PostgreSQL** as the relational database engine because:
- **ACID Compliance**: Ensures critical academic notification receipts (like placement updates) are never lost or corrupted.
- **Relational Integrity**: Enforces foreign keys mapping notifications to user-read states.
- **Rich Indexing Options**: Supports B-Tree, GIN, and Partial indexes for rapid priority searches.

### 2. Schema Design (DDL)

```sql
-- Main Notifications Table
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(100) NOT NULL,
    message TEXT NOT NULL,
    type VARCHAR(20) NOT NULL CHECK (type IN ('placement', 'result', 'event')),
    priority VARCHAR(10) NOT NULL CHECK (priority IN ('high', 'medium', 'low')),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- User Read/View status Mapping Table
CREATE TABLE user_notifications (
    id BIGSERIAL PRIMARY KEY,
    user_id UUID NOT NULL,
    notification_id UUID REFERENCES notifications(id) ON DELETE CASCADE NOT NULL,
    viewed BOOLEAN DEFAULT FALSE NOT NULL,
    viewed_at TIMESTAMP WITH TIME ZONE,
    CONSTRAINT unique_user_notification UNIQUE (user_id, notification_id)
);
```

### 3. Index Strategy
To optimize sorting and retrieval:
- **Composite Index**: `CREATE INDEX idx_notif_type_created ON notifications(type, created_at DESC);`
  - Speeds up filtered search by type and chronological sorting.
- **Partial/Conditional Index**: `CREATE INDEX idx_user_notif_unviewed ON user_notifications(user_id) WHERE viewed = FALSE;`
  - Dramatically speeds up retrieving "unread" notifications for a user.

### 4. SQL Queries
* **Fetch Priority Sorted Inbox**:
```sql
SELECT n.*, un.viewed
FROM notifications n
LEFT JOIN user_notifications un ON n.id = un.notification_id AND un.user_id = $1
ORDER BY 
    CASE WHEN n.type = 'placement' THEN 1
         WHEN n.type = 'result' THEN 2
         WHEN n.type = 'event' THEN 3
         ELSE 4
    END ASC,
    n.created_at DESC
LIMIT $2 OFFSET $3;
```

### 5. Scaling Discussion
To scale to millions of notifications:
- **Database Partitioning**: Partition the `notifications` and `user_notifications` tables by month/year using PostgreSQL table partitioning.
- **Read Replicas**: Route heavy `GET` queries to read replicas while directing writes to a single primary database.
- **Sharding**: Shard the `user_notifications` table based on a hash of the `user_id`, distributing read/write traffic across separate database clusters.

---

## STAGE 3: Query Analysis & Optimization

### 1. Slow Query Analysis
Consider the following initial slow query:
```sql
SELECT * FROM notifications 
WHERE type = 'placement' 
ORDER BY created_at DESC;
```
* **Why it is slow**: Without an index covering both `type` and `created_at`, PostgreSQL performs a **Sequential Scan (Seq Scan)**. It reads every row from disk, filters rows by `type`, and then performs an in-memory or disk-based **External Merge Sort** to arrange them chronologically.

### 2. Recommended Indexes
We create a multi-column B-Tree index:
```sql
CREATE INDEX idx_notif_type_created_desc ON notifications(type, created_at DESC);
```

### 3. Cost Analysis & Execution Plan (EXPLAIN ANALYZE)

#### Before Indexing (Seq Scan):
```
EXPLAIN ANALYZE SELECT * FROM notifications WHERE type = 'placement' ORDER BY created_at DESC;
-> Sort (cost=1420.50..1440.00 rows=7800 width=120) (actual time=45.12..48.55 rows=7800 loops=1)
     Sort Key: created_at DESC
     Sort Method: quicksort  Memory: 1250kB
     -> Seq Scan on notifications  (cost=0.00..890.00 rows=7800 width=120) (actual time=0.01..12.30 rows=7800 loops=1)
          Filter: (type = 'placement'::text)
Total runtime: 51.20 ms
```

#### After Indexing (Index Scan):
```
EXPLAIN ANALYZE SELECT * FROM notifications WHERE type = 'placement' ORDER BY created_at DESC;
-> Index Scan using idx_notif_type_created_desc on notifications (cost=0.42..320.10 rows=7800 width=120) (actual time=0.03..3.20 rows=7800 loops=1)
     Index Cond: (type = 'placement'::text)
Total runtime: 3.55 ms
```

* **Cost Reduction**: The cost drops from **1440.00** to **320.10** units, and execution time drops from **51ms** to **3.5ms** (an ~14x speedup) by converting the scan into a direct Index Scan with no separate sort step.

---

## STAGE 4: Performance Optimizations

### 1. Redis Caching
To reduce database load, cache user notifications in Redis:
- **Key Strategy**: `user: {userId}:notifications:page:{pageNum}`
- **TTL**: 300 seconds (5 minutes).
- **Eviction/Invalidation**: On creating a new notification, purge keys matching `user:{userId}:notifications:*` using cache tags.

### 2. Pagination Strategy: Cursor vs. Offset
- **Offset Pagination (`LIMIT 10 OFFSET 100000`)**:
  - *Cons*: Database must scan and discard `100,000` rows first. Performance degrades linearly ($O(N)$). Also suffers from item skipping/duplicate errors if new notifications are added while paginating.
- **Cursor Pagination (`WHERE created_at < '2026-06-03T16:15:00Z' LIMIT 10`)**:
  - *Pros*: Scans directly from index starting at cursor. Constant time $O(1)$ lookup. Immune to pagination shifts. Used in this portal.

### 3. Lazy Loading & Push vs. Pull Strategies
- **Pull Strategy (Polling)**:
  - Client polls the server every 30s. Easy to implement but wastes bandwidth and database connection pools.
- **Push Strategy (SSE/WebSockets)**:
  - Server maintains active connection and pushes updates immediately. Highly efficient but requires keeping thousands of persistent connections open.
- **Hybrid (Chosen)**: Clients load list on-demand via cursor-paginated API calls (Lazy Loading) and listen via **SSE** strictly for live update signals that increment an unread badge, rather than pushing full lists over sockets.

---

## STAGE 5: Notify All Architecture (Queue-Based)

When broadcasting to millions of students simultaneously, doing inline database writes will crash the application server. We implement a decoupled queue-based pipeline:

### 1. Message Queue Architecture

```
[API Server] 
     │ (Publish Broadcast Event)
     ▼
[RabbitMQ / Kafka Exchange]
     │
     ├──► [Queue: Student Workers] ──► [Worker Group A] ──► Write to DB (user_notifications)
     │
     └──► [Queue: Push Services]   ──► [Worker Group B] ──► Dispatch iOS/Android Push Alerts
```

### 2. Retry Policy & Dead Letter Queue (DLQ)
- **Retry Mechanism**: If a worker fails to write to DB or push alerts (e.g. database connection pool exhausted), the message is retried with **Exponential Backoff** (e.g., 2s, 4s, 8s).
- **Dead Letter Queue (DLQ)**: If a log/write fails 5 consecutive times, the message is routed to `notifications.broadcast.dlq` for administrator inspection and manual resolution, preventing queue blockages.

### 3. Idempotency Guarantee
To prevent sending duplicate alerts, each message carries an `idempotencyKey` consisting of `md5(notificationId + userId)`. Workers store processed keys in Redis with a 24-hour TTL:
```
IF EXISTS redis.get(idempotencyKey):
    SKIP processing (duplicate)
ELSE:
    redis.set(idempotencyKey, "processed", TTL=86400)
    EXECUTE database insert
```

### 4. Revised Pseudocode for Queue-Based Worker
```typescript
interface BroadcastMessage {
  notificationId: string;
  userId: string;
  idempotencyKey: string;
}

async function processBroadcast(message: BroadcastMessage): Promise<void> {
  const { notificationId, userId, idempotencyKey } = message;

  // 1. Check idempotency in Redis cache
  const isDuplicate = await redis.get(idempotencyKey);
  if (isDuplicate) {
    console.log(`Skipping duplicate message: ${idempotencyKey}`);
    return;
  }

  try {
    // 2. Perform transactional database write
    await db.transaction(async (tx) => {
      await tx.insert(user_notifications).values({
        userId,
        notificationId,
        viewed: false
      });
    });

    // 3. Mark idempotency key as processed in Redis
    await redis.set(idempotencyKey, 'processed', 'EX', 86400);

  } catch (error) {
    console.error(`Error saving user notification: ${error}`);
    // Throwing error triggers message negative acknowledgment (nack) and retries
    throw error; 
  }
}
```
