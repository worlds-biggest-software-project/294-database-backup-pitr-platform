# Data Model Suggestion 1: Normalized Relational (PostgreSQL)

> Project: Database Backup & PITR Platform
> Approach: Traditional normalized relational schema using PostgreSQL

---

## Summary

A fully normalized relational data model using PostgreSQL as the primary metadata store. All backup catalog metadata, scheduling configuration, audit trails, and operational state are modeled as strongly typed tables with foreign key constraints. This approach prioritises data integrity, queryability, and familiarity for the platform engineering audience.

This is the most conventional and widely understood approach. It aligns with how pgBackRest, Barman, and AWS Backup internally track backup state (though those tools use files or proprietary stores rather than a relational database).

---

## Key Entities and Relationships

### Entity-Relationship Overview

```
Organization
  └── Team
       └── DatabaseInstance
            ├── BackupPolicy (retention, schedule, encryption)
            ├── BackupJob
            │    ├── BackupArtifact (full/incremental/WAL segment)
            │    └── BackupValidation
            ├── RestoreJob
            │    └── RestoreTarget
            ├── WALStream
            │    └── WALSegment
            └── AnomalyAlert
                 └── ProtectiveSnapshot

User ──── Role ──── Permission (RBAC)
AuditLog (cross-cutting)
```

### Core Schema

```sql
-- Multi-tenancy and access control
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT UNIQUE NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           TEXT UNIQUE NOT NULL,
    display_name    TEXT NOT NULL,
    password_hash   TEXT,
    auth_provider   TEXT NOT NULL DEFAULT 'local', -- 'local', 'oidc', 'saml'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL, -- 'admin', 'operator', 'viewer'
    permissions     TEXT[] NOT NULL DEFAULT '{}'
);

CREATE TABLE user_roles (
    user_id UUID NOT NULL REFERENCES users(id),
    role_id UUID NOT NULL REFERENCES roles(id),
    PRIMARY KEY (user_id, role_id)
);

-- Database instances (the things being backed up)
CREATE TYPE db_engine AS ENUM ('postgresql', 'mysql', 'mongodb');

CREATE TABLE database_instances (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    engine          db_engine NOT NULL,
    engine_version  TEXT,
    host            TEXT NOT NULL,
    port            INTEGER NOT NULL,
    connection_params TEXT, -- encrypted connection string
    environment     TEXT NOT NULL DEFAULT 'production', -- 'production', 'staging', 'dev'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Storage targets (where backups go)
CREATE TYPE storage_provider AS ENUM ('s3', 'gcs', 'azure_blob', 'sftp', 'minio');

CREATE TABLE storage_locations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    provider        storage_provider NOT NULL,
    bucket          TEXT NOT NULL,
    prefix          TEXT,
    region          TEXT,
    endpoint_url    TEXT, -- for S3-compatible stores
    credentials_ref TEXT NOT NULL, -- reference to secrets manager
    is_failover     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Backup policies (scheduling and retention rules)
CREATE TABLE backup_policies (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    database_id         UUID NOT NULL REFERENCES database_instances(id),
    storage_location_id UUID NOT NULL REFERENCES storage_locations(id),
    failover_storage_id UUID REFERENCES storage_locations(id),
    name                TEXT NOT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    -- Scheduling
    full_backup_cron    TEXT NOT NULL, -- cron expression
    incr_backup_cron    TEXT,          -- cron expression for incremental
    wal_archive_enabled BOOLEAN NOT NULL DEFAULT true,
    -- Retention
    retention_days      INTEGER NOT NULL DEFAULT 30,
    retention_full_count INTEGER,      -- keep N full backups regardless of age
    -- Encryption
    encryption_enabled  BOOLEAN NOT NULL DEFAULT true,
    encryption_algorithm TEXT NOT NULL DEFAULT 'AES-256-GCM',
    kms_key_ref        TEXT,           -- reference to KMS key
    -- Compression
    compression_algo    TEXT NOT NULL DEFAULT 'zstd', -- 'lz4', 'zstd', 'brotli', 'none'
    compression_level   INTEGER NOT NULL DEFAULT 3,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Backup jobs (individual backup executions)
CREATE TYPE backup_type AS ENUM ('full', 'incremental', 'differential');
CREATE TYPE job_status AS ENUM (
    'pending', 'running', 'completed', 'failed', 'cancelled', 'expired'
);

CREATE TABLE backup_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id       UUID NOT NULL REFERENCES backup_policies(id),
    database_id     UUID NOT NULL REFERENCES database_instances(id),
    backup_type     backup_type NOT NULL,
    status          job_status NOT NULL DEFAULT 'pending',
    triggered_by    TEXT NOT NULL DEFAULT 'schedule', -- 'schedule', 'manual', 'anomaly'
    triggered_by_user_id UUID REFERENCES users(id),
    -- Timing
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    -- WAL range covered
    wal_start_lsn   TEXT, -- e.g., '0/16000028' for PostgreSQL
    wal_end_lsn     TEXT,
    -- Size and storage
    size_bytes       BIGINT,
    compressed_bytes BIGINT,
    storage_path     TEXT, -- path within the bucket
    -- Error tracking
    error_message    TEXT,
    retry_count      INTEGER NOT NULL DEFAULT 0,
    parent_backup_id UUID REFERENCES backup_jobs(id), -- for incremental chains
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_backup_jobs_database_status ON backup_jobs(database_id, status);
CREATE INDEX idx_backup_jobs_completed ON backup_jobs(completed_at DESC);

-- WAL segments (transaction log archive tracking)
CREATE TABLE wal_segments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    database_id     UUID NOT NULL REFERENCES database_instances(id),
    segment_name    TEXT NOT NULL, -- e.g., '000000010000000000000001'
    timeline_id     INTEGER NOT NULL DEFAULT 1,
    lsn_start       TEXT NOT NULL,
    lsn_end         TEXT NOT NULL,
    size_bytes       BIGINT NOT NULL,
    compressed_bytes BIGINT,
    storage_path     TEXT NOT NULL,
    archived_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    checksum         TEXT NOT NULL, -- SHA-256
    UNIQUE(database_id, segment_name, timeline_id)
);

CREATE INDEX idx_wal_segments_database_time ON wal_segments(database_id, archived_at);

-- Backup validations (synthetic restore tests)
CREATE TYPE validation_result AS ENUM ('passed', 'failed', 'warning');

CREATE TABLE backup_validations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    backup_id       UUID NOT NULL REFERENCES backup_jobs(id),
    validation_type TEXT NOT NULL, -- 'checksum', 'synthetic_restore', 'wal_continuity'
    result          validation_result NOT NULL,
    started_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    details         TEXT, -- human-readable result description
    shadow_instance TEXT, -- identifier of shadow restore target
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Restore jobs
CREATE TABLE restore_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    database_id     UUID NOT NULL REFERENCES database_instances(id),
    backup_id       UUID REFERENCES backup_jobs(id), -- base backup used
    requested_by    UUID NOT NULL REFERENCES users(id),
    status          job_status NOT NULL DEFAULT 'pending',
    -- PITR target
    target_time     TIMESTAMPTZ, -- point-in-time target
    target_lsn      TEXT,        -- or specific LSN
    target_name     TEXT,        -- named restore point
    -- NL query support
    original_query  TEXT, -- natural-language restore request
    resolved_target TEXT, -- how the NL query was resolved
    -- Execution
    restore_mode    TEXT NOT NULL DEFAULT 'new_instance', -- 'new_instance', 'in_place', 'branch'
    target_host     TEXT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Anomaly detection alerts
CREATE TYPE alert_severity AS ENUM ('info', 'warning', 'critical');
CREATE TYPE alert_status AS ENUM ('open', 'acknowledged', 'resolved', 'false_positive');

CREATE TABLE anomaly_alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    database_id     UUID NOT NULL REFERENCES database_instances(id),
    severity        alert_severity NOT NULL,
    status          alert_status NOT NULL DEFAULT 'open',
    alert_type      TEXT NOT NULL, -- 'ransomware', 'bulk_delete', 'schema_corruption', 'write_spike'
    description     TEXT NOT NULL,
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ,
    resolved_by     UUID REFERENCES users(id),
    -- Protective action taken
    snapshot_id     UUID REFERENCES backup_jobs(id), -- protective snapshot triggered
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Comprehensive audit log
CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    user_id         UUID REFERENCES users(id),
    action          TEXT NOT NULL, -- 'backup.create', 'restore.initiate', 'policy.update', etc.
    resource_type   TEXT NOT NULL, -- 'backup_job', 'restore_job', 'policy', etc.
    resource_id     UUID,
    ip_address      INET,
    user_agent      TEXT,
    details         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_logs_org_time ON audit_logs(organization_id, created_at DESC);
CREATE INDEX idx_audit_logs_resource ON audit_logs(resource_type, resource_id);

-- AI/ML cost forecasting data
CREATE TABLE cost_forecasts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    policy_id       UUID REFERENCES backup_policies(id),
    forecast_date   DATE NOT NULL,
    storage_cost_usd NUMERIC(12,4),
    compute_cost_usd NUMERIC(12,4),
    total_cost_usd   NUMERIC(12,4),
    recommendation   TEXT,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Pros

- **Strong data integrity**: Foreign keys, constraints, and ENUM types enforce consistency across the entire backup catalog. Invalid states (e.g., a restore referencing a nonexistent backup) are impossible at the database level.
- **Powerful querying**: Complex reporting queries (e.g., "show all failed backups in the last 7 days for production databases with retention under 30 days") are natural SQL joins.
- **Mature tooling**: PostgreSQL has decades of operational tooling, monitoring, and expertise. The team already understands the engine since the platform targets PostgreSQL users.
- **ACID compliance**: Backup metadata operations (creating a job, updating status, recording WAL segments) are transactional, preventing partial state.
- **Schema-as-documentation**: The normalized schema itself documents the domain model. New team members can read the DDL to understand the system.
- **Compliance-friendly**: Typed audit logs with relational links to resources satisfy SOC 2, HIPAA, and GDPR audit requirements.
- **Battle-tested pattern**: Nearly all production backup management systems (AWS Backup, Percona Everest) use relational metadata stores internally.

## Cons

- **Schema rigidity**: Adding new database engines, backup types, or metadata fields requires migrations. Every new feature that changes the data shape needs an ALTER TABLE.
- **Performance at scale**: With millions of WAL segments and backup jobs across thousands of databases, join-heavy queries may slow down without careful indexing and partitioning.
- **Operational metrics stored poorly**: Time-series data (backup duration trends, WAL lag over time, storage growth) fits awkwardly in a relational model. Separate tables or a dedicated time-series store may be needed.
- **Multi-engine metadata divergence**: PostgreSQL WAL, MySQL binlog, and MongoDB oplog have different metadata shapes. Forcing them into the same normalized columns either loses detail or requires many nullable columns.
- **No built-in event history**: The current state overwrites previous state (e.g., `status` changes from `running` to `completed`). State transition history requires explicit audit logging or a separate events table.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Primary database | PostgreSQL 17+ |
| Connection pooling | PgBouncer or Supavisor |
| Migrations | golang-migrate, Flyway, or Atlas |
| ORM / query builder | sqlc (Go), Prisma (TypeScript), or SQLAlchemy (Python) |
| Secrets management | HashiCorp Vault or AWS Secrets Manager for `credentials_ref` |
| Encryption | AES-256-GCM via pgcrypto or application-level encryption |
| Observability | OpenTelemetry SDK emitting to Prometheus + Grafana |

---

## Migration and Scaling Considerations

### Partitioning Strategy

- **`backup_jobs`**: Partition by `created_at` (monthly range partitions). Old partitions can be archived or moved to cheaper storage.
- **`wal_segments`**: Partition by `database_id` and `archived_at`. This is the highest-volume table and benefits most from partitioning.
- **`audit_logs`**: Partition by `created_at` (monthly). Critical for compliance queries that span date ranges.

### Scaling Path

1. **Vertical scaling** (single PostgreSQL instance) handles up to ~100 managed databases and ~10M WAL segments comfortably.
2. **Read replicas** for dashboard queries and reporting, keeping the primary available for write-path operations (backup job creation, WAL segment registration).
3. **Connection pooling** via PgBouncer to handle bursty agent connections from backup workers.
4. **Table partitioning** (declarative range partitioning in PostgreSQL 17) for high-volume tables.
5. **Citus or PostgreSQL 17 built-in sharding** for very large deployments (1000+ databases) where single-node partitioning is insufficient.

### Migration Path from MVP to Scale

- Start with a single PostgreSQL instance for the metadata store.
- Add read replicas when dashboard query load exceeds write capacity.
- Introduce partitioning for `wal_segments` and `audit_logs` when row counts exceed 50M.
- Consider extracting time-series metrics (backup duration, WAL lag, storage growth) to a dedicated TimescaleDB hypertable or Prometheus/VictoriaMetrics if operational dashboards need sub-second refresh.

---

## Sources

- [PostgreSQL WAL Archiving and PITR Documentation](https://www.postgresql.org/docs/current/continuous-archiving.html)
- [Velero CRD Architecture](https://velero.io/docs/v1.8/how-velero-works/)
- [pgBackRest Configuration Reference](https://pgbackrest.org/configuration.html)
- [WAL-G Documentation](https://wal-g.readthedocs.io/PostgreSQL/)
