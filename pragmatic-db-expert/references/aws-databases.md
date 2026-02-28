# AWS Database Reference: DynamoDB, DocumentDB, Aurora DSQL

## Table of Contents
1. [DynamoDB](#dynamodb)
2. [DocumentDB](#documentdb)
3. [Aurora DSQL](#aurora-dsql)
4. [Decision Matrix](#decision-matrix)
5. [Combination Patterns](#combination-patterns)

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

## Decision Matrix

| Criterion | DynamoDB | DocumentDB | Aurora DSQL |
|-----------|----------|------------|-------------|
| Data model | Key-value / flat document | Rich document (nested, arrays) | Relational (tables, JOINs) |
| Query flexibility | Low (access patterns defined upfront) | High (MQL, aggregations) | Very high (full SQL) |
| Throughput (scale-out) | Unlimited (auto-sharding) | Medium-High (replicas) | High (distributed) |
| Serverless friendly | Excellent (HTTP, no connections) | Moderate (needs proxy) | Good (scales to zero, needs proxy) |
| ACID transactions | Limited (100 items) | Multi-document | Full distributed ACID |
| Multi-region writes | Yes (Global Tables) | No (DR only) | Yes (active-active) |
| SQL | No | No | Yes (PostgreSQL) |
| Team skill requirement | DynamoDB / NoSQL patterns | MongoDB / MQL | SQL / PostgreSQL |
| Cost model | Pay per read/write unit | Pay per instance + storage | Pay per request + storage (serverless) |
| Maturity | Very mature | Mature | Newer (growing) |

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

### Anti-patterns to avoid
- Do not use DocumentDB as a cache layer — it's not designed for that; use ElastiCache
- Do not use DynamoDB for complex join-heavy reporting — query patterns become unmaintainable
- Do not use Aurora DSQL when you need tens of millions of simple key lookups per second — DynamoDB's horizontal scale is unmatched for that
