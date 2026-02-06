# The Ultimate System Design Encyclopedia
## From Zero to Staff Engineer — Complete Reference Guide

---

# PHASE 1: FOUNDATIONS (Weeks 1–4)

---

## 1.1 Scalability — The Core Problem

Every system design question boils down to: "How do you handle more load?"

### Vertical Scaling (Scale Up)
- Add more CPU, RAM, disk to a single machine.
- **Limit**: The biggest AWS instance (u-24tb1.112xlarge) has 448 vCPUs and 24TB RAM. That's the ceiling.
- **When it works**: PostgreSQL reads, in-memory caches, early-stage startups. Stack Overflow runs on just 9 web servers (vertical scaling + great architecture).
- **When it fails**: Single point of failure. Costs grow exponentially. Cannot scale beyond hardware limits.

### Horizontal Scaling (Scale Out)
- Add more machines.
- **Challenge**: Now you need load balancing, data partitioning, consensus, distributed transactions.
- **Every company at scale uses this**: Netflix (100,000+ EC2 instances), Google (millions of servers across 40+ data centers), Facebook (hundreds of thousands of servers).

### Key Metrics to Understand

**QPS (Queries Per Second)**: How many requests your system handles.
- Google Search: ~100,000 QPS
- Twitter: ~300,000 QPS for reads, ~6,000 QPS for writes
- WhatsApp: ~600,000 QPS for messages

**Latency Percentiles**:
- **p50 (median)**: Half of requests are faster than this. Typical target: 50-200ms.
- **p99**: 99% of requests are faster. This is what matters for user experience. If p99 = 2s, 1 in 100 users waits 2+ seconds.
- **p99.9**: Critical for payment systems. Amazon found every 100ms of latency cost 1% in sales.

**Throughput**: Data processed per unit time. Measured in MB/s, GB/s, or events/sec.

**Back-of-Envelope Estimation Cheat Sheet**:
```
1 million requests/day = ~12 requests/sec
1 billion requests/day = ~12,000 requests/sec

1 KB per request × 12,000 QPS = 12 MB/s bandwidth
1 MB per request × 12,000 QPS = 12 GB/s bandwidth (need CDN)

Storage:
- 1 million users × 1KB profile = 1 GB
- 1 billion users × 1KB profile = 1 TB
- 1 million users × 10 photos × 5 MB = 50 TB

Common latencies:
- L1 cache: 0.5 ns
- RAM: 100 ns
- SSD random read: 150 μs
- HDD random read: 10 ms
- Send 1 KB over 1 Gbps network: 10 μs
- Intra-datacenter round trip: 0.5 ms
- Cross-datacenter round trip: 30-100 ms
- Cross-continent round trip: 100-300 ms
```

---

## 1.2 CAP Theorem and Its Real-World Implications

### The Theory
In a distributed system, during a network partition, you must choose between Consistency and Availability.

- **Consistency (C)**: Every read receives the most recent write or an error.
- **Availability (A)**: Every request receives a response (no errors), without guarantee it's the most recent.
- **Partition Tolerance (P)**: System operates despite network failures between nodes.

**Network partitions ALWAYS happen** (cable cuts, switch failures, cloud zone outages). So in practice, you choose between CP and AP.

### Real-World Choices

**CP Systems (Consistency over Availability)**:
- **Google Spanner**: Globally consistent. Uses TrueTime (GPS + atomic clocks) to assign globally meaningful timestamps. When a partition occurs, Spanner will block writes until consistency can be guaranteed. Used for Google Ads, Google Play.
- **HBase**: Strong consistency within regions. Used by Facebook Messages (historically).
- **ZooKeeper**: CP coordination service. If a majority of nodes can't communicate, the minority becomes unavailable.
- **MongoDB (default config)**: Strong consistency with single primary. Reads from primary always consistent.
- **Use when**: Financial transactions, inventory management, seat booking, anything where stale reads cause real problems.

**AP Systems (Availability over Consistency)**:
- **Cassandra**: Tunable consistency but typically runs as AP. Writes always succeed to available nodes. Reconciles later via anti-entropy (Merkle trees, read repair, hinted handoff). Used by Netflix, Apple, Discord.
- **DynamoDB**: Eventual consistency by default (strong consistency available at 2x cost). Amazon.com shopping cart — better to add an item twice than lose an add.
- **CouchDB**: Multi-master replication with conflict resolution. Great for offline-first apps.
- **DNS**: Eventually consistent. TTL-based propagation. Your DNS update might take hours to propagate globally.
- **Use when**: Social media feeds, product catalogs, recommendation caches, analytics. A slightly stale read is acceptable.

### PACELC — The Practical Extension
CAP only covers failure scenarios. PACELC asks: "When there's NO partition (Else), do you optimize for Latency or Consistency?"

| System | Partition: C or A | Else: L or C |
|--------|-------------------|--------------|
| Spanner | PC | EC (globally consistent, higher latency) |
| DynamoDB | PA | EL (fast, eventually consistent) |
| Cassandra | PA | EL (tunable, defaults to fast) |
| MongoDB | PC | EC (consistent reads from primary) |
| PostgreSQL (single) | PC | EC (ACID) |

---

## 1.3 Networking Deep Dive

### DNS (Domain Name System)

**How it works**:
```
Browser → Local DNS Resolver (ISP) → Root Name Server
  → TLD Name Server (.com) → Authoritative Name Server (netflix.com)
  → Returns IP address → Cached at each level
```

**Record Types**:
- **A**: Domain → IPv4 (e.g., netflix.com → 54.237.226.164)
- **AAAA**: Domain → IPv6
- **CNAME**: Alias (e.g., www.netflix.com → netflix.com)
- **NS**: Delegation to name servers
- **MX**: Mail routing
- **TXT**: Verification, SPF, DKIM
- **SRV**: Service discovery (used in microservices)

**Industry Patterns**:
- **Netflix**: Uses AWS Route 53 with latency-based routing. Users in Europe automatically routed to EU servers.
- **Google**: Runs its own authoritative DNS (Google Public DNS: 8.8.8.8). Anycast routing — same IP address advertised from multiple locations, traffic goes to nearest.
- **Cloudflare**: 1.1.1.1. Claims fastest public DNS resolver. Uses anycast from 300+ cities.

**DNS-based Load Balancing**:
- Round-robin DNS: Return different IPs for each query. Simple but no health checking.
- Weighted DNS: Route 90% traffic to primary, 10% to canary.
- Geolocation DNS: Route based on client location. Route 53 supports this.
- Failover DNS: Health-check primary, switch to secondary on failure.

### CDN (Content Delivery Network)

**How CDNs Work**:
```
User in Tokyo requests image.jpg
  → CDN Edge in Tokyo: Cache HIT → Return image (5ms)
  → CDN Edge in Tokyo: Cache MISS → Request from Origin (US)
    → Origin returns image → Edge caches it → Return to user (200ms first time, 5ms after)
```

**Push vs Pull CDN**:
- **Pull (origin-pull)**: CDN fetches from origin on first request, caches. Most common. CloudFront, Cloudflare.
- **Push**: You upload content to CDN proactively. Good for large files, predictable content. Netflix Open Connect.

**Netflix Open Connect — The Most Advanced CDN**:
- Netflix serves **95%+ of traffic** from Open Connect Appliances (OCAs) placed inside ISP networks.
- OCAs are custom servers with 100-200 TB of SSD/HDD storage.
- Content pre-positioned overnight based on ML predictions of what will be watched tomorrow.
- During peak hours (evening), almost zero traffic goes to Netflix's origin servers.
- Over 1,000 ISP partners, 17,000+ servers worldwide.
- Result: Consistent 4K streaming quality regardless of user location.

**CDN Invalidation Strategies**:
- **TTL-based**: Set Cache-Control headers. `max-age=3600` = cached for 1 hour.
- **Versioned URLs**: `style.v2.css` or `style.css?v=abc123`. New version = new URL = cache bypass.
- **Purge API**: Explicitly invalidate cached content. CloudFront invalidation takes 5-15 minutes.
- **Stale-while-revalidate**: Serve stale content while fetching fresh in background. Best user experience.

### Load Balancers

**Layer 4 (Transport Layer)**:
- Routes based on IP + port. No inspection of HTTP content.
- Faster, less CPU-intensive. Can handle millions of connections.
- **Google Maglev**: Custom L4 LB. Each machine handles 10M+ packets/sec. Uses consistent hashing to distribute across backends. Published 2016 paper.
- **AWS NLB**: Network Load Balancer. L4. Millions of requests/sec. Static IPs. Best for gRPC, WebSocket.

**Layer 7 (Application Layer)**:
- Routes based on HTTP content: URL path, headers, cookies, request body.
- Can do smart routing: `/api/v1/*` → service A, `/api/v2/*` → service B.
- Can inject/modify headers. SSL termination.
- **AWS ALB**: Application Load Balancer. L7. Path-based and host-based routing. WebSocket support.
- **Nginx**: Most popular L7 LB. Also reverse proxy, web server, caching. Uber, Airbnb, Dropbox use it.
- **HAProxy**: Battle-tested L7 LB. GitHub, Stack Overflow, Reddit use it.
- **Envoy**: Modern L7 proxy built by Lyft. gRPC-native, automatic retries, circuit breaking, distributed tracing. Foundation of service meshes (Istio). Used by Spotify, Uber, Airbnb.

**Load Balancing Algorithms**:
| Algorithm | How It Works | Best For |
|-----------|-------------|----------|
| Round Robin | Rotate through servers sequentially | Equal-capacity servers |
| Weighted Round Robin | More traffic to higher-capacity servers | Mixed hardware |
| Least Connections | Send to server with fewest active connections | Long-lived connections |
| Least Response Time | Send to fastest-responding server | Heterogeneous latencies |
| IP Hash | Hash client IP → consistent server | Session affinity |
| Consistent Hashing | Minimal redistribution when servers added/removed | Caching layers, stateful services |
| Random with Two Choices | Pick 2 random servers, send to less loaded one | Simple yet effective (proven by research) |

**Health Checking**:
- **Active**: LB periodically pings backends (`GET /health`). Remove unhealthy nodes.
- **Passive**: Monitor response codes/latency. If server returns 500s, remove it.
- **Graceful degradation**: Don't remove all backends. Keep minimum even if "unhealthy."

### API Gateway

**What it does**: Single entry point for all API requests. Handles cross-cutting concerns.

**Capabilities**:
- **Rate limiting**: Protect backends from overload
- **Authentication/Authorization**: Validate tokens before forwarding
- **Request routing**: Route to appropriate microservice
- **Protocol translation**: REST ↔ gRPC, HTTP ↔ WebSocket
- **Request/response transformation**: Add headers, reshape payloads
- **Caching**: Response caching at the edge
- **Circuit breaking**: Stop forwarding to failing services
- **Logging/monitoring**: Centralized request logging

**Industry Examples**:
- **Netflix Zuul** (now Spring Cloud Gateway): Custom API gateway. Handles all Netflix API traffic. Plugin architecture for filters (auth, routing, logging). One of the pioneering microservices gateways.
- **Kong**: Open-source, built on Nginx/OpenResty. Plugin ecosystem. Used by many companies.
- **AWS API Gateway**: Fully managed. REST, HTTP, WebSocket APIs. Integrates with Lambda.
- **Envoy + custom filters**: Lyft, Uber. Envoy as both gateway and sidecar.

---

## 1.4 Communication Protocols — Complete Guide

### REST (Representational State Transfer)

**Principles**: Stateless, resource-oriented, standard HTTP methods (GET, POST, PUT, DELETE, PATCH).

**Best Practices**:
```
GET    /api/v1/users              → List users (with pagination)
GET    /api/v1/users/123          → Get user 123
POST   /api/v1/users              → Create user
PUT    /api/v1/users/123          → Replace user 123
PATCH  /api/v1/users/123          → Partial update user 123
DELETE /api/v1/users/123          → Delete user 123

GET    /api/v1/users/123/orders   → Get orders for user 123
GET    /api/v1/users/123/orders?status=pending&page=2&limit=20
```

**Pagination Patterns**:
- **Offset-based**: `?page=3&limit=20`. Simple but slow for large offsets (DB skips rows).
- **Cursor-based**: `?cursor=eyJpZCI6MTIzfQ&limit=20`. Encode last seen ID. Twitter, Facebook use this. O(1) regardless of page depth.
- **Keyset**: `?after_id=123&limit=20`. Similar to cursor but transparent. Best for chronological feeds.

**Versioning**: URL path (`/v1/users`), header (`Accept: application/vnd.api.v1+json`), or query param (`?version=1`). URL path is most common.

**Used by**: Spotify (public API), Stripe, Twilio, GitHub (v3). The default choice for public APIs.

### GraphQL

**What**: Query language for APIs. Client specifies exactly what data it needs.

```graphql
# Client sends query:
query {
  user(id: "123") {
    name
    email
    orders(last: 5) {
      id
      total
      items {
        name
        price
      }
    }
  }
}

# Server returns exactly that shape — no over-fetching, no under-fetching
```

**Advantages**:
- No over-fetching (REST returns whole object even if you need 2 fields)
- No under-fetching (one query replaces multiple REST calls)
- Self-documenting schema
- Strong typing

**Challenges**:
- N+1 query problem: Naive resolvers make separate DB calls per field. Solution: DataLoader (batching + caching).
- Caching is harder: Every query is unique (unlike REST where URLs are cache keys).
- Rate limiting complexity: One "query" might be trivial or might request the entire database.
- Security: Malicious deep/wide queries can overwhelm servers. Need query depth limiting, cost analysis.

**Industry**:
- **GitHub API v4**: Fully GraphQL. Users query exactly the repo/issue/PR data they need.
- **Shopify**: Storefront API is GraphQL. Merchants query only the product data they display.
- **Netflix**: Internal GraphQL federation across 100s of microservices. One query can span user data, content catalog, recommendations.
- **Facebook**: Invented GraphQL (2012, open-sourced 2015). Powers the Facebook app.
- **Airbnb**: Migrated from REST to GraphQL for mobile apps. Reduced data transfer significantly.

### gRPC (Google Remote Procedure Call)

**What**: High-performance RPC framework using Protocol Buffers (binary serialization) over HTTP/2.

```protobuf
// Define service in .proto file
service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc ListUsers (ListUsersRequest) returns (stream User);  // server streaming
  rpc CreateUsers (stream CreateUserRequest) returns (CreateUsersResponse);  // client streaming
  rpc Chat (stream ChatMessage) returns (stream ChatMessage);  // bidirectional
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
}
```

**Why it's fast**:
- Binary serialization (Protobuf): 3-10x smaller than JSON, 20-100x faster to serialize.
- HTTP/2: Multiplexing (multiple requests on one connection), header compression, server push.
- Streaming: Native support for all 4 patterns (unary, server, client, bidirectional).
- Code generation: Type-safe clients in any language generated from .proto files.

**When to use**: Internal microservice-to-microservice communication where performance matters.

**Industry**:
- **Google**: Everything internal uses gRPC (evolved from Stubby, Google's internal RPC).
- **Netflix**: Uses gRPC between microservices for performance-critical paths.
- **Uber**: gRPC for inter-service communication. Built custom middleware for tracing, auth.
- **Square**: Heavy gRPC usage for internal services.
- **Cloudflare**: Uses gRPC between edge and origin.

### WebSocket

**What**: Full-duplex, bidirectional communication over a single TCP connection. Starts as HTTP upgrade, then becomes persistent.

```
Client: GET /chat HTTP/1.1
        Upgrade: websocket
Server: HTTP/1.1 101 Switching Protocols

[Now full-duplex communication on this connection]
Client ←→ Server (both can send at any time)
```

**Industry**:
- **Slack**: All real-time messaging. Each client maintains WebSocket to Slack servers.
- **Discord**: Real-time messages, presence, typing indicators. Gateway servers handle millions of WebSocket connections using Elixir/Erlang.
- **Uber**: Driver location updates (every 4 seconds). Rider → driver real-time tracking.
- **Coinbase/Binance**: Real-time price feeds. Order book updates via WebSocket.
- **Figma**: Real-time collaborative editing. Cursor positions, edits, selections — all WebSocket.

**Scaling Challenge**: WebSocket connections are long-lived and stateful. A typical HTTP server handles 10,000-100,000+ concurrent WebSocket connections. To handle millions, you need:
- Multiple gateway servers behind L4 load balancer.
- Connection registry (Redis) to track which user is on which server.
- Pub/Sub (Redis, Kafka) to route messages to the right server.
- Sticky sessions or connection migration.

### Server-Sent Events (SSE)

**What**: Server pushes data to client over HTTP. One-directional (server → client). Simpler than WebSocket.

```
Client: GET /events HTTP/1.1
        Accept: text/event-stream

Server: HTTP/1.1 200 OK
        Content-Type: text/event-stream

data: {"token": "Hello"}

data: {"token": " world"}

data: {"token": "!"}

data: [DONE]
```

**When to use**: When you only need server → client push. Simpler than WebSocket, auto-reconnection built into browser API.

**Industry**:
- **ChatGPT/Claude**: Token-by-token streaming responses use SSE. Each token is a server-sent event.
- **GitHub**: Notifications, live updates on PR pages.
- **Stock tickers**: Simple price update streams.

### Apache Kafka — Deep Dive

**What**: Distributed event streaming platform. Not just a message queue — a distributed commit log.

**Core Concepts**:
```
Producer → [Topic: "user-events"]
             ├── Partition 0: [msg1, msg4, msg7, ...]
             ├── Partition 1: [msg2, msg5, msg8, ...]
             └── Partition 2: [msg3, msg6, msg9, ...]
                                    ↓
             Consumer Group A (service-1): 3 consumers, one per partition
             Consumer Group B (analytics): 3 consumers, one per partition
```

**Key Properties**:
- **Ordered within partition**: Messages with same key always go to same partition. If you partition by user_id, all events for a user are ordered.
- **Durable**: Messages persisted to disk. Can be replayed. Retention configurable (days, weeks, forever).
- **Consumer groups**: Multiple consumers in a group share partitions. Adding consumers = more parallelism (up to number of partitions).
- **At-least-once delivery** (default): Consumer might process a message twice after crash. Design for idempotency.
- **Exactly-once semantics**: Available with transactional producers + idempotent consumers. Higher overhead.

**Why Not a Traditional Queue (RabbitMQ)**:
| Feature | Kafka | RabbitMQ |
|---------|-------|----------|
| Model | Distributed log | Message broker |
| Replay | Yes (retain messages) | No (consumed = gone) |
| Ordering | Per-partition | Per-queue |
| Throughput | Millions/sec | Tens of thousands/sec |
| Consumer model | Pull (consumers control pace) | Push (broker pushes) |
| Use case | Event streaming, data pipelines | Task queues, routing |

**Industry at Scale**:
- **LinkedIn**: Created Kafka. Processes 7+ trillion messages/day across 100+ clusters.
- **Uber**: 1+ trillion messages/day. Trip events, driver locations, payments, surge pricing — all through Kafka.
- **Netflix**: 1+ trillion messages/day. Every user action (play, pause, browse) is a Kafka event. Powers real-time analytics and recommendation training.
- **Spotify**: User listening events → Kafka → real-time analytics + recommendation updates.
- **Confluent**: Company built by Kafka creators. Managed Kafka service.

**Kafka Connect**: Pre-built connectors to/from databases, S3, Elasticsearch, etc. CDC (Change Data Capture) from PostgreSQL → Kafka using Debezium.

**Kafka Streams / ksqlDB**: Process streams in real-time. Windowed aggregations, joins, filtering. Stream processing without Spark/Flink.

---

## 1.5 API Design Best Practices

### Idempotency

An operation is idempotent if performing it multiple times has the same effect as performing it once.

**Why it matters**: In distributed systems, requests can be retried due to timeouts, network issues. If a payment request is retried, you don't want to charge twice.

**Implementation**:
```
# Client generates unique idempotency key
POST /api/v1/payments
Headers: Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

# Server stores: idempotency_key → response
# If same key sent again, return cached response without re-executing
```

**Stripe** uses this for all POST requests. Store idempotency keys in Redis with 24-hour TTL.

### Rate Limiting

**Token Bucket** (most common):
```
Bucket capacity: 100 tokens
Refill rate: 10 tokens/second

Request arrives:
  - If tokens > 0: consume 1 token, process request
  - If tokens = 0: reject with 429 Too Many Requests

Allows bursts (up to 100 at once) while maintaining average rate (10/sec)
```

**Distributed Rate Limiting**: Use Redis with atomic INCR + EXPIRE.
```
Key: rate_limit:{user_id}:{window}
INCR key → if count > limit → reject
EXPIRE key {window_seconds} (set only on first request via NX flag)
```

**Stripe's approach**: Rate limits per API key. Different limits for different endpoints. Returns `Retry-After` header.

### Pagination Deep Dive

**Cursor-based (recommended for feeds)**:
```json
GET /api/v1/posts?limit=20&cursor=eyJ0IjoiMjAyNC0wMS0xNSIsImlkIjoiYWJjMTIzIn0=

Response:
{
  "data": [...20 posts...],
  "pagination": {
    "next_cursor": "eyJ0IjoiMjAyNC0wMS0xNCIsImlkIjoiZGVmNDU2In0=",
    "has_more": true
  }
}

# Cursor is base64-encoded: {"t": "2024-01-15", "id": "abc123"}
# Server queries: WHERE (created_at, id) < ('2024-01-15', 'abc123') ORDER BY created_at DESC, id DESC LIMIT 20
```

**Why cursor > offset**:
- Offset `SKIP 10000` means DB reads and discards 10,000 rows. O(n).
- Cursor uses index seek. O(1) regardless of "page number."
- Cursor handles insertions/deletions correctly (no skipped/duplicate items).

---

# PHASE 2: DATA LAYER DEEP DIVE (Weeks 5–10)

---

## 2.1 Relational Databases

### PostgreSQL — The Swiss Army Knife

**Why PostgreSQL dominates**:
- ACID transactions with MVCC (Multi-Version Concurrency Control)
- Rich data types: JSON/JSONB, arrays, hstore, geometric, full-text search, UUID
- Extensions: PostGIS (geospatial), pgvector (vector search), TimescaleDB (time-series), pg_cron (scheduled jobs)
- Materialized views, CTEs, window functions, lateral joins
- Logical replication, table partitioning, parallel query execution

**Instagram's PostgreSQL at Scale**:
- Billions of rows, sharded by user ID.
- Each shard is a PostgreSQL instance.
- Custom Django ORM layer for routing queries to correct shard.
- Used consistent hashing for shard assignment.
- Handled 1B+ monthly active users on sharded PostgreSQL.

**PostgreSQL Indexing**:
```sql
-- B-tree (default): equality and range queries
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_date ON orders(created_at DESC);

-- Composite index: order matters! Leftmost prefix used for queries
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);
-- ✅ WHERE user_id = 1 AND created_at > '2024-01-01'
-- ✅ WHERE user_id = 1
-- ❌ WHERE created_at > '2024-01-01' (can't use, user_id is first)

-- Partial index: index only relevant rows
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;
-- Smaller index, faster for queries that filter on is_active = true

-- GIN index: for JSONB, arrays, full-text search
CREATE INDEX idx_users_metadata ON users USING GIN(metadata jsonb_path_ops);
-- Enables: WHERE metadata @> '{"role": "admin"}'

-- GiST index: for geometric/spatial data, ranges
CREATE INDEX idx_locations ON places USING GiST(coordinates);

-- BRIN index: for naturally ordered data (timestamps in append-only tables)
CREATE INDEX idx_events_time ON events USING BRIN(created_at);
-- Tiny index, very efficient for time-series scans

-- pgvector: vector similarity search
CREATE INDEX idx_embeddings ON items USING ivfflat(embedding vector_cosine_ops) WITH (lists = 100);
-- Enables: ORDER BY embedding <=> '[0.1, 0.2, ...]' LIMIT 10
```

**Connection Pooling**: PostgreSQL spawns a process per connection (~10MB each). Use PgBouncer (lightweight, preferred) or Pgpool-II. Supabase uses PgBouncer. Connection pool size: typically 20-100 connections per application instance.

**PostgreSQL VACUUM**: MVCC means old row versions accumulate. VACUUM reclaims space. autovacuum runs automatically but may need tuning for high-write workloads. Dead tuple monitoring is critical.

### MySQL + Vitess — YouTube's Secret

**Vitess**:
- MySQL sharding middleware built by YouTube in 2010.
- Solves MySQL's limitations at scale: no native sharding, connection limits, lack of online DDL.
- Architecture: `Client → VTGate (proxy) → VTTablet (per-shard) → MySQL`
- Features: Automatic query routing, cross-shard queries, online schema changes (zero-downtime ALTER TABLE), connection pooling.
- **Used by**: YouTube, Slack, Square, HubSpot, GitHub (partial).
- Open-source, CNCF graduated project.

---

## 2.2 NoSQL Databases — Complete Guide

### MongoDB — Document Store

**Data Model**:
```json
// Instead of separate tables with joins:
{
  "_id": "user_123",
  "name": "Piyush",
  "email": "piyush@example.com",
  "addresses": [
    {"type": "home", "city": "Delhi", "pin": "110001"},
    {"type": "work", "city": "Gurugram", "pin": "122001"}
  ],
  "orders": [
    {"order_id": "ord_1", "total": 599, "items": [
      {"sku": "SKU001", "name": "Headphones", "qty": 1}
    ]}
  ]
}
```

**When MongoDB shines**:
- Rapidly evolving schema (startups, MVPs)
- Embedded/nested data that's read together
- High write throughput with flexible consistency
- Geospatial queries (2dsphere index)

**When to avoid MongoDB**:
- Heavy joins across collections (no real joins until $lookup)
- Strict ACID across multiple documents (multi-document transactions added in 4.0 but slower)
- Complex analytical queries

**MongoDB at Scale**:
- **Uber (historically)**: Trip data in MongoDB. Later moved to Schemaless (custom DB on MySQL).
- **Coinbase**: Stores crypto transaction data.
- **Forbes**: Content management.

**Aggregation Pipeline**: MongoDB's equivalent of SQL GROUP BY, JOIN, subqueries:
```javascript
db.orders.aggregate([
  { $match: { status: "completed", date: { $gte: ISODate("2024-01-01") } } },
  { $unwind: "$items" },
  { $group: { _id: "$items.category", total_revenue: { $sum: "$items.price" }, count: { $sum: 1 } } },
  { $sort: { total_revenue: -1 } },
  { $limit: 10 }
])
```

### Cassandra — The Write Machine

**Architecture**: Peer-to-peer (no master). Every node is equal. Uses consistent hashing (token ring).

**Data Modeling Philosophy**: "Model your queries, not your entities." Denormalize aggressively. Each table is designed for one specific query pattern.

```cql
-- Table designed for: "Get all videos watched by a user, sorted by time"
CREATE TABLE user_viewing_history (
    user_id UUID,
    watched_at TIMESTAMP,
    video_id UUID,
    video_title TEXT,
    duration_watched INT,
    PRIMARY KEY (user_id, watched_at)
) WITH CLUSTERING ORDER BY (watched_at DESC);

-- Partition key: user_id (determines which node stores the data)
-- Clustering key: watched_at (determines sort order within partition)
-- Query: SELECT * FROM user_viewing_history WHERE user_id = ? LIMIT 50;
```

**Write Path**: Write → Commit log (disk) + Memtable (memory) → Acknowledge. Memtables periodically flushed to SSTables (sorted string tables) on disk. Compaction merges SSTables.

**Read Path**: Check Memtable → Check Bloom filters (probabilistic, quickly skips irrelevant SSTables) → Read SSTables → Merge results.

**Consistency Levels**:
| Level | Reads | Writes | Guarantee |
|-------|-------|--------|-----------|
| ONE | Read from 1 node | Write to 1 node | Lowest latency, weakest consistency |
| QUORUM | Read from majority | Write to majority | Strong enough for most cases |
| ALL | Read from all replicas | Write to all replicas | Strongest, lowest availability |
| LOCAL_QUORUM | Quorum within local DC | Quorum within local DC | Good for multi-DC |

**Rule**: If `R + W > N` (replicas), you get strong consistency. E.g., N=3, R=2, W=2 → always read latest write.

**Netflix + Cassandra**:
- 30+ PB of data across thousands of nodes.
- Stores viewing history, bookmarks, user preferences.
- Custom tooling: Priam (cluster management), Aegisthus (backup/analytics).
- Chose Cassandra for: write-heavy workload, multi-datacenter replication, linear scalability.

**Discord + Cassandra → ScyllaDB**:
- Initially Cassandra for message storage.
- Hit issues: GC pauses (Java), high p99 latency (200ms), compaction storms.
- Migrated to **ScyllaDB** (C++ rewrite of Cassandra): p99 dropped from 200ms to 15ms.
- ScyllaDB uses shared-nothing architecture, user-space scheduling (no OS thread context switches).

### DynamoDB — Serverless NoSQL

**What**: Fully managed key-value and document store by AWS. Zero operational overhead.

**Data Model**:
- **Partition Key**: Hash determines which partition stores the data.
- **Sort Key (optional)**: Enables range queries within a partition.
- **GSI (Global Secondary Index)**: Alternate partition + sort key. Like a full copy of the table with different key structure.
- **LSI (Local Secondary Index)**: Same partition key, different sort key.

```
Table: Orders
Partition Key: customer_id
Sort Key: order_date

GSI: orders-by-status
  Partition Key: status
  Sort Key: order_date
```

**Pricing Model**: Pay for Read Capacity Units (RCU) and Write Capacity Units (WCU). Or use on-demand mode (pay per request, more expensive but no capacity planning).

**Single-Table Design**: DynamoDB experts (Rick Houlihan) advocate putting ALL entities in one table using generic PK/SK with prefixes:
```
PK: USER#123        SK: PROFILE          → User profile
PK: USER#123        SK: ORDER#2024-01-15 → User's order
PK: ORDER#456       SK: ITEM#1           → Order line item
```

**Industry**:
- **Amazon.com**: Shopping cart, session management, product catalog.
- **Lyft**: Core ride data.
- **Samsung**: IoT device data (billions of devices).
- **Nike**: SNKRS app (high-concurrency shoe drops).
- **Capital One**: Banking transactions.

### Redis — The Speed Demon

**Data Structures** (what makes Redis special — not just GET/SET):

| Structure | Operations | Use Case |
|-----------|-----------|----------|
| **Strings** | GET, SET, INCR, EXPIRE | Caching, counters, rate limiting |
| **Hashes** | HGET, HSET, HGETALL | User profiles, session data |
| **Lists** | LPUSH, RPUSH, LRANGE, LTRIM | Activity feeds, queues |
| **Sets** | SADD, SISMEMBER, SINTER, SUNION | Tags, unique visitors, mutual friends |
| **Sorted Sets** | ZADD, ZRANGE, ZRANGEBYSCORE, ZRANK | Leaderboards, timelines, priority queues |
| **Streams** | XADD, XREAD, XREADGROUP | Event streaming, log aggregation |
| **HyperLogLog** | PFADD, PFCOUNT | Unique count estimation (12KB for billions of elements!) |
| **Bloom Filter** | BF.ADD, BF.EXISTS | "Does this exist?" probabilistic check |
| **Geospatial** | GEOADD, GEODIST, GEORADIUS | Nearby search, distance calculation |

**Redis as a Primary Database**:
- **Redis Persistence**: RDB snapshots (periodic), AOF (append-only file, every write logged). Both can be used together.
- **Redis Cluster**: Automatic sharding across nodes using hash slots (0-16383). Each key hashes to a slot.
- **Redis Sentinel**: High availability. Monitors master, promotes replica if master fails.

**Industry Examples**:
- **Twitter**: Timeline caching using sorted sets. Score = tweet timestamp. Each user's home timeline is a Redis sorted set.
- **GitHub**: Session storage, job queues (Resque/Sidekiq backed by Redis).
- **Snapchat**: Real-time features, rate limiting.
- **Pinterest**: Object cache, rate limiting, pub/sub.
- **Stack Overflow**: Caching layer. Famously runs on very few servers thanks to aggressive Redis caching.

### Elasticsearch — Search and Analytics

**What**: Distributed search and analytics engine built on Apache Lucene.

**How it works**:
- Documents indexed into **inverted index** (word → list of documents containing it).
- Stored in **shards** distributed across nodes.
- Queries fan out to all relevant shards, results merged.

**Key Capabilities**:
```json
// Full-text search with relevance scoring
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "wireless headphones" } },
        { "range": { "price": { "lte": 200 } } }
      ],
      "should": [
        { "match": { "brand": "Sony" } }
      ],
      "filter": [
        { "term": { "in_stock": true } }
      ]
    }
  },
  "highlight": { "fields": { "title": {} } },
  "aggs": {
    "brands": { "terms": { "field": "brand.keyword" } },
    "avg_price": { "avg": { "field": "price" } }
  }
}
```

**Industry**:
- **Netflix**: Search across catalog. Typo tolerance ("Stanger Things" → "Stranger Things").
- **Uber**: Search trips, drivers, restaurants. Geo-filtered search for Uber Eats.
- **Wikipedia**: Powers the search bar. Full-text search across millions of articles.
- **GitHub**: Code search (migrated to custom Blackbird engine, but previously Elasticsearch).
- **ELK Stack**: Elasticsearch + Logstash + Kibana. Industry standard for log aggregation and analysis.

### Vector Databases

**What**: Specialized databases for storing and searching high-dimensional vectors (embeddings).

**How similarity search works**:
```
Query: "comfortable running shoes for flat feet"
  → Embedding model → [0.12, -0.45, 0.78, ...] (768 or 1536 dimensions)
  → Vector DB: Find nearest neighbors using cosine similarity or L2 distance
  → Returns: Top-K most semantically similar items
```

**Indexing Algorithms**:
| Algorithm | Speed | Accuracy | Memory | Used By |
|-----------|-------|----------|--------|---------|
| **HNSW** | Very Fast | High | High | Pinecone, Weaviate, pgvector |
| **IVF** | Fast | Medium-High | Medium | Milvus, FAISS |
| **PQ (Product Quantization)** | Fast | Medium | Low | FAISS, Milvus |
| **ScaNN** | Very Fast | High | Medium | Google (developed by Google Research) |

**Options**:
- **Pinecone**: Fully managed. Simplest to use. Pay per usage.
- **Weaviate**: Open-source. Hybrid search (vector + keyword). GraphQL API.
- **Milvus**: Open-source. Massive scale (billions of vectors). Used by Shopee, Tokopedia.
- **Qdrant**: Open-source. Rust-based. Fast, memory-efficient.
- **pgvector**: PostgreSQL extension. Add vector search to existing PostgreSQL. Great for <10M vectors.
- **FAISS**: Library (not database) by Meta. Building block for other systems. Powers Facebook's similarity search.
- **ChromaDB**: Lightweight, developer-friendly. Popular for RAG prototypes.

---

## 2.3 Caching — The Complete Encyclopedia

### Why Caching Changes Everything

**The numbers**:
- Database read: 1-10ms (with index), 100ms+ (complex query, table scan)
- Redis read: 0.1-0.5ms
- Local in-memory cache: 0.001ms (nanoseconds)
- CDN edge: 5-20ms (vs 200ms from origin)

**Rule of thumb**: If 20% of your data serves 80% of requests (Pareto principle), caching that 20% eliminates 80% of database load.

### Caching Strategies — When to Use Each

#### 1. Cache-Aside (Lazy Loading)

```python
def get_user(user_id):
    # 1. Check cache
    cached = redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)
    
    # 2. Cache miss → read from DB
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    
    # 3. Populate cache
    redis.setex(f"user:{user_id}", 3600, json.dumps(user))  # TTL: 1 hour
    
    return user

def update_user(user_id, data):
    # 1. Update DB
    db.execute("UPDATE users SET ... WHERE id = %s", user_id)
    
    # 2. Invalidate cache (NOT update — avoids race conditions)
    redis.delete(f"user:{user_id}")
```

**Pros**: Only caches data that's actually requested. Cache failures don't break the system (fallback to DB).
**Cons**: First request always has cache miss. Possible stale data between DB update and cache invalidation.
**Used by**: Reddit, GitHub, most applications. Default choice.

#### 2. Read-Through

```
App → Cache.get(key) → Cache checks itself
  → If HIT: return cached value
  → If MISS: Cache itself queries DB → stores result → returns to app
```

**Difference from Cache-Aside**: The cache library/service handles the DB read, not the application. Application code is simpler.
**Used by**: DynamoDB Accelerator (DAX) — transparent read-through cache for DynamoDB. Hibernate second-level cache.

#### 3. Write-Through

```
App → Write to Cache → Cache synchronously writes to DB → Acknowledge
```

**Pros**: Cache is always consistent with DB. No stale reads.
**Cons**: Every write has cache + DB latency. Caches data that might never be read (wasted memory).
**Used by**: Amazon DynamoDB with DAX (write-through + read-through). Banking systems where consistency is critical.

#### 4. Write-Behind (Write-Back)

```
App → Write to Cache → Acknowledge immediately
       └→ Cache asynchronously flushes to DB (batched)
```

**Pros**: Lowest write latency (just cache write). Batching reduces DB writes.
**Cons**: **Data loss risk** if cache crashes before flushing. Consistency lag.
**Used by**: CPU L1/L2 caches (hardware). Write-heavy systems where some data loss is acceptable. Logging, analytics, view counters.

#### 5. Write-Around

```
App → Write to DB directly (bypass cache)
       Reads still go through cache (cache-aside for reads)
```

**Pros**: Doesn't pollute cache with data that might not be read.
**Cons**: Read-after-write will always be a cache miss (until next read populates cache).
**Used by**: Systems with high write volume but reads only for recent data. Log ingestion.

### Cache Invalidation — The Hard Problem

"There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

#### Strategies:

**1. TTL (Time-To-Live)**:
- Set expiration on cache entries. After TTL, entry is evicted.
- Simple, predictable. But: data could be stale for up to TTL duration.
- `redis.setex("user:123", 3600, data)` — expires in 1 hour.

**2. Event-Driven Invalidation**:
```
DB Write → Publish event to Kafka → Cache invalidation consumer → Delete cache key
```
- Near-real-time invalidation. Used by Uber, Netflix.
- Requires event infrastructure (Kafka, Redis Pub/Sub).

**3. Version-Based Keys**:
```
Cache key: "product:123:v5"
On update: increment version → "product:123:v6"
Old key auto-expires via TTL. No explicit invalidation needed.
```

**4. Facebook's Lease-Based Invalidation**:
- Problem: **Thundering herd** — cache key expires → 1000 requests all miss cache → 1000 DB queries.
- Solution: On cache miss, Memcached gives a "lease" (token) to ONE requester. Other requesters for same key wait or get stale data.
- Result: Only 1 DB query even with 1000 concurrent requests.
- Published in: "Scaling Memcache at Facebook" (2013 paper). Essential reading.

### Multi-Level Caching in Production

**Netflix's Complete Caching Stack**:
```
Level 1: Client-side cache (browser, app)
    ↓ (miss)
Level 2: CDN (Open Connect appliances in ISP networks)
    ↓ (miss)  
Level 3: API Gateway cache
    ↓ (miss)
Level 4: EVCache (distributed Memcached)
    - 30 million+ requests/second
    - Multi-zone replication for HA
    - Zone-aware routing (prefer local zone)
    ↓ (miss)
Level 5: Cassandra / MySQL
    ↓ (for analytics)
Level 6: S3 (cold storage)
```

**Facebook's TAO Cache**:
- Graph-aware caching layer between PHP services and MySQL.
- Two-tier: Leader cache (per-region) + Follower caches (per-datacenter).
- Handles **1 billion+ reads/second** (yes, billion with a B).
- Objects (nodes) and Associations (edges) in the social graph.
- Write-through to MySQL. Invalidation via pub/sub.

**Twitter's Timeline Architecture**:
```
Write path: User tweets
  → Fan-out service writes to all follower timelines in Redis
  → Each timeline: Redis sorted set (score = tweet ID/timestamp)
  → Limit: Keep last 800 entries per timeline

Read path: User opens home feed
  → Read from Redis sorted set → instant
  → For celebrity follows: merge celebrity tweets at read time

Cache structure per user:
  timeline:{user_id} → Sorted Set of (tweet_id, score)
  ZREVRANGE timeline:123 0 19 → Latest 20 tweets
```

### Cache Eviction Policies

| Policy | Description | When to Use |
|--------|-------------|------------|
| **LRU** | Evict least recently accessed | General purpose. Best default. Redis `allkeys-lru`. |
| **LFU** | Evict least frequently accessed | When some items are consistently popular (search results, product pages). Redis `allkeys-lfu`. |
| **FIFO** | Evict oldest | Simple, but ignores access patterns |
| **Random** | Evict random entry | Surprisingly effective. Low overhead. |
| **TTL-based** | Evict expired entries | Session data, temporary tokens |
| **Size-based** | Evict largest entries | When memory is constrained and sizes vary |

**Redis eviction configuration**:
```
maxmemory 4gb
maxmemory-policy allkeys-lru   # Recommended for caching
# Other options: volatile-lru (only keys with TTL), allkeys-lfu, noeviction
```

---

# PHASE 3: DISTRIBUTED SYSTEMS PATTERNS (Weeks 11–16)

---

## 3.1 Consistency Models in Practice

### Strong Consistency — Google Spanner

**The problem**: How do you achieve globally consistent transactions across data centers on different continents?

**TrueTime API**:
- Google placed GPS receivers and atomic clocks in every data center.
- TrueTime returns a time interval `[earliest, latest]` with guaranteed real time in that interval.
- Uncertainty typically 1-7ms.
- Transaction rule: Wait out the uncertainty interval before committing.
- If TrueTime says `[T, T+6ms]`, wait 6ms. After that, guaranteed no other transaction started before you.
- This gives **serializable** consistency across the globe.

**Why it's revolutionary**: Before Spanner, global strong consistency was considered impractical. CAP theorem said you must choose. Spanner proved you can have strong consistency with high availability if you're willing to accept small latency penalties.

### Eventual Consistency — Dynamo-Style

**How it works**: Write to any node. Changes propagate asynchronously. Temporarily, different nodes may return different values.

**Conflict Resolution**:
- **Last-Writer-Wins (LWW)**: Highest timestamp wins. Simple but loses data. Cassandra default.
- **Vector Clocks**: Track causal order. Detect conflicts. Let application resolve. Original DynamoDB paper.
- **CRDTs (Conflict-free Replicated Data Types)**: Data structures that automatically merge without conflicts.

**CRDT Examples**:
```
G-Counter (grow-only counter):
  Node A: {A: 5, B: 0}
  Node B: {A: 0, B: 3}
  Merge: {A: 5, B: 3} → Total: 8
  
  No conflicts possible! Each node only increments its own slot.

OR-Set (observed-remove set):
  Handles concurrent add + remove of same element.
  Used by: Figma (real-time collaboration), Riak, Redis Enterprise
```

**Figma's CRDTs**: Figma uses CRDTs for real-time collaborative design. Multiple designers editing the same file simultaneously. Changes merge automatically. No last-writer-wins data loss.

---

## 3.2 Consensus Algorithms

### Raft — The Understandable Consensus Algorithm

**Problem**: How do distributed nodes agree on a value even if some nodes fail?

**How Raft works**:
```
1. Leader Election:
   - Nodes start as Followers
   - If no heartbeat from Leader → become Candidate → request votes
   - Majority votes → become Leader
   - Only Leader accepts writes

2. Log Replication:
   - Client sends write to Leader
   - Leader appends to its log
   - Leader sends AppendEntries RPC to all Followers
   - Once majority acknowledge → entry is "committed"
   - Leader responds to client: success

3. Safety:
   - Only nodes with the most up-to-date log can become Leader
   - Committed entries are never lost
```

**Used by**:
- **etcd**: Kubernetes control plane state store. All K8s cluster state (pods, services, configs) stored in etcd using Raft.
- **CockroachDB**: Each range (shard) has its own Raft group. Provides strong consistency.
- **HashiCorp Consul**: Service discovery and configuration. Raft for leader election and state replication.
- **TiKV**: Distributed KV store (used by TiDB). Raft per region.

### Paxos

**The OG consensus algorithm** (Leslie Lamport, 1989). Mathematically proven correct but notoriously hard to understand and implement.

**Used by**:
- **Google Chubby**: Distributed lock service. Foundation of Google's infrastructure. Paxos-based.
- **Google Spanner**: Uses Paxos for replication within each shard.
- **Apache Mesos**: Uses Paxos variant.

---

## 3.3 Distributed Transaction Patterns

### Saga Pattern

**Problem**: In microservices, a "transaction" spans multiple services. You can't use a single database transaction.

**Example: E-commerce Order**:
```
1. Order Service:    Create order (PENDING)
2. Payment Service:  Charge customer
3. Inventory Service: Reserve items
4. Shipping Service:  Schedule delivery
5. Order Service:    Update order (CONFIRMED)

If step 3 fails (out of stock):
  → Compensate step 2: Refund payment
  → Compensate step 1: Cancel order
```

**Two styles**:
- **Choreography**: Each service listens for events and acts. Decoupled but hard to debug.
```
Order Created → [Payment Service] → Payment Charged → [Inventory Service] → ...
```
- **Orchestration**: Central orchestrator directs the flow. Easier to understand and monitor. Uber uses this for trip lifecycle.
```
Orchestrator → Order Service: Create
Orchestrator → Payment Service: Charge
Orchestrator → Inventory Service: Reserve
(if failure) → Orchestrator → Payment Service: Refund
```

### CQRS (Command Query Responsibility Segregation)

**Idea**: Separate the write model from the read model. They can use different databases, schemas, and scaling strategies.

```
WRITES (Commands):                 READS (Queries):
  Client → API → Command Handler    Client → API → Query Handler
    → Write to PostgreSQL              → Read from Elasticsearch
    → Publish event to Kafka              (or Redis, or DynamoDB)
                ↓
    Kafka Consumer → Update read store
```

**Why separate**:
- Write model optimized for consistency and validation.
- Read model optimized for query patterns (denormalized, pre-computed).
- Scale reads and writes independently.

**LinkedIn Feed**:
- Writes: Post published → Kafka → stored in primary DB.
- Read model: Feed service reads from pre-computed, personalized feed store (denormalized).
- Different storage, different scaling, different optimization.

### Event Sourcing

**Instead of storing current state, store the events that led to it**:
```
Traditional: User { balance: 750 }

Event Sourced:
  Event 1: AccountCreated { amount: 0 }
  Event 2: Deposited { amount: 1000 }
  Event 3: Withdrawn { amount: 200 }
  Event 4: Withdrawn { amount: 50 }
  
  Current state = replay all events → balance: 750
```

**Advantages**:
- Complete audit trail (every change is recorded).
- Time travel (reconstruct state at any point in time).
- Event replay (reprocess events to build new read models).
- Natural fit with CQRS.

**Used by**: Banking systems, financial trading platforms, supply chain tracking, any system requiring complete audit trails.

---

## 3.4 Resilience Patterns

### Circuit Breaker (Netflix Hystrix → Resilience4j)

```
States:
  CLOSED (normal): Requests pass through. Track failures.
    → If failure rate > threshold (e.g., 50% in last 10 calls)
  OPEN (tripped): All requests fail immediately. Don't call the failing service.
    → After timeout (e.g., 30 seconds)
  HALF-OPEN (testing): Let ONE request through.
    → If success → CLOSED
    → If failure → OPEN again
```

**Netflix's approach**: Every microservice-to-microservice call is wrapped in a circuit breaker. If a recommendation service is down, the UI shows generic recommendations instead of failing entirely. "The show must go on."

### Bulkhead Pattern

**Idea**: Isolate failures. Don't let one slow dependency kill everything.

```
Thread Pool for Service A: [■■■□□□□□□□] (3/10 active)
Thread Pool for Service B: [■■■■■■■■■■] (10/10 — THIS IS SLOW)
Thread Pool for Service C: [■■□□□□□□□□] (2/10 active)

Without bulkhead: Service B slowness uses up ALL threads → everything fails
With bulkhead: Service B uses its own pool → Services A & C unaffected
```

### Retry with Exponential Backoff + Jitter

```python
def retry_with_backoff(func, max_retries=5):
    for attempt in range(max_retries):
        try:
            return func()
        except TransientError:
            # Exponential backoff: 1s, 2s, 4s, 8s, 16s
            delay = min(2 ** attempt, 60)  # cap at 60 seconds
            
            # Jitter: randomize to avoid thundering herd
            jitter = random.uniform(0, delay)
            time.sleep(jitter)
    
    raise MaxRetriesExceeded()
```

**AWS SDK** implements this by default. All AWS API calls use exponential backoff with jitter.

### Idempotency Keys for Exactly-Once Processing

```
Producer → Kafka → Consumer

Problem: Consumer crashes after processing but before committing offset.
  → Kafka resends the message → processed twice!

Solution: Every message has an idempotency key.
  Consumer checks: "Have I processed message_id X?"
  If yes → skip. If no → process + store message_id.

Storage for processed IDs: Redis SET with TTL, or database unique constraint.
```

---

# PHASE 4: REAL-WORLD SYSTEM DESIGNS (Weeks 17–28)

---

## 4.1 Netflix — Complete Architecture Deep Dive

### Content Ingestion Pipeline
```
Studio uploads master file (e.g., 4K ProRes, 100+ GB)
  → Stored in S3
  → Encoding pipeline (runs on thousands of EC2 instances):
    → Video analysis: scene complexity, shot boundaries
    → Per-shot encoding: Complex scenes get more bitrate, simple scenes get less
    → Encode into ~1,200 streams:
      - Resolutions: 240p, 360p, 480p, 720p, 1080p, 4K, HDR
      - Codecs: H.264 (compatibility), VP9 (Chrome/Android), AV1 (next-gen, 30% better)
      - Bitrates: Multiple per resolution (e.g., 1080p at 3, 4.5, 5.8, 7.5 Mbps)
    → Audio encoding: Dolby Digital, Dolby Atmos, AAC stereo
    → Subtitle processing: 30+ languages, timed text
  → Manifests generated (what chunks available at what quality)
  → Distributed to Open Connect CDN
```

**Per-Shot Encoding (Netflix's Innovation)**:
- Traditional: Encode entire video at fixed bitrate.
- Netflix: Analyze each shot. Dark talking scene needs less data. Action scene needs more.
- Result: Same perceived quality at 20% less bandwidth. Saves billions in CDN costs.

### Microservices Architecture (500+)
```
Each team owns 1-5 microservices.
Service registry: Netflix Eureka (service discovery)
Load balancing: Netflix Ribbon (client-side LB)
Circuit breaker: Hystrix (now Resilience4j)
API Gateway: Zuul → Spring Cloud Gateway
Configuration: Netflix Archaius (dynamic config without restart)
Inter-service comm: REST + gRPC
Observability: Atlas (metrics), Edgar (distributed tracing), Mantis (real-time stream processing)
```

### Netflix Recommendation Engine — Deep Dive

**The Problem**: 200+ million subscribers. 17,000+ titles. Users give up after 60-90 seconds of browsing. How to show the right content instantly?

**Stage 1: Candidate Generation**
- Multiple retrieval models run in parallel, each generating hundreds of candidates:
  - **Collaborative Filtering**: "Users like you watched..."
    - Implicit feedback (watch history, completion rate, browse behavior)
    - Matrix Factorization: Decompose user-item matrix into latent factors
    - User vector (128-dim) × Item vector (128-dim) → predicted rating
  - **Content-Based**: "Because you watched [genre/actor/director]..."
    - Video metadata: genre, cast, director, tags
    - Human-curated tags: "cerebral," "witty," "dark"
    - Audio/visual features extracted by ML models
  - **Trending/Popular**: Regional and global trends
  - **Personal Trending**: What's picking up among users similar to you
  - **Because You Watched (BYW)**: Direct item-to-item similarity

**Stage 2: Ranking**
- Deep neural network scores each candidate
- Features include:
  - User features: demographics, watch history, time since last watch
  - Item features: genre, popularity, age, average completion rate
  - Context: time of day, day of week, device (TV vs phone)
  - Interaction features: user's history with this genre, user's engagement patterns
- Model predicts: Probability of play AND Probability of completion (not just clicks!)
- **Wide & Deep architecture**: Wide path memorizes specific user-item interactions. Deep path generalizes across features.

**Stage 3: Row Assembly + Re-ranking**
- Homepage is rows: "Trending Now," "Because You Watched X," "Top 10 in India"
- Row selection: ML model chooses which rows to show and in what order
- **Diversity injection**: Avoid showing same genre in consecutive rows
- **Explore/Exploit**: 80% exploit (show what model is confident user likes), 20% explore (test new content to gather data)

**Stage 4: Artwork Personalization**
- Same show, different thumbnails for different users
- User who watches action → sees action scene thumbnail for a drama
- User who watches romance → sees romantic moment thumbnail for same drama
- **Contextual Bandits**: Multi-armed bandit that learns which artwork drives engagement per user
- A/B tested: Personalized artwork increased engagement by ~20%

**Training Pipeline**:
```
User events (Kafka) → Data Lake (S3)
  → Feature Engineering (Spark)
  → Training (PyTorch on GPU clusters)
  → Model Evaluation (offline metrics: precision, recall, NDCG)
  → A/B Test (allocate 1% traffic to new model)
  → Champion/Challenger: If new model wins → promote to 100%
```

### Chaos Engineering

**Chaos Monkey**: Randomly terminates EC2 instances in production during business hours. Forces engineers to build resilient services. "If your service can't handle a random instance dying, fix it."

**Chaos Kong**: Simulates entire AWS region failure. Netflix can failover between US regions in minutes. Forces multi-region design.

**FIT (Failure Injection Testing)**: Inject specific failures (latency, errors, network partition) between specific services. Test graceful degradation.

**Principle**: "The best way to avoid failure is to fail constantly." — Netflix

---

## 4.2 Spotify — Discovery Engine Deep Dive

### Audio Analysis Pipeline

```
New Song Uploaded → Audio Analysis Service:
  1. Raw audio → Mel Spectrogram (visual representation of frequency over time)
  2. CNN processes spectrogram → Audio embedding (128-256 dim vector)
  
Features extracted:
  - Tempo: 60-200 BPM
  - Key: C major, A minor, etc.
  - Energy: 0.0-1.0 (quiet acoustic vs loud EDM)
  - Danceability: 0.0-1.0
  - Valence: 0.0-1.0 (sad to happy)
  - Instrumentalness: 0.0-1.0
  - Speechiness: 0.0-1.0
  
  3. Audio embedding stored in vector database
  4. Used for: similar song recommendations, radio feature, mood-based playlists
```

**Why this matters (Cold Start Problem)**: New songs have zero listens. Collaborative filtering can't recommend them. But audio analysis can immediately say "this sounds like Coldplay" and recommend to Coldplay fans.

### Discover Weekly — How It's Built

```
Weekly Pipeline (every Monday):
  1. Collaborative Filtering:
     - Build user-track matrix (500M+ users × 100M+ tracks)
     - ALS (Alternating Least Squares) matrix factorization
     - Find "taste neighbors" — users with similar listening patterns
     - Collect their listens that you haven't heard

  2. NLP on Playlists:
     - Scrape 4B+ playlist titles and descriptions
     - Word2Vec / BERT on playlist text
     - "Chill Sunday Morning" playlists tend to contain [specific songs]
     - Learn associations: playlist theme → songs

  3. Audio Model:
     - For tracks without enough collaborative data
     - CNN embedding captures musical qualities
     - Find similar-sounding tracks to user's favorites

  4. Merge & Rank:
     - Combine candidates from all three sources
     - Ranking model (gradient boosted trees / neural network):
       Features: collaborative score, audio similarity, freshness,
                 user's genre preferences, listening context
     - Diversity filter: Don't include 5 songs from same artist

  5. Output: 30 songs per user, refreshed every Monday
```

### Spotify's Backend Architecture

```
Client (Desktop/Mobile/Web) → Edge Proxy (Envoy)
  → Access Point (AP): Long-lived TCP connection per client
     Handles: Authentication, encryption, audio streaming, metadata
  → Microservices (1000+):
     - User Service (profile, preferences)
     - Catalog Service (tracks, albums, artists, metadata)
     - Playlist Service (user playlists, collaborative playlists)
     - Search Service (Elasticsearch-based)
     - Social Service (followers, activity feed)
     - Ads Service (free tier ad insertion)
     - Payment Service (subscriptions)
     - Recommendation Service (multiple sub-services)
  → Data:
     - Google BigQuery: Analytics data warehouse
     - Google Bigtable: Low-latency serving (user preferences, features)
     - Google Cloud Dataflow: Stream processing (real-time listening events)
     - PostgreSQL: Transactional data (user accounts, payments)
     - Cassandra: High-write workloads (listening history)
```

---

## 4.3 Google Search — Deep Dive

### Indexing Pipeline

```
1. Crawling (Googlebot):
   - Starts from seed URLs
   - BFS crawl with priority queue
   - Politeness rules: Respect robots.txt, rate limit per domain
   - Render JavaScript (headless Chrome for SPAs)
   - Fresh crawl (frequent for news sites) + Deep crawl (thorough for static content)

2. Processing:
   - Parse HTML → extract text, links, metadata
   - Language detection
   - Duplicate detection (SimHash — detect near-duplicate pages)
   - Spam detection (ML models for link spam, keyword stuffing)
   - Entity extraction (people, places, events)

3. Indexing:
   - Build inverted index: word → [(doc1, position), (doc2, position), ...]
   - Stored in custom format on Google's Colossus file system
   - Index sharded across thousands of machines
   - Forward index: doc → [word1, word2, ...] (for document features)

4. Knowledge Graph:
   - Extract entities and relationships
   - 500 billion+ facts about 5 billion+ entities
   - Powers knowledge panels, featured snippets, direct answers
```

### Ranking Evolution

**PageRank (1998)**:
- Treat web as a graph. Each link is a "vote."
- Pages linked to by many important pages are important.
- Iterative computation: `PR(A) = (1-d) + d × Σ(PR(T)/L(T))` for all pages T linking to A.
- Revolutionary at the time but insufficient alone.

**RankBrain (2015)**:
- ML model for query understanding.
- Handles never-before-seen queries (15% of daily queries are new).
- Converts queries to vectors. Finds similar known queries.
- Example: "What's the title of the consumer at the highest level of a food chain?" → maps to "apex predator."

**BERT (2019)**:
- Transformer-based language understanding.
- Understands context and word relationships.
- "2019 brazil traveler to usa need a visa" — BERT understands "to" means traveling TO the USA (not from).
- Applied to 10% of English queries initially, expanded globally.

**MUM (Multitask Unified Model, 2021)**:
- 1000x more powerful than BERT.
- Multimodal: understands text AND images.
- Multilingual: trained across 75 languages.
- Can transfer knowledge across languages.
- Example: "I've hiked Mt. Adams and now want to hike Mt. Fuji next fall, what should I do differently to prepare?" — MUM understands hiking, Mt. Fuji's specific conditions, seasonal factors, AND finds answers across Japanese language sources.

**Gemini Integration (2024+)**:
- LLM-powered "AI Overviews" in search results.
- Generates synthesized answers from multiple sources.
- Represents fundamental shift from "10 blue links" to direct AI answers.

### Serving Architecture

```
User types query → Browser/App
  → DNS (Google's anycast DNS) → Nearest Google data center
  → Google Frontend (GFE): SSL termination, request routing
  → Web Server: Query parsing, spell correction, query expansion
  → Index Serving System:
     Query → Sent to ALL relevant index shards in parallel (1000+ shards)
     Each shard returns top candidates with scores
     Results merged centrally
  → Ranking:
     1000+ signals: PageRank, content quality, freshness, user location,
       device, language, query-document relevance (BERT/MUM), 
       user search history, site authority
     Multiple passes: Coarse ranking → Fine ranking → Re-ranking
  → Ads System:
     Parallel: Ad auction runs simultaneously
     Ads ranked by: bid × quality score × expected CTR
     Ads inserted into results
  → Response Assembly:
     Merge organic results + ads + knowledge panel + featured snippet + images + videos
     Personalized based on user history and location
  → Response → User (total time: typically 200-500ms)
```

---

## 4.4 Uber — Real-Time Systems Deep Dive

### Geospatial Indexing

**The Problem**: Given a rider's location, find the nearest available drivers in real-time among millions of active drivers.

**Google S2 Geometry**:
```
Earth's surface → Projected onto a cube → Each face divided into cells
Cell levels: Level 0 (face) → Level 30 (sub-centimeter)

Uber uses Level 12 cells (~3.3 km²) for initial driver indexing.

When rider requests:
1. Determine rider's S2 cell
2. Query that cell + neighboring cells for available drivers
3. Compute ETA (not straight-line distance — actual road distance via routing engine)
4. Rank drivers by ETA, driver rating, acceptance probability
5. Offer trip to best driver
```

**H3 (Uber's Hexagonal Grid)**:
- Uber later developed H3, a hexagonal hierarchical spatial index.
- Why hexagons > squares: Every neighbor is equidistant (squares have diagonal neighbors that are √2 farther).
- Used for: surge pricing, marketplace analysis, demand prediction.
- Open-sourced. Widely adopted in geospatial industry.

### Surge Pricing System

```
Every 2 minutes:
  1. Compute demand per H3 hexagon:
     Count: rider requests in last 5 minutes
  2. Compute supply per hexagon:
     Count: available drivers in hexagon + adjacent hexagons
  3. Supply-Demand Ratio → Surge Multiplier:
     ratio < 0.8 → surge 1.0x (no surge)
     ratio 0.8-0.6 → surge 1.3x
     ratio 0.6-0.4 → surge 1.8x
     ratio < 0.4 → surge 2.5x+ 
  4. ML model adjusts for:
     - Time of day, day of week
     - Weather (rain → higher demand)
     - Events (concert ending → demand spike)
     - Historical patterns
  5. Smoothing: Gradual increase/decrease to avoid oscillation
```

### Uber's Data Architecture

```
Events (1 trillion+/day):
  Trip events, driver location pings, payment events, rider actions
  → Apache Kafka (core event backbone)
  → Apache Flink (real-time processing):
    - Real-time ETA computation
    - Fraud detection
    - Surge pricing signals
  → HDFS/S3 (batch storage):
    - Apache Spark for batch analytics
    - ML model training data

Databases:
  - Schemaless (custom): Append-only on MySQL. Trip data.
  - Cassandra: Driver/rider profiles at scale.
  - MySQL (Vitess-like sharding): Transactional data.
  - Redis: Caching, rate limiting, session data.
  - Elasticsearch: Search (restaurants, places).
  - CacheFront: Custom CDN for dynamic content.

ML Platform (Michelangelo):
  - Feature store: Online (Redis) + Offline (Hive/HDFS)
  - Model training: Distributed TensorFlow, XGBoost
  - Model serving: Real-time predictions via gRPC
  - Experiment tracking: A/B testing framework
  - Models in production: ETA prediction, surge pricing, fraud detection, 
    driver-rider matching, food delivery time estimation
```

---

# PHASE 5: AI/ML SYSTEM DESIGN (Weeks 29–36)

---

## 5.1 Recommendation Systems — Production Architecture

### TikTok's Recommendation Engine (The Gold Standard)

TikTok's recommendation is considered the most advanced in the industry. It's why the app is so addictive.

```
For-You Page Algorithm:

1. Candidate Generation (millions → thousands):
   - Collaborative: Users with similar watch patterns
   - Content-based: Videos similar to ones you liked
   - Social: What your friends/contacts engage with
   - Trending: Regionally popular content
   - Creator: Diversify content sources

2. Pre-Ranking (thousands → hundreds):
   - Lightweight model (logistic regression or small NN)
   - Quick filter: Remove low-quality, policy violations
   - Score on: predicted watch time, like probability

3. Ranking (hundreds → tens):
   - Deep learning model with HUNDREDS of features:
    
   User features:
     - Demographics (age, gender, location)
     - Device (iPhone vs Android, screen size)
     - Interaction history (last 500 videos: watch time, likes, shares, comments)
     - Session context (time of day, day of week, session length)
     - Creator follows, hashtag preferences
   
   Video features:
     - Duration, resolution, audio features
     - Hashtags, captions (NLP embeddings)
     - Creator reputation, follower count
     - Video age, virality velocity
     - Visual features (CNN embeddings from frames)
     - Audio features (music, speech, effects)
   
   Cross features:
     - User-creator affinity
     - User-hashtag affinity
     - Similar-user engagement with this video
   
   Multi-task prediction heads:
     - P(watch to completion)    ← Most important
     - P(like)
     - P(comment)
     - P(share)
     - P(follow creator)
     - P(long dwell time)
     - Negative signals: P(not interested), P(report)
   
   Final score = Weighted combination of all predictions
   Weights tuned via A/B testing and long-term user retention metrics

4. Re-Ranking:
   - Diversity: Don't show same creator/song/hashtag consecutively
   - Freshness: Mix in new content to prevent filter bubbles
   - Explore/Exploit: ~10% exploration of unfamiliar content types
   - Business rules: Promote partner content, demote borderline content
   - Safety filters: Policy compliance check
```

**Key Insight**: TikTok optimizes for **watch time**, not likes. This better captures genuine interest (you might not "like" every video you enjoy watching).

### Pinterest — Visual Recommendation with PinSage

**PinSage** (published 2018): Graph neural network over billions of nodes.

```
Pinterest's Graph:
  Nodes: 3 billion+ pins (images)
  Edges: Pin saved to same board = connected
  
  PinSage generates embeddings for every pin:
  1. Random walk on graph to sample neighbors
  2. Aggregate neighbor features through GNN layers
  3. Each pin gets a 256-dim embedding
  
  Recommendation:
  "Find me pins similar to this one"
  → Nearest neighbor search on pin embeddings
  → ANN (Approximate Nearest Neighbor) using Faiss
  
  Result: Visual and conceptual similarity
  (A photo of a wooden deck → recommends deck designs, outdoor furniture, garden layouts)
```

### Amazon — E-commerce Recommendation

```
Amazon's recommendation generates 35%+ of total revenue.

"Customers who bought this also bought":
  - Item-to-item collaborative filtering
  - For each item pair, compute co-purchase frequency
  - Precomputed offline, stored in DynamoDB
  - Extremely fast at serving time (just a lookup)

"Recommended for you" (homepage):
  - User embedding from purchase/browse history
  - Two-tower model: User tower + Item tower
  - Candidate retrieval → Ranking → Business rules

"Frequently bought together" (cart page):
  - Association rule mining (Apriori-like but at scale)
  - Market basket analysis on billions of orders
  - Precomputed item bundles

Real-time signals:
  - "You viewed X 5 minutes ago" → boost related items
  - Price changes → re-rank based on price sensitivity
  - Stock levels → suppress out-of-stock items
```

### Feature Store — The ML Infrastructure Backbone

```
OFFLINE Feature Store:
  - Computed via batch pipelines (Spark, Airflow)
  - Stored in: S3, BigQuery, Hive
  - Examples:
    - user_avg_watch_time_30d: Average watch time over 30 days
    - item_popularity_score: Normalized popularity
    - user_genre_affinity_vector: 50-dim genre preference embedding
  - Updated: hourly or daily

ONLINE Feature Store:
  - Served at prediction time with <10ms latency
  - Stored in: Redis, DynamoDB, Bigtable
  - Examples:
    - user_last_5_interactions: Last 5 items user interacted with
    - user_current_session_length: How long user has been browsing
    - item_current_view_count: Real-time popularity signal
  - Updated: real-time via Kafka consumers

Feature Store Platforms:
  - Feast (open-source): Offline (BigQuery/S3) + Online (Redis/DynamoDB)
  - Tecton: Managed feature platform (founded by Uber ML team)
  - Hopsworks: Open-source, full feature pipeline
  - Uber Michelangelo: Custom-built, powers all Uber ML
  - Spotify Feature Hub: Custom-built for Spotify's ML models
```

---

## 5.2 LLM System Design — Production Guide

### LLM Serving Architecture

```
                    ┌────────────────────┐
                    │   API Gateway      │
                    │  (Rate limit, Auth)│
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │   Load Balancer    │
                    │  (Least connections)│
                    └─────────┬──────────┘
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
    ┌───────▼───────┐ ┌──────▼──────┐ ┌───────▼───────┐
    │ Inference      │ │ Inference   │ │ Inference      │
    │ Server 1       │ │ Server 2    │ │ Server 3       │
    │ (vLLM on 8xH100)│ │ (vLLM)     │ │ (vLLM)        │
    └───────┬───────┘ └──────┬──────┘ └───────┬───────┘
            │                 │                 │
    ┌───────▼───────┐        │         ┌───────▼───────┐
    │ KV Cache       │        │         │ KV Cache       │
    │ (PagedAttention)│       │         │ (PagedAttention)│
    └───────────────┘        │         └───────────────┘
                              │
                    ┌─────────▼──────────┐
                    │  Prompt Cache      │
                    │  (Redis/custom)    │
                    │  System prompts,   │
                    │  common prefixes   │
                    └────────────────────┘
```

### Key Optimization Deep Dives

**PagedAttention (vLLM's Innovation)**:

```
Problem: KV cache memory management is wasteful.
  Traditional: Pre-allocate contiguous memory per request for max sequence length.
  For max_seq_len=2048 and batch_size=32: Need 32 × 2048 × (key_size + value_size) pre-allocated.
  Most of this is wasted (requests don't reach max length).

PagedAttention:
  Inspired by OS virtual memory paging.
  KV cache divided into fixed-size "pages" (blocks).
  Pages allocated on-demand as tokens are generated.
  Non-contiguous pages → no fragmentation.
  Pages can be shared across requests with same prefix (system prompt).

Result: 2-4x higher throughput. Can serve 2-4x more concurrent requests per GPU.
```

**Speculative Decoding**:
```
Problem: LLM generation is sequential. Each token depends on all previous tokens.
  For a 100-token response: 100 sequential forward passes. Can't parallelize.

Solution:
  1. Small "draft" model (e.g., 1B parameters) quickly generates N draft tokens.
  2. Large target model (e.g., 70B parameters) verifies ALL N tokens in ONE parallel forward pass.
  3. Accept all tokens up to first disagreement. Reject the rest.
  4. Repeat.

Example:
  Draft model generates: "The quick brown fox jumps over the lazy dog"
  Target model verifies in one pass: "The quick brown fox jumps over the" ✓✓✓✓✓✓✓
  "lazy" ✗ (target would say "sleeping")
  Accept first 7 tokens, reject rest, continue from "over the"

Speedup: 2-3x with good draft model. Works best when draft model agrees frequently.
```

**Quantization**:
```
FP32 (32-bit float): Full precision. 4 bytes per parameter.
  70B model = 280 GB. Needs 4+ A100 80GB GPUs.

FP16 (16-bit float): Half precision. 2 bytes per parameter.  
  70B model = 140 GB. Needs 2+ A100 80GB GPUs.
  Minimal quality loss. Standard for inference.

INT8 (8-bit integer): 1 byte per parameter.
  70B model = 70 GB. Fits on 1 A100 80GB!
  Small quality loss. GPTQ, SmoothQuant.

INT4 (4-bit integer): 0.5 bytes per parameter.
  70B model = 35 GB. Fits on 1 A100 40GB!
  Noticeable quality loss for complex reasoning.
  AWQ, GPTQ-4bit, GGUF (llama.cpp).

Methods:
  - GPTQ: Post-training quantization. One-shot calibration.
  - AWQ: Activation-aware quantization. Protects important weights.
  - GGUF: Format for llama.cpp. CPU-friendly quantization.
  - bitsandbytes: Easy integration with HuggingFace. QLoRA uses 4-bit base + LoRA adapters.
```

### RAG (Retrieval-Augmented Generation) — Production Architecture

```
                    ┌─────────────────────────────┐
                    │     OFFLINE INDEXING          │
                    │                               │
                    │  Documents → Chunking          │
                    │    → Embedding Model           │
                    │    → Vector DB (Pinecone)      │
                    │    → BM25 Index (Elasticsearch)│
                    └──────────────┬────────────────┘
                                   │
┌──────────────────────────────────▼────────────────────────────────┐
│                      ONLINE QUERY PIPELINE                        │
│                                                                    │
│  User Query                                                        │
│    → Query Understanding:                                          │
│       - Query classification (needs retrieval? or general knowledge?)│
│       - Query expansion (add synonyms, related terms)              │
│       - HyDE: Generate hypothetical answer → embed THAT            │
│                                                                    │
│    → Hybrid Retrieval:                                             │
│       - Vector search: Query embedding → cosine similarity → Top 20│
│       - Keyword search: BM25 on Elasticsearch → Top 20            │
│       - Reciprocal Rank Fusion: Merge both result sets             │
│                                                                    │
│    → Reranking:                                                    │
│       - Cross-encoder model (e.g., bge-reranker-v2-m3)            │
│       - Scores query-document pairs with full attention            │
│       - Much more accurate than bi-encoder (embedding) similarity  │
│       - Select Top 5-10 chunks                                     │
│                                                                    │
│    → Context Assembly:                                             │
│       - Concatenate retrieved chunks                               │
│       - Add metadata (source, date, relevance score)               │
│       - Respect context window limits                              │
│                                                                    │
│    → LLM Generation:                                               │
│       - System prompt with instructions                            │
│       - Retrieved context                                          │
│       - User query                                                 │
│       - Generate answer with citations                             │
│                                                                    │
│    → Post-processing:                                              │
│       - Citation verification (do cited chunks support claims?)    │
│       - Hallucination detection                                    │
│       - Source attribution                                         │
└────────────────────────────────────────────────────────────────────┘
```

**Chunking Strategies**:
| Strategy | How | Pros | Cons |
|----------|-----|------|------|
| Fixed-size | 512 tokens with 50 token overlap | Simple, predictable | May split mid-sentence |
| Sentence-based | Split on sentence boundaries | Coherent chunks | Variable sizes |
| Semantic | Use embedding similarity to detect topic shifts | Best coherence | Expensive, complex |
| Recursive | Split by paragraph → sentence → character until target size | Good balance | Implementation complexity |
| Document-structure | Split by headers, sections, pages | Preserves document structure | Requires structured docs |

**Advanced RAG Patterns**:
- **Multi-Query RAG**: Generate 3-5 query variations → retrieve for each → merge results. Improves recall.
- **Parent Document Retrieval**: Index small chunks but retrieve parent (larger) chunks for context.
- **Self-RAG**: LLM decides whether it needs retrieval. Retrieves only when unsure.
- **Corrective RAG (CRAG)**: After retrieval, evaluate relevance. If low → web search fallback.
- **Graph RAG**: Build knowledge graph from documents → traverse graph for multi-hop reasoning.

### Agentic Systems — Production Architecture

```
┌────────────────────────────────────────────────────────────┐
│                    AGENT ORCHESTRATOR                        │
│                                                              │
│  User Query → [Planning LLM]                                 │
│    "Book me a flight from Delhi to NYC next Friday"          │
│                                                              │
│  Plan:                                                       │
│    1. Search flights (Flight API tool)                        │
│    2. Check user preferences (Memory/Profile tool)           │
│    3. Compare options (Reasoning)                            │
│    4. Book selected flight (Booking API tool)                │
│    5. Send confirmation (Email tool)                         │
│                                                              │
│  Execution Loop (ReAct):                                     │
│    Thought: Need to search flights Delhi → NYC, Friday       │
│    Action: flight_search(from="DEL", to="JFK", date="...")   │
│    Observation: [3 flights found: AI101, UA83, DL19]         │
│    Thought: User prefers direct flights (from memory)        │
│    Action: get_user_preference("flight_preference")          │
│    Observation: {airline: "any", class: "economy", stops: 0} │
│    Thought: AI101 is direct, 18h, $850. Best match.         │
│    Action: book_flight(flight="AI101", class="economy")      │
│    Observation: Booking confirmed. PNR: ABC123               │
│    Action: send_email(to=user, subject="Flight Booked", ...) │
│    Final Answer: "Booked! AI101 DEL→JFK, Friday. PNR: ABC123│
│                                                              │
│  Safety Layer:                                               │
│    - Tool call validation (is this tool call reasonable?)    │
│    - Cost guardrails (don't exceed budget)                   │
│    - Human-in-the-loop for high-stakes actions (payments)    │
│    - Max iterations limit (prevent infinite loops)           │
│    - Output content filtering                                │
└────────────────────────────────────────────────────────────┘
```

**Agent Memory Architecture**:
```
Short-Term Memory:
  - Current conversation history (in context window)
  - Working memory (scratchpad for intermediate results)
  
Long-Term Memory:
  - Vector store of past interactions
  - Structured knowledge base (user preferences, facts)
  - Episodic memory: "Last time user asked about flights, they preferred..."
  
Procedural Memory:
  - Learned tool usage patterns
  - Successful action sequences for common tasks
  - Error recovery strategies
```

**MCP (Model Context Protocol)**:
- Anthropic's standard for LLM ↔ Tool communication.
- Defines how tools expose their capabilities, inputs, outputs.
- Enables plug-and-play tool integration.
- Rapidly becoming an industry standard.

---

# PHASE 6: INFRASTRUCTURE & DEVOPS (Weeks 37–42)

---

## 6.1 Kubernetes Deep Dive

### Core Concepts

```
Cluster:
  ├── Control Plane:
  │   ├── API Server (all K8s commands go through here)
  │   ├── etcd (distributed KV store, cluster state, Raft consensus)
  │   ├── Scheduler (assigns pods to nodes based on resources, affinity)
  │   └── Controller Manager (ensures desired state = actual state)
  │
  └── Worker Nodes:
      ├── Kubelet (agent on each node, manages pods)
      ├── Container Runtime (containerd, CRI-O)
      ├── Kube-proxy (networking, service discovery)
      └── Pods:
          ├── Pod 1: [Container A, Container B (sidecar)]
          ├── Pod 2: [Container C]
          └── Pod 3: [Container D]
```

**Key Resources**:

| Resource | Purpose | Example |
|----------|---------|---------|
| **Pod** | Smallest unit. One or more containers sharing network/storage. | Single instance of your app. |
| **Deployment** | Manages pod replicas. Rolling updates, rollbacks. | "Run 3 replicas of my web server." |
| **Service** | Stable network endpoint for pods (which are ephemeral). | ClusterIP (internal), LoadBalancer (external), NodePort. |
| **Ingress** | HTTP routing rules. Path/host-based routing to services. | `/api` → api-service, `/web` → web-service. |
| **StatefulSet** | For stateful workloads. Stable names, persistent storage. | Databases, Kafka brokers, Elasticsearch nodes. |
| **DaemonSet** | Run one pod per node. | Log collectors (Fluentd), monitoring agents (Datadog). |
| **CronJob** | Scheduled tasks. | Nightly data cleanup, weekly report generation. |
| **ConfigMap / Secret** | Configuration and sensitive data. | DB connection strings, API keys. |
| **HPA** | Horizontal Pod Autoscaler. Scale replicas based on metrics. | Scale from 3 → 20 pods when CPU > 70%. |
| **VPA** | Vertical Pod Autoscaler. Adjust resource requests. | Increase pod memory from 512MB → 1GB. |
| **PVC** | Persistent Volume Claim. Request storage. | 100GB SSD for database. |

### Deployment Strategies in K8s

**Rolling Update** (default):
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # Allow 25% extra pods during update
      maxUnavailable: 25%   # Allow 25% to be unavailable
```

**Canary with Istio Service Mesh**:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
spec:
  http:
  - route:
    - destination:
        host: my-service
        subset: stable
      weight: 95          # 95% to current version
    - destination:
        host: my-service
        subset: canary
      weight: 5           # 5% to new version
```

---

## 6.2 Observability — The Three Pillars

### Metrics (Prometheus + Grafana)

**The RED Method** (for request-driven services):
- **Rate**: Requests per second
- **Errors**: Error rate (HTTP 5xx / total requests)
- **Duration**: Latency distribution (p50, p95, p99)

**The USE Method** (for infrastructure):
- **Utilization**: CPU usage %, memory usage %, disk I/O %
- **Saturation**: Queue length, thread pool saturation
- **Errors**: Hardware errors, kernel errors

**SLOs (Service Level Objectives)**:
```
SLI (Indicator): p99 latency of /api/search requests
SLO (Objective): p99 latency < 200ms for 99.9% of time
SLA (Agreement): If SLO breached for >4 hours/month, customer gets credits

Error Budget: 100% - 99.9% = 0.1% allowed downtime = 43.2 minutes/month
  If you've used 30 minutes of error budget → slow down deployments
  If you have budget remaining → deploy faster, take more risks
```

**Google's SRE approach**: Measure SLIs → Set SLOs → Calculate error budget → Error budget drives deployment velocity.

### Distributed Tracing

```
User Request → API Gateway (Trace ID: abc-123)
  → Auth Service (Span 1: 5ms)
  → User Service (Span 2: 12ms)
    → PostgreSQL query (Span 2.1: 8ms)
    → Redis cache check (Span 2.2: 0.5ms)
  → Recommendation Service (Span 3: 45ms)
    → Feature Store lookup (Span 3.1: 3ms)
    → ML Model inference (Span 3.2: 38ms)
  → Response assembly (Span 4: 2ms)

Total: 64ms. Bottleneck clearly visible: ML inference at 38ms.
```

**Tools**: OpenTelemetry (standard), Jaeger (open-source), Zipkin, Datadog APT, AWS X-Ray.

---

# PHASE 7: ADVANCED PATTERNS (Weeks 43–48)

---

## 7.1 Data Architecture Patterns

### Lambda Architecture

```
              ┌─────────────────┐
              │   Raw Data       │
              │  (Events, Logs)  │
              └────────┬────────┘
                       │
          ┌────────────┼────────────┐
          │                         │
  ┌───────▼───────┐     ┌──────────▼──────────┐
  │ Batch Layer    │     │ Speed Layer          │
  │ (Spark, Hive)  │     │ (Flink, Kafka Streams)│
  │ Complete,      │     │ Approximate,          │
  │ accurate       │     │ real-time              │
  │ Hours latency  │     │ Seconds latency        │
  └───────┬───────┘     └──────────┬──────────┘
          │                         │
  ┌───────▼─────────────────────────▼───────┐
  │          Serving Layer                   │
  │  Merge batch results + speed results     │
  │  (Druid, Elasticsearch, custom)          │
  └──────────────────────────────────────────┘
```

**LinkedIn**: Uses Lambda for analytics. Batch (Spark on HDFS) provides accurate daily numbers. Speed (Kafka Streams) provides real-time approximate counts. Serving layer merges both.

### Kappa Architecture

```
All data → Kafka (single source of truth, immutable log)
  → Stream Processing (Flink / Kafka Streams)
  → Serving Layer

Reprocessing: Replay Kafka from beginning with new logic.
No batch layer needed!
```

**Simpler but requires**: Kafka retention long enough for reprocessing. Stream processing powerful enough for all computations.

### Data Mesh

```
Instead of centralized data team owning one data warehouse:

Domain A (Orders Team):
  - Owns order data
  - Publishes "Orders Data Product" with SLOs, schema, docs
  - Self-serve infrastructure

Domain B (Users Team):
  - Owns user data
  - Publishes "Users Data Product"
  
Domain C (Payments Team):
  - Owns payment data
  - Consumes Orders + Users data products as needed

Central Platform Team:
  - Provides self-serve data infrastructure
  - Standardized tooling, compute, governance
  - NOT responsible for data quality (domains are)
```

**Principles**: Domain ownership, Data as a product, Self-serve platform, Federated governance.
**Adopted by**: Zalando, Intuit, JPMorgan, Thoughtworks clients.

---

## 7.2 AI-Native Architecture Patterns (2024–2025)

### AI Gateway / Router

```
Application → AI Gateway → Route to best model:
  │
  ├── Simple queries → Claude Haiku (cheap, fast)
  ├── Complex reasoning → Claude Opus (expensive, smart)
  ├── Code generation → Specialized code model
  ├── Image understanding → Vision model
  └── Fallback: If primary fails → secondary provider
  
Features:
  - Cost optimization: Route by complexity
  - Latency optimization: Choose fastest available model
  - Rate limit management: Distribute across providers
  - Caching: Cache identical prompts
  - Monitoring: Token usage, latency, error rates per model
  - Guardrails: Input/output filtering

Tools: LiteLLM, Portkey, Martian, custom routers
```

### Voice AI Pipeline

```
User speaks → 
  1. VAD (Voice Activity Detection): Silero VAD
     Detect when user starts/stops speaking
  
  2. ASR (Automatic Speech Recognition):
     Options: Whisper (OpenAI), Deepgram, AssemblyAI, Google STT
     Streaming ASR for real-time (partial results as user speaks)
     Latency target: <300ms from end of speech to transcript
  
  3. LLM Processing:
     Transcript → LLM (Claude, GPT-4, or fine-tuned model)
     Streaming response (token by token)
     Latency target: First token in <500ms
  
  4. TTS (Text-to-Speech):
     Options: ElevenLabs, PlayHT, XTTS, CosyVoice, Cartesia
     Streaming TTS: Start speaking as soon as first sentence ready
     Latency target: First audio chunk in <200ms from first LLM token
  
  5. Audio playback to user

Total pipeline latency target: <1 second from user stops speaking to AI starts responding

Key optimizations:
  - Streaming everything: ASR streams → LLM streams → TTS streams
  - Sentence buffering: Send complete sentences to TTS (better prosody)
  - Interruption handling: User interrupts → stop TTS, process new input
  - WebSocket: Persistent connection for low-latency bidirectional audio
  - Turn detection: ML model to detect conversational turn-taking
```

---

# PHASE 8: PRACTICE & INTERVIEW PREPARATION

---

## 8.1 System Design Interview Framework (Detailed)

### Step 1: Requirements Gathering (5 min)

**Functional Requirements** — What does the system DO?
```
"Design a URL shortener"
- Can users create short URLs? → Yes
- Do short URLs expire? → Optional, user can set TTL
- Can users see analytics (click counts)? → Yes, basic analytics
- Custom short URLs? → Nice to have
- API or web UI? → Both
```

**Non-Functional Requirements** — HOW WELL does it do it?
```
- Scale: How many URLs created/day? → 100 million
- Read:Write ratio? → 100:1 (read-heavy)
- Latency: Redirect latency? → <100ms
- Availability: How critical? → 99.99% (high availability, AP)
- Durability: Can we lose URLs? → No, must be durable
- Security: Protection against abuse? → Rate limiting, spam detection
```

**Back-of-Envelope Estimation**:
```
Writes: 100M URLs/day = ~1,200/sec
Reads: 100:1 ratio = 10B reads/day = ~120,000/sec
Storage: 100M/day × 365 days × 5 years × 500 bytes = ~91 TB
Bandwidth: 120,000 reads/sec × 500 bytes = 60 MB/s (manageable)
Cache: 20% of URLs serve 80% of traffic
  → Cache 20% of 91TB × active period URLs ≈ few TB in Redis
```

### Step 2: High-Level Design (10 min)

Draw the architecture, identify components, define APIs, data model.

### Step 3: Deep Dive (20 min)

Pick 2-3 areas to deep dive based on the problem's unique challenges.

### Step 4: Trade-offs & Extensions (5 min)

Discuss what you'd improve, trade-offs you made, monitoring approach.

---

## 8.2 Essential Reading List

### Must-Read Papers
| Paper | By | Key Concept |
|-------|-----|------------|
| MapReduce | Google (2004) | Distributed batch processing |
| GFS | Google (2003) | Distributed file system |
| BigTable | Google (2006) | Wide-column store |
| Dynamo | Amazon (2007) | Eventually consistent KV store, consistent hashing |
| Cassandra | Facebook (2009) | Dynamo + BigTable hybrid |
| Spanner | Google (2012) | Globally consistent distributed DB |
| Kafka | LinkedIn (2011) | Distributed commit log |
| Raft | Stanford (2014) | Understandable consensus |
| Scaling Memcache at Facebook | Facebook (2013) | Caching at scale, thundering herd |
| TAO | Facebook (2013) | Graph-aware cache, social graph at scale |
| Zanzibar | Google (2019) | Global authorization system |
| Borg | Google (2015) | Container orchestration (predecessor to K8s) |
| Dapper | Google (2010) | Distributed tracing |
| Deep Neural Networks for YouTube | YouTube (2016) | Production recommendation system |
| PinSage | Pinterest (2018) | Graph neural networks at billion scale |
| Attention Is All You Need | Google (2017) | Transformer architecture |
| Scaling Laws for Neural LMs | OpenAI (2020) | How model performance scales with compute |
| FAISS | Meta (2024) | Billion-scale vector similarity search |
| Efficient Memory Management for LLMs (vLLM) | UC Berkeley (2023) | PagedAttention |

### Books (Priority Order)
1. **"Designing Data-Intensive Applications"** — Martin Kleppmann. THE system design bible. Covers replication, partitioning, consistency, batch/stream processing.
2. **"System Design Interview Vol 1 & 2"** — Alex Xu. Practical design walkthroughs.
3. **"Site Reliability Engineering"** — Google SRE Book. Free online. Monitoring, on-call, error budgets.
4. **"Building Microservices"** — Sam Newman. Microservice patterns and practices.
5. **"Machine Learning System Design"** — Chip Huyen. ML in production.
6. **"Database Internals"** — Alex Petrov. How databases actually work under the hood.
7. **"Understanding Distributed Systems"** — Roberto Vitillo. Clear explanation of distributed patterns.

### Blogs to Follow
- Netflix Tech Blog — Architecture, ML, chaos engineering
- Uber Engineering — Real-time systems, ML platforms, data infrastructure
- Spotify Engineering — Recommendations, backend, data engineering
- Meta Engineering — Infrastructure, AI, social graph
- LinkedIn Engineering — Kafka, data systems, ML
- Cloudflare Blog — CDN, networking, edge computing, DDoS
- ByteByteGo Newsletter — Weekly system design summaries (Alex Xu)
- The Morning Paper — CS paper summaries (Adrian Colyer)
- High Scalability — Architecture case studies

---

## Quick Reference: Complete Decision Matrix

### Database Selection
```
Need ACID + complex joins + well-understood?     → PostgreSQL
Need ACID + massive scale + MySQL compatible?    → Vitess (MySQL sharding)
Need global strong consistency?                  → CockroachDB or Google Spanner
Need flexible schema + fast iteration?           → MongoDB
Need massive write throughput + linear scale?    → Cassandra or ScyllaDB
Need serverless + single-digit ms + AWS native?  → DynamoDB
Need sub-ms reads + complex data structures?     → Redis
Need full-text search + analytics?               → Elasticsearch
Need graph traversals + relationships?           → Neo4j
Need time-series + metrics + IoT?                → TimescaleDB or InfluxDB
Need vector similarity + embeddings?             → Pinecone / pgvector / Milvus
Need offline-first + sync?                       → CouchDB / PouchDB
```

### Communication Protocol Selection
```
Standard client-server CRUD API?                 → REST
Mobile apps needing flexible queries?            → GraphQL
Internal microservice ↔ microservice (perf)?     → gRPC
Real-time bidirectional (chat, gaming)?          → WebSocket
Server → client streaming (LLM tokens)?          → SSE
Async event-driven decoupled?                    → Kafka
Task queue with routing?                         → RabbitMQ
```

### Caching Strategy Selection
```
General purpose (most apps)?                     → Cache-Aside + Redis + TTL
Read-heavy, rarely updated?                      → Read-Through + long TTL + CDN
Write-heavy, need consistency?                   → Write-Through
Write-heavy, eventual consistency OK?            → Write-Behind (batch flush)
Write-heavy, rarely read after?                  → Write-Around
User sessions?                                   → Redis with TTL
Static assets (images, JS, CSS)?                 → CDN with versioned URLs
API responses?                                   → API Gateway cache + Redis
Database query results?                          → Application-level cache (Redis)
ML model predictions?                            → Feature store online cache
```

### ML Model Serving Selection
```
Real-time predictions (<100ms)?                  → Online serving (Triton, TorchServe)
Batch processing (millions of items)?            → Batch (Spark ML, AWS Batch)
Edge/mobile deployment?                          → TF Lite, CoreML, ONNX Runtime
LLM serving (high throughput)?                   → vLLM, TensorRT-LLM, TGI
LLM serving (multi-model)?                       → AI Gateway (LiteLLM, custom)
Streaming predictions?                           → Flink + embedded model
```

---

*Estimated study time: 48 weeks at 10-15 hours/week. Adjust based on your level. The single most important skill: understanding WHY each technology exists and what trade-offs it makes. Build things. Read papers. Study production architectures. That's how you truly learn system design.*
