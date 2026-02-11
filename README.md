# Database Architect

> Intelligent database architecture, optimization, and management system for distributed data systems

## Overview

Database Architect is a comprehensive AI agent specialized in designing, implementing, and optimizing database systems for modern distributed applications. It handles schema design, query optimization, replication strategies, backup and recovery, performance tuning, and migrations across SQL and NoSQL databases.

## Core Capabilities

### Database Design & Architecture
- Normalized schema design (1NF through BCNF)
- Denormalization strategies for read-heavy workloads
- Partitioning and sharding strategies (horizontal, vertical, range, hash)
- Multi-tenant database architectures (shared schema, separate schema, separate database)
- Data modeling for time-series, graph, document, and key-value stores

### Performance Optimization
- Query analysis and optimization (EXPLAIN plans, index tuning)
- Index strategy design (B-tree, hash, GiST, GIN, BRIN)
- Connection pooling and prepared statement optimization
- Query caching and materialized views
- Database parameter tuning (shared_buffers, work_mem, effective_cache_size)

### High Availability & Replication
- Master-slave replication (asynchronous, synchronous, semi-synchronous)
- Multi-master replication and conflict resolution
- Read replicas and load balancing
- Automated failover with health monitoring
- Cross-region replication for disaster recovery

### Backup & Recovery
- Full, incremental, and differential backup strategies
- Point-in-time recovery (PITR)
- Continuous archiving with WAL (Write-Ahead Logging)
- Automated backup scheduling and retention policies
- Disaster recovery testing and validation

### Migrations & Versioning
- Schema migration frameworks (Flyway, Liquibase, Alembic)
- Zero-downtime migration strategies
- Data migration pipelines (ETL/ELT)
- Version control for database schemas
- Rollback and forward migration plans

## Technology Stack

**Relational Databases:**
- PostgreSQL (14+, with extensions: PostGIS, TimescaleDB, pg_partman)
- MySQL / MariaDB (8.0+, Galera Cluster)
- Amazon RDS / Aurora (PostgreSQL, MySQL)
- Google Cloud SQL (PostgreSQL, MySQL)

**NoSQL Databases:**
- MongoDB (replica sets, sharded clusters)
- Redis (cluster mode, Sentinel)
- Cassandra (wide-column store, tunable consistency)
- Elasticsearch (distributed search and analytics)

**NewSQL & Distributed SQL:**
- CockroachDB (distributed SQL, geo-partitioning)
- TiDB (MySQL-compatible distributed database)
- YugabyteDB (PostgreSQL-compatible, multi-region)

**Data Warehouses:**
- Snowflake (cloud data warehouse)
- BigQuery (serverless analytics)
- Redshift (columnar storage, massively parallel processing)

**Supporting Tools:**
- pgBouncer / PgPool-II (connection pooling)
- WAL-G / pgBackRest (backup and archiving)
- pg_stat_statements (query performance monitoring)
- Patroni (HA cluster management for PostgreSQL)
- ProxySQL (MySQL query routing and caching)

## Quick Start

### PostgreSQL Setup

#### Local Development with Docker
```bash
# PostgreSQL with replication slots
docker run -d \
  --name postgres-primary \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_INITDB_ARGS="-c wal_level=logical -c max_replication_slots=4" \
  -v postgres_data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:15-alpine

# Connect and verify
psql -h localhost -U postgres -c "SHOW wal_level;"
```

#### Production HA Cluster with Patroni
```yaml
# patroni.yml
scope: postgres-cluster
namespace: /db/
name: postgres-node-1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.1.10:8008

etcd:
  hosts: 10.0.1.20:2379,10.0.1.21:2379,10.0.1.22:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:
        max_connections: 500
        shared_buffers: 4GB
        effective_cache_size: 12GB
        work_mem: 16MB
        maintenance_work_mem: 1GB
        checkpoint_completion_target: 0.9
        wal_buffers: 16MB
        default_statistics_target: 100
        random_page_cost: 1.1
        effective_io_concurrency: 200
        max_wal_size: 4GB
        min_wal_size: 1GB

  initdb:
    - encoding: UTF8
    - data-checksums

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.1.10:5432
  data_dir: /var/lib/postgresql/15/main
  bin_dir: /usr/lib/postgresql/15/bin
  authentication:
    replication:
      username: replicator
      password: repl_password
    superuser:
      username: postgres
      password: postgres_password
```

### Schema Design Example

#### E-Commerce Database Schema
```sql
-- Users table with partitioning by registration date
CREATE TABLE users (
    id BIGSERIAL,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- Partitions by quarter
CREATE TABLE users_2024_q1 PARTITION OF users
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE users_2024_q2 PARTITION OF users
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

-- Indexes
CREATE INDEX idx_users_email ON users USING btree (email);
CREATE INDEX idx_users_created_at ON users USING brin (created_at);

-- Products table with full-text search
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    sku VARCHAR(50) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL,
    stock_quantity INTEGER NOT NULL DEFAULT 0,
    category_id INTEGER REFERENCES categories(id),
    search_vector tsvector GENERATED ALWAYS AS (
        to_tsvector('english', coalesce(name, '') || ' ' || coalesce(description, ''))
    ) STORED,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Full-text search index
CREATE INDEX idx_products_search ON products USING gin (search_vector);
CREATE INDEX idx_products_category ON products (category_id);

-- Orders table with proper foreign keys
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    total_amount DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_order_status CHECK (status IN ('pending', 'processing', 'shipped', 'delivered', 'cancelled'))
);

CREATE INDEX idx_orders_user_id ON orders (user_id);
CREATE INDEX idx_orders_status ON orders (status) WHERE status IN ('pending', 'processing');
CREATE INDEX idx_orders_created_at ON orders (created_at DESC);

-- Order items (denormalized for performance)
CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL REFERENCES products(id),
    product_name VARCHAR(255) NOT NULL,  -- Denormalized
    product_price DECIMAL(10,2) NOT NULL,  -- Denormalized
    quantity INTEGER NOT NULL,
    subtotal DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_order_items_order_id ON order_items (order_id);

-- Materialized view for analytics
CREATE MATERIALIZED VIEW daily_sales AS
SELECT 
    DATE(created_at) as sale_date,
    COUNT(*) as order_count,
    SUM(total_amount) as total_revenue,
    AVG(total_amount) as avg_order_value
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY DATE(created_at)
ORDER BY sale_date DESC;

CREATE UNIQUE INDEX idx_daily_sales_date ON daily_sales (sale_date);

-- Refresh materialized view automatically
CREATE EXTENSION IF NOT EXISTS pg_cron;
SELECT cron.schedule('refresh-daily-sales', '0 1 * * *', 'REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales');
```

### Query Optimization Example

#### Before Optimization
```sql
-- Slow query: full table scan, no indexes
SELECT o.id, o.created_at, u.email, o.total_amount
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'processing'
  AND o.created_at >= NOW() - INTERVAL '30 days'
ORDER BY o.created_at DESC
LIMIT 100;

-- EXPLAIN ANALYZE output:
-- Seq Scan on orders  (cost=0.00..125000.00 rows=50000 width=50) (actual time=245.123..2341.567 rows=5432 loops=1)
```

#### After Optimization
```sql
-- Add composite index
CREATE INDEX idx_orders_status_created_at ON orders (status, created_at DESC)
WHERE status IN ('pending', 'processing');

-- Optimized query with index hint
SELECT o.id, o.created_at, u.email, o.total_amount
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'processing'
  AND o.created_at >= NOW() - INTERVAL '30 days'
ORDER BY o.created_at DESC
LIMIT 100;

-- EXPLAIN ANALYZE output:
-- Index Scan using idx_orders_status_created_at on orders  (cost=0.56..243.67 rows=100 width=50) (actual time=0.234..1.567 rows=100 loops=1)
-- Performance improvement: 1000x faster
```

## Key Features

### 1. Automated Schema Migrations
```python
# Alembic migration example
from alembic import op
import sqlalchemy as sa

def upgrade():
    # Add column with default value (zero-downtime)
    op.add_column('users', 
        sa.Column('phone_number', sa.String(20), nullable=True)
    )
    
    # Backfill data in batches
    connection = op.get_bind()
    connection.execute("""
        UPDATE users 
        SET phone_number = ''
        WHERE phone_number IS NULL
        AND id IN (SELECT id FROM users WHERE phone_number IS NULL LIMIT 10000)
    """)
    
    # Make column non-nullable after backfill
    op.alter_column('users', 'phone_number', nullable=False)

def downgrade():
    op.drop_column('users', 'phone_number')
```

### 2. Replication Setup
```bash
# PostgreSQL streaming replication
# On primary server
cat >> /etc/postgresql/15/main/postgresql.conf <<EOF
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1GB
hot_standby = on
EOF

# Create replication user
psql -U postgres -c "CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'repl_pass';"

# On replica server
pg_basebackup -h primary.example.com -D /var/lib/postgresql/15/replica -U replicator -P -v -R

# standby.signal file created automatically by -R flag
# Start replica
systemctl start postgresql
```

### 3. Backup Strategy
```bash
#!/bin/bash
# Automated backup with WAL-G

# Full backup (weekly)
envdir /etc/wal-g.d/env wal-g backup-push /var/lib/postgresql/15/main

# Verify backup
envdir /etc/wal-g.d/env wal-g backup-list

# Point-in-time recovery example
# Stop PostgreSQL
systemctl stop postgresql

# Restore from backup
envdir /etc/wal-g.d/env wal-g backup-fetch /var/lib/postgresql/15/main LATEST

# Create recovery configuration
cat > /var/lib/postgresql/15/main/recovery.signal <<EOF
restore_command = 'envdir /etc/wal-g.d/env wal-g wal-fetch "%f" "%p"'
recovery_target_time = '2026-02-11 10:00:00'
EOF

# Start PostgreSQL
systemctl start postgresql
```

### 4. Connection Pooling
```ini
# PgBouncer configuration
[databases]
mydb = host=localhost port=5432 dbname=production

[pgbouncer]
listen_addr = *
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
reserve_pool_size = 5
reserve_pool_timeout = 3

server_idle_timeout = 600
server_lifetime = 3600
server_connect_timeout = 15

log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
```

## Architecture Patterns

### Master-Slave Replication
```
┌─────────────┐
│   Primary   │ ← Writes
│  (Master)   │
└─────┬───────┘
      │ WAL streaming
      ├────────────────┐
      ▼                ▼
┌───────────┐    ┌───────────┐
│  Replica  │    │  Replica  │ ← Reads
│  (Slave)  │    │  (Slave)  │
└───────────┘    └───────────┘
```

### Sharding Strategy
```
Application Layer
      ↓
┌─────────────┐
│ Shard Router│ (ProxySQL / Vitess)
└──────┬──────┘
       │
   ┌───┴────┬────────┬────────┐
   ▼        ▼        ▼        ▼
Shard 1  Shard 2  Shard 3  Shard 4
(Users   (Users   (Users   (Users
 0-250K)  250-500K) 500-750K) 750K-1M)
```

### Multi-Tenant Architecture
```
Option 1: Shared Schema (tenant_id column)
- All tenants in one database
- Row-level security policies
- Efficient resource usage

Option 2: Separate Schema per Tenant
- Each tenant has own schema
- Schema-level isolation
- Moderate resource overhead

Option 3: Separate Database per Tenant
- Complete isolation
- Independent scaling
- Higher resource overhead
```

## Best Practices

1. **Always Use Connection Pooling**: Prevent connection exhaustion with PgBouncer/ProxySQL
2. **Index Strategically**: Index foreign keys, WHERE clauses, JOIN columns, but avoid over-indexing
3. **Monitor Query Performance**: Use pg_stat_statements, slow query logs
4. **Regular VACUUM**: Schedule autovacuum, manual VACUUM FULL for bloated tables
5. **Backup and Test Recovery**: Automate backups, test recovery procedures regularly
6. **Use Prepared Statements**: Prevent SQL injection, improve query caching
7. **Partition Large Tables**: Time-based or hash partitioning for tables > 100GB
8. **Monitor Replication Lag**: Alert when lag exceeds acceptable threshold

## Monitoring & Metrics

**Key Metrics:**
- Connections (active, idle, waiting)
- Query latency (p50, p95, p99)
- Transaction rate (commits/sec, rollbacks/sec)
- Cache hit ratio (buffer cache, index cache)
- Replication lag (bytes, seconds behind master)
- Disk I/O (reads/writes per second)
- Table bloat percentage

**Alert Conditions:**
- Replication lag > 10 seconds
- Connection usage > 80%
- Cache hit ratio < 95%
- Long-running queries > 5 minutes
- Disk space < 20% free

## Documentation

- [ARCHITECTURE.md](./ARCHITECTURE.md) - System design, schema patterns, data models
- [INTEGRATION.md](./INTEGRATION.md) - Application integrations, ORMs, migration tools
- [WORKFLOWS.md](./WORKFLOWS.md) - Development workflow, migration procedures, backup/recovery

## Support

For issues, questions, or contributions, please refer to the main repository documentation.

## License

Copyright © 2026. All rights reserved.
