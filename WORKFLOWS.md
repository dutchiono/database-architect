# Database Workflows

## Development Workflow

### Schema Design Process

#### 1. Requirements Gathering
```markdown
## Feature: User Preferences System

### Data Requirements
- Store user preferences as flexible key-value pairs
- Support nested structures (theme, notifications, privacy)
- Enable fast lookups by user_id
- Track preference change history
- Support default values per preference type

### Performance Requirements
- Read latency: < 50ms (p95)
- Write latency: < 100ms (p95)
- Support 10,000 concurrent users
- 1M+ preference records

### Compliance
- GDPR: User preferences must be exportable and deletable
- Data retention: Keep history for 2 years
```

#### 2. Schema Design
```sql
-- V001__create_user_preferences.sql

-- Main preferences table (current state)
CREATE TABLE user_preferences (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    preferences JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    CONSTRAINT uniq_user_preferences UNIQUE (user_id)
);

-- Indexes for performance
CREATE INDEX idx_user_preferences_user_id ON user_preferences (user_id);
CREATE INDEX idx_user_preferences_jsonb ON user_preferences USING gin (preferences jsonb_path_ops);

-- Audit trail (history)
CREATE TABLE user_preferences_history (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    preferences JSONB NOT NULL,
    changed_at TIMESTAMP NOT NULL DEFAULT NOW(),
    changed_by VARCHAR(100)
) PARTITION BY RANGE (changed_at);

-- Monthly partitions for history
CREATE TABLE user_preferences_history_2026_02 PARTITION OF user_preferences_history
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

CREATE INDEX idx_prefs_history_user_id ON user_preferences_history (user_id, changed_at DESC);

-- Trigger to track changes
CREATE OR REPLACE FUNCTION log_preference_changes()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO user_preferences_history (user_id, preferences, changed_by)
    VALUES (NEW.user_id, NEW.preferences, current_user);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_log_preference_changes
    AFTER INSERT OR UPDATE ON user_preferences
    FOR EACH ROW
    EXECUTE FUNCTION log_preference_changes();

-- Helper function for querying preferences
CREATE OR REPLACE FUNCTION get_user_preference(
    p_user_id BIGINT,
    p_key TEXT,
    p_default TEXT DEFAULT NULL
)
RETURNS TEXT AS $$
BEGIN
    RETURN COALESCE(
        (SELECT preferences->>p_key FROM user_preferences WHERE user_id = p_user_id),
        p_default
    );
END;
$$ LANGUAGE plpgsql STABLE;
```

#### 3. Review Checklist
- [ ] Indexes on all foreign keys
- [ ] Appropriate constraints (NOT NULL, CHECK, UNIQUE)
- [ ] Cascade delete behavior defined
- [ ] Partitioning strategy for large tables
- [ ] Audit/history tracking implemented
- [ ] Performance considerations (denormalization if needed)
- [ ] Migration rollback script prepared
- [ ] Data retention policy defined

### Local Development Setup

#### 1. Docker Compose Environment
```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: dev-postgres
    environment:
      POSTGRES_DB: development
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: devpass
      PGDATA: /var/lib/postgresql/data/pgdata
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init.sql
    command:
      - "postgres"
      - "-c" "log_statement=all"
      - "-c" "log_duration=on"
      - "-c" "shared_preload_libraries=pg_stat_statements"
      - "-c" "pg_stat_statements.track=all"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U devuser -d development"]
      interval: 10s
      timeout: 5s
      retries: 5

  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: dev-pgadmin
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    depends_on:
      - postgres

volumes:
  postgres_data:
```

```bash
# Start development environment
docker-compose -f docker-compose.dev.yml up -d

# Run migrations
alembic upgrade head

# Seed test data
psql -h localhost -U devuser -d development -f scripts/seed-data.sql
```

#### 2. Migration Development
```bash
# Create new migration
alembic revision -m "add_user_preferences_table"

# Edit the generated migration file
# alembic/versions/001_add_user_preferences_table.py

# Test migration (up)
alembic upgrade head

# Test rollback (down)
alembic downgrade -1

# Verify database state
psql -h localhost -U devuser -d development -c "\dt"
psql -h localhost -U devuser -d development -c "\d user_preferences"
```

#### 3. Query Testing
```sql
-- Test query performance locally
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM user_preferences WHERE user_id = 123;

-- Check index usage
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan as index_scans,
    idx_tup_read as tuples_read,
    idx_tup_fetch as tuples_fetched
FROM pg_stat_user_indexes
WHERE tablename = 'user_preferences'
ORDER BY idx_scan DESC;
```

## Migration Workflow

### Safe Migration Process

#### Phase 1: Pre-Deployment Validation
```bash
#!/bin/bash
# scripts/validate-migration.sh

set -e

echo "=== Migration Validation ==="

# 1. Syntax check
echo "Checking SQL syntax..."
psql -h localhost -U devuser -d development --dry-run < migrations/V001__add_column.sql

# 2. Test on replica of production
echo "Testing on staging (prod replica)..."
psql -h staging-db.internal -U app_user -d staging < migrations/V001__add_column.sql

# 3. Rollback test
echo "Testing rollback..."
psql -h staging-db.internal -U app_user -d staging < migrations/V001__add_column.rollback.sql

# 4. Performance test
echo "Checking migration duration..."
START_TIME=$(date +%s)
psql -h staging-db.internal -U app_user -d staging < migrations/V001__add_column.sql
END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))

if [ $DURATION -gt 300 ]; then
    echo "WARNING: Migration took ${DURATION}s (>5 minutes)"
    exit 1
fi

echo "Validation passed (${DURATION}s)"
```

#### Phase 2: Zero-Downtime Migration Strategy
```sql
-- Example: Adding NOT NULL column

-- Step 1: Add nullable column (fast, no locks)
ALTER TABLE users ADD COLUMN phone_number VARCHAR(20);

-- Step 2: Add default constraint (fast)
ALTER TABLE users ALTER COLUMN phone_number SET DEFAULT '';

-- Step 3: Backfill in batches (avoid lock escalation)
DO $$
DECLARE
    batch_size INT := 10000;
    updated_rows INT;
BEGIN
    LOOP
        UPDATE users
        SET phone_number = ''
        WHERE id IN (
            SELECT id FROM users 
            WHERE phone_number IS NULL 
            LIMIT batch_size
        );
        
        GET DIAGNOSTICS updated_rows = ROW_COUNT;
        EXIT WHEN updated_rows = 0;
        
        RAISE NOTICE 'Updated % rows', updated_rows;
        COMMIT;
        PERFORM pg_sleep(0.1);  -- Throttle to avoid overload
    END LOOP;
END $$;

-- Step 4: Verify backfill complete
SELECT COUNT(*) FROM users WHERE phone_number IS NULL;
-- Should return 0

-- Step 5: Add NOT NULL constraint (fast, validated data)
ALTER TABLE users ALTER COLUMN phone_number SET NOT NULL;

-- Step 6: Add index if needed
CREATE INDEX CONCURRENTLY idx_users_phone ON users (phone_number);
```

#### Phase 3: Deployment Checklist
```markdown
## Migration Deployment Checklist

### Pre-Deployment
- [ ] Migration tested on staging (production replica)
- [ ] Rollback script prepared and tested
- [ ] Downtime estimate calculated (< 30s acceptable)
- [ ] Team notified (on-call engineers)
- [ ] Monitoring alerts adjusted (expect temp spike)
- [ ] Database backup verified (< 24h old)

### Deployment Window
- [ ] Enable maintenance mode if needed
- [ ] Monitor connection count
- [ ] Execute migration
- [ ] Verify schema changes
- [ ] Run smoke tests
- [ ] Check application logs for errors

### Post-Deployment
- [ ] Disable maintenance mode
- [ ] Verify application functionality
- [ ] Monitor query performance (pg_stat_statements)
- [ ] Check replication lag
- [ ] Update documentation
- [ ] Notify team of completion
```

## Backup & Recovery Workflow

### Automated Backup

#### Daily Backup Script
```bash
#!/bin/bash
# scripts/backup-database.sh

set -e

# Configuration
BACKUP_DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/postgresql"
DB_HOST="db-primary.internal"
DB_NAME="production"
S3_BUCKET="s3://company-db-backups/postgresql"

# Create backup directory
mkdir -p "${BACKUP_DIR}"

echo "Starting backup: ${BACKUP_DATE}"

# Full backup using pg_dump
pg_dump -h "${DB_HOST}" \
    -U postgres \
    -d "${DB_NAME}" \
    -F custom \
    -Z 9 \
    -f "${BACKUP_DIR}/backup_${BACKUP_DATE}.dump" \
    --verbose

# Verify backup integrity
pg_restore --list "${BACKUP_DIR}/backup_${BACKUP_DATE}.dump" > /dev/null
if [ $? -eq 0 ]; then
    echo "Backup integrity verified"
else
    echo "Backup verification failed!"
    exit 1
fi

# Upload to S3
aws s3 cp "${BACKUP_DIR}/backup_${BACKUP_DATE}.dump" \
    "${S3_BUCKET}/full/backup_${BACKUP_DATE}.dump" \
    --storage-class STANDARD_IA

# WAL archiving (continuous backup)
envdir /etc/wal-g.d/env wal-g backup-push "${PGDATA}"

# Cleanup old local backups (keep 7 days)
find "${BACKUP_DIR}" -name "backup_*.dump" -mtime +7 -delete

echo "Backup completed successfully"

# Send notification
curl -X POST https://hooks.slack.com/services/xxx \
    -H 'Content-Type: application/json' \
    -d "{\"text\":\"Database backup completed: ${BACKUP_DATE}\"}"
```

### Point-in-Time Recovery (PITR)

#### Recovery Procedure
```bash
#!/bin/bash
# scripts/restore-database.sh

set -e

TARGET_TIME="2026-02-11 10:00:00 UTC"
BACKUP_ID="backup_20260211_020000"
PGDATA="/var/lib/postgresql/15/main"

echo "=== Point-in-Time Recovery ==="
echo "Target time: ${TARGET_TIME}"

# 1. Stop PostgreSQL
systemctl stop postgresql

# 2. Backup current data (just in case)
mv "${PGDATA}" "${PGDATA}.bak.$(date +%s)"

# 3. Restore base backup
envdir /etc/wal-g.d/env wal-g backup-fetch "${PGDATA}" "${BACKUP_ID}"

# 4. Create recovery configuration
cat > "${PGDATA}/recovery.signal" <<EOF
# Point-in-time recovery configuration
restore_command = 'envdir /etc/wal-g.d/env wal-g wal-fetch "%f" "%p"'
recovery_target_time = '${TARGET_TIME}'
recovery_target_action = 'promote'
EOF

# 5. Set permissions
chown -R postgres:postgres "${PGDATA}"
chmod 700 "${PGDATA}"

# 6. Start PostgreSQL (will replay WAL logs)
echo "Starting recovery process..."
systemctl start postgresql

# 7. Monitor recovery progress
while [ -f "${PGDATA}/recovery.signal" ]; do
    echo "Recovery in progress..."
    sleep 5
done

echo "Recovery completed!"

# 8. Verify database state
psql -U postgres -d production -c "SELECT NOW() as current_time, pg_last_wal_replay_lsn() as replay_lsn;"

# 9. Verify data integrity
psql -U postgres -d production -c "SELECT COUNT(*) FROM users;"
psql -U postgres -d production -c "SELECT COUNT(*) FROM orders WHERE created_at < '${TARGET_TIME}';"
```

## Performance Optimization Workflow

### Query Analysis

#### 1. Identify Slow Queries
```sql
-- Top 10 slowest queries by mean execution time
SELECT 
    substring(query, 1, 100) as short_query,
    round(mean_exec_time::numeric, 2) as avg_time_ms,
    calls,
    round(total_exec_time::numeric, 2) as total_time_ms,
    round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) as pct_total_time
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat_statements%'
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Queries with lowest cache hit ratio
SELECT 
    substring(query, 1, 100) as short_query,
    calls,
    shared_blks_hit,
    shared_blks_read,
    round(
        (shared_blks_hit::numeric / NULLIF(shared_blks_hit + shared_blks_read, 0) * 100)::numeric, 
        2
    ) as cache_hit_ratio
FROM pg_stat_statements
WHERE (shared_blks_hit + shared_blks_read) > 0
ORDER BY cache_hit_ratio ASC
LIMIT 10;
```

#### 2. Analyze Query Plans
```sql
-- Get detailed execution plan
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, COSTS, TIMING) 
SELECT o.id, o.created_at, u.email, o.total_amount
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'pending'
  AND o.created_at >= NOW() - INTERVAL '30 days'
ORDER BY o.created_at DESC
LIMIT 100;
```

#### 3. Index Optimization
```sql
-- Find missing indexes (tables without indexes on foreign keys)
SELECT 
    c.conrelid::regclass AS table_name,
    string_agg(a.attname, ', ' ORDER BY x.n) AS columns,
    'CREATE INDEX idx_' || c.conrelid::regclass || '_' || 
        array_to_string(array_agg(a.attname ORDER BY x.n), '_') ||
    ' ON ' || c.conrelid::regclass || ' (' || 
        string_agg(a.attname, ', ' ORDER BY x.n) || ');' AS create_index
FROM pg_constraint c
CROSS JOIN LATERAL unnest(c.conkey) WITH ORDINALITY AS x(attnum, n)
JOIN pg_attribute a ON a.attnum = x.attnum AND a.attrelid = c.conrelid
WHERE c.contype = 'f'
  AND NOT EXISTS (
      SELECT 1 FROM pg_index i
      WHERE i.indrelid = c.conrelid
        AND c.conkey::int[] <@ i.indkey::int[]
  )
GROUP BY c.conrelid, c.conname
ORDER BY table_name;

-- Find unused indexes (candidates for removal)
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan as scans,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE 'pg_toast%'
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Database Tuning

#### PostgreSQL Configuration Tuning
```ini
# postgresql.conf - Production optimizations

# Memory
shared_buffers = 8GB                    # 25% of RAM
effective_cache_size = 24GB             # 75% of RAM
work_mem = 64MB                         # RAM / max_connections / 4
maintenance_work_mem = 2GB              # For VACUUM, CREATE INDEX
wal_buffers = 16MB

# Checkpoints
checkpoint_completion_target = 0.9
checkpoint_timeout = 15min
max_wal_size = 4GB
min_wal_size = 1GB

# Query Planning
default_statistics_target = 100
random_page_cost = 1.1                  # SSD storage
effective_io_concurrency = 200          # SSD storage

# Connections
max_connections = 200
superuser_reserved_connections = 3

# Logging
log_min_duration_statement = 1000       # Log queries > 1 second
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_autovacuum_min_duration = 0

# Autovacuum
autovacuum = on
autovacuum_max_workers = 4
autovacuum_naptime = 30s
autovacuum_vacuum_threshold = 50
autovacuum_vacuum_scale_factor = 0.1
autovacuum_analyze_threshold = 50
autovacuum_analyze_scale_factor = 0.05

# Replication
wal_level = replica
max_wal_senders = 10
wal_keep_size = 10GB
hot_standby = on
```

## Monitoring Workflow

### Daily Health Check
```bash
#!/bin/bash
# scripts/db-health-check.sh

echo "=== PostgreSQL Health Check ==="

# Connection count
echo -e "\n1. Connection Status:"
psql -U postgres -d production -c "
SELECT 
    state,
    COUNT(*) as connections
FROM pg_stat_activity
WHERE datname = 'production'
GROUP BY state;"

# Database size
echo -e "\n2. Database Size:"
psql -U postgres -d production -c "
SELECT 
    pg_size_pretty(pg_database_size('production')) as size;"

# Table bloat
echo -e "\n3. Table Bloat (top 5):"
psql -U postgres -d production -c "
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size,
    round(100 * (pg_total_relation_size(schemaname||'.'||tablename) - 
        pg_relation_size(schemaname||'.'||tablename))::numeric / 
        NULLIF(pg_total_relation_size(schemaname||'.'||tablename), 0), 2) as bloat_pct
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 5;"

# Replication lag
echo -e "\n4. Replication Lag:"
psql -U postgres -d production -c "
SELECT 
    client_addr,
    state,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;"

# Cache hit ratio
echo -e "\n5. Cache Hit Ratio:"
psql -U postgres -d production -c "
SELECT 
    round(100.0 * sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit + heap_blks_read), 0), 2) as cache_hit_ratio
FROM pg_statio_user_tables;"

echo -e "\n=== Health Check Complete ==="
```

---

**Related Documentation:**
- [README.md](./README.md) - System overview and quick start
- [ARCHITECTURE.md](./ARCHITECTURE.md) - Detailed architecture and data models
- [INTEGRATION.md](./INTEGRATION.md) - Application and tool integrations
