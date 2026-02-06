# Complete System Design Roadmap — From Zero to Staff Engineer Level

---

## Phase 1: Foundations (Weeks 1–4)

### 1.1 Core Concepts

**Scalability** — Vertical (bigger machine) vs Horizontal (more machines). Every system you'll study uses horizontal scaling. Netflix runs 100,000+ instances on AWS. Google Search handles 8.5 billion queries/day — no single server can do that.

**Latency vs Throughput** — Latency is how fast one request completes. Throughput is how many requests per second. Spotify targets <200ms for song playback start. Google targets <500ms for search results. These two often trade off against each other.

**Availability vs Consistency (CAP Theorem)** — You can only guarantee two of three: Consistency, Availability, Partition Tolerance. In practice, partitions always happen, so you choose between CP (banking systems) and AP (social media feeds). Netflix chooses AP — a slightly stale recommendation is better than no recommendation.

**PACELC Extension** — When there's no partition, you still choose between Latency and Consistency. DynamoDB (used by Amazon) chooses ELC — eventual consistency for lower latency during normal operations.

### 1.2 Networking Essentials

- **DNS** — How netflix.com resolves to an IP. Netflix uses Route 53 with latency-based routing.
- **CDN** — Netflix Open Connect: custom CDN appliances placed inside ISPs. Serves 95%+ of traffic.
- **Load Balancers** — L4 (TCP/UDP) vs L7 (HTTP). Google uses Maglev (custom L4 LB handling 10M+ packets/sec per machine).
- **Reverse Proxy** — Nginx, Envoy. Spotify uses Envoy as service mesh proxy.
- **API Gateway** — Rate limiting, auth, routing. Kong, AWS API Gateway. Netflix Zuul → Spring Cloud Gateway.

### 1.3 Communication Protocols

| Protocol | Use Case | Industry Example |
|----------|----------|-----------------|
| REST | Standard APIs | Spotify Public API |
| GraphQL | Flexible queries, mobile-first | GitHub API v4, Netflix (internal) |
| gRPC | Low-latency microservices | Google internal (Stubby → gRPC) |
| WebSocket | Real-time bidirectional | Slack, Discord, Uber driver tracking |
| SSE | Server push, one-way streaming | ChatGPT token streaming |
| Kafka/Event Streams | Async decoupled communication | LinkedIn (invented Kafka), Uber |

---

## Phase 2: Data Layer Deep Dive (Weeks 5–10)

### 2.1 Database Selection Guide

#### Relational (SQL)
- **PostgreSQL** — Most versatile. Used by Instagram (scaled to billions of rows with sharding), Notion, Supabase.
- **MySQL/Vitess** — YouTube (Vitess was built by Google to shard MySQL for YouTube). PlanetScale uses Vitess.
- **When to use**: ACID transactions, complex joins, well-defined schema. Banking, e-commerce orders, user accounts.

#### Document Store (NoSQL)
- **MongoDB** — Flexible schema. Used by Uber (trip data), Coinbase.
- **DynamoDB** — Serverless, single-digit ms latency. Amazon.com's shopping cart, Lyft.
- **When to use**: Rapidly evolving schema, nested/hierarchical data, high write throughput.

#### Wide-Column Store
- **Cassandra** — Netflix (viewing history, 30+ PB), Discord (messages, trillions of rows), Apple (400,000+ instances).
- **HBase** — Facebook Messenger (was used for message storage).
- **When to use**: Massive write throughput, time-series-like access patterns, no complex queries.

#### Graph Database
- **Neo4j** — Social networks, fraud detection. Used by eBay, NASA.
- **Amazon Neptune** — Knowledge graphs at AWS.
- **When to use**: Highly connected data — social graphs, recommendation engines, fraud rings.

#### Key-Value Store
- **Redis** — Session storage, caching, leaderboards. Twitter (timeline caching), GitHub, Snapchat.
- **Memcached** — Simple caching. Facebook uses both Redis and Memcached (TAO cache).
- **When to use**: Sub-millisecond reads, simple lookups, caching, rate limiting.

#### Time-Series Database
- **InfluxDB / TimescaleDB** — Monitoring, IoT. Tesla vehicle telemetry.
- **When to use**: Metrics, logs, sensor data, financial tick data.

#### Search Engine
- **Elasticsearch** — Full-text search, log analytics. Netflix (search), Uber (trip search), Wikipedia.
- **When to use**: Full-text search, fuzzy matching, log aggregation (ELK stack).

#### Vector Database
- **Pinecone / Weaviate / Milvus / pgvector** — Semantic search, RAG, recommendations.
- **When to use**: Embedding similarity search, LLM-powered applications, recommendation systems.

### 2.2 Database Scaling Patterns

**Indexing** — B-tree (default for most DBs), Hash index (exact match), GIN/GiST (PostgreSQL full-text/geo), LSM-tree (Cassandra, RocksDB — write-optimized).

**Replication**
- Single-leader: One write node, multiple read replicas. PostgreSQL streaming replication.
- Multi-leader: Multiple write nodes. CockroachDB, Cassandra.
- Leaderless: Quorum reads/writes. DynamoDB, Cassandra.

**Sharding (Partitioning)**
- Hash-based: Consistent hashing (DynamoDB, Cassandra). Avoids hotspots.
- Range-based: Good for time-series. HBase uses range partitioning.
- Directory-based: Lookup table maps keys → shards. More flexible but single point of failure.
- **Instagram's sharding**: Sharded PostgreSQL by user ID using consistent hashing. Each shard is a PostgreSQL instance.

**Denormalization** — Store redundant data to avoid joins at read time. YouTube denormalizes video metadata into Vitess for fast reads.

### 2.3 Caching — The Complete Guide

#### Cache Strategies

| Strategy | How It Works | Best For |
|----------|-------------|----------|
| **Cache-Aside (Lazy)** | App checks cache → miss → read DB → populate cache | General purpose. Most common. Reddit, GitHub. |
| **Read-Through** | Cache itself fetches from DB on miss | Simplified app code. DynamoDB Accelerator (DAX). |
| **Write-Through** | Write to cache AND DB simultaneously | Strong consistency needed. Banking systems. |
| **Write-Behind (Write-Back)** | Write to cache, async flush to DB | High write throughput. Risk of data loss. |
| **Write-Around** | Write to DB only, cache populated on read | Write-heavy, rarely-read data. Log ingestion. |

#### Eviction Policies
- **LRU** (Least Recently Used) — Most common. Redis default.
- **LFU** (Least Frequently Used) — Better for skewed access (popular vs long-tail).
- **TTL** (Time-To-Live) — Expire after fixed time. DNS caching, session tokens.

#### Multi-Level Caching (Real-World Architecture)
```
Browser Cache (HTTP headers, Service Workers)
    → CDN Edge Cache (Netflix Open Connect, CloudFront)
        → API Gateway Cache
            → Application-Level Cache (Redis/Memcached)
                → Database Query Cache
                    → OS Page Cache / Buffer Pool
```

**Facebook's Caching Stack**: Clients → CDN → TAO (graph-aware cache) → Memcached → MySQL. TAO handles 10 billion+ reads/sec.

**Netflix's EVCache**: Distributed Memcached layer. Serves 30 million requests/sec. Every microservice at Netflix reads from EVCache before hitting Cassandra.

#### Cache Invalidation Patterns
- **Event-driven invalidation**: Kafka event on data change → consumer invalidates cache. Used by Uber.
- **Version-based**: Append version to cache key. New version = new key = old cache auto-expires.
- **Lease-based**: Memcached at Facebook uses "leases" to prevent thundering herd on cache miss.

---

## Phase 3: Distributed Systems Patterns (Weeks 11–16)

### 3.1 Consistency Patterns

- **Strong Consistency** — After a write, all reads see the latest value. Google Spanner uses TrueTime (atomic clocks + GPS) for global strong consistency.
- **Eventual Consistency** — Reads may return stale data temporarily. DynamoDB, Cassandra. DNS is eventually consistent.
- **Causal Consistency** — If A causes B, everyone sees A before B. MongoDB supports causal sessions.
- **Read-Your-Writes** — You always see your own writes immediately. Session affinity or sticky sessions achieve this.

### 3.2 Consensus Algorithms
- **Raft** — etcd (Kubernetes control plane), CockroachDB, HashiCorp Consul.
- **Paxos** — Google Chubby (distributed lock service), Spanner.
- **ZAB** — Apache ZooKeeper (used by Kafka for controller election).

### 3.3 Distributed Patterns

- **Saga Pattern** — Distributed transactions via sequence of local transactions + compensating actions. Uber's trip lifecycle uses sagas.
- **CQRS** — Command Query Responsibility Segregation. Separate read/write models. LinkedIn feed: writes go to Kafka → read-optimized stores.
- **Event Sourcing** — Store events, not state. Replay to reconstruct. Used in financial systems, audit trails.
- **Circuit Breaker** — Stop calling failing services. Netflix Hystrix (now Resilience4j).
- **Bulkhead** — Isolate failures. Separate thread pools per dependency.
- **Sidecar / Service Mesh** — Envoy proxy as sidecar. Istio (Google/Lyft). Handles retries, tracing, mTLS.

### 3.4 Message Queues & Streaming

| System | Model | Industry Example |
|--------|-------|-----------------|
| **Apache Kafka** | Distributed log, replay, ordered | LinkedIn (created it), Uber (1 trillion+ events/day), Netflix |
| **RabbitMQ** | Traditional message broker, routing | Robinhood, Mozilla |
| **Amazon SQS** | Managed queue, at-least-once | AWS-native workloads |
| **Apache Pulsar** | Multi-tenant, geo-replication | Yahoo (created it), Splunk |
| **Redis Streams** | Lightweight streaming | Real-time leaderboards, chat |

**Kafka Deep Dive**: Topics → Partitions → Consumer Groups. Partitions enable parallelism. Key-based partitioning ensures order per key. Uber routes trip events by trip_id to ensure ordered processing.

### 3.5 Rate Limiting Algorithms

| Algorithm | How It Works | Used By |
|-----------|-------------|---------|
| **Token Bucket** | Tokens added at fixed rate, consumed per request. Allows bursts. | AWS API Gateway, Stripe |
| **Leaky Bucket** | Requests drain at constant rate. Smooths bursts. | Nginx |
| **Sliding Window Log** | Track timestamps of requests. Exact but memory-heavy. | — |
| **Sliding Window Counter** | Approximation using current + weighted previous window. | Cloudflare |
| **Fixed Window** | Count requests per time window. Simple but edge-case bursts. | Basic implementations |

---

## Phase 4: Real-World System Designs (Weeks 17–28)

### 4.1 Netflix — Video Streaming Platform

**Architecture Overview:**
```
Client → CDN (Open Connect) → API Gateway (Zuul/Spring Cloud Gateway)
  → Microservices (500+) → EVCache → Cassandra/MySQL
  → Kafka → Real-time/Batch Processing → S3
```

**Key Components:**
- **Content Ingestion**: Videos encoded into 1000+ different formats/bitrates (H.264, VP9, AV1) using AWS EC2. Stored in S3.
- **Open Connect CDN**: Custom hardware appliances deployed inside ISP networks. Handles 95%+ of traffic. Pre-positions popular content based on ML predictions.
- **Adaptive Bitrate Streaming (ABR)**: Client measures bandwidth → requests appropriate quality chunk. DASH protocol.
- **Microservices**: 500+ microservices. Each team owns their service. Deployed on AWS using Titus (custom container runtime).
- **Data Pipeline**: Kafka → Apache Flink (real-time) + Spark (batch) → Data warehouse for analytics and ML training.
- **Chaos Engineering**: Chaos Monkey (random instance termination), Chaos Kong (region failure simulation). Ensures resilience.

**Netflix Recommendation System:**
- **Collaborative Filtering**: Users who watched X also watched Y.
- **Content-Based Filtering**: Analyze genres, actors, directors per user taste profile.
- **Deep Learning**: Wide & Deep networks. Two-tower models for candidate retrieval.
- **Contextual Bandits**: For artwork personalization — different users see different thumbnails for the same show. A/B testing at massive scale.
- **Architecture**: Candidate Generation (cheap model, 1000s of candidates) → Ranking (expensive model, top 100) → Re-ranking (business rules, diversity) → Presentation.

### 4.2 Spotify — Music Streaming & Discovery

**Architecture Overview:**
- Migrated from on-prem to **Google Cloud Platform** (2018).
- 1000+ microservices, primarily Java/Python.
- **Backend**: Java microservices using their custom "Apollo" framework (now largely Spring Boot).
- **Data**: Google BigQuery (analytics), Bigtable (low-latency), Cloud Dataflow (stream processing).

**Spotify's Recommendation & Discovery Engine:**
- **Discover Weekly / Release Radar**: Core discovery features, updated weekly.
- **Collaborative Filtering**: Matrix factorization on user-track interaction matrix. If User A and User B have similar listening patterns, recommend tracks User B liked to User A.
- **NLP on Playlists**: Analyze playlist titles and descriptions using Word2Vec/BERT. "Chill Vibes" playlist → learn what "chill" means musically.
- **Audio Analysis (CNN)**: Raw audio → Convolutional Neural Network → audio embeddings. Captures tempo, key, energy, "mood." Crucial for new/unpopular songs (cold-start problem).
- **Knowledge Graph**: Artists → genres → moods → events. Enables "Because you listened to X" explanations.
- **Two-Tower Retrieval**: User tower (listening history embedding) + Item tower (song embedding) → dot product similarity → top candidates.
- **Reinforcement Learning**: Optimize for long-term engagement, not just clicks. Bandits for homepage slot optimization.

**Audio Streaming:**
- **Ogg Vorbis** (free tier) and **AAC** (premium) formats.
- Pre-fetches next song while current is playing.
- Offline mode: Encrypted downloads with DRM, synced via protobuf.

### 4.3 Google Search — The OG Distributed System

**Architecture:**
```
Query → DNS → Google Frontend (GFE)
  → Web Server → Index Serving (Bigtable/SSTable)
  → Ranking (1000+ signals) → Ads Auction → Response Assembly
```

**Key Systems:**
- **Crawling**: Googlebot. Caffeine indexer processes pages in near-real-time.
- **Indexing**: Inverted index stored in custom SSTable format on GFS/Colossus (successor to GFS).
- **Serving**: Query hits multiple index shards in parallel. Each shard returns candidates. Merged and ranked centrally.
- **Ranking Evolution**: PageRank (1998) → RankBrain (2015, ML-based) → BERT (2019, NLU) → MUM (2021, multimodal) → Gemini integration (2024+, LLM-powered).
- **Infrastructure**: Borg (container orchestration, predecessor to Kubernetes), Spanner (globally consistent DB), BigTable (NoSQL), MapReduce → Flume → Dataflow.

### 4.4 YouTube — Video Platform at Scale

- **Vitess**: MySQL sharding middleware, built by YouTube. Now used by Slack, Square, GitHub.
- **Video Processing**: Upload → transcoding pipeline (multiple resolutions, codecs) → thumbnail generation → content moderation (ML) → CDN distribution.
- **Recommendation**: Deep Neural Networks for YouTube Recommendations (2016 paper). Two-stage: candidate generation (millions → hundreds) → ranking (hundreds → dozens). Features: watch history, search history, demographics, freshness.
- **Live Streaming**: RTMP ingest → HLS/DASH output. Edge caching for popular streams.

### 4.5 Uber — Real-Time Ride Matching

**Key Challenges:** Real-time geospatial matching at millions of events/sec.

- **Geospatial Indexing**: Google S2 geometry library. Earth divided into cells. Drivers indexed by S2 cell for fast nearest-neighbor queries.
- **Ringpop**: Consistent hashing ring for distributing real-time state across nodes.
- **Dispatch**: When rider requests, system queries nearby cells → ranks available drivers by ETA (predicted using ML) → offers trip.
- **Schemaless**: Uber's custom append-only DB on MySQL. Write-optimized for trip data.
- **Kafka**: 1 trillion+ messages/day. Core event backbone for trip events, driver locations, payments.
- **Surge Pricing**: Real-time supply-demand model. Geospatial demand prediction using ML.
- **H3**: Uber's open-source hexagonal hierarchical spatial index. Better than square grids for distance calculations.

### 4.6 Twitter/X — Real-Time Feed

**Fan-out Problem:**
- **Fan-out on Write**: When a user tweets, push to all followers' timelines (Redis). Fast reads, expensive writes. Used for users with <500K followers.
- **Fan-out on Read**: For celebrities (>500K followers), don't push. Merge their tweets at read time. Hybrid approach.
- **Timeline**: Redis sorted sets. Score = timestamp. ZRANGEBYSCORE for pagination.
- **Snowflake ID**: Distributed unique ID generator. 64-bit: timestamp (41 bits) + machine ID (10 bits) + sequence (12 bits). Sortable by time.

### 4.7 WhatsApp — Messaging at Scale

- **Erlang/BEAM VM**: Handles millions of concurrent connections per server. Lightweight processes.
- **Protocol**: Custom XMPP variant over TCP. Persistent connections.
- **Delivery**: Message → Server → Check recipient online → Deliver or store. Receipts (single tick, double tick, blue tick).
- **End-to-End Encryption**: Signal Protocol. Keys exchanged via server but server cannot read messages.
- **Scale**: 2 billion users, 100 billion messages/day, ~50 engineers (at acquisition). Efficiency through Erlang's concurrency model.

### 4.8 Discord — Real-Time Communication

- **Database Migration**: Cassandra → ScyllaDB (C++ rewrite of Cassandra). Reduced p99 latency from 200ms to 15ms for message reads.
- **Elixir/Erlang**: Real-time gateway servers handling millions of WebSocket connections.
- **Message Storage**: Bucket-based partitioning. Messages grouped into time-based buckets for efficient retrieval.
- **Voice**: WebRTC for peer-to-peer (small groups), SFU (Selective Forwarding Unit) for larger groups.
- **Lazy Loading**: Channels only load messages when opened. Virtual scrolling for message lists.

---

## Phase 5: AI/ML System Design (Weeks 29–36)

### 5.1 Recommendation Systems Architecture

#### General Pattern (Used by Netflix, Spotify, YouTube, Amazon, TikTok)
```
┌─────────────────────────────────────────────────────┐
│                   OFFLINE PIPELINE                   │
│                                                      │
│  Raw Data → Feature Engineering → Model Training     │
│  (User behavior, item metadata, context)             │
│  → Model Validation → Model Registry                 │
└──────────────────────┬──────────────────────────────┘
                       │ Deploy
┌──────────────────────▼──────────────────────────────┐
│                   ONLINE SERVING                     │
│                                                      │
│  Request → Candidate Retrieval (fast, approximate)   │
│    → Filtering (business rules, already seen)        │
│    → Ranking (precise ML model)                      │
│    → Re-ranking (diversity, freshness, fairness)     │
│    → Response                                        │
└─────────────────────────────────────────────────────┘
```

#### Key Techniques
- **Collaborative Filtering**: User-item interaction matrix. Matrix Factorization (SVD, ALS). Spotify, Netflix.
- **Content-Based**: Item features → user preference model. Useful for cold-start.
- **Two-Tower Model**: User encoder + Item encoder → dot product similarity. YouTube, Google, Pinterest.
- **Wide & Deep**: Wide (memorization of specific interactions) + Deep (generalization). Google Play Store recommendations.
- **Transformers for Recommendations**: Self-attention on user interaction sequence. SASRec, BERT4Rec. Alibaba, TikTok.
- **Graph Neural Networks**: User-item bipartite graph. PinSage (Pinterest) — billions of nodes.
- **Multi-Armed Bandits / Contextual Bandits**: Explore-exploit tradeoff. Netflix thumbnail optimization. Spotify homepage.
- **Reinforcement Learning**: Optimize long-term user satisfaction, not just immediate clicks. YouTube.

#### Feature Store
- **Online Feature Store**: Low-latency serving (Redis, DynamoDB). Real-time features like "items viewed in last 5 minutes."
- **Offline Feature Store**: Batch-computed features (S3, BigQuery). Historical aggregates.
- **Tools**: Feast (open-source), Tecton, Hopsworks. Uber's Michelangelo, Spotify's Feature Hub.

### 5.2 LLM System Design

#### Serving LLMs in Production
```
Client → API Gateway (rate limit, auth)
  → Load Balancer
  → Inference Server Cluster (vLLM / TensorRT-LLM / TGI)
  → GPU Nodes (A100/H100)
```

**Key Optimization Techniques:**
- **KV Cache**: Store key-value pairs from previous tokens. Avoid recomputation. Essential for autoregressive generation.
- **Continuous Batching**: Don't wait for all requests to finish. New requests join batch as slots free up. vLLM, TGI.
- **PagedAttention**: vLLM's innovation. Manage KV cache like OS virtual memory pages. Eliminates memory fragmentation. 2-4x throughput improvement.
- **Speculative Decoding**: Small "draft" model generates N tokens → large model verifies in parallel. 2-3x speedup.
- **Quantization**: FP16 → INT8 → INT4. GPTQ, AWQ, GGUF. Reduce memory 2-4x with minimal quality loss.
- **Tensor Parallelism**: Split model layers across GPUs. For models that don't fit on one GPU.
- **Pipeline Parallelism**: Split model stages across GPUs. Different microbatches in different stages.

#### RAG (Retrieval-Augmented Generation) Architecture
```
Query → Query Encoder → Vector Search (Pinecone/Weaviate/pgvector)
  → Top-K Documents Retrieved
  → Reranker (Cross-encoder, Cohere Rerank)
  → Context + Query → LLM → Response
```

**Best Practices:**
- **Chunking**: Split documents into 256-512 token chunks with overlap. Sentence-based or semantic chunking.
- **Hybrid Search**: Vector search (semantic) + BM25 (keyword) → Reciprocal Rank Fusion.
- **Embedding Models**: text-embedding-3-large (OpenAI), voyage-3 (Anthropic), BGE, GTE.
- **Reranking**: Cross-encoder reranker dramatically improves precision. Cohere Rerank, bge-reranker.
- **Query Transformation**: HyDE (Hypothetical Document Embeddings), query decomposition, step-back prompting.

#### Agentic Systems Architecture
```
User Query → Orchestrator (LLM as planner)
  → Tool Selection → Tool Execution
  → Observation → Re-planning (if needed)
  → Final Response
```

**Patterns:**
- **ReAct**: Reason + Act loop. LLM generates thought → action → observation → repeat.
- **Plan-and-Execute**: Generate full plan first → execute steps → revise if needed.
- **Multi-Agent**: Specialized agents collaborate. AutoGen, CrewAI pattern.
- **Tool Use**: Function calling / tool use APIs. MCP (Model Context Protocol) for standardized tool interfaces.
- **Memory**: Short-term (conversation buffer), Long-term (vector store), Episodic (past interactions).
- **Guardrails**: Input/output validation, content filtering, hallucination detection.

### 5.3 ML Infrastructure

**Model Training Pipeline:**
```
Data Lake (S3/GCS) → ETL/Feature Engineering (Spark/Dataflow)
  → Training (PyTorch/JAX on GPU clusters)
  → Experiment Tracking (MLflow/W&B)
  → Model Registry → CI/CD → Deployment
```

**Model Serving Patterns:**
- **Online (Real-time)**: REST/gRPC endpoint. <100ms latency. TensorRT, ONNX Runtime, Triton Inference Server.
- **Batch**: Process large datasets offline. Spark ML, AWS Batch.
- **Streaming**: Process events in real-time. Flink + embedded model.
- **Edge**: Model on device. TensorFlow Lite, Core ML, ONNX.

**MLOps Tools:**
| Category | Tools |
|----------|-------|
| Experiment Tracking | MLflow, Weights & Biases, Neptune |
| Feature Store | Feast, Tecton, Hopsworks |
| Model Serving | Triton, TorchServe, TFServing, vLLM, TGI |
| Pipeline Orchestration | Kubeflow, Airflow, Prefect, Dagster |
| Monitoring | Evidently AI, WhyLabs, Arize |

---

## Phase 6: Infrastructure & DevOps (Weeks 37–42)

### 6.1 Containerization & Orchestration

- **Docker**: Package apps with dependencies. Standard unit of deployment.
- **Kubernetes**: Container orchestration. Pods, Services, Deployments, StatefulSets, DaemonSets.
  - **HPA**: Horizontal Pod Autoscaler. Scale based on CPU/memory/custom metrics.
  - **Service Mesh**: Istio/Linkerd. mTLS, observability, traffic management.
  - **Helm**: Package manager for K8s. Charts for reproducible deployments.
- **Alternatives**: AWS ECS/Fargate (simpler), Docker Swarm (simpler), Nomad (HashiCorp).

### 6.2 CI/CD Pipeline
```
Code Push → Build → Unit Tests → Integration Tests
  → Security Scan → Container Build → Push to Registry
  → Deploy to Staging → Smoke Tests
  → Canary Deploy (1% traffic) → Gradual Rollout → 100%
```

**Deployment Strategies:**
- **Blue-Green**: Two identical environments. Switch traffic instantly. Easy rollback.
- **Canary**: Route small % of traffic to new version. Monitor metrics. Gradually increase.
- **Rolling**: Replace instances one by one. K8s default.
- **Feature Flags**: Deploy code but toggle features independently. LaunchDarkly. Netflix does 100+ deployments/day.

### 6.3 Observability

**Three Pillars:**
- **Metrics**: Prometheus + Grafana. RED method (Rate, Errors, Duration). USE method (Utilization, Saturation, Errors).
- **Logs**: ELK Stack (Elasticsearch, Logstash, Kibana) or Loki + Grafana. Structured JSON logging.
- **Traces**: Jaeger, Zipkin, OpenTelemetry. Distributed tracing across microservices. Google Dapper paper inspired this.

**Alerting**: PagerDuty, OpsGenie. Alert on SLO violations, not individual metrics. Google's SRE book: error budgets.

### 6.4 Security

- **Authentication**: OAuth 2.0, OpenID Connect, JWT. Auth0, Okta.
- **Authorization**: RBAC (Role-Based), ABAC (Attribute-Based), OPA (Open Policy Agent).
- **API Security**: Rate limiting, input validation, CORS, HTTPS everywhere.
- **Secrets Management**: HashiCorp Vault, AWS Secrets Manager.
- **Zero Trust**: Never trust, always verify. BeyondCorp (Google's model).

---

## Phase 7: Advanced Patterns & Emerging Trends (Weeks 43–48)

### 7.1 Data-Intensive Application Patterns

- **Lambda Architecture**: Batch layer (accuracy) + Speed layer (real-time) + Serving layer. LinkedIn, Netflix.
- **Kappa Architecture**: Everything is a stream. Simplification of Lambda. Kafka Streams only.
- **Data Mesh**: Domain-oriented, decentralized data ownership. Treat data as a product. Zalando, Intuit.
- **Lakehouse**: Combine data lake flexibility with data warehouse structure. Delta Lake (Databricks), Apache Iceberg, Apache Hudi.

### 7.2 Edge Computing

- **Edge Functions**: Cloudflare Workers, Vercel Edge Functions, Deno Deploy. Run code at CDN edge. <50ms latency globally.
- **Use Cases**: A/B testing, personalization, geolocation-based routing, authentication.
- **TikTok**: Edge processing for content moderation and recommendation pre-filtering.

### 7.3 Multi-Region & Global Architecture

- **Active-Active**: Multiple regions serve traffic simultaneously. Requires conflict resolution (CRDTs, last-writer-wins).
- **Active-Passive**: One region active, others on standby. Simpler but higher failover time.
- **CRDTs** (Conflict-free Replicated Data Types): Data structures that auto-merge. Used by Figma (real-time collaboration), Redis Enterprise.
- **Google Spanner**: Globally consistent, globally distributed. TrueTime API with atomic clocks.

### 7.4 Serverless Architecture

- **AWS Lambda / Google Cloud Functions / Azure Functions**: Pay per invocation. Auto-scales to zero.
- **Limitations**: Cold starts (100ms-2s), 15 min timeout (Lambda), vendor lock-in, hard to debug.
- **Best For**: Event-driven processing, APIs with bursty traffic, scheduled jobs.
- **Step Functions**: Orchestrate serverless workflows. AWS Step Functions, Google Workflows.

### 7.5 AI-Native Architectures (2024-2025 Trends)

- **Compound AI Systems**: Multiple models + retrieval + tools. Not just one LLM.
- **Mixture of Experts (MoE)**: Sparse models. Only activate relevant "experts" per token. GPT-4, Mixtral.
- **Inference Optimization**: Speculative decoding, continuous batching, prefix caching, structured output.
- **AI Gateway**: Route between multiple LLM providers. Load balance, fallback, cost optimization. LiteLLM, Portkey.
- **Evaluation-Driven Development**: LLM-as-judge, systematic benchmarks, regression testing for AI outputs.
- **MCP (Model Context Protocol)**: Standardized interface for LLM tool use. Anthropic-led standard.
- **Voice AI Pipeline**: ASR (Whisper/Deepgram) → LLM → TTS (ElevenLabs/XTTS). Real-time via WebSocket.

---

## Phase 8: Practice & Interview Prep (Ongoing)

### 8.1 System Design Interview Framework

**Step 1: Clarify Requirements (3-5 min)**
- Functional requirements (what does it do?)
- Non-functional requirements (scale, latency, consistency, availability)
- Back-of-envelope estimation (users, QPS, storage, bandwidth)

**Step 2: High-Level Design (10-15 min)**
- Core components and data flow
- API design (endpoints, request/response)
- Data model (schema, database choice)

**Step 3: Deep Dive (15-20 min)**
- Scale bottlenecks and solutions
- Database sharding, caching strategy
- Failure modes and handling
- Specific algorithm or data structure choices

**Step 4: Wrap-up (3-5 min)**
- Summarize trade-offs
- Mention monitoring, alerting
- Future improvements

### 8.2 Must-Practice Designs

| Design Problem | Key Concepts |
|---------------|--------------|
| URL Shortener | Hashing, base62, read-heavy, caching |
| Rate Limiter | Token bucket, distributed counting, Redis |
| Notification System | Pub-sub, priority queues, delivery guarantees |
| Chat System | WebSocket, presence, message ordering, E2E encryption |
| News Feed | Fan-out, ranking, caching, hybrid approach |
| Search Autocomplete | Trie, prefix tree, caching, ranking |
| Web Crawler | BFS, URL frontier, politeness, deduplication |
| Distributed KV Store | Consistent hashing, replication, conflict resolution |
| Payment System | ACID, idempotency, saga pattern, reconciliation |
| File Storage (Dropbox) | Chunking, deduplication, sync protocol, conflict resolution |
| Video Streaming | CDN, adaptive bitrate, transcoding pipeline |
| Location-Based Service | Geohash, quadtree, proximity search |
| Ticket Booking | Distributed locking, seat reservation, eventual consistency |
| Metrics/Monitoring | Time-series DB, aggregation, downsampling, alerting |
| LLM Serving Platform | GPU scheduling, batching, caching, multi-model routing |
| Recommendation System | Two-tower, feature store, A/B testing framework |

### 8.3 Essential Reading

**Papers:**
- Google: MapReduce, GFS, BigTable, Spanner, Borg, Dapper, Dremel
- Amazon: Dynamo, Aurora
- Facebook: TAO, Memcache at Scale, Cassandra
- Netflix: Zuul, EVCache, Chaos Engineering
- LinkedIn: Kafka, Espresso, Rest.li
- YouTube: Deep Neural Networks for YouTube Recommendations
- Meta: FAISS (vector search), LLaMA, Scaling Laws

**Books:**
- "Designing Data-Intensive Applications" (Martin Kleppmann) — The Bible
- "System Design Interview" Vol 1 & 2 (Alex Xu)
- "Building Microservices" (Sam Newman)
- "Site Reliability Engineering" (Google SRE Book)
- "Machine Learning System Design" (Chip Huyen)

**Blogs:**
- Netflix Tech Blog, Uber Engineering, Spotify Engineering, Meta Engineering
- The Morning Paper (summaries of CS papers)
- High Scalability, ByteByteGo Newsletter

---

## Quick Reference: Decision Cheat Sheet

### When to Use What Database
```
Need ACID + Joins?                    → PostgreSQL
Need flexible schema + fast dev?      → MongoDB
Need massive write throughput?        → Cassandra / ScyllaDB
Need sub-ms key-value lookups?        → Redis / DynamoDB
Need full-text search?                → Elasticsearch
Need graph traversals?                → Neo4j
Need time-series data?                → TimescaleDB / InfluxDB
Need vector similarity search?        → pgvector / Pinecone / Milvus
Need global strong consistency?       → CockroachDB / Spanner
```

### When to Use What Communication
```
Client-Server CRUD?                   → REST
Mobile with varied data needs?        → GraphQL
Internal microservice-to-microservice? → gRPC
Real-time bidirectional?              → WebSocket
Server-push only?                     → SSE
Async decoupled processing?           → Kafka / RabbitMQ
```

### When to Use What Cache
```
General purpose caching?              → Redis (Cache-Aside)
Read-heavy, rarely updated?           → CDN + Redis + long TTL
Write-heavy, read-after-write?        → Write-Through
High write throughput, eventual OK?   → Write-Behind
Session storage?                      → Redis with TTL
API response caching?                 → API Gateway cache + CDN
```

---

*Total estimated time: 48 weeks (12 months) at ~10-15 hours/week. Adjust based on your current level. Focus on understanding WHY each technology exists and what trade-offs it makes — that's the real skill.*
