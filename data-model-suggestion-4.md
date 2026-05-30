# Data Model Suggestion 4: Time-Series Core with Relational Catalog (TimescaleDB + PostgreSQL)

> Project: Database Backup & PITR Platform
> Approach: TimescaleDB hypertables for operational metrics and backup telemetry, PostgreSQL relational tables for catalog and configuration

---

## Summary

A dual-model architecture that uses TimescaleDB (a PostgreSQL extension) for time-series operational data and standard PostgreSQL relational tables for the backup catalog, policies, and configuration. This approach recognises that a backup and PITR platform generates two fundamentally different categories of data:

1. **Catalog data** (backup jobs, policies, storage locations, users) -- entity-centric, relationship-heavy, moderately sized, queried by key lookup and filtered joins. Best served by a normalized relational model.

2. **Operational telemetry** (WAL lag measurements, backup throughput over time, storage consumption trends, write-rate anomaly detection metrics, cost tracking) -- time-stamped, append-heavy, queried by time range with aggregations. Best served by a time-series-optimized store.

The key insight is that the AI-powered features (anomaly detection, cost forecasting, backup schedule optimisation) depend heavily on time-series data. Anomaly detection requires comparing current write rates against historical baselines. Cost forecasting requires storage consumption trends over time. Backup schedule optimisation requires analysis of data-change velocity patterns. TimescaleDB hypertables provide the compression, continuous aggregates, and retention policies that make these workloads efficient.

Since TimescaleDB is a PostgreSQL extension (not a separate database), both the relational catalog and the time-series telemetry live in the same PostgreSQL instance, share the same connection, and can be joined in a single query.

---

## Key Entities and Relationships

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     PostgreSQL 17 + TimescaleDB                  │
│                                                                  │
│  ┌──────────────────────────┐  ┌──────────────────────────────┐ │
│  │   Relational Tables       │  │   TimescaleDB Hypertables     │ │
│  │   (standard PostgreSQL)   │  │   (time-series optimized)     │ │
│  │                           │  │                               │ │
│  │  organizations            │  │  backup_metrics               │ │
│  │  users / roles            │  │  wal_lag_measurements         │ │
│  │  database_instances       │  │  write_rate_samples           │ │
│  │  storage_locations        │  │  storage_consumption          │ │
│  │  backup_policies          │  │  restore_performance          │ │
│  │  backup_jobs              │  │  anomaly_scores               │ │
│  │  restore_jobs             │  │  cost_samples                 │ │
│  │  audit_logs               │  │  agent_heartbeats             │ │
│  │  anomaly_alerts           │  │                               │ │
│  └──────────────────────────┘  └──────────────────────────────┘ │
│              │                              │                    │
│              └────── JOIN across both ───────┘                    │
│                    (single connection)                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┴──────────┐
                    │  Grafana / Dashboard │
                    │  (queries both)      │
                    └─────────────────────┘
```

### Relational Catalog Tables

The relational tables follow the same structure as Suggestion 1, but simplified here to focus on the time-series additions. See Suggestion 1 for the full relational schema.

```sql
-- Core relational tables (abbreviated -- see Suggestion 1 for full schema)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE database_instances (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name TEXT NOT NULL,
    engine TEXT NOT NULL CHECK (engine IN ('postgresql', 'mysql', 'mongodb')),
    engine_version TEXT,
    host TEXT NOT NULL,
    port INTEGER NOT NULL,
    environment TEXT NOT NULL DEFAULT 'production',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE backup_policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    database_id UUID NOT NULL REFERENCES database_instances(id),
    name TEXT NOT NULL,
    full_backup_cron TEXT NOT NULL,
    incr_backup_cron TEXT,
    retention_days INTEGER NOT NULL DEFAULT 30,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE backup_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id UUID NOT NULL REFERENCES backup_policies(id),
    database_id UUID NOT NULL REFERENCES database_instances(id),
    backup_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    triggered_by TEXT NOT NULL DEFAULT 'schedule',
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    size_bytes BIGINT,
    compressed_bytes BIGINT,
    storage_path TEXT,
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE restore_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    database_id UUID NOT NULL REFERENCES database_instances(id),
    backup_id UUID REFERENCES backup_jobs(id),
    requested_by UUID NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    target_time TIMESTAMPTZ,
    restore_mode TEXT NOT NULL DEFAULT 'new_instance',
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE anomaly_alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    database_id UUID NOT NULL REFERENCES database_instances(id),
    severity TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'open',
    alert_type TEXT NOT NULL,
    description TEXT NOT NULL,
    detected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    snapshot_id UUID REFERENCES backup_jobs(id)
);
```

### TimescaleDB Hypertables (Time-Series Data)

```sql
-- Enable TimescaleDB extension
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- =================================================================
-- HYPERTABLE 1: Backup job metrics (one row per backup execution)
-- =================================================================
CREATE TABLE backup_metrics (
    time            TIMESTAMPTZ NOT NULL,
    database_id     UUID NOT NULL,
    backup_job_id   UUID NOT NULL,
    backup_type     TEXT NOT NULL, -- 'full', 'incremental', 'differential'
    -- Duration metrics
    duration_seconds    DOUBLE PRECISION,
    data_transfer_mbps  DOUBLE PRECISION,
    -- Size metrics
    raw_size_bytes      BIGINT,
    compressed_size_bytes BIGINT,
    compression_ratio   DOUBLE PRECISION,
    -- WAL/binlog/oplog metrics
    log_segments_count  INTEGER,
    log_bytes_archived  BIGINT,
    -- Validation metrics
    validation_duration_seconds DOUBLE PRECISION,
    checksum_verified   BOOLEAN,
    -- Resource usage
    cpu_percent         DOUBLE PRECISION,
    memory_mb           DOUBLE PRECISION,
    io_read_mbps        DOUBLE PRECISION,
    io_write_mbps       DOUBLE PRECISION
);

SELECT create_hypertable('backup_metrics', 'time');

-- Continuous aggregate: daily backup statistics per database
CREATE MATERIALIZED VIEW backup_stats_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    database_id,
    count(*) AS total_backups,
    count(*) FILTER (WHERE duration_seconds IS NOT NULL) AS successful_backups,
    avg(duration_seconds) AS avg_duration_seconds,
    max(duration_seconds) AS max_duration_seconds,
    sum(raw_size_bytes) AS total_raw_bytes,
    sum(compressed_size_bytes) AS total_compressed_bytes,
    avg(compression_ratio) AS avg_compression_ratio,
    avg(data_transfer_mbps) AS avg_transfer_mbps
FROM backup_metrics
GROUP BY bucket, database_id;

-- Refresh policy: update every hour, covering last 3 hours
SELECT add_continuous_aggregate_policy('backup_stats_daily',
    start_offset => INTERVAL '3 days',
    end_offset   => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);

-- =================================================================
-- HYPERTABLE 2: WAL / transaction log lag measurements
-- =================================================================
CREATE TABLE wal_lag_measurements (
    time            TIMESTAMPTZ NOT NULL,
    database_id     UUID NOT NULL,
    -- Lag metrics (how far behind is archiving from the live database)
    lag_bytes       BIGINT NOT NULL,
    lag_seconds     DOUBLE PRECISION NOT NULL,
    -- Archive throughput
    archive_rate_bytes_per_sec BIGINT,
    -- Pending segments
    pending_segments INTEGER,
    -- Current LSN/position (engine-agnostic as text)
    current_position TEXT, -- WAL LSN, binlog position, or oplog timestamp
    archived_position TEXT
);

SELECT create_hypertable('wal_lag_measurements', 'time');

-- Continuous aggregate: 5-minute WAL lag summary
CREATE MATERIALIZED VIEW wal_lag_5min
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('5 minutes', time) AS bucket,
    database_id,
    avg(lag_seconds) AS avg_lag_seconds,
    max(lag_seconds) AS max_lag_seconds,
    avg(lag_bytes) AS avg_lag_bytes,
    max(lag_bytes) AS max_lag_bytes,
    avg(archive_rate_bytes_per_sec) AS avg_archive_rate
FROM wal_lag_measurements
GROUP BY bucket, database_id;

SELECT add_continuous_aggregate_policy('wal_lag_5min',
    start_offset => INTERVAL '1 day',
    end_offset   => INTERVAL '5 minutes',
    schedule_interval => INTERVAL '5 minutes'
);

-- Retention policy: keep raw measurements for 30 days
SELECT add_retention_policy('wal_lag_measurements', INTERVAL '30 days');

-- =================================================================
-- HYPERTABLE 3: Database write-rate samples (for anomaly detection)
-- =================================================================
CREATE TABLE write_rate_samples (
    time            TIMESTAMPTZ NOT NULL,
    database_id     UUID NOT NULL,
    -- Write volume metrics
    writes_per_second   DOUBLE PRECISION NOT NULL,
    transactions_per_second DOUBLE PRECISION,
    rows_inserted       BIGINT,
    rows_updated        BIGINT,
    rows_deleted        BIGINT,
    -- Data change metrics
    data_change_bytes   BIGINT,
    -- Entropy analysis (for ransomware detection)
    entropy_score       DOUBLE PRECISION, -- 0.0 = low entropy, 1.0 = high entropy (suspicious)
    -- Schema change detection
    ddl_operations      INTEGER DEFAULT 0,
    -- Computed anomaly indicators
    z_score             DOUBLE PRECISION, -- standard deviations from baseline
    is_anomalous        BOOLEAN DEFAULT false
);

SELECT create_hypertable('write_rate_samples', 'time');

-- Continuous aggregate: hourly write-rate baselines for anomaly detection
CREATE MATERIALIZED VIEW write_rate_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    database_id,
    avg(writes_per_second) AS avg_writes,
    stddev(writes_per_second) AS stddev_writes,
    max(writes_per_second) AS max_writes,
    avg(entropy_score) AS avg_entropy,
    max(entropy_score) AS max_entropy,
    sum(rows_deleted) AS total_deletes,
    sum(ddl_operations) AS total_ddl_ops,
    count(*) FILTER (WHERE is_anomalous) AS anomaly_count
FROM write_rate_samples
GROUP BY bucket, database_id;

SELECT add_continuous_aggregate_policy('write_rate_hourly',
    start_offset => INTERVAL '3 days',
    end_offset   => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);

-- Retention: keep raw samples for 7 days, hourly aggregates for 1 year
SELECT add_retention_policy('write_rate_samples', INTERVAL '7 days');

-- =================================================================
-- HYPERTABLE 4: Storage consumption tracking (for cost forecasting)
-- =================================================================
CREATE TABLE storage_consumption (
    time                TIMESTAMPTZ NOT NULL,
    organization_id     UUID NOT NULL,
    database_id         UUID NOT NULL,
    storage_location_id UUID NOT NULL,
    -- Storage metrics
    total_bytes         BIGINT NOT NULL,
    full_backup_bytes   BIGINT NOT NULL DEFAULT 0,
    incr_backup_bytes   BIGINT NOT NULL DEFAULT 0,
    wal_archive_bytes   BIGINT NOT NULL DEFAULT 0,
    -- Object counts
    total_objects       INTEGER NOT NULL DEFAULT 0,
    -- Cost metrics (sampled from cloud provider APIs)
    estimated_cost_usd  NUMERIC(12,4),
    -- Storage class breakdown
    standard_bytes      BIGINT DEFAULT 0,
    infrequent_bytes    BIGINT DEFAULT 0, -- STANDARD_IA / Nearline / Cool
    archive_bytes       BIGINT DEFAULT 0  -- Glacier / Coldline / Archive
);

SELECT create_hypertable('storage_consumption', 'time');

-- Continuous aggregate: daily storage trends
CREATE MATERIALIZED VIEW storage_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    organization_id,
    database_id,
    storage_location_id,
    last(total_bytes, time) AS total_bytes,
    last(estimated_cost_usd, time) AS daily_cost_usd,
    max(total_bytes) - min(total_bytes) AS growth_bytes
FROM storage_consumption
GROUP BY bucket, organization_id, database_id, storage_location_id;

SELECT add_continuous_aggregate_policy('storage_daily',
    start_offset => INTERVAL '7 days',
    end_offset   => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day'
);

-- =================================================================
-- HYPERTABLE 5: Restore performance tracking
-- =================================================================
CREATE TABLE restore_performance (
    time            TIMESTAMPTZ NOT NULL,
    database_id     UUID NOT NULL,
    restore_job_id  UUID NOT NULL,
    -- Performance metrics
    phase           TEXT NOT NULL, -- 'base_restore', 'wal_replay', 'validation', 'promotion'
    duration_seconds DOUBLE PRECISION NOT NULL,
    data_restored_bytes BIGINT,
    wal_segments_replayed INTEGER,
    throughput_mbps  DOUBLE PRECISION,
    -- Target metrics
    target_time     TIMESTAMPTZ,
    actual_restored_to TIMESTAMPTZ,
    time_accuracy_seconds DOUBLE PRECISION -- how close to target
);

SELECT create_hypertable('restore_performance', 'time');

-- =================================================================
-- HYPERTABLE 6: Backup agent heartbeats
-- =================================================================
CREATE TABLE agent_heartbeats (
    time            TIMESTAMPTZ NOT NULL,
    agent_id        TEXT NOT NULL,
    database_id     UUID NOT NULL,
    -- Agent status
    status          TEXT NOT NULL, -- 'healthy', 'degraded', 'unreachable'
    -- Resource usage
    cpu_percent     DOUBLE PRECISION,
    memory_mb       DOUBLE PRECISION,
    disk_free_bytes BIGINT,
    -- Current operation
    current_operation TEXT, -- 'idle', 'backup', 'wal_archive', 'restore'
    operation_progress_pct DOUBLE PRECISION
);

SELECT create_hypertable('agent_heartbeats', 'time');

-- Short retention for heartbeats
SELECT add_retention_policy('agent_heartbeats', INTERVAL '7 days');

-- =================================================================
-- HYPERTABLE 7: Anomaly detection model scores
-- =================================================================
CREATE TABLE anomaly_scores (
    time            TIMESTAMPTZ NOT NULL,
    database_id     UUID NOT NULL,
    model_name      TEXT NOT NULL, -- 'write_volume_v2', 'entropy_v1', 'schema_change_v1'
    -- Scores
    score           DOUBLE PRECISION NOT NULL, -- 0.0 = normal, 1.0 = anomalous
    threshold       DOUBLE PRECISION NOT NULL, -- alert threshold
    is_triggered    BOOLEAN NOT NULL DEFAULT false,
    -- Model context
    input_features  JSONB NOT NULL DEFAULT '{}',
    -- Link to alert if triggered
    alert_id        UUID
);

SELECT create_hypertable('anomaly_scores', 'time');

SELECT add_retention_policy('anomaly_scores', INTERVAL '90 days');
```

### Compression Policies

```sql
-- Enable TimescaleDB compression on older data for massive storage savings

-- Compress backup_metrics older than 7 days
ALTER TABLE backup_metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'database_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('backup_metrics', INTERVAL '7 days');

-- Compress WAL lag measurements older than 3 days
ALTER TABLE wal_lag_measurements SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'database_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('wal_lag_measurements', INTERVAL '3 days');

-- Compress write rate samples older than 2 days
ALTER TABLE write_rate_samples SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'database_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('write_rate_samples', INTERVAL '2 days');

-- Compress storage consumption older than 7 days
ALTER TABLE storage_consumption SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'organization_id, database_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('storage_consumption', INTERVAL '7 days');
```

### Cross-Model Query Examples

```sql
-- Dashboard query: backup health with performance trends
-- Joins relational catalog with time-series metrics
SELECT
    d.name AS database_name,
    d.engine,
    d.environment,
    bj.status AS last_backup_status,
    bj.completed_at AS last_backup_time,
    bs.avg_duration_seconds,
    bs.avg_compression_ratio,
    wl.avg_lag_seconds AS current_wal_lag,
    sc.total_bytes AS storage_used,
    sc.daily_cost_usd
FROM database_instances d
LEFT JOIN LATERAL (
    SELECT status, completed_at FROM backup_jobs
    WHERE database_id = d.id ORDER BY completed_at DESC LIMIT 1
) bj ON true
LEFT JOIN backup_stats_daily bs ON bs.database_id = d.id
    AND bs.bucket = date_trunc('day', now())
LEFT JOIN LATERAL (
    SELECT avg_lag_seconds FROM wal_lag_5min
    WHERE database_id = d.id ORDER BY bucket DESC LIMIT 1
) wl ON true
LEFT JOIN LATERAL (
    SELECT total_bytes, daily_cost_usd FROM storage_daily
    WHERE database_id = d.id ORDER BY bucket DESC LIMIT 1
) sc ON true
WHERE d.organization_id = $1;

-- Anomaly detection: compare current write rate to 7-day baseline
WITH baseline AS (
    SELECT
        database_id,
        avg(avg_writes) AS baseline_avg,
        stddev(avg_writes) AS baseline_stddev
    FROM write_rate_hourly
    WHERE bucket > now() - INTERVAL '7 days'
      AND bucket < now() - INTERVAL '1 hour'
    GROUP BY database_id
),
current_rate AS (
    SELECT
        database_id,
        avg(writes_per_second) AS current_avg
    FROM write_rate_samples
    WHERE time > now() - INTERVAL '5 minutes'
    GROUP BY database_id
)
SELECT
    d.name,
    d.engine,
    cr.current_avg,
    bl.baseline_avg,
    bl.baseline_stddev,
    (cr.current_avg - bl.baseline_avg) / NULLIF(bl.baseline_stddev, 0) AS z_score
FROM current_rate cr
JOIN baseline bl ON bl.database_id = cr.database_id
JOIN database_instances d ON d.id = cr.database_id
WHERE (cr.current_avg - bl.baseline_avg) / NULLIF(bl.baseline_stddev, 0) > 3.0
ORDER BY z_score DESC;

-- Cost forecasting: project storage costs 30 days forward
-- using linear regression on daily consumption
WITH daily_growth AS (
    SELECT
        database_id,
        bucket,
        total_bytes,
        total_bytes - lag(total_bytes) OVER (
            PARTITION BY database_id ORDER BY bucket
        ) AS daily_growth_bytes
    FROM storage_daily
    WHERE bucket > now() - INTERVAL '30 days'
),
growth_stats AS (
    SELECT
        database_id,
        avg(daily_growth_bytes) AS avg_daily_growth,
        last(total_bytes, bucket) AS current_bytes
    FROM daily_growth
    WHERE daily_growth_bytes IS NOT NULL
    GROUP BY database_id
)
SELECT
    d.name,
    gs.current_bytes / (1024^3) AS current_gb,
    (gs.current_bytes + gs.avg_daily_growth * 30) / (1024^3) AS projected_gb_30d,
    gs.avg_daily_growth * 30 / (1024^3) AS growth_30d_gb,
    -- Assuming $0.023/GB/month (S3 Standard)
    (gs.current_bytes + gs.avg_daily_growth * 30) / (1024^3) * 0.023 AS projected_cost_usd
FROM growth_stats gs
JOIN database_instances d ON d.id = gs.database_id
ORDER BY projected_cost_usd DESC;

-- Restore SLA tracking: are we meeting our RTO targets?
SELECT
    d.name,
    d.environment,
    bp.name AS policy_name,
    avg(rp.duration_seconds) AS avg_restore_seconds,
    max(rp.duration_seconds) AS worst_restore_seconds,
    percentile_cont(0.95) WITHIN GROUP (ORDER BY rp.duration_seconds) AS p95_restore_seconds,
    avg(rp.time_accuracy_seconds) AS avg_pitr_accuracy_seconds
FROM restore_performance rp
JOIN database_instances d ON d.id = rp.database_id
JOIN backup_policies bp ON bp.database_id = d.id
WHERE rp.time > now() - INTERVAL '90 days'
  AND rp.phase = 'wal_replay'
GROUP BY d.name, d.environment, bp.name
ORDER BY avg_restore_seconds DESC;
```

---

## Pros

- **Purpose-built for the AI workload**: Anomaly detection, cost forecasting, and schedule optimisation are fundamentally time-series problems. TimescaleDB's continuous aggregates, compression, and retention policies provide exactly the primitives these AI features need without custom application code.
- **Massive storage efficiency**: TimescaleDB compression achieves 90-95% compression on time-series data. Millions of write-rate samples and WAL lag measurements can be stored for months at minimal cost, enabling longer historical baselines for anomaly detection.
- **Automatic data lifecycle**: Retention policies automatically delete old raw samples while continuous aggregates preserve the statistical summaries. No application-level cleanup jobs needed.
- **Single database technology**: TimescaleDB is a PostgreSQL extension. Both the relational catalog and time-series hypertables share the same PostgreSQL instance, connection pool, and transaction guarantees. No operational overhead of a separate time-series database.
- **Native Grafana integration**: TimescaleDB hypertables are directly queryable from Grafana via the standard PostgreSQL data source. Dashboard creation is immediate -- no custom data connectors needed.
- **Continuous aggregates eliminate query latency**: Pre-computed daily backup statistics, hourly write-rate baselines, and 5-minute WAL lag summaries mean dashboard queries hit aggregated data, not raw samples.
- **Standard SQL**: Unlike InfluxQL or PromQL, TimescaleDB queries are standard PostgreSQL SQL with minor extensions (`time_bucket()`, `first()`, `last()`). The team does not need to learn a new query language.

## Cons

- **Extension dependency**: TimescaleDB is a PostgreSQL extension that must be compiled, installed, and version-managed alongside PostgreSQL. Some managed PostgreSQL providers (e.g., Supabase, Neon) do not support TimescaleDB extensions.
- **Hypertable limitations**: Hypertables have restrictions compared to regular PostgreSQL tables: no foreign keys FROM other tables TO hypertables, unique indexes must include the partitioning column, and some DDL operations behave differently.
- **Dual-model cognitive load**: Developers must understand both relational and time-series paradigms. Knowing when to use a relational table vs. a hypertable, when to create a continuous aggregate vs. a materialized view, and how compression affects query patterns requires training.
- **Compression trade-offs**: Compressed chunks are read-only. Late-arriving data or updates to compressed data require decompression, modification, and recompression. For write-rate samples that may be retroactively annotated as anomalous, the compression window must account for this.
- **Backup of TimescaleDB itself**: The irony of a backup platform: TimescaleDB hypertables require specific flags in pg_dump to preserve hypertable metadata. The platform's own backup strategy must account for this.
- **License considerations**: TimescaleDB Community Edition is Apache 2.0, but advanced features (continuous aggregates, compression, retention policies) require the Timescale License (TSL), which is source-available but not OSI-approved open source. This may affect self-hosted deployment licensing.
- **Overkill for small deployments**: If a customer manages 5 databases, the time-series infrastructure is unnecessary overhead. A simple relational model with periodic metric snapshots would suffice.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Primary database | PostgreSQL 17 + TimescaleDB 2.x |
| Relational ORM | sqlc (Go), Prisma (TypeScript), or SQLAlchemy (Python) |
| Time-series queries | Raw SQL with `time_bucket()`, `first()`, `last()` functions |
| Dashboards | Grafana with PostgreSQL data source (native TimescaleDB support) |
| Anomaly detection | Python (scikit-learn, Prophet) reading from continuous aggregates |
| Cost forecasting | SQL-based linear regression on `storage_daily` continuous aggregate |
| Metrics collection | OpenTelemetry Collector writing to hypertables via OTLP-to-SQL adapter |
| Agent telemetry | gRPC streaming from backup agents to a collector service |

---

## Migration and Scaling Considerations

### Deployment Topology

```
MVP (single instance):
┌──────────────────────────────┐
│  PostgreSQL 17 + TimescaleDB  │
│  ┌────────┐  ┌────────────┐  │
│  │Relational│  │Hypertables  │  │
│  │ tables  │  │(compressed) │  │
│  └────────┘  └────────────┘  │
└──────────────────────────────┘

Scale-out (multi-node):
┌──────────────────┐  ┌───────────────────────┐
│  PostgreSQL 17    │  │  Timescale Cloud       │
│  (catalog only)   │  │  (hypertables,         │
│                   │  │   continuous aggregates,│
│                   │  │   compression)          │
└──────────────────┘  └───────────────────────┘
        │                        │
        └───── Application ──────┘
```

### Scaling Path

1. **Single instance** (MVP): PostgreSQL + TimescaleDB handles both relational and time-series workloads. Sufficient for up to ~500 managed databases and ~6 months of raw telemetry.
2. **Compression and retention tuning**: As data grows, tune compression policies (compress after 2-3 days) and retention policies (drop raw samples after 7-30 days, keep continuous aggregates for 1-2 years).
3. **Read replicas**: Offload Grafana dashboards and anomaly detection queries to read replicas. Write-path (agent telemetry ingestion) stays on the primary.
4. **Timescale Cloud**: For very large deployments (1000+ databases, >1M samples/day), migrate hypertables to Timescale Cloud's managed multi-node service while keeping the relational catalog on a standard PostgreSQL instance.
5. **Separate instances**: If the time-series workload dominates, split into a dedicated PostgreSQL instance for the catalog and a dedicated TimescaleDB instance for telemetry. Cross-instance queries are handled at the application layer.

### Data Lifecycle Management

| Data Type | Raw Retention | Aggregate Retention | Compression After |
|-----------|---------------|--------------------|--------------------|
| WAL lag measurements | 30 days | 2 years (5-min buckets) | 3 days |
| Write-rate samples | 7 days | 1 year (hourly buckets) | 2 days |
| Backup metrics | 90 days | 2 years (daily buckets) | 7 days |
| Storage consumption | 90 days | 3 years (daily buckets) | 7 days |
| Agent heartbeats | 7 days | N/A | 1 day |
| Anomaly scores | 90 days | 1 year (hourly buckets) | 7 days |
| Restore performance | 1 year | 3 years | 30 days |

### Migration from Pure Relational

If starting with Suggestion 1 (pure relational) and later adding time-series capabilities:

1. Install TimescaleDB extension (`CREATE EXTENSION timescaledb`)
2. Create hypertables for the new time-series data
3. Backfill historical metrics from application logs or monitoring systems
4. Update backup agents to stream telemetry to hypertables
5. Create continuous aggregates for dashboard queries
6. Configure compression and retention policies

This migration is non-destructive and can be done incrementally.

---

## Sources

- [TimescaleDB Documentation](https://timescaledb.org/)
- [How to Design TimescaleDB Hypertables](https://oneuptime.com/blog/post/2026-01-26-timescaledb-hypertables/view)
- [How to Implement Real-Time Analytics with TimescaleDB](https://oneuptime.com/blog/post/2026-02-02-timescaledb-real-time-analytics/view)
- [How to Backup TimescaleDB Databases](https://oneuptime.com/blog/post/2026-01-27-timescaledb-backup/view)
- [How to Instrument TimescaleDB with OpenTelemetry](https://oneuptime.com/blog/post/2026-02-06-instrument-timescaledb-opentelemetry/view)
- [Schema Design for Time Series Data - Google Cloud Bigtable](https://cloud.google.com/bigtable/docs/schema-design-time-series)
- [Time Series Databases for System Design](https://www.hellointerview.com/learn/system-design/deep-dives/time-series-databases)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
