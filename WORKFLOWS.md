# Database Architect - Operational Workflows

## Workflow Overview

This document defines standard operational procedures for database architecture, schema management, performance optimization, disaster recovery, and routine maintenance tasks.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Database Workflow Lifecycle                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Design Phase          Implementation        Operations          │
│  ┌────────────┐       ┌────────────┐       ┌────────────┐      │
│  │  Schema    │──────→│ Migration  │──────→│ Monitoring │      │
│  │  Design    │       │ Execution  │       │ & Tuning   │      │
│  └────────────┘       └────────────┘       └────────────┘      │
│        │                     │                     │             │
│        ↓                     ↓                     ↓             │
│  Requirements          Testing              Optimization         │
│  Validation            & Rollback           & Scaling            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

## Core Workflows

### Workflow 1: Schema Design & Review

**Purpose:** Design database schemas following best practices and normalization principles

**Triggers:**
- New feature requiring database changes
- Performance optimization needs
- Data model refactoring

**Steps:**

**1. Gather Requirements**
```yaml
requirements_checklist:
  - [ ] Identify entities and relationships
  - [ ] Define data types and constraints
  - [ ] Estimate data volume and growth
  - [ ] Specify query patterns (read/write ratio)
  - [ ] Define performance requirements (latency, throughput)
  - [ ] Identify compliance requirements (PII, audit trails)
```

**2. Design Entity-Relationship Diagram**
```sql
-- Example ERD for e-commerce system

-- Users table
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Orders table
CREATE TABLE orders (
    order_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id),
    status VARCHAR(20) NOT NULL CHECK (status IN ('pending', 'paid', 'shipped', 'delivered', 'cancelled')),
    total_amount DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Order items table
CREATE TABLE order_items (
    order_item_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(order_id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(product_id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10,2) NOT NULL,
    subtotal DECIMAL(10,2) GENERATED ALWAYS AS (quantity * unit_price) STORED
);

-- Indexes for common queries
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
```

**3. Normalization Analysis**
```yaml
normalization_review:
  first_normal_form:
    - [ ] All attributes contain atomic values
    - [ ] No repeating groups
    
  second_normal_form:
    - [ ] No partial dependencies on composite keys
    
  third_normal_form:
    - [ ] No transitive dependencies
    
  denormalization_decisions:
    - table: order_items
      reason: Include product_name for faster queries (avoid join)
      trade_off: Update anomaly if product renamed
```

**4. Peer Review**
```yaml
review_checklist:
  - [ ] Naming conventions followed (snake_case)
  - [ ] Appropriate data types selected
  - [ ] Primary keys defined
  - [ ] Foreign key constraints specified
  - [ ] Indexes planned for query patterns
  - [ ] Constraints enforce business rules
  - [ ] Migration path from existing schema documented
```

**5. Approval & Documentation**
- Document schema in data dictionary
- Update ER diagrams in architecture docs
- Create JIRA ticket for implementation
- Get sign-off from tech lead

**Tools:**
- dbdiagram.io, draw.io (ERD creation)
- SQLFluff (SQL linting)
- PostgreSQL documentation

---

### Workflow 2: Database Migration Management

**Purpose:** Safely apply schema changes to production databases

**Triggers:**
- Approved schema design
- Bug fix requiring schema change
- Performance optimization (index addition)

**Steps:**

**1. Create Migration File**
```bash
# Use migration tool (Flyway, Liquibase, or custom)
flyway migrate -locations=migrations/

# Migration naming convention: V{version}__{description}.sql
# Example: V001__create_users_table.sql
```

**Migration Template:**
```sql
-- V005__add_email_verification.sql
-- Description: Add email verification fields to users table
-- Author: jane.doe@company.com
-- Date: 2026-02-11
-- Estimated downtime: 0 seconds (non-blocking)

-- Forward migration
BEGIN;

-- Add new columns (nullable initially for backward compatibility)
ALTER TABLE users 
  ADD COLUMN email_verified BOOLEAN DEFAULT FALSE,
  ADD COLUMN verification_token VARCHAR(255),
  ADD COLUMN verification_sent_at TIMESTAMP;

-- Create index for token lookup
CREATE INDEX CONCURRENTLY idx_users_verification_token 
  ON users(verification_token) 
  WHERE verification_token IS NOT NULL;

-- Backfill existing users as verified
UPDATE users 
  SET email_verified = TRUE 
  WHERE created_at < '2026-02-11';

COMMIT;

-- Rollback script (separate file or comment)
-- ALTER TABLE users DROP COLUMN email_verified, DROP COLUMN verification_token, DROP COLUMN verification_sent_at;
-- DROP INDEX idx_users_verification_token;
```

**2. Test Migration on Development Environment**
```bash
# Run migration
flyway migrate -url=jdbc:postgresql://dev-db/myapp -user=admin

# Verify schema
psql -h dev-db -d myapp -c "\d users"

# Test application functionality
./run_integration_tests.sh

# Test rollback
flyway undo
```

**3. Test Migration on Staging Environment**
```yaml
staging_migration_checklist:
  pre_migration:
    - [ ] Backup staging database
    - [ ] Verify application health (all services green)
    - [ ] Check replication lag (< 1 second)
    
  migration:
    - [ ] Apply migration (monitor for errors)
    - [ ] Verify migration status (flyway info)
    - [ ] Check application logs (no errors)
    
  post_migration:
    - [ ] Run smoke tests
    - [ ] Verify query performance (no regressions)
    - [ ] Document any issues observed
```

**4. Production Migration Planning**
```yaml
production_migration_plan:
  change_window: "2026-02-11 02:00-04:00 UTC"
  estimated_duration: "15 minutes"
  rollback_time: "5 minutes"
  
  prerequisites:
    - [ ] Staging migration successful
    - [ ] Change request approved (CR-12345)
    - [ ] DBA assigned (on-call available)
    - [ ] Communication sent (status page update)
    
  risk_assessment:
    risk_level: medium
    impact: "New columns added, backward compatible"
    rollback_plan: "DROP COLUMN commands prepared"
    
  monitoring:
    - Watch replication lag
    - Monitor query latency (p95, p99)
    - Check error rates in application logs
    - Verify connection pool health
```

**5. Execute Production Migration**
```bash
# Pre-migration checks
./pre_migration_checks.sh

# Backup database (if not already automated)
pg_dump -h prod-db -d myapp -Fc -f backup_pre_migration_$(date +%Y%m%d).dump

# Apply migration
flyway migrate -url=jdbc:postgresql://prod-db/myapp

# Post-migration verification
./post_migration_checks.sh

# Update status page
curl -X POST https://status.company.com/api/incidents/close \
  -d '{"incident_id": "12345", "status": "resolved"}'
```

**6. Post-Migration Monitoring**
```yaml
monitoring_duration: 24_hours
metrics_to_watch:
  - query_latency_p95
  - error_rate
  - replication_lag
  - connection_pool_utilization
  
alert_conditions:
  - metric: query_latency_p95
    threshold: "> 200ms"
    action: Investigate slow queries
    
  - metric: error_rate
    threshold: "> 1%"
    action: Consider rollback
```

**Tools:**
- Flyway, Liquibase (migration frameworks)
- pt-online-schema-change, gh-ost (online schema changes for MySQL)
- pgBackRest, Barman (PostgreSQL backup)

---

### Workflow 3: Query Performance Optimization

**Purpose:** Identify and resolve slow queries

**Triggers:**
- Slow query alerts (> 1 second)
- High CPU usage on database server
- User complaints about slow page loads

**Steps:**

**1. Identify Slow Queries**
```sql
-- PostgreSQL: Enable pg_stat_statements
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Find top 10 slowest queries
SELECT 
    query,
    calls,
    total_exec_time / calls AS avg_time_ms,
    total_exec_time,
    rows / calls AS avg_rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

**2. Analyze Query Execution Plan**
```sql
-- Get detailed execution plan
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) 
SELECT u.username, COUNT(o.order_id) 
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
WHERE u.created_at > '2026-01-01'
GROUP BY u.username;

-- Look for:
-- • Seq Scan (should be Index Scan for large tables)
-- • High cost estimates
-- • Large buffer reads
-- • Nested loops with many iterations
```

**3. Optimization Strategies**

**Strategy A: Add Missing Index**
```sql
-- Before: Seq Scan on orders (cost=0.00..5432.10 rows=50000)
-- Problem: Filtering by user_id without index

-- Solution: Create index
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);

-- After: Index Scan using idx_orders_user_id (cost=0.29..8.45 rows=15)
```

**Strategy B: Rewrite Query**
```sql
-- Before (slow): Subquery in SELECT
SELECT 
    u.username,
    (SELECT COUNT(*) FROM orders WHERE user_id = u.user_id) AS order_count
FROM users u;

-- After (fast): JOIN with aggregation
SELECT 
    u.username,
    COUNT(o.order_id) AS order_count
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
GROUP BY u.username;
```

**Strategy C: Use Materialized View**
```sql
-- For expensive aggregations that don't need real-time data
CREATE MATERIALIZED VIEW user_order_summary AS
SELECT 
    u.user_id,
    u.username,
    COUNT(o.order_id) AS total_orders,
    SUM(o.total_amount) AS lifetime_value
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
GROUP BY u.user_id, u.username;

-- Create index on materialized view
CREATE INDEX idx_user_order_summary_user_id ON user_order_summary(user_id);

-- Refresh materialized view (can be scheduled)
REFRESH MATERIALIZED VIEW CONCURRENTLY user_order_summary;
```

**4. Test Optimization**
```sql
-- Compare before/after performance
-- Before optimization
\timing on
SELECT ... FROM users WHERE ...;
-- Time: 3421.563 ms

-- After optimization
SELECT ... FROM users WHERE ...;
-- Time: 45.231 ms  (75x improvement)
```

**5. Deploy to Production**
```yaml
deployment_plan:
  change_type: index_addition
  impact: low (online operation)
  
  steps:
    - [ ] Create index CONCURRENTLY (no table lock)
    - [ ] Monitor index creation progress
    - [ ] Verify query uses new index (EXPLAIN)
    - [ ] Monitor query latency improvement
    
  rollback:
    - Drop index if performance degrades
```

**Tools:**
- pg_stat_statements (PostgreSQL)
- MySQL slow query log
- EXPLAIN plans
- pgBadger (PostgreSQL log analyzer)

---

### Workflow 4: Database Scaling

**Purpose:** Scale database to handle increased load

**Triggers:**
- CPU usage > 80% sustained
- Connection pool exhaustion
- Query latency degradation
- Storage capacity warnings

**Steps:**

**1. Diagnose Bottleneck**
```yaml
bottleneck_analysis:
  cpu_bound:
    symptoms:
      - High CPU usage
      - Many concurrent queries
    solutions:
      - Add read replicas
      - Optimize slow queries
      - Cache frequently accessed data
      
  memory_bound:
    symptoms:
      - High memory usage
      - Disk swapping
      - Low buffer cache hit ratio
    solutions:
      - Increase RAM
      - Tune shared_buffers, work_mem
      - Optimize queries to use less memory
      
  io_bound:
    symptoms:
      - High disk I/O wait
      - Slow query execution
    solutions:
      - Upgrade to SSD/NVMe
      - Optimize indexes
      - Partition large tables
      
  connection_bound:
    symptoms:
      - "Too many connections" errors
      - High connection pool utilization
    solutions:
      - Implement connection pooling (PgBouncer)
      - Increase max_connections (with more RAM)
      - Optimize connection lifetime
```

**2. Vertical Scaling (Scale Up)**
```yaml
# Upgrade database instance size
vertical_scaling_plan:
  current_instance: db.r5.xlarge (4 vCPU, 32 GB RAM)
  target_instance: db.r5.2xlarge (8 vCPU, 64 GB RAM)
  
  downtime: ~5 minutes (failover to standby)
  
  steps:
    - [ ] Create snapshot before resize
    - [ ] Schedule maintenance window
    - [ ] Resize instance (AWS console or CLI)
    - [ ] Verify new resources available
    - [ ] Monitor performance improvement
```

**3. Horizontal Scaling (Scale Out)**

**A. Add Read Replicas**
```yaml
# Route read traffic to replicas
read_replica_setup:
  primary: db-primary.internal (handles writes)
  replicas:
    - db-replica-1.internal (read-only)
    - db-replica-2.internal (read-only)
    - db-replica-3.internal (read-only)
    
  application_changes:
    - Configure read/write split in ORM
    - Use connection string with read endpoints
    - Handle eventual consistency
```

```python
# Application code for read/write split
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# Write connection (primary)
write_engine = create_engine('postgresql://db-primary.internal/myapp')
WriteSession = sessionmaker(bind=write_engine)

# Read connection (replicas with load balancing)
read_engine = create_engine('postgresql://db-replicas.internal/myapp')
ReadSession = sessionmaker(bind=read_engine)

# Usage
def get_user(user_id):
    session = ReadSession()  # Read from replica
    return session.query(User).filter_by(id=user_id).first()

def create_user(data):
    session = WriteSession()  # Write to primary
    user = User(**data)
    session.add(user)
    session.commit()
```

**B. Implement Sharding**
```yaml
# Shard by user_id range
sharding_strategy:
  shard_key: user_id
  shards:
    - name: shard_1
      range: 0000-4999
      database: db-shard-1.internal
      
    - name: shard_2
      range: 5000-9999
      database: db-shard-2.internal
```

```python
# Shard routing logic
def get_shard(user_id):
    if user_id < 5000:
        return 'db-shard-1.internal'
    else:
        return 'db-shard-2.internal'

def query_user(user_id):
    shard_db = get_shard(user_id)
    connection = connect(shard_db)
    return connection.execute("SELECT * FROM users WHERE user_id = ?", user_id)
```

**4. Monitor Scaling Impact**
```yaml
post_scaling_metrics:
  duration: 7_days
  
  metrics:
    - cpu_utilization (should decrease)
    - query_latency_p95 (should improve)
    - connection_pool_usage (should decrease)
    - replication_lag (should remain < 1s)
    
  success_criteria:
    - CPU usage < 70%
    - Query latency p95 < 100ms
    - Connection pool < 60% utilized
    - No errors in application logs
```

**Tools:**
- AWS RDS, Azure Database, Google Cloud SQL (managed databases)
- PgBouncer, ProxySQL (connection pooling)
- Vitess, Citus (sharding)

---

### Workflow 5: Backup & Disaster Recovery

**Purpose:** Protect data and recover from disasters

**Triggers:**
- Scheduled backups (daily, weekly)
- Before major schema changes
- Disaster recovery drills (quarterly)

**Steps:**

**1. Configure Automated Backups**
```yaml
# Backup configuration
backup_config:
  base_backups:
    frequency: daily
    time: 02:00 UTC
    retention: 30 days
    destination: s3://backups/postgres/
    
  wal_archiving:
    enabled: true
    archive_command: "aws s3 cp %p s3://wal-archive/%f"
    retention: 7 days
    
  point_in_time_recovery:
    enabled: true
    recovery_window: 7 days
```

**2. Execute Backup**
```bash
# PostgreSQL base backup
pg_basebackup \
  -h db-primary.internal \
  -D /backup/base/$(date +%Y%m%d) \
  -Ft -z -P \
  -U backup_user

# Upload to S3
aws s3 sync /backup/base/$(date +%Y%m%d) \
  s3://backups/postgres/$(date +%Y%m%d)/

# Verify backup integrity
pg_verifybackup /backup/base/$(date +%Y%m%d)
```

**3. Test Restore (Monthly)**
```yaml
restore_test_plan:
  frequency: monthly
  environment: staging
  
  steps:
    - [ ] Provision new database instance
    - [ ] Restore from latest backup
    - [ ] Apply WAL archives (PITR)
    - [ ] Verify data integrity (row counts, checksums)
    - [ ] Test application connectivity
    - [ ] Document restore time (RTO metric)
    - [ ] Destroy test instance
```

**4. Disaster Recovery Procedure**
```yaml
# Primary database failure scenario
disaster_recovery_steps:
  1_detect_failure:
    - Health check fails for 3 consecutive attempts
    - Alert sent to on-call DBA
    
  2_assess_damage:
    - Check if primary is recoverable
    - Determine data loss extent (RPO)
    
  3_failover_to_standby:
    - Promote standby replica to primary
    - Update DNS records
    - Redirect application traffic
    
  4_verify_operations:
    - Test write operations
    - Check replication to remaining replicas
    - Monitor error rates
    
  5_post_incident:
    - Root cause analysis
    - Update runbooks
    - Schedule DR drill review
```

**5. Point-in-Time Recovery**
```bash
# Restore to specific timestamp (e.g., before bad deployment)
# Step 1: Restore base backup
pg_restore \
  -h db-recovery.internal \
  -d myapp \
  /backup/base/20260211

# Step 2: Configure recovery target
cat > /var/lib/postgresql/data/recovery.conf << EOF
restore_command = 'aws s3 cp s3://wal-archive/%f %p'
recovery_target_time = '2026-02-11 09:30:00'
recovery_target_action = 'promote'
EOF

# Step 3: Start PostgreSQL (applies WAL up to target time)
systemctl start postgresql

# Step 4: Verify recovered data
psql -c "SELECT NOW();"  # Should show recovery target time
```

**Tools:**
- pgBackRest, Barman (PostgreSQL backup tools)
- AWS Backup, Azure Backup (cloud-native)
- Percona XtraBackup (MySQL)

---

### Workflow 6: Security Audit & Compliance

**Purpose:** Ensure database security and regulatory compliance

**Triggers:**
- Quarterly security audits
- Compliance requirements (SOC 2, HIPAA, GDPR)
- Security incident

**Steps:**

**1. Access Control Audit**
```sql
-- Review all database users and roles
SELECT 
    rolname AS role,
    rolsuper AS is_superuser,
    rolcreatedb AS can_create_db,
    rolcanlogin AS can_login
FROM pg_roles
ORDER BY rolname;

-- Review table-level permissions
SELECT 
    grantee,
    table_schema,
    table_name,
    privilege_type
FROM information_schema.role_table_grants
WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY grantee, table_name;
```

**Checklist:**
```yaml
access_control_audit:
  - [ ] No unnecessary superuser accounts
  - [ ] Service accounts follow least privilege
  - [ ] No shared credentials
  - [ ] Unused accounts disabled
  - [ ] Default passwords changed
  - [ ] Strong password policy enforced
```

**2. Encryption Verification**
```yaml
encryption_audit:
  at_rest:
    - [ ] Disk encryption enabled (LUKS, AWS EBS encryption)
    - [ ] TDE (Transparent Data Encryption) configured
    - [ ] Backup encryption enabled
    - [ ] Key rotation schedule defined
    
  in_transit:
    - [ ] SSL/TLS enforced for all connections
    - [ ] Certificate expiration monitored
    - [ ] Weak ciphers disabled
    - [ ] mTLS for service-to-service communication
```

**3. Audit Logging Review**
```sql
-- Enable audit logging
ALTER SYSTEM SET log_statement = 'ddl';  -- Log DDL commands
ALTER SYSTEM SET log_connections = on;   -- Log connections
ALTER SYSTEM SET log_disconnections = on;
ALTER SYSTEM SET log_duration = on;      -- Log query duration
SELECT pg_reload_conf();

-- Review audit logs
SELECT * FROM pg_read_file('pg_log/postgresql-2026-02-11.log');
```

**4. Vulnerability Scan**
```bash
# Scan for known vulnerabilities
nmap -sV -p 5432 db-primary.internal

# Check PostgreSQL version
psql -c "SELECT version();"

# Verify no default/weak passwords
./check_weak_passwords.sh

# Review network exposure
netstat -an | grep 5432
```

**5. Compliance Reporting**
```yaml
# GDPR compliance example
gdpr_compliance:
  data_inventory:
    - [ ] PII fields documented (email, name, address)
    - [ ] Data retention policies defined
    - [ ] Data processing agreements signed
    
  user_rights:
    - [ ] Right to access (data export capability)
    - [ ] Right to erasure (delete user data)
    - [ ] Right to rectification (update user data)
    - [ ] Right to portability (JSON/CSV export)
    
  security_measures:
    - [ ] Encryption at rest and in transit
    - [ ] Access controls and audit logging
    - [ ] Regular security audits
    - [ ] Incident response plan
```

**Tools:**
- OpenSCAP, Lynis (security scanners)
- pgAudit (PostgreSQL audit extension)
- Vault, AWS Secrets Manager (secrets management)

---

## Operational Runbooks

### Runbook: Database Connection Issues

**Symptoms:**
- Application errors: "Too many connections"
- High connection pool utilization
- Slow query performance

**Diagnosis:**
```sql
-- Check current connections
SELECT 
    COUNT(*) AS total_connections,
    MAX(max_connections) AS max_allowed
FROM pg_stat_activity, pg_settings
WHERE name = 'max_connections';

-- Identify idle connections
SELECT 
    pid,
    usename,
    application_name,
    state,
    NOW() - state_change AS idle_duration
FROM pg_stat_activity
WHERE state = 'idle'
ORDER BY idle_duration DESC;
```

**Resolution:**
```sql
-- Terminate idle connections (> 1 hour idle)
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
  AND NOW() - state_change > INTERVAL '1 hour';

-- Increase max_connections (requires restart)
ALTER SYSTEM SET max_connections = 300;
SELECT pg_reload_conf();  -- or restart PostgreSQL
```

---

### Runbook: Replication Lag

**Symptoms:**
- Stale data on replicas
- Replication lag metric increasing
- Read queries returning old data

**Diagnosis:**
```sql
-- Check replication status (on primary)
SELECT 
    client_addr AS replica,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    sync_state,
    (sent_lsn - replay_lsn) AS lag_bytes
FROM pg_stat_replication;

-- Check replication lag in seconds
SELECT 
    NOW() - pg_last_xact_replay_timestamp() AS replication_lag
FROM pg_stat_replication;
```

**Resolution:**
```yaml
short_term:
  - Reduce write load on primary
  - Scale up replica resources (CPU, I/O)
  - Optimize slow queries on replica
  
long_term:
  - Upgrade replica instance size
  - Implement connection pooling
  - Consider sharding to distribute writes
```

---

## Monitoring Dashboards

### Dashboard: Database Health
```yaml
panels:
  - title: Query Latency (p95)
    query: histogram_quantile(0.95, rate(pg_query_duration_seconds_bucket[5m]))
    threshold_warning: 100ms
    threshold_critical: 500ms
    
  - title: Connection Pool Utilization
    query: pg_connections_active / pg_connections_max * 100
    threshold_warning: 70%
    threshold_critical: 90%
    
  - title: Replication Lag
    query: pg_replication_lag_seconds
    threshold_warning: 5s
    threshold_critical: 30s
    
  - title: Disk Usage
    query: (1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100
    threshold_warning: 80%
    threshold_critical: 90%
```

## Best Practices Summary

1. **Schema Design**
   - Follow normalization principles (3NF minimum)
   - Use appropriate data types
   - Define constraints and indexes
   - Document all schema changes

2. **Migrations**
   - Test on staging before production
   - Use backward-compatible changes when possible
   - Always have rollback plan
   - Monitor post-migration performance

3. **Performance**
   - Monitor slow queries (> 1 second)
   - Use EXPLAIN for query optimization
   - Create indexes for frequent queries
   - Cache frequently accessed data

4. **Backups**
   - Automate daily backups
   - Test restores monthly
   - Enable point-in-time recovery
   - Store backups in separate region

5. **Security**
   - Follow least privilege principle
   - Enable SSL/TLS encryption
   - Audit access regularly
   - Rotate credentials quarterly

## References

- [PostgreSQL Administration](https://www.postgresql.org/docs/current/admin.html)
- [MySQL Performance Tuning](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)
- [Database Reliability Engineering](https://www.oreilly.com/library/view/database-reliability-engineering/9781491925935/)
- [Google SRE Book - Managing Database Reliability](https://sre.google/sre-book/managing-critical-state/)
