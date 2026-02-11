# Database Architect - Integration Specifications

## Integration Overview

Database Architect serves as the foundational data layer for all system architects, providing persistent storage, query optimization, and data integrity enforcement. It integrates bidirectionally with application services, monitoring systems, and infrastructure components.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Integration Ecosystem                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│   Application Layer                                              │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│   │   API    │  │  Service │  │  Cache   │  │  Queue   │      │
│   │ Gateway  │──│  Mesh    │──│  Layer   │──│  Manager │      │
│   └──────────┘  └──────────┘  └──────────┘  └──────────┘      │
│        │             │              │              │             │
│        └─────────────┴──────────────┴──────────────┘            │
│                          │                                        │
│                          ↓                                        │
│   ┌─────────────────────────────────────────────────────┐       │
│   │           Database Architect (Core)                 │       │
│   │  • Connection Management                            │       │
│   │  • Query Routing & Optimization                     │       │
│   │  • Schema Validation & Migration                    │       │
│   │  • Transaction Coordination                         │       │
│   └─────────────────────────────────────────────────────┘       │
│                          │                                        │
│        ┌─────────────────┼─────────────────┬──────────┐         │
│        │                 │                 │          │          │
│        ↓                 ↓                 ↓          ↓          │
│   ┌────────┐      ┌──────────┐      ┌─────────┐  ┌────────┐   │
│   │Monitor │      │ Deployment│      │ Security│  │ Backup │   │
│   │        │──────│           │──────│         │──│        │   │
│   └────────┘      └──────────┘      └─────────┘  └────────┘   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

## Integration Points

### 1. API Gateway Integration

**Purpose:** Route database requests through API layer with authentication and rate limiting

**Integration Pattern: Database-per-Service with Shared Gateway**

```yaml
# Gateway → Database Flow
gateway_config:
  database_routing:
    - service: user-service
      database: users_db
      connection_pool:
        min: 5
        max: 20
      rate_limit: 1000req/min
      
    - service: order-service
      database: orders_db
      connection_pool:
        min: 10
        max: 50
      rate_limit: 2000req/min
```

**API Gateway Actions:**
- Authenticate requests before database access
- Route queries to appropriate database shard
- Apply query-level rate limiting
- Log all database operations
- Transform database responses to API format

**Database Architect Responsibilities:**
- Expose connection endpoints per service
- Provide health check endpoints
- Return standardized error codes
- Support connection pooling
- Enable query timeout configuration

**Data Flow:**
```
Client Request
  → API Gateway (auth, validation)
    → Database Architect (route to correct DB)
      → Primary/Replica Selection
        → Execute Query
          → Response (formatted)
            → API Gateway (transform)
              → Client Response
```

### 2. Service Mesh Integration

**Purpose:** Enable database service discovery, load balancing, and observability

**Integration Pattern: Sidecar Proxy for Database Connections**

```yaml
# Service Mesh Database Config
service_mesh:
  database_services:
    - name: postgres-primary
      endpoints:
        - host: db1.internal
          port: 5432
          weight: 100
      health_check:
        interval: 10s
        timeout: 3s
        
    - name: postgres-replicas
      endpoints:
        - host: db2.internal
          port: 5432
          weight: 50
        - host: db3.internal
          port: 5432
          weight: 50
      health_check:
        interval: 10s
        timeout: 3s
```

**Service Mesh Provides:**
- Automatic service discovery for database instances
- Client-side load balancing across replicas
- Circuit breaking for failed database connections
- Retry logic with exponential backoff
- mTLS encryption for database connections

**Database Architect Provides:**
- Health check endpoints (/health, /ready)
- Connection metadata (read/write capability)
- Performance metrics (query latency, throughput)
- Replica lag information
- Connection pool status

**Connection Pattern:**
```python
# Application code with service mesh
from database_client import DatabaseClient

# Service mesh handles discovery and load balancing
db = DatabaseClient(
    service_name="postgres-primary",  # mesh resolves to actual endpoint
    connection_pool_size=20,
    timeout=5000
)

# Reads automatically routed to replicas
result = db.query("SELECT * FROM users", read_preference="replica")
```

### 3. Cache Layer Integration

**Purpose:** Reduce database load with intelligent caching strategies

**Integration Pattern: Cache-Aside with Write-Through**

```yaml
# Cache Integration Config
cache_config:
  provider: redis
  strategy: cache_aside
  ttl_policies:
    - pattern: "user:*"
      ttl: 3600  # 1 hour
      invalidate_on: ["user.update", "user.delete"]
      
    - pattern: "product:*"
      ttl: 7200  # 2 hours
      invalidate_on: ["product.update", "inventory.change"]
      
    - pattern: "session:*"
      ttl: 1800  # 30 minutes
      invalidate_on: ["session.logout"]
```

**Cache Strategies:**

**1. Cache-Aside (Lazy Loading):**
```python
def get_user(user_id):
    # Check cache first
    user = cache.get(f"user:{user_id}")
    if user:
        return user
    
    # Cache miss - fetch from database
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    
    # Populate cache
    cache.set(f"user:{user_id}", user, ttl=3600)
    return user
```

**2. Write-Through:**
```python
def update_user(user_id, data):
    # Update database first
    db.execute("UPDATE users SET ... WHERE id = ?", user_id)
    
    # Update cache
    cache.set(f"user:{user_id}", data, ttl=3600)
```

**3. Write-Behind (Async):**
```python
def track_event(event_data):
    # Write to cache immediately
    cache.lpush("events:pending", event_data)
    
    # Background worker flushes to database
    # (batch writes every 10 seconds)
```

**Cache Invalidation Patterns:**
- Time-based (TTL)
- Event-based (on update/delete)
- Tag-based (invalidate by category)
- Version-based (increment version number)

**Database Architect Responsibilities:**
- Provide change data capture (CDC) events
- Support cache warming queries
- Optimize for cache-friendly access patterns
- Document cacheable vs non-cacheable queries

### 4. Queue Manager Integration

**Purpose:** Decouple database writes and enable async processing

**Integration Pattern: Event-Driven Database Updates**

```yaml
# Queue Integration Config
queue_config:
  provider: rabbitmq
  exchanges:
    - name: database.events
      type: topic
      queues:
        - name: user.created
          binding_key: user.created
          consumer: user_service
          
        - name: order.placed
          binding_key: order.placed
          consumer: order_service
          
        - name: inventory.updated
          binding_key: inventory.*
          consumer: inventory_service
```

**Event-Driven Patterns:**

**1. Transactional Outbox:**
```sql
-- Write to database and outbox in same transaction
BEGIN;
  INSERT INTO orders (...) VALUES (...);
  INSERT INTO outbox_events (event_type, payload) 
    VALUES ('order.placed', '{"order_id": 123}');
COMMIT;

-- Background worker publishes events from outbox
-- Then marks them as processed
```

**2. Change Data Capture (CDC):**
```yaml
# Debezium connector config
cdc_config:
  connector: debezium-postgres
  tables:
    - users
    - orders
    - inventory
  publish_to: kafka
  topics:
    - database.users.changes
    - database.orders.changes
    - database.inventory.changes
```

**3. CQRS with Event Sourcing:**
```
Command → Write to Event Store (append-only)
            ↓
        Event Queue
            ↓
    Event Handlers → Update Read Models (denormalized)
```

**Queue Manager Provides:**
- Reliable message delivery (at-least-once)
- Dead letter queues for failed processing
- Message ordering guarantees
- Backpressure handling

**Database Architect Provides:**
- Outbox table for transactional messaging
- CDC event stream
- Event schema registry
- Idempotency support (duplicate detection)

### 5. Monitoring & Observability Integration

**Purpose:** Track database performance, health, and anomalies

**Integration Pattern: Multi-Layer Monitoring**

```yaml
# Monitoring Integration Config
monitoring_config:
  exporters:
    - type: prometheus
      port: 9187
      metrics:
        - pg_stat_database
        - pg_stat_statements
        - connection_pool_stats
        
    - type: datadog
      api_key: ${DD_API_KEY}
      metrics:
        - query_latency_p95
        - slow_query_count
        - replication_lag
        
  log_shipping:
    destination: elasticsearch
    log_types:
      - slow_queries
      - error_logs
      - audit_logs
    
  alerting:
    channels:
      - slack
      - pagerduty
    rules:
      - name: high_replication_lag
        condition: replication_lag > 10s
        severity: warning
        
      - name: connection_pool_exhausted
        condition: pool_utilization > 90%
        severity: critical
```

**Monitoring Data Flow:**
```
Database Metrics
  → Prometheus Exporter (scrape every 15s)
    → Prometheus TSDB
      → Grafana Dashboards
      → Alertmanager (threshold violations)
        → Notification Channels
```

**Key Metrics Exposed:**
- Query performance (latency, throughput)
- Connection pool (active, idle, waiting)
- Replication lag (primary to replica delay)
- Disk usage (data, WAL, temp files)
- Cache hit ratio (buffer cache, query cache)
- Lock contention (waiting queries, deadlocks)

**Log Integration:**
```json
// Structured log format
{
  "timestamp": "2026-02-11T10:15:32Z",
  "level": "warn",
  "component": "database-architect",
  "message": "Slow query detected",
  "query": "SELECT * FROM orders WHERE created_at > ...",
  "duration_ms": 3421,
  "rows_returned": 15000,
  "trace_id": "abc123..."
}
```

### 6. Deployment Integration

**Purpose:** Automate database provisioning, migrations, and updates

**Integration Pattern: GitOps with Database Migrations**

```yaml
# Deployment Pipeline
deployment_config:
  source_control: github
  repository: database-schemas
  
  environments:
    - name: development
      database: dev_db
      auto_migrate: true
      backup_before_migrate: true
      
    - name: staging
      database: staging_db
      auto_migrate: true
      backup_before_migrate: true
      require_approval: false
      
    - name: production
      database: prod_db
      auto_migrate: false  # manual approval required
      backup_before_migrate: true
      require_approval: true
      rollback_on_error: true
```

**Migration Pipeline:**
```
Git Push (migration files)
  → CI Pipeline Triggered
    → Lint Migration Files
      → Test on Staging DB
        → Generate Migration Report
          → Manual Approval (production)
            → Backup Production DB
              → Apply Migration
                → Verify Schema
                  → Update Schema Registry
```

**Deployment Tools Integration:**
- **Terraform:** Provision database infrastructure
- **Kubernetes Operators:** Manage database lifecycles
- **Flyway/Liquibase:** Execute migrations
- **Ansible:** Configure database servers
- **Helm Charts:** Deploy database containers

**Blue-Green Database Deployment:**
```yaml
# Blue-Green Strategy
blue_green:
  current: blue_db  # production traffic
  standby: green_db  # receives migrations
  
  deployment_steps:
    1. Apply migrations to green_db
    2. Replicate data from blue_db to green_db
    3. Verify green_db health
    4. Switch application traffic to green_db
    5. Monitor for errors
    6. Keep blue_db as rollback target (24 hours)
```

### 7. Security Integration

**Purpose:** Enforce access control, encryption, and audit compliance

**Integration Pattern: Defense in Depth**

```yaml
# Security Integration Config
security_config:
  authentication:
    method: certificate  # mTLS
    certificate_authority: internal-ca
    rotation_days: 90
    
  authorization:
    model: RBAC
    roles:
      - name: read_only
        permissions: [SELECT]
        
      - name: application
        permissions: [SELECT, INSERT, UPDATE, DELETE]
        restrictions:
          - no_ddl
          - no_truncate
          
      - name: admin
        permissions: [ALL]
        require_mfa: true
        
  encryption:
    at_rest:
      enabled: true
      provider: aws_kms
      key_rotation: 90d
      
    in_transit:
      tls_version: 1.3
      cipher_suites: [TLS_AES_256_GCM_SHA384]
      
  audit_logging:
    enabled: true
    log_targets:
      - splunk
      - s3
    events:
      - DDL_COMMANDS
      - PRIVILEGE_CHANGES
      - FAILED_LOGINS
      - DATA_EXPORT
```

**Access Control Flow:**
```
Application Request
  → API Gateway (JWT validation)
    → Service Identity (mTLS cert)
      → Database Architect (role mapping)
        → PostgreSQL (row-level security)
          → Audit Log (record access)
```

**Row-Level Security (RLS):**
```sql
-- Enable RLS on sensitive tables
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only see their own data
CREATE POLICY user_isolation ON users
  USING (user_id = current_user_id());
  
-- Policy: Admins can see all data
CREATE POLICY admin_access ON users
  USING (current_role = 'admin');
```

### 8. Backup & Disaster Recovery Integration

**Purpose:** Ensure data durability and rapid recovery

**Integration Pattern: Continuous Backup with PITR**

```yaml
# Backup Integration Config
backup_config:
  strategy: continuous
  
  base_backups:
    frequency: daily
    retention: 30d
    destination: s3://backups/database/
    compression: zstd
    encryption: aes-256
    
  wal_archiving:
    enabled: true
    destination: s3://wal-archive/
    compression: gzip
    retention: 7d
    
  point_in_time_recovery:
    enabled: true
    recovery_window: 7d
    
  cross_region_replication:
    enabled: true
    target_region: us-west-2
    sync_mode: async
```

**Backup Data Flow:**
```
Primary Database
  → Continuous WAL Archiving
    → S3 (encrypted, versioned)
      → Cross-Region Replication
        → S3 Secondary Region
  
  → Daily Base Backup (pg_basebackup)
    → S3 (compressed, encrypted)
      → Lifecycle Policy (move to Glacier after 30d)
```

**Recovery Scenarios:**

**1. Point-in-Time Recovery:**
```bash
# Restore to specific timestamp
pg_restore \
  --target-time="2026-02-11 09:30:00" \
  --base-backup=s3://backups/2026-02-11/ \
  --wal-archive=s3://wal-archive/
```

**2. Cross-Region Failover:**
```yaml
# Automated failover process
failover_procedure:
  1. Detect primary region outage (health checks)
  2. Promote standby in secondary region to primary
  3. Update DNS records (Route53)
  4. Redirect application traffic
  5. Monitor replication lag (should be zero)
  6. Alert operations team
```

## Integration APIs

### Database Connection API

```yaml
# Connection endpoint
POST /api/v1/connections
Authorization: Bearer <token>

Request:
  database: users_db
  mode: read  # or 'write'
  pool_size: 10
  timeout_ms: 5000

Response:
  connection_string: postgresql://...
  pool_id: pool_abc123
  endpoints:
    - host: db1.internal
      port: 5432
      role: primary
    - host: db2.internal
      port: 5432
      role: replica
```

### Migration API

```yaml
# Execute migration
POST /api/v1/migrations
Authorization: Bearer <token>

Request:
  migration_files:
    - V001__create_users_table.sql
    - V002__add_email_index.sql
  environment: staging
  dry_run: false

Response:
  migration_id: mig_xyz789
  status: success
  applied_migrations: 2
  execution_time_ms: 1523
```

### Health Check API

```yaml
# Check database health
GET /api/v1/health

Response:
  status: healthy
  checks:
    - name: primary_database
      status: up
      latency_ms: 12
      
    - name: replica_1
      status: up
      latency_ms: 15
      replication_lag_ms: 100
      
    - name: connection_pool
      status: healthy
      active: 12
      idle: 8
      max: 50
```

## Integration Testing

### Test Scenarios

**1. Connection Pool Exhaustion:**
```python
# Simulate high concurrent load
async def test_connection_pool():
    tasks = [execute_query() for _ in range(100)]
    results = await asyncio.gather(*tasks)
    assert all(r.status == 'success' for r in results)
```

**2. Replication Lag Handling:**
```python
# Write to primary, read from replica with eventual consistency
def test_eventual_consistency():
    db.write("INSERT INTO users ...")
    time.sleep(0.5)  # allow replication
    user = db.read("SELECT * FROM users WHERE id = ?")
    assert user is not None
```

**3. Failover Simulation:**
```python
# Test automatic failover to standby
def test_failover():
    primary_host = db.get_primary()
    simulate_outage(primary_host)
    
    # Wait for failover (should be < 30s)
    time.sleep(30)
    
    new_primary = db.get_primary()
    assert new_primary != primary_host
    assert db.execute("SELECT 1").success
```

## Integration Troubleshooting

### Common Issues

**1. Connection Pool Starvation**
- **Symptom:** Requests timing out, pool exhausted errors
- **Diagnosis:** Check active connections vs pool limit
- **Resolution:** Increase pool size, reduce query duration, implement connection timeout

**2. Replication Lag**
- **Symptom:** Stale data on replicas, inconsistent reads
- **Diagnosis:** Monitor replication lag metric
- **Resolution:** Scale replica resources, optimize write load, use read-after-write consistency

**3. Cache Inconsistency**
- **Symptom:** Cached data doesn't match database
- **Diagnosis:** Check cache invalidation events
- **Resolution:** Implement cache versioning, reduce TTL, use write-through strategy

**4. Migration Failures**
- **Symptom:** Schema out of sync, migration rollback needed
- **Diagnosis:** Review migration logs, check database locks
- **Resolution:** Test migrations on staging first, implement idempotent migrations

## Integration Best Practices

1. **Connection Management**
   - Always use connection pooling
   - Set appropriate timeout values
   - Close connections explicitly
   - Monitor pool utilization

2. **Query Optimization**
   - Use prepared statements
   - Implement query result caching
   - Batch INSERT/UPDATE operations
   - Index frequently queried columns

3. **Error Handling**
   - Implement retry logic with exponential backoff
   - Use circuit breakers for cascading failures
   - Log all errors with trace IDs
   - Provide meaningful error messages

4. **Security**
   - Never hardcode credentials
   - Use secrets management (Vault, AWS Secrets Manager)
   - Enable SSL/TLS for all connections
   - Implement least privilege access
   - Rotate credentials regularly

5. **Monitoring**
   - Track query latency (p50, p95, p99)
   - Monitor connection pool metrics
   - Alert on replication lag
   - Log slow queries (> 1s)
   - Track error rates

## References

- [Database Connection Pooling Best Practices](https://example.com)
- [PostgreSQL Replication Documentation](https://www.postgresql.org/docs/current/replication.html)
- [Cache Invalidation Strategies](https://example.com)
- [Microservices Database Patterns](https://microservices.io/patterns/data/)
