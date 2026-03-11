# AWS Database Reference: DynamoDB, DocumentDB, Aurora DSQL, ElastiCache, MemoryDB, Timestream

## Table of Contents
1. [DynamoDB](#dynamodb)
2. [DocumentDB](#documentdb)
3. [Aurora DSQL](#aurora-dsql)
4. [ElastiCache](#elasticache)
5. [MemoryDB for Redis](#memorydb-for-redis)
6. [Amazon Timestream](#amazon-timestream)
7. [Decision Matrix](#decision-matrix)
8. [Combination Patterns](#combination-patterns)

---

## DynamoDB

**Type**: NoSQL key-value and document store
**Model**: Single-table design with partition key + optional sort key
**Consistency**: Eventual (default) or strong consistency per-read
**Transactions**: Yes (TransactWriteItems / TransactGetItems) — up to 100 items
**Latency**: Single-digit milliseconds at any scale
**Scaling**: Fully automatic (on-demand) or provisioned with auto-scaling
**Connections**: HTTP-based (no persistent connections — ideal for Lambda/serverless)
**Global**: Global Tables for active-active multi-region replication
**SQL**: No — access via DynamoDB API or PartiQL (limited SQL-like syntax)

### When DynamoDB wins
- Access patterns are known upfront and few (1–5 primary patterns)
- High and variable throughput with unpredictable spikes
- Serverless architecture (Lambda, Fargate) — no connection pool issues
- Sub-10ms latency is a hard requirement
- Data grows horizontally (sharding by partition key is automatic)
- Event-driven patterns: DynamoDB Streams + Lambda for CDC
- Mobile/gaming/IoT/e-commerce carts — session management

### When DynamoDB struggles
- Ad-hoc queries, complex filters, aggregations, or JOINs across entities
- Schema changes are frequent (requires re-thinking access patterns)
- Team is SQL-native and NoSQL modeling expertise is limited
- Documents are deeply nested with many access paths
- Maximum item size is 400KB (large payloads need S3 + reference)

### Key design considerations
- **Partition key choice is critical** — bad partition keys cause hot partitions. Distribute writes evenly.
- **Single-table design**: Combine multiple entity types in one table using composite keys (PK + SK patterns like `USER#123` and `ORDER#456`)
- **GSI (Global Secondary Index)**: Up to 20 per table. Each GSI is billed separately for read/write capacity.
- **DynamoDB Streams**: Real-time change capture for event-driven processing or replication

---

## DocumentDB

**Type**: Document database (MongoDB-compatible API)
**Model**: JSON documents organized in collections
**Consistency**: Strong consistency within primary, eventual with read replicas
**Transactions**: Multi-document ACID transactions (since 4.0 compatibility)
**Latency**: Milliseconds — slightly higher than DynamoDB for simple lookups
**Scaling**: Vertical (instance size) + read replicas (up to 15) + storage auto-scales
**Connections**: Persistent TCP connections — not ideal for very short-lived Lambda functions (use connection pooling or proxy)
**Global**: Global Clusters for disaster recovery (async replication)
**SQL**: No — uses MongoDB query language (MQL) and aggregation pipeline

### When DocumentDB wins
- Data is inherently document-shaped: nested objects, arrays, variable schema
- Query patterns are varied and not always known upfront
- Complex aggregations, text search within documents (Atlas Search not available, use basic regex/indexes)
- Team has MongoDB experience or existing MongoDB workloads being migrated
- Content management systems, catalogs, user profiles, configuration stores
- Flexible schema where documents evolve over time

### When DocumentDB struggles
- Serverless Lambda at scale — persistent connections require connection pooling (use DocumentDB with Proxy or Mongoose with connection caching)
- Very high write throughput (DynamoDB scales more elastically)
- Strong SQL requirements or complex relational queries
- Not 100% MongoDB-compatible: some operators and aggregation stages differ — validate compatibility before migrating
- Cost can be high for large clusters — minimum cluster has a fixed cost

### Key design considerations
- **Indexes**: Create indexes on frequently queried fields. Compound indexes help complex queries.
- **Aggregation pipeline**: Powerful for transformation but runs on the primary — consider read replicas for heavy analytics
- **Connection pooling**: Critical in Lambda environments. Reuse connections across invocations. Set `maxPoolSize` appropriately.
- **Storage**: Auto-scales in 10GB increments. No need to pre-provision storage.

---

## Aurora DSQL

**Type**: Distributed relational SQL database (PostgreSQL-compatible)
**Model**: Fully relational — tables, indexes, foreign keys, JOINs
**Consistency**: Strong consistency, ACID transactions with serializable isolation
**Transactions**: Full ACID — distributed transactions across multiple regions
**Latency**: Low (single-digit ms for simple queries) — higher than DynamoDB for single-row lookups
**Scaling**: Serverless auto-scaling (compute scales to zero) — no instance management
**Connections**: PostgreSQL wire protocol — use connection pooling (RDS Proxy) for Lambda workloads
**Global**: Active-active multi-region by design — writes accepted in any region
**SQL**: Full PostgreSQL SQL dialect

### When Aurora DSQL wins
- Team is SQL-native — relational modeling, JOINs, stored procedures
- ACID transactions across multiple entities are non-negotiable
- Serverless requirement: scales to zero (no cost when idle), great for startups
- Multi-region active-active writes needed with strong consistency
- Replacing a PostgreSQL workload with no managed infrastructure burden
- Transactional microservices that need SQL flexibility without managing RDS instances

### When Aurora DSQL struggles
- Extremely high single-key throughput (DynamoDB still wins at millions of simple ops/sec)
- Very large data volumes with complex sharding needs (DynamoDB's horizontal model is more natural)
- Write latency sensitive to cross-region coordination (active-active has coordination overhead)
- Applications that need PostgreSQL extensions not yet supported by DSQL (validate extension compatibility)
- Aurora DSQL is newer — ecosystem maturity and tooling depth is still growing vs. Aurora PostgreSQL

### Key design considerations
- **Serverless billing**: Pay per request unit + storage. Very cost-effective for variable or low-traffic workloads.
- **Connection pooling**: Required for Lambda/short-lived workloads. Use RDS Proxy or built-in pooling.
- **Multi-region writes**: Conflicts resolved via optimistic concurrency — design for conflict-free access patterns by region when possible
- **Migration**: Straightforward from PostgreSQL — standard tools (pg_dump, DMS) apply

---

## ElastiCache

**Type**: Managed in-memory cache (Redis or Memcached engine)
**Model**: Key-value (Memcached) or rich data structures (Redis: strings, hashes, lists, sets, sorted sets, streams)
**Consistency**: Eventual consistency across replicas; primary reads are consistent
**Transactions**: Redis supports basic transactions (MULTI/EXEC) — not ACID in the relational sense
**Latency**: Sub-millisecond (microseconds for simple reads/writes)
**Scaling**: Redis — cluster mode with sharding; Memcached — horizontal node scaling
**Connections**: Persistent TCP connections — use connection pooling or cluster-aware clients
**Durability**: Optional (Redis AOF/RDB snapshots) — but data loss on failure is possible; not a primary store
**Global**: Redis Global Datastore for cross-region replication (active-passive)
**SQL**: No

### When ElastiCache wins
- Offloading repeated, expensive reads from a primary database (DynamoDB, Aurora, DocumentDB)
- User sessions, API rate limiting, feature flags, leaderboards, real-time counters
- Sub-millisecond response required on the hot path of user-facing requests
- Caching the result of complex queries or aggregations
- Pub/Sub messaging for lightweight real-time notifications
- Temporary data with TTL (shopping carts, OTP tokens, temporary locks)

### When ElastiCache struggles
- Durable long-term storage — data may be lost on node failure (use MemoryDB instead)
- Data larger than memory capacity — not suitable for large datasets without careful eviction policies
- Complex querying, filtering, or aggregations beyond Redis data structures
- ACID transactional requirements or relational data modeling
- Team unfamiliar with cache invalidation patterns — stale data bugs are common

### Key design considerations
- **Eviction policy**: Choose the right policy (`allkeys-lru`, `volatile-lru`, etc.) based on cache access patterns
- **TTL strategy**: Always set TTL on cache entries to avoid stale data accumulation
- **Cache-aside vs write-through**: Cache-aside is most common (app reads cache first, falls back to DB); write-through updates cache on every write
- **Connection pooling**: Critical for Lambda and serverless — use Redis cluster-aware clients and keep connections warm
- **Redis vs Memcached**: Use Redis when you need persistence, pub/sub, rich data structures, or replication. Use Memcached for simple, horizontal caching with minimal overhead.

---

## MemoryDB for Redis

**Type**: Durable in-memory database with Redis API compatibility
**Model**: Redis data structures (strings, hashes, lists, sets, sorted sets, streams, JSON)
**Consistency**: Strong consistency — synchronous replication to a Multi-AZ transaction log before ACK
**Transactions**: Redis MULTI/EXEC supported
**Latency**: Sub-millisecond reads; slightly higher write latency than ElastiCache (due to durable replication)
**Scaling**: Cluster mode with sharding across nodes; up to hundreds of shards
**Connections**: Persistent TCP connections — use cluster-aware clients
**Durability**: Yes — data is persisted to a Multi-AZ transaction log; survives node failures and reboots
**Global**: Multi-AZ within a region; cross-region not natively supported as of 2024
**SQL**: No

### When MemoryDB wins
- Primary data store where data must survive failures — not just a cache
- Microservices needing a fast, durable store with Redis API (sessions, game state, real-time leaderboards)
- Applications already using Redis as primary store that need durability guarantees
- Sub-millisecond reads are required AND data loss is not acceptable
- Event sourcing or activity streams using Redis Streams with durable retention
- Feature stores, rate limiters, or distributed locks that need strong consistency + durability

### When MemoryDB struggles
- Pure caching use case — ElastiCache is cheaper and simpler (MemoryDB has durability overhead cost)
- Large datasets that exceed memory budget — in-memory databases are expensive per GB
- Complex relational data or query-heavy workloads — not a replacement for relational databases
- Write-intensive workloads at extremely high throughput (durability adds latency per write)

### Key design considerations
- **MemoryDB vs ElastiCache**: MemoryDB is a primary data store; ElastiCache is a cache. If you can tolerate data loss on failure, use ElastiCache — it's cheaper. If not, use MemoryDB.
- **Cluster topology**: Plan sharding strategy upfront based on key distribution and access patterns
- **Data eviction**: Unlike ElastiCache, MemoryDB does not evict data — manage data lifecycle explicitly with TTLs or application-level deletion
- **Cost model**: More expensive than ElastiCache per GB — only use when durability of in-memory data is truly required

---

## Amazon Timestream

**Type**: Serverless time series database
**Model**: Time series — each record has a timestamp, dimensions (metadata), and measures (values)
**Consistency**: Strong consistency for recent data (in-memory store); eventual for historical (magnetic store)
**Transactions**: Not supported — designed for high-ingest append-only workloads
**Latency**: Low for recent data (in-memory); higher for historical queries (magnetic tier)
**Scaling**: Fully serverless — scales automatically with ingest rate and query load
**Connections**: HTTP-based (HTTPS endpoints) — compatible with serverless architectures
**Durability**: Yes — data is persisted across memory and magnetic tiers with configurable retention per tier
**Global**: Single-region per database; cross-region replication not natively supported
**SQL**: Yes — Timestream Query Language (SQL-compatible with time series extensions: time_series(), interpolate(), bin(), etc.)

### When Timestream wins
- IoT sensor data, telemetry, and device metrics at high ingest rates
- Application performance monitoring (APM): latency, error rates, throughput over time
- Infrastructure monitoring: CPU, memory, disk, network metrics
- Financial tick data, trading signals, or price history
- Any workload where data is inherently ordered by time and queried by time ranges
- Downsampling, rollups, and retention policies needed (e.g., keep 7 days at second granularity, 1 year at hourly)

### When Timestream struggles
- Data is not time-stamped or time is not the primary query dimension — use DynamoDB or Aurora instead
- Complex entity relationships, JOINs across non-temporal data — use relational databases
- Very low cardinality, simple counter-style data that fits in DynamoDB (simpler and cheaper)
- Write patterns that update existing records (Timestream is append-only — no UPDATE support)
- Cross-region requirements or multi-region replication needs

### Key design considerations
- **Dual-store architecture**: Timestream has an in-memory store (recent data, fast queries) and a magnetic store (historical data, cheaper). Configure retention per tier based on query patterns.
- **Dimensions vs measures**: Dimensions are metadata identifiers (device_id, region, service); measures are the actual values. High-cardinality dimensions increase storage costs.
- **Ingest rate**: Designed for millions of data points per second. Batch writes for efficiency.
- **Query model**: SQL-like with time series functions — `bin()` for time bucketing, `interpolate_linear()` for gap filling, `time_series()` for time series objects.
- **Scheduled queries**: Pre-aggregate data into derived tables to reduce query latency and cost for dashboards.

---

## Decision Matrix

| Criterion | DynamoDB | DocumentDB | Aurora DSQL | ElastiCache | MemoryDB | Timestream |
|-----------|----------|------------|-------------|-------------|----------|------------|
| Primary use case | Operational NoSQL store | Document store | Relational SQL store | Cache / acceleration | Durable in-memory store | Time series data |
| Data model | Key-value / flat document | Rich document (nested, arrays) | Relational (tables, JOINs) | Key-value / Redis structures | Redis data structures | Timestamped metrics/events |
| Query flexibility | Low (access patterns defined upfront) | High (MQL, aggregations) | Very high (full SQL) | Low (key lookup, Redis commands) | Low (Redis commands) | High (SQL + time series functions) |
| Throughput (scale-out) | Unlimited (auto-sharding) | Medium-High (replicas) | High (distributed) | Very high (cluster mode) | High (cluster mode) | Very high (serverless auto-scale) |
| Serverless friendly | Excellent (HTTP, no connections) | Moderate (needs proxy) | Good (scales to zero, needs proxy) | Moderate (persistent TCP) | Moderate (persistent TCP) | Excellent (HTTP, serverless) |
| ACID transactions | Limited (100 items) | Multi-document | Full distributed ACID | No (basic MULTI/EXEC) | No (basic MULTI/EXEC) | No (append-only) |
| Multi-region writes | Yes (Global Tables) | No (DR only) | Yes (active-active) | Yes (Global Datastore, passive) | No | No |
| SQL | No | No | Yes (PostgreSQL) | No | No | Yes (time series SQL) |
| Latency | Single-digit ms | Low ms | Single-digit ms | Sub-ms (microseconds) | Sub-ms (microseconds) | Low ms (recent data) |
| Durability | Yes | Yes | Yes | Optional (snapshots) | Yes (Multi-AZ log) | Yes (dual-store) |
| TTL / expiration | Yes (TTL attribute) | No (application-managed) | No (application-managed) | Yes (per-key TTL) | Yes (per-key TTL) | Yes (per-tier retention) |
| Team skill requirement | DynamoDB / NoSQL patterns | MongoDB / MQL | SQL / PostgreSQL | Redis commands | Redis commands | SQL + time series concepts |
| Cost model | Pay per read/write unit | Pay per instance + storage | Pay per request + storage (serverless) | Pay per node/hour | Pay per node/hour | Pay per ingest + query + storage |
| Maturity | Very mature | Mature | Newer (growing) | Very mature | Mature | Mature |

---

## Combination Patterns

### Pattern 1: DynamoDB (hot path) + DocumentDB (complex queries)
**Use case**: High-speed writes and simple reads use DynamoDB. Complex reporting, search, or aggregation queries use DocumentDB.
**Integration**: DynamoDB Streams → Lambda → DocumentDB sync (CDC pattern). Keep DocumentDB eventually consistent with DynamoDB.
**Example**: E-commerce order management — DynamoDB for order creation/status updates (millisecond writes), DocumentDB for order history queries with complex filters.

### Pattern 2: Aurora DSQL (primary) + DynamoDB (session/cache layer)
**Use case**: Relational transactional core on Aurora DSQL, with DynamoDB for user sessions, caching hot data, or feature flags.
**Integration**: Application writes session to DynamoDB TTL-based store; transactional data goes to Aurora DSQL.
**Example**: SaaS application — Aurora DSQL for billing and subscription data, DynamoDB for API rate limiting and session tokens.

### Pattern 3: DynamoDB (primary) + Aurora DSQL (reporting replica)
**Use case**: DynamoDB as the operational store for high-throughput workloads, Aurora DSQL for analytics and reporting that needs SQL joins.
**Integration**: CDC pipeline (DynamoDB Streams → Lambda → Aurora DSQL) to sync relevant data for reporting use cases.

### Pattern 4: Any primary DB + ElastiCache (read acceleration)
**Use case**: Reduce read latency and offload repeated queries from a primary database.
**Integration**: Application checks ElastiCache first (cache-aside pattern); on miss, reads from primary DB and populates cache with TTL.
**Example**: Product catalog on DynamoDB or Aurora → ElastiCache caches popular product pages. Reduces P99 latency from 10ms to <1ms and cuts primary DB read costs significantly.

### Pattern 5: MemoryDB for Redis (primary) + DynamoDB (cold storage)
**Use case**: Active, durable game state or session data in MemoryDB; older/inactive data archived to DynamoDB.
**Integration**: Application reads/writes active sessions to MemoryDB. A background job archives expired sessions to DynamoDB for audit or recovery.
**Example**: Online gaming — active match state and player sessions in MemoryDB (sub-ms + durable), historical match records in DynamoDB for leaderboards.

### Pattern 6: IoT/App (primary DB) + Timestream (metrics)
**Use case**: Operational data (device registry, user accounts) in DynamoDB or Aurora; time series telemetry in Timestream.
**Integration**: Application writes events/metrics to Timestream independently from business data. Dashboards query Timestream; transactional flows use primary DB.
**Example**: IoT platform — device configuration in DynamoDB, sensor readings (temperature, humidity, pressure at 1s intervals) in Timestream. Grafana dashboards query Timestream; Lambda functions managing device state use DynamoDB.

### Pattern 7: Aurora DSQL / DocumentDB + ElastiCache + Timestream (full stack)
**Use case**: Full-featured SaaS application with relational core, cache layer, and observability.
**Integration**: Aurora DSQL for business entities and transactions; ElastiCache for session management and hot data caching; Timestream for application metrics, error rates, and performance telemetry.
**Example**: Multi-tenant SaaS — Aurora DSQL for tenant data and billing, ElastiCache for user sessions and rate limiting, Timestream for monitoring response times and error rates per tenant.

### Anti-patterns to avoid
- Do not use DocumentDB as a cache layer — it's not designed for that; use ElastiCache
- Do not use ElastiCache as a primary database — data loss on node failure without MemoryDB's durability guarantees
- Do not use DynamoDB for complex join-heavy reporting — query patterns become unmaintainable
- Do not use Aurora DSQL when you need tens of millions of simple key lookups per second — DynamoDB's horizontal scale is unmatched for that
- Do not use Timestream for operational data with UPDATE patterns — it is append-only and not suitable for mutable records
- Do not use MemoryDB when a cache (ElastiCache) suffices — MemoryDB costs more due to durability; only pay for it when data loss is unacceptable
