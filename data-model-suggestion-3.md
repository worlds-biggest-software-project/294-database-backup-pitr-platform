# Data Model Suggestion 3: Hybrid Relational + JSONB/Document Approach

> Project: Database Backup & PITR Platform
> Approach: PostgreSQL with structured columns for core data and JSONB for engine-specific metadata

---

## Summary

A hybrid data model that uses PostgreSQL's relational capabilities for well-defined, frequently queried fields (status, timestamps, relationships) while leveraging JSONB columns for engine-specific metadata, configuration, and extensible attributes. This approach addresses one of the central challenges of a multi-database backup platform: PostgreSQL WAL segments, MySQL binlog events, and MongoDB oplog slices have fundamentally different metadata shapes that resist a single normalized schema.

The relational backbone provides strong typing, foreign key integrity, and efficient joins for the core backup catalog. JSONB columns absorb the variability between database engines, backup tool configurations, and evolving feature requirements without schema migrations.

---

## Key Entities and Relationships

### Design Principle: Structured Core + Flexible Envelope

```
┌──────────────────────────────────────────────────────────┐
│                    Relational Core                        │
│  (typed columns, foreign keys, indexes, constraints)      │
│                                                           │
│   id, database_id, status, backup_type, started_at,       │
│   completed_at, size_bytes, storage_path                   │
├──────────────────────────────────────────────────────────┤
│                    JSONB Envelope                          │
│  (engine-specific details, tool config, extensible)        │
│                                                           │
│   engine_metadata: { wal_start_lsn, timeline_id, ... }    │
│   tool_config: { compression, parallel_jobs, ... }         │
│   validation_details: { checks: [...], warnings: [...] }   │
│   ai_context: { anomaly_scores, recommendations, ... }     │
└──────────────────────────────────────────────────────────┘
```

### Core Schema

```sql
-- Multi-tenancy
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT UNIQUE NOT NULL,
    settings        JSONB NOT NULL DEFAULT '{}', -- org-level preferences
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users with flexible auth attributes
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           TEXT UNIQUE NOT NULL,
    display_name    TEXT NOT NULL,
    auth_config     JSONB NOT NULL DEFAULT '{}', -- provider-specific: OIDC claims, SAML attrs
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- RBAC with flexible permission model
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]', -- flexible permission structure
    UNIQUE(organization_id, name)
);

CREATE TABLE user_roles (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by UUID REFERENCES users(id),
    PRIMARY KEY (user_id, role_id)
);

-- Database instances with engine-specific connection config
CREATE TABLE database_instances (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    engine          TEXT NOT NULL CHECK (engine IN ('postgresql', 'mysql', 'mongodb')),
    engine_version  TEXT,
    environment     TEXT NOT NULL DEFAULT 'production',
    -- Core connection fields (common to all engines)
    host            TEXT NOT NULL,
    port            INTEGER NOT NULL,
    -- Engine-specific connection parameters in JSONB
    connection_config JSONB NOT NULL DEFAULT '{}',
    /*
     * PostgreSQL: {"database": "mydb", "sslmode": "verify-full", "replication_slot": "backup_slot"}
     * MySQL:      {"database": "mydb", "binlog_format": "ROW", "server_id": 100}
     * MongoDB:    {"replica_set": "rs0", "auth_db": "admin", "read_preference": "secondary"}
     */
    -- Health and status
    last_health_check TIMESTAMPTZ,
    health_status   TEXT DEFAULT 'unknown',
    health_details  JSONB DEFAULT '{}',
    tags            JSONB NOT NULL DEFAULT '[]', -- user-defined tags for grouping
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_db_instances_org ON database_instances(organization_id);
CREATE INDEX idx_db_instances_engine ON database_instances(engine);
CREATE INDEX idx_db_instances_tags ON database_instances USING GIN (tags);

-- Storage locations with provider-specific config
CREATE TABLE storage_locations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    provider        TEXT NOT NULL CHECK (provider IN ('s3', 'gcs', 'azure_blob', 'sftp', 'minio')),
    -- Common fields
    bucket          TEXT NOT NULL,
    prefix          TEXT DEFAULT '',
    region          TEXT,
    -- Provider-specific configuration
    provider_config JSONB NOT NULL DEFAULT '{}',
    /*
     * S3:    {"endpoint": "...", "storage_class": "STANDARD_IA", "server_side_encryption": "aws:kms", "kms_key_id": "..."}
     * GCS:   {"project": "...", "storage_class": "NEARLINE"}
     * Azure: {"account_name": "...", "container": "...", "access_tier": "Cool"}
     * SFTP:  {"key_path": "/path/to/key", "known_hosts": "..."}
     */
    credentials_ref TEXT NOT NULL, -- vault reference
    is_failover     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Backup policies with flexible scheduling and engine-specific options
CREATE TABLE backup_policies (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    database_id         UUID NOT NULL REFERENCES database_instances(id),
    storage_location_id UUID NOT NULL REFERENCES storage_locations(id),
    failover_storage_id UUID REFERENCES storage_locations(id),
    name                TEXT NOT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    -- Core scheduling (common to all engines)
    full_backup_cron    TEXT NOT NULL,
    incr_backup_cron    TEXT,
    -- Core retention (common to all engines)
    retention_days      INTEGER NOT NULL DEFAULT 30,
    retention_full_count INTEGER,
    -- Encryption (common)
    encryption_enabled  BOOLEAN NOT NULL DEFAULT true,
    encryption_algorithm TEXT NOT NULL DEFAULT 'AES-256-GCM',
    kms_key_ref         TEXT,
    -- Engine-specific and tool-specific configuration
    engine_config       JSONB NOT NULL DEFAULT '{}',
    /*
     * PostgreSQL: {
     *   "wal_archive_enabled": true,
     *   "compression": "zstd",
     *   "compression_level": 3,
     *   "parallel_jobs": 4,
     *   "block_level_incremental": true,
     *   "archive_mode": "async",
     *   "max_wal_senders": 3
     * }
     * MySQL: {
     *   "binlog_streaming": true,
     *   "compression": "lz4",
     *   "single_transaction": true,
     *   "flush_logs": true,
     *   "gtid_mode": true
     * }
     * MongoDB: {
     *   "oplog_archive_enabled": true,
     *   "oplog_span_min": 10,
     *   "compression": "snappy",
     *   "priority_weight": 1.0,
     *   "mongodump_options": {"readPreference": "secondary"}
     * }
     */
    -- AI-driven schedule recommendations
    ai_recommendations  JSONB DEFAULT '{}',
    /*
     * {
     *   "recommended_full_cron": "0 2 * * 0",
     *   "recommended_retention_days": 45,
     *   "estimated_monthly_cost_usd": 127.50,
     *   "confidence": 0.87,
     *   "last_analyzed_at": "2026-05-25T00:00:00Z"
     * }
     */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Backup jobs with engine-specific result metadata
CREATE TABLE backup_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id       UUID NOT NULL REFERENCES backup_policies(id),
    database_id     UUID NOT NULL REFERENCES database_instances(id),
    -- Core fields (common to all engines)
    backup_type     TEXT NOT NULL CHECK (backup_type IN ('full', 'incremental', 'differential')),
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'running', 'completed', 'failed', 'cancelled', 'expired')),
    triggered_by    TEXT NOT NULL DEFAULT 'schedule',
    triggered_by_user_id UUID REFERENCES users(id),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    size_bytes      BIGINT,
    compressed_bytes BIGINT,
    storage_path    TEXT,
    error_message   TEXT,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    parent_backup_id UUID REFERENCES backup_jobs(id),
    -- Engine-specific result metadata in JSONB
    engine_metadata JSONB NOT NULL DEFAULT '{}',
    /*
     * PostgreSQL: {
     *   "wal_start_lsn": "0/16000028",
     *   "wal_end_lsn": "0/18000000",
     *   "timeline_id": 1,
     *   "system_identifier": "7123456789012345678",
     *   "pg_version": "17.2",
     *   "tablespace_map": {...},
     *   "block_level_stats": {"changed_blocks": 45000, "total_blocks": 1200000}
     * }
     * MySQL: {
     *   "binlog_file": "mysql-bin.000042",
     *   "binlog_position": 12345678,
     *   "gtid_executed": "3E11FA47-71CA-11E1-9E33-C80AA9429562:1-42",
     *   "server_id": 100,
     *   "innodb_lsn": "987654321"
     * }
     * MongoDB: {
     *   "oplog_start": {"ts": 1716595200, "t": 1},
     *   "oplog_end": {"ts": 1716681600, "t": 1},
     *   "replica_set": "rs0",
     *   "cluster_time": {"ts": 1716681600, "t": 1},
     *   "collections_backed_up": ["orders", "users", "products"]
     * }
     */
    -- Validation results summary
    validation_summary JSONB DEFAULT '{}',
    /*
     * {
     *   "last_validated_at": "2026-05-25T06:00:00Z",
     *   "checksum_verified": true,
     *   "synthetic_restore_passed": true,
     *   "wal_continuity_verified": true,
     *   "warnings": ["WAL segment 000000010000000000000042 took 45s to archive (threshold: 30s)"]
     * }
     */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_backup_jobs_db_status ON backup_jobs(database_id, status);
CREATE INDEX idx_backup_jobs_completed ON backup_jobs(completed_at DESC);
CREATE INDEX idx_backup_jobs_engine_meta ON backup_jobs USING GIN (engine_metadata);

-- Transaction log segments (WAL/binlog/oplog) -- unified tracking
CREATE TABLE log_segments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    database_id     UUID NOT NULL REFERENCES database_instances(id),
    -- Common fields
    segment_name    TEXT NOT NULL,
    size_bytes      BIGINT NOT NULL,
    compressed_bytes BIGINT,
    storage_path    TEXT NOT NULL,
    checksum        TEXT NOT NULL,
    archived_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Engine-specific segment metadata
    segment_metadata JSONB NOT NULL DEFAULT '{}',
    /*
     * PostgreSQL: {
     *   "type": "wal",
     *   "timeline_id": 1,
     *   "lsn_start": "0/16000000",
     *   "lsn_end": "0/17000000"
     * }
     * MySQL: {
     *   "type": "binlog",
     *   "binlog_file": "mysql-bin.000042",
     *   "position_start": 4,
     *   "position_end": 12345678,
     *   "gtid_range": "3E11FA47-...:1-42"
     * }
     * MongoDB: {
     *   "type": "oplog_slice",
     *   "ts_start": {"ts": 1716595200, "t": 1},
     *   "ts_end": {"ts": 1716598800, "t": 1},
     *   "replica_set": "rs0"
     * }
     */
    UNIQUE(database_id, segment_name)
);

CREATE INDEX idx_log_segments_db_time ON log_segments(database_id, archived_at);
CREATE INDEX idx_log_segments_metadata ON log_segments USING GIN (segment_metadata);

-- Restore jobs with NL query tracking
CREATE TABLE restore_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    database_id     UUID NOT NULL REFERENCES database_instances(id),
    backup_id       UUID REFERENCES backup_jobs(id),
    requested_by    UUID NOT NULL REFERENCES users(id),
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'running', 'completed', 'failed', 'cancelled')),
    -- Core restore parameters
    restore_mode    TEXT NOT NULL DEFAULT 'new_instance'
                    CHECK (restore_mode IN ('new_instance', 'in_place', 'branch')),
    -- PITR target (engine-agnostic wrapper with engine-specific details)
    target_time     TIMESTAMPTZ,
    -- Engine-specific PITR parameters
    pitr_config     JSONB NOT NULL DEFAULT '{}',
    /*
     * PostgreSQL: {"target_lsn": "0/17500000", "target_timeline": 1, "recovery_target_action": "promote"}
     * MySQL:      {"target_gtid": "3E11FA47-...:1-35", "target_binlog": "mysql-bin.000042", "target_position": 9876543}
     * MongoDB:    {"target_oplog_ts": {"ts": 1716597000, "t": 1}, "target_replica_set": "rs0"}
     */
    -- NL restore query support
    nl_query        JSONB DEFAULT '{}',
    /*
     * {
     *   "original_text": "restore before the accidental orders table truncation yesterday",
     *   "resolved_target_time": "2026-05-24T14:32:17Z",
     *   "resolution_method": "ai_nlp",
     *   "confidence": 0.94,
     *   "candidate_events": [
     *     {"time": "2026-05-24T14:32:17Z", "event": "TRUNCATE TABLE orders", "source": "pg_stat_activity"},
     *     {"time": "2026-05-24T14:31:50Z", "event": "BEGIN", "source": "wal_analysis"}
     *   ]
     * }
     */
    target_host     TEXT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    error_message   TEXT,
    restore_details JSONB DEFAULT '{}', -- post-restore validation results
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Anomaly detection with flexible alert payloads
CREATE TABLE anomaly_alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    database_id     UUID NOT NULL REFERENCES database_instances(id),
    severity        TEXT NOT NULL CHECK (severity IN ('info', 'warning', 'critical')),
    status          TEXT NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open', 'acknowledged', 'resolved', 'false_positive')),
    alert_type      TEXT NOT NULL,
    description     TEXT NOT NULL,
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ,
    resolved_by     UUID REFERENCES users(id),
    snapshot_id     UUID REFERENCES backup_jobs(id),
    -- Rich anomaly context in JSONB
    anomaly_context JSONB NOT NULL DEFAULT '{}',
    /*
     * {
     *   "detection_model": "write_volume_anomaly_v2",
     *   "baseline_metrics": {"avg_writes_per_sec": 3000, "stddev": 450},
     *   "current_metrics": {"writes_per_sec": 45000, "duration_seconds": 120},
     *   "z_score": 93.3,
     *   "entropy_analysis": {"score": 0.97, "pattern": "sequential_encryption"},
     *   "related_queries": ["DELETE FROM orders WHERE 1=1", "UPDATE users SET email = 'x'"],
     *   "recommended_action": "isolate_immutable_snapshot"
     * }
     */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_anomaly_alerts_db_status ON anomaly_alerts(database_id, status);
CREATE INDEX idx_anomaly_alerts_severity ON anomaly_alerts(severity, detected_at DESC);

-- Audit log with structured + flexible fields
CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    user_id         UUID REFERENCES users(id),
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID,
    ip_address      INET,
    user_agent      TEXT,
    -- Structured audit context
    context         JSONB NOT NULL DEFAULT '{}',
    /*
     * {
     *   "previous_state": {"retention_days": 30},
     *   "new_state": {"retention_days": 45},
     *   "reason": "Compliance team requested extended retention",
     *   "approval_ticket": "JIRA-1234"
     * }
     */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_logs_org_time ON audit_logs(organization_id, created_at DESC);
CREATE INDEX idx_audit_logs_resource ON audit_logs(resource_type, resource_id);
CREATE INDEX idx_audit_logs_action ON audit_logs(action, created_at DESC);

-- Cost tracking and forecasting
CREATE TABLE cost_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    database_id     UUID REFERENCES database_instances(id),
    storage_location_id UUID REFERENCES storage_locations(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    -- Core cost fields
    storage_bytes   BIGINT NOT NULL DEFAULT 0,
    storage_cost_usd NUMERIC(12,4) NOT NULL DEFAULT 0,
    compute_cost_usd NUMERIC(12,4) NOT NULL DEFAULT 0,
    transfer_cost_usd NUMERIC(12,4) NOT NULL DEFAULT 0,
    -- Breakdown and forecast data
    cost_breakdown  JSONB NOT NULL DEFAULT '{}',
    /*
     * {
     *   "by_backup_type": {"full": 85.00, "incremental": 12.50, "wal": 30.00},
     *   "by_storage_class": {"STANDARD": 50.00, "STANDARD_IA": 77.50},
     *   "forecast_next_30d": 135.00,
     *   "optimization_potential_usd": 22.50,
     *   "recommendations": [
     *     "Move backups older than 14 days to STANDARD_IA to save ~$22.50/month",
     *     "Increase compression level from 3 to 6 for 15% additional savings"
     *   ]
     * }
     */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### JSONB Query Examples

```sql
-- Find all PostgreSQL backups with WAL LSN in a specific range
SELECT b.id, b.status, b.completed_at,
       b.engine_metadata->>'wal_start_lsn' AS wal_start,
       b.engine_metadata->>'wal_end_lsn' AS wal_end
FROM backup_jobs b
JOIN database_instances d ON d.id = b.database_id
WHERE d.engine = 'postgresql'
  AND b.engine_metadata->>'wal_start_lsn' >= '0/16000000'
  AND b.status = 'completed'
ORDER BY b.completed_at DESC;

-- Find MongoDB backups for a specific replica set
SELECT b.id, b.completed_at,
       b.engine_metadata->'oplog_start' AS oplog_start,
       b.engine_metadata->'oplog_end' AS oplog_end
FROM backup_jobs b
JOIN database_instances d ON d.id = b.database_id
WHERE d.engine = 'mongodb'
  AND b.engine_metadata->>'replica_set' = 'rs0';

-- Find databases tagged for a specific team
SELECT * FROM database_instances
WHERE tags @> '["team:payments"]';

-- Get anomalies with high z-scores
SELECT id, database_id, alert_type, description,
       anomaly_context->'current_metrics'->>'writes_per_sec' AS write_rate,
       (anomaly_context->>'z_score')::numeric AS z_score
FROM anomaly_alerts
WHERE (anomaly_context->>'z_score')::numeric > 10
  AND status = 'open'
ORDER BY detected_at DESC;

-- Cost optimization recommendations across an organization
SELECT d.name AS database_name,
       cr.cost_breakdown->'recommendations' AS recommendations,
       (cr.cost_breakdown->>'optimization_potential_usd')::numeric AS savings
FROM cost_records cr
JOIN database_instances d ON d.id = cr.database_id
WHERE cr.organization_id = $1
  AND (cr.cost_breakdown->>'optimization_potential_usd')::numeric > 10
ORDER BY savings DESC;
```

---

## Pros

- **Engine-agnostic core with engine-specific depth**: The relational core handles queries common to all engines (list backups, filter by status, retention management), while JSONB columns store the full richness of engine-specific metadata without schema compromises.
- **Schema evolution without migrations**: Adding new fields to engine metadata, anomaly context, or AI recommendations requires no ALTER TABLE. The JSONB columns absorb new attributes automatically.
- **Single database technology**: Everything runs on PostgreSQL. No need for a separate document store (MongoDB) or search engine. GIN indexes on JSONB columns provide efficient querying.
- **Best of both worlds for querying**: Core fields use standard relational indexes (B-tree on status, timestamps). JSONB fields use GIN indexes for containment queries and expression indexes for frequently accessed paths.
- **Natural fit for multi-engine backup**: The platform's central challenge -- unifying PostgreSQL WAL, MySQL binlog, and MongoDB oplog metadata -- is elegantly handled by typed JSONB columns rather than sparse nullable columns or separate per-engine tables.
- **Practical for compliance**: Audit logs have structured relational fields (who, when, what) for standard queries plus JSONB context for rich detail. This satisfies both automated compliance checks and human auditor review.
- **Familiar to the team**: Any developer comfortable with PostgreSQL can work with this model. JSONB is a well-understood feature, not an exotic technology choice.

## Cons

- **JSONB lacks schema enforcement**: Unlike typed columns, JSONB fields can contain any valid JSON. Application-level validation (or CHECK constraints with jsonb_typeof) is required to prevent malformed data.
- **Partial index limitations**: While GIN indexes support containment queries (`@>`, `?`, `?|`), complex path queries (`engine_metadata->'block_level_stats'->>'changed_blocks' > 1000`) require expression indexes that must be explicitly created.
- **Query complexity**: Accessing nested JSONB fields requires `->>` and `->` operators, which are less readable than column references. ORMs and query builders have varying levels of JSONB support.
- **No foreign key into JSONB**: You cannot create a foreign key constraint on a field inside a JSONB column. If `engine_metadata` references another entity by ID, referential integrity must be enforced at the application level.
- **Performance trade-offs**: JSONB columns are stored as a single binary blob per row. Updating a single field within a large JSONB document rewrites the entire column value. For frequently updated JSONB fields, this can be slower than updating a dedicated column.
- **Reporting complexity**: Business intelligence tools and report builders generally expect flat relational schemas. JSONB fields may need to be flattened into views or CTEs for reporting integration.
- **Discipline required**: Without clear conventions, teams may put too much data into JSONB ("the junk drawer problem") or too little (defeating the purpose). Clear guidelines on what belongs in structured columns vs. JSONB are essential.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Primary database | PostgreSQL 17+ (native JSONB, GIN indexes, partitioning) |
| ORM / query builder | sqlc with custom JSONB types (Go), Prisma with Json fields (TS), or SQLAlchemy with JSONB dialect (Python) |
| Schema validation | JSON Schema validation at the application layer; pg_jsonschema extension for database-level validation |
| Migrations | Atlas or golang-migrate with JSONB-aware diff tooling |
| JSONB indexing | GIN indexes for containment queries; expression indexes for hot paths |
| API layer | REST with OpenAPI 3.1 (JSONB fields map naturally to JSON in API responses) |
| Observability | OpenTelemetry with custom metrics for JSONB query performance |

### Schema Validation Strategy

```sql
-- Optional: Use pg_jsonschema extension for database-level JSONB validation
-- (Available as a PostgreSQL extension)

-- Or use CHECK constraints for critical fields
ALTER TABLE backup_jobs ADD CONSTRAINT chk_pg_metadata
  CHECK (
    (SELECT engine FROM database_instances WHERE id = database_id) != 'postgresql'
    OR (engine_metadata ? 'wal_start_lsn' AND engine_metadata ? 'wal_end_lsn')
  );
```

---

## Migration and Scaling Considerations

### Indexing Strategy for JSONB

```sql
-- GIN index for general containment queries on engine_metadata
CREATE INDEX idx_backup_engine_meta_gin ON backup_jobs USING GIN (engine_metadata);

-- Expression index for a frequently queried JSONB path
CREATE INDEX idx_backup_pg_timeline ON backup_jobs (
    (engine_metadata->>'timeline_id')
) WHERE engine_metadata ? 'timeline_id';

-- Expression index for MySQL GTID lookups
CREATE INDEX idx_backup_mysql_gtid ON backup_jobs (
    (engine_metadata->>'gtid_executed')
) WHERE engine_metadata ? 'gtid_executed';

-- Partial GIN index for anomaly context (only open alerts)
CREATE INDEX idx_anomaly_context_open ON anomaly_alerts
    USING GIN (anomaly_context)
    WHERE status = 'open';
```

### Partitioning

```sql
-- Partition backup_jobs by month
CREATE TABLE backup_jobs (
    -- ... columns as above ...
) PARTITION BY RANGE (created_at);

-- Partition log_segments by month
CREATE TABLE log_segments (
    -- ... columns as above ...
) PARTITION BY RANGE (archived_at);

-- Partition audit_logs by month
CREATE TABLE audit_logs (
    -- ... columns as above ...
) PARTITION BY RANGE (created_at);
```

### Scaling Path

1. **Single PostgreSQL instance** (MVP): Handles the metadata store comfortably. JSONB operations are CPU-bound but well-optimized in PostgreSQL 17.
2. **Read replicas**: For dashboard and reporting queries that do heavy JSONB path extraction.
3. **Materialized views**: Pre-compute flattened views of JSONB data for dashboard and BI tool consumption.
4. **JSONB to columns migration**: If specific JSONB fields are queried in >80% of queries, promote them to dedicated columns. This can be done incrementally without breaking existing queries (add column, backfill, update queries, drop JSONB field).
5. **TimescaleDB hypertables**: For time-series operational metrics (backup duration, WAL lag, storage growth), consider promoting the `cost_records` or a new `metrics` table to a TimescaleDB hypertable while keeping the rest relational+JSONB.

### JSONB Field Promotion Path

When a JSONB field becomes critical enough to warrant a dedicated column:

```sql
-- Step 1: Add the new column
ALTER TABLE backup_jobs ADD COLUMN wal_start_lsn TEXT;

-- Step 2: Backfill from JSONB
UPDATE backup_jobs SET wal_start_lsn = engine_metadata->>'wal_start_lsn'
WHERE engine_metadata ? 'wal_start_lsn';

-- Step 3: Update application code to write both
-- Step 4: Drop from JSONB (optional, or keep for backward compat)
```

---

## Sources

- [PostgreSQL JSONB: Powerful Storage for Semi-Structured Data](https://www.architecture-weekly.com/p/postgresql-jsonb-powerful-storage)
- [Building a Document Store with PostgreSQL JSONB](https://www.cloudthat.com/resources/blog/building-a-document-store-with-postgresql-jsonb)
- [EF Core 10 Turns PostgreSQL into a Hybrid Relational-Document DB](https://trailheadtechnology.com/ef-core-10-turns-postgresql-into-a-hybrid-relational-document-db/)
- [JSONB: PostgreSQL's Secret Weapon for Flexible Data Modeling](https://medium.com/@richardhightower/jsonb-postgresqls-secret-weapon-for-flexible-data-modeling-cf2f5087168f)
- [PostgreSQL JSONB Guide](https://rivestack.io/blog/postgresql-jsonb-guide)
