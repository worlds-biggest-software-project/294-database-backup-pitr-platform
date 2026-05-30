# Data Model Suggestion 2: Event-Sourced / CQRS Approach

> Project: Database Backup & PITR Platform
> Approach: Event sourcing with Command Query Responsibility Segregation (CQRS)

---

## Summary

An event-sourced architecture where every state change in the backup lifecycle is captured as an immutable event in an append-only event store. The system maintains read-optimized projections (materialized views) for dashboard queries, reporting, and API responses using the CQRS pattern. Commands (schedule backup, initiate restore, acknowledge alert) produce events; queries read from projections.

This approach is uniquely fitting for a backup and PITR platform because the domain itself is about preserving and replaying history. The event store provides a complete, immutable audit trail of every backup operation -- precisely what regulated-industry customers (SOC 2, HIPAA, GDPR) require. It also enables "time-travel" over the platform's own metadata: you can reconstruct what the backup catalog looked like at any past point in time.

---

## Key Entities and Relationships

### Event Store Architecture

```
Command Bus                          Event Store (append-only)
  │                                       │
  ├── ScheduleBackup ──────────────→ BackupScheduled
  ├── StartBackup   ──────────────→ BackupStarted
  ├── CompleteBackup ─────────────→ BackupCompleted / BackupFailed
  ├── ArchiveWALSegment ──────────→ WALSegmentArchived
  ├── InitiateRestore ────────────→ RestoreInitiated
  ├── CompleteRestore ────────────→ RestoreCompleted / RestoreFailed
  ├── DetectAnomaly  ─────────────→ AnomalyDetected
  ├── TriggerProtectiveSnapshot ──→ ProtectiveSnapshotTriggered
  ├── UpdatePolicy   ─────────────→ PolicyUpdated
  └── AcknowledgeAlert ───────────→ AlertAcknowledged
                                          │
                                    Projection Engine
                                          │
                          ┌───────────────┼───────────────┐
                          ▼               ▼               ▼
                   BackupCatalog    DashboardView    AuditTrail
                   (read model)    (read model)     (read model)
```

### Event Store Schema

```sql
-- The single source of truth: an append-only event log
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,         -- aggregate root ID (e.g., backup_job_id)
    stream_type     TEXT NOT NULL,          -- 'BackupJob', 'RestoreJob', 'BackupPolicy', 'DatabaseInstance'
    event_type      TEXT NOT NULL,          -- 'BackupStarted', 'BackupCompleted', 'WALSegmentArchived', etc.
    event_version   INTEGER NOT NULL,       -- monotonically increasing per stream
    payload         JSONB NOT NULL,         -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}', -- correlation IDs, causation, user context
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_id, event_version)
);

CREATE INDEX idx_events_stream ON events(stream_id, event_version);
CREATE INDEX idx_events_type ON events(event_type, created_at);
CREATE INDEX idx_events_created ON events(created_at);

-- Snapshots for aggregate rehydration performance
CREATE TABLE snapshots (
    stream_id       UUID PRIMARY KEY,
    stream_type     TEXT NOT NULL,
    event_version   INTEGER NOT NULL,       -- version at which snapshot was taken
    state           JSONB NOT NULL,          -- serialized aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Example Events

```json
// BackupStarted event
{
    "stream_id": "a1b2c3d4-...",
    "stream_type": "BackupJob",
    "event_type": "BackupStarted",
    "event_version": 1,
    "payload": {
        "database_id": "db-001",
        "policy_id": "pol-001",
        "backup_type": "full",
        "triggered_by": "schedule",
        "storage_location": "s3://backups/pg-prod/",
        "encryption_algorithm": "AES-256-GCM",
        "compression": "zstd"
    },
    "metadata": {
        "correlation_id": "corr-xyz",
        "user_id": null,
        "source_ip": "10.0.1.5"
    }
}

// BackupCompleted event
{
    "stream_id": "a1b2c3d4-...",
    "stream_type": "BackupJob",
    "event_type": "BackupCompleted",
    "event_version": 2,
    "payload": {
        "size_bytes": 5368709120,
        "compressed_bytes": 1073741824,
        "wal_start_lsn": "0/16000028",
        "wal_end_lsn": "0/18000000",
        "storage_path": "s3://backups/pg-prod/full/2026-05-25T00:00:00Z",
        "checksum": "sha256:abc123...",
        "duration_seconds": 342
    },
    "metadata": {
        "correlation_id": "corr-xyz"
    }
}

// AnomalyDetected event
{
    "stream_id": "alert-001",
    "stream_type": "AnomalyAlert",
    "event_type": "AnomalyDetected",
    "event_version": 1,
    "payload": {
        "database_id": "db-001",
        "alert_type": "ransomware",
        "severity": "critical",
        "description": "Write volume 15x above 7-day average; encryption pattern detected",
        "detected_metrics": {
            "current_write_rate": 45000,
            "baseline_write_rate": 3000,
            "entropy_score": 0.97
        }
    }
}

// RestoreInitiated event (with NL query)
{
    "stream_id": "restore-001",
    "stream_type": "RestoreJob",
    "event_type": "RestoreInitiated",
    "event_version": 1,
    "payload": {
        "database_id": "db-001",
        "original_query": "restore before the accidental orders table truncation yesterday",
        "resolved_target_time": "2026-05-24T14:32:17Z",
        "resolution_method": "ai_nlp",
        "resolution_confidence": 0.94,
        "base_backup_id": "a1b2c3d4-...",
        "restore_mode": "branch",
        "target_host": "shadow-pg-001"
    }
}
```

### Read Model Projections (CQRS Query Side)

```sql
-- Projection: Current backup catalog (materialized from events)
CREATE TABLE backup_catalog_view (
    backup_id       UUID PRIMARY KEY,
    database_id     UUID NOT NULL,
    database_name   TEXT NOT NULL,
    engine          TEXT NOT NULL,
    backup_type     TEXT NOT NULL,
    status          TEXT NOT NULL, -- derived from latest event
    triggered_by    TEXT NOT NULL,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    size_bytes      BIGINT,
    compressed_bytes BIGINT,
    storage_path    TEXT,
    wal_start_lsn   TEXT,
    wal_end_lsn     TEXT,
    retention_expires_at TIMESTAMPTZ,
    last_validated_at TIMESTAMPTZ,
    validation_result TEXT,
    last_event_version INTEGER NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_catalog_db_status ON backup_catalog_view(database_id, status);
CREATE INDEX idx_catalog_retention ON backup_catalog_view(retention_expires_at);

-- Projection: Dashboard summary (aggregated metrics)
CREATE TABLE dashboard_summary_view (
    organization_id UUID NOT NULL,
    database_id     UUID NOT NULL,
    date            DATE NOT NULL,
    total_backups   INTEGER NOT NULL DEFAULT 0,
    successful_backups INTEGER NOT NULL DEFAULT 0,
    failed_backups  INTEGER NOT NULL DEFAULT 0,
    total_size_bytes BIGINT NOT NULL DEFAULT 0,
    wal_segments_archived INTEGER NOT NULL DEFAULT 0,
    avg_backup_duration_seconds NUMERIC(10,2),
    anomalies_detected INTEGER NOT NULL DEFAULT 0,
    restores_performed INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (organization_id, database_id, date)
);

-- Projection: WAL continuity tracker
CREATE TABLE wal_continuity_view (
    database_id     UUID NOT NULL,
    timeline_id     INTEGER NOT NULL,
    earliest_lsn    TEXT NOT NULL,
    latest_lsn      TEXT NOT NULL,
    earliest_time   TIMESTAMPTZ NOT NULL,
    latest_time     TIMESTAMPTZ NOT NULL,
    segment_count   BIGINT NOT NULL,
    total_size_bytes BIGINT NOT NULL,
    has_gaps         BOOLEAN NOT NULL DEFAULT false,
    gap_details     JSONB, -- array of {start_lsn, end_lsn} gaps
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (database_id, timeline_id)
);

-- Projection: Compliance audit trail (immutable by design)
CREATE TABLE audit_trail_view (
    event_id        UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    user_id         UUID,
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID,
    timestamp       TIMESTAMPTZ NOT NULL,
    ip_address      TEXT,
    details         JSONB NOT NULL
);

CREATE INDEX idx_audit_org_time ON audit_trail_view(organization_id, timestamp DESC);
```

### Projection Rebuild Mechanism

```python
# Pseudocode: Rebuilding a projection from the event store
class BackupCatalogProjection:
    """Processes events to maintain the backup_catalog_view."""

    def handle(self, event):
        match event.event_type:
            case "BackupStarted":
                self.insert_or_update(backup_id=event.stream_id, {
                    "status": "running",
                    "database_id": event.payload["database_id"],
                    "backup_type": event.payload["backup_type"],
                    "triggered_by": event.payload["triggered_by"],
                    "started_at": event.created_at,
                })
            case "BackupCompleted":
                self.update(backup_id=event.stream_id, {
                    "status": "completed",
                    "completed_at": event.created_at,
                    "size_bytes": event.payload["size_bytes"],
                    "compressed_bytes": event.payload["compressed_bytes"],
                    "storage_path": event.payload["storage_path"],
                    "wal_start_lsn": event.payload["wal_start_lsn"],
                    "wal_end_lsn": event.payload["wal_end_lsn"],
                })
            case "BackupFailed":
                self.update(backup_id=event.stream_id, {
                    "status": "failed",
                    "completed_at": event.created_at,
                })
            case "BackupValidated":
                self.update(backup_id=event.stream_id, {
                    "last_validated_at": event.created_at,
                    "validation_result": event.payload["result"],
                })

    def rebuild_from_scratch(self):
        """Drop and recreate the projection from all events."""
        self.truncate_view()
        for event in self.event_store.read_all(stream_type="BackupJob"):
            self.handle(event)
```

---

## Pros

- **Perfect audit trail**: The event store IS the audit trail. Every state change is permanently recorded with full context (who, what, when, why). This directly satisfies SOC 2, HIPAA, and GDPR audit requirements without a separate audit logging mechanism.
- **Temporal queries over platform state**: You can reconstruct the backup catalog as it existed at any past point in time by replaying events up to a timestamp. This is uniquely powerful for a PITR platform -- the platform itself supports "point-in-time" queries over its own metadata.
- **Domain alignment**: The backup lifecycle is inherently event-driven (backup started, WAL archived, restore initiated, anomaly detected). Event sourcing models the domain naturally without impedance mismatch.
- **Projection flexibility**: New dashboard views, reports, or API response shapes can be added by creating new projections that replay the existing event history. No schema migration needed for read-side changes.
- **Resilient read models**: If a projection becomes corrupted or a bug is found, it can be completely rebuilt from the event store without data loss.
- **Natural decoupling**: Commands (write side) and queries (read side) scale independently. Backup agents write events at high throughput; dashboard queries hit read-optimized projections.
- **Compliance evidence export**: The event stream can be exported as-is for compliance audits, providing cryptographically verifiable proof of every backup operation.

## Cons

- **Complexity overhead**: Event sourcing adds significant architectural complexity. The team must maintain the event store, projection engine, event handlers, and snapshot mechanism. This is a larger investment than a simple CRUD application.
- **Eventual consistency**: Read models (projections) lag behind the event store. Dashboard data may be seconds behind reality. For a backup platform where "is my backup running?" needs a real-time answer, this can cause user confusion.
- **Event schema evolution**: As the platform evolves, event schemas change. Old events must still be deserializable. This requires versioning events and maintaining upcasters (transformers from old event formats to new ones).
- **Storage growth**: The event store grows indefinitely (by design -- events are never deleted). For a platform managing thousands of databases with millions of WAL segments, the event store itself becomes a significant storage concern.
- **Debugging difficulty**: Tracing a bug requires replaying events to reproduce state, which can be more challenging than inspecting a row in a relational table.
- **Team skill requirement**: Event sourcing and CQRS are less widely understood than relational CRUD. Hiring and onboarding are harder.
- **Overkill for configuration data**: Backup policies, storage locations, and user accounts are CRUD-like entities that don't benefit from event sourcing. A hybrid approach (event-sourced operations + CRUD for configuration) adds further complexity.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Event store | PostgreSQL with the `events` table (simplest); or EventStoreDB for dedicated event store features |
| Message broker | Apache Kafka or NATS JetStream for event distribution to projection workers |
| Projection engine | Custom projection workers in Go or Rust; or Marten (if .NET) |
| Read model store | PostgreSQL for relational projections; Redis for hot dashboard caches |
| Snapshot store | PostgreSQL (same instance as event store) or S3 for large aggregate snapshots |
| API layer | GraphQL or REST for query-side APIs; gRPC for internal command dispatch |
| Serialization | Protocol Buffers or JSON with schema registry for event versioning |

### Recommended Event Store: PostgreSQL vs EventStoreDB

For the MVP, using PostgreSQL as the event store is recommended. It avoids introducing a new database technology, and PostgreSQL's JSONB, indexing, and partitioning capabilities are more than sufficient for the expected event volume. EventStoreDB can be evaluated later if the event store exceeds 1 billion events or if built-in projections and subscriptions become valuable.

---

## Migration and Scaling Considerations

### Event Store Partitioning

```sql
-- Partition the event store by creation time (monthly)
CREATE TABLE events (
    event_id    UUID NOT NULL DEFAULT gen_random_uuid(),
    stream_id   UUID NOT NULL,
    stream_type TEXT NOT NULL,
    event_type  TEXT NOT NULL,
    event_version INTEGER NOT NULL,
    payload     JSONB NOT NULL,
    metadata    JSONB NOT NULL DEFAULT '{}',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_id, event_version)
) PARTITION BY RANGE (created_at);

-- Monthly partitions
CREATE TABLE events_2026_05 PARTITION OF events
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE events_2026_06 PARTITION OF events
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
-- ... automated partition creation via pg_partman
```

### Scaling Path

1. **Single PostgreSQL instance** for both event store and projections (MVP). Handles up to ~100M events.
2. **Separate read replicas** for projection queries. Backup agents write to the primary; dashboards query replicas.
3. **Kafka/NATS for event distribution** when projection lag becomes unacceptable. Events are written to PostgreSQL AND published to a message broker for real-time projection updates.
4. **Archive old event partitions** to object storage (S3/GCS) for cost control. Events older than the retention window are rarely queried but must be preserved for compliance.
5. **Snapshot frequency tuning**: For aggregates with long event streams (e.g., a database instance with 100K+ backup events), increase snapshot frequency to reduce rehydration time.

### Migration from Event-Sourced to Hybrid

If event sourcing proves too complex for certain subdomains, configuration entities (backup policies, storage locations, users) can be migrated to a traditional CRUD model while keeping the operational event store for backup jobs, restores, WAL tracking, and alerts. This hybrid approach is common in production event-sourced systems.

### Compliance Export

The event store can be exported to immutable storage (S3 with Object Lock, or WORM-compliant storage) for regulatory retention. Each event is self-describing and includes full context, making it ideal for auditor review.

---

## Sources

- [Event Sourcing Pattern - AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/event-sourcing.html)
- [CQRS Pattern - Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [Event Sourcing Pattern - Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
- [Building Robust Systems With Immutable Event Logs](https://dzone.com/articles/event-sourcing-explained-building-robust-systems)
- [It's Time to Rethink Event Sourcing](https://blog.bemi.io/rethinking-event-sourcing/)
- [Event Sourcing Database Architecture - Redpanda](https://www.redpanda.com/guides/event-stream-processing-event-sourcing-database)
