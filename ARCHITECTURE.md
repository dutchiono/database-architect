# Database Architect - System Architecture

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Database Architect System                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐  │
│  │   Schema     │      │    Query     │      │   Migration  │  │
│  │  Designer    │──────│  Optimizer   │──────│   Manager    │  │
│  └──────────────┘      └──────────────┘      └──────────────┘  │
│         │                      │                      │          │
│         │                      │                      │          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Database Strategy Engine                     │  │
│  │  • Type Selection Logic (SQL/NoSQL/Graph/Time-Series)    │  │
│  │  • Partitioning & Sharding Strategies                    │  │
│  │  • Replication & HA Configuration                        │  │
│  │  • Performance Tuning Rules                              │  │
│  └──────────────────────────────────────────────────────────┘  │
│         │                      │                      │          │
│  ┌──────────────┐      ┌──────────────┐      �┌──────────────┐ │
│  │  Indexing    │      │   Backup &   │      │   Security   │  │
│  │  Strategy    │──────│   Recovery   │──────│  & Access    │  │
│  └──────────────┘      └──────────────┘      └──────────────┘  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │
        ┌─────────────────────┴─────────────────────┐
        │                                             │
┌───────▼────────┐                          ┌────────▼───────┐
│   Database     │                          │   Monitoring   │
│   Platforms    │                          │   & Analytics  │
├────────────────┤                          ├────────────────┤
│ • PostgreSQL   │                          │ • Slow Query   │
│ • MySQL        │                          │ • Connection   │
│ • MongoDB      │                          │   Pool Stats   │
│ • Redis        │                          │ • Disk Usage   │
│ • Cassandra    │                          │ • Replication  │
│ • TimescaleDB  │                          │   Lag          │
│ • Neo4j        │                          │ • Performance  │
└────────────────┘                          │   Metrics      │
                                            └────────────────┘
```

## Core Components

### 1. Schema Designer
**Purpose:** Design and validate database schemas with best practices

**Key Features:**
- Entity-relationship modeling
- Normalization analysis (1NF through 5NF)
- Denormalization strategies for performance
- Data type optimization
- Constraint definition (PK, FK, unique, check)
- JSON schema validation for document stores

**Technologies:**
- ERD generation tools
- Schema validation libraries
- Type inference engines

### 2. Query Optimizer
**Purpose:** Analyze and optimize database queries for performance

**Key Features:**
- EXPLAIN plan analysis
- Index recommendation engine
- Query rewriting suggestions
- N+1 query detection
- Batch operation optimization
- Prepared statement management

**Optimization Techniques:**
- Index covering strategies
- Join order optimization
- Subquery to JOIN conversion
- Materialized view candidates
- Partition pruning

### 3. Migration Manager
**Purpose:** Version control and safe deployment of schema changes

**Key Features:**
- Versioned migration files
- Rollback capabilities
- Zero-downtime migration strategies
- Data transformation scripts
- Dependency tracking
- Multi-environment deployment

**Migration Types:**
- Additive (columns, indexes, tables)
- Destructive (drops, renames)
- Data migrations (ETL, transformations)
- Online schema changes (pt-online-schema-change, gh-ost)

### 4. Database Strategy Engine
**Purpose:** Select optimal database technology and configuration

**Decision Matrix:**

| Use Case | Recommended DB | Rationale |
|----------|---------------|-----------|
| Transactional (ACID) | PostgreSQL, MySQL | Strong consistency, ACID guarantees |
| Document Store | MongoDB, CouchDB | Flexible schema, JSON documents |
| Key-Value Cache | Redis, Memcached | Ultra-fast lookups, TTL support |
| Time-Series | TimescaleDB, InfluxDB | Optimized for time-based data |
| Graph Relationships | Neo4j, ArangoDB | Complex relationship queries |
| Wide-Column | Cassandra, HBase | Massive scale, high writes |
| Search Engine | Elasticsearch, Meilisearch | Full-text search, fuzzy matching |

**Configuration Strategies:**
- Connection pooling (PgBouncer, ProxySQL)
- Read replicas for scaling reads
- Master-slave vs multi-master replication
- Sharding key selection
- Partition strategies (range, hash, list)

### 5. Indexing Strategy
**Purpose:** Design and maintain optimal indexes

**Index Types:**
- B-tree (default, range queries)
- Hash (exact match, equality)
- GIN (full-text, JSONB, arrays)
- GiST (geometric, full-text)
- BRIN (very large tables, natural order)
- Covering indexes (include columns)

**Best Practices:**
- Index cardinality analysis
- Composite index column order
- Partial indexes for filtered queries
- Index maintenance (REINDEX, VACUUM)
- Cost-benefit analysis (write penalty vs read gain)

### 6. Backup & Recovery
**Purpose:** Data protection and disaster recovery

**Backup Strategies:**
- Logical backups (pg_dump, mysqldump)
- Physical backups (base backups, snapshots)
- Continuous archiving (WAL, binlog)
- Point-in-time recovery (PITR)
- Cross-region replication

**Recovery Procedures:**
- RTO (Recovery Time Objective) targets
- RPO (Recovery Point Objective) targets
- Backup verification and testing
- Automated restore testing
- Disaster recovery runbooks

### 7. Security & Access Control
**Purpose:** Protect data and enforce access policies

**Security Layers:**
- Authentication (passwords, certificates, LDAP)
- Authorization (RBAC, ABAC)
- Encryption at rest (TDE, disk encryption)
- Encryption in transit (SSL/TLS)
- Network isolation (VPC, firewall rules)
- Audit logging (DDL, DML, DCL)

**Access Patterns:**
- Least privilege principle
- Service account management
- Connection string security (secrets management)
- SQL injection prevention
- Parameterized queries enforcement

## Data Architecture Patterns

### Pattern 1: CQRS (Command Query Responsibility Segregation)
```
Write Model (Commands) → PostgreSQL (normalized)
                              ↓
                         Event Stream
                              ↓
Read Model (Queries) ← MongoDB (denormalized views)
```

### Pattern 2: Lambda Architecture
```
Batch Layer (historical) → Data Warehouse
Speed Layer (real-time) → Stream Processing → Redis
                                ↓
                         Serving Layer (combined view)
```

### Pattern 3: Polyglot Persistence
```
User Auth → PostgreSQL (ACID transactions)
Session Data → Redis (TTL, fast access)
Product Catalog → Elasticsearch (search)
User Activity → Cassandra (high write volume)
Social Graph → Neo4j (relationships)
```

## Performance Tuning

### Database Server Tuning
**PostgreSQL:**
```sql
-- Connection settings
max_connections = 200
shared_buffers = 4GB
effective_cache_size = 12GB

-- WAL settings
wal_buffers = 16MB
checkpoint_completion_target = 0.9

-- Query planner
random_page_cost = 1.1  -- for SSD
effective_io_concurrency = 200
```

**MySQL:**
```ini
# Buffer pool (70-80% of RAM)
innodb_buffer_pool_size = 8G
innodb_log_file_size = 1G

# Query cache (use with caution in 5.7+)
query_cache_type = 0  # disabled in 8.0+

# Connections
max_connections = 200
```

### Query Optimization Checklist
- [ ] Proper indexes on WHERE, JOIN, ORDER BY columns
- [ ] Avoid SELECT * (specify needed columns)
- [ ] Use LIMIT for pagination
- [ ] Batch INSERT/UPDATE operations
- [ ] Avoid correlated subqueries
- [ ] Use EXISTS instead of COUNT(*) > 0
- [ ] Optimize JOIN order (smallest table first)
- [ ] Partition large tables
- [ ] Use connection pooling
- [ ] Cache frequently accessed data

## Scalability Strategies

### Vertical Scaling (Scale Up)
- Increase CPU, RAM, disk I/O
- Upgrade to SSD or NVMe
- Optimize for single-server performance

### Horizontal Scaling (Scale Out)

**Read Replicas:**
```
Master (writes) → Replica 1 (reads)
                → Replica 2 (reads)
                → Replica 3 (reads)
```

**Sharding:**
```
User Hash → Shard 1 (users A-M)
          → Shard 2 (users N-Z)
```

**Federation:**
```
Service A → DB A (orders)
Service B → DB B (inventory)
Service C → DB C (users)
```

## Monitoring & Observability

### Key Metrics
- **Performance:** Query latency, throughput (QPS)
- **Resources:** CPU, memory, disk I/O, network
- **Connections:** Active, idle, max reached
- **Replication:** Lag, status, conflicts
- **Errors:** Failed queries, deadlocks, constraint violations

### Monitoring Tools
- Native: pg_stat_statements, MySQL slow query log
- APM: New Relic, Datadog, Dynatrace
- Open Source: Prometheus + Grafana, Zabbix
- Cloud: AWS RDS Performance Insights, Azure Database Insights

### Alerting Thresholds
- Replication lag > 5 seconds
- Connection pool utilization > 80%
- Slow queries > 1 second
- Disk usage > 85%
- CPU usage > 80% sustained

## Schema Evolution Best Practices

### Safe Migration Steps
1. **Additive Phase:** Add new columns/tables (backward compatible)
2. **Dual-Write Phase:** Write to both old and new schema
3. **Backfill Phase:** Migrate historical data
4. **Dual-Read Phase:** Read from new schema, fallback to old
5. **Cleanup Phase:** Remove old schema (after validation)

### Zero-Downtime Techniques
- Use online schema change tools (pt-osc, gh-ost)
- Add indexes CONCURRENTLY (PostgreSQL)
- Avoid table locks during migrations
- Blue-green database deployment
- Feature flags for schema-dependent code

## Technology Selection Matrix

| Requirement | PostgreSQL | MongoDB | Redis | Cassandra | Neo4j |
|-------------|-----------|---------|-------|-----------|-------|
| ACID Transactions | ✓✓✓ | ✓ | ✗ | ✗ | ✓✓ |
| Horizontal Scale | ✓ | ✓✓✓ | ✓✓ | ✓✓✓ | ✓ |
| Complex Queries | ✓✓✓ | ✓✓ | ✗ | ✓ | ✓✓✓ |
| Write Performance | ✓✓ | ✓✓✓ | ✓✓✓ | ✓✓✓ | ✓ |
| Read Performance | ✓✓ | ✓✓ | ✓✓✓ | ✓✓ | ✓✓ |
| Schema Flexibility | ✓ | ✓✓✓ | ✓✓ | ✓✓ | ✓✓ |
| Operational Maturity | ✓✓✓ | ✓✓ | ✓✓✓ | ✓✓ | ✓ |

## Disaster Recovery Architecture

```
Primary Region (us-east-1)
  ├─ Master Database
  ├─ Read Replica 1
  └─ Read Replica 2
         │
         │ Async Replication
         ↓
Secondary Region (us-west-2)
  ├─ Standby Master (read-only)
  └─ Read Replica
         │
         │ Backup Stream
         ↓
  S3 (WAL archives, daily backups)
```

**Failover Process:**
1. Detect primary failure (health checks)
2. Promote standby to master
3. Update DNS/load balancer
4. Redirect application traffic
5. Monitor replication lag
6. Restore primary as replica when recovered

## References

- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [MySQL Performance Tuning](https://dev.mysql.com/doc/)
- [MongoDB Best Practices](https://docs.mongodb.com/manual/)
- [Database Reliability Engineering](https://www.oreilly.com/library/view/database-reliability-engineering/9781491925935/)
- [Designing Data-Intensive Applications](https://dataintensive.net/)
