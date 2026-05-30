# Database Backup & PITR Platform — Phased Development Plan

> Project: 294-database-backup-pitr-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files. The data model adopts **Suggestion 4 (TimescaleDB time-series core + relational catalog)** as the backbone, fused with the **JSONB engine-metadata envelope from Suggestion 3** to absorb PostgreSQL WAL / MySQL binlog / MongoDB oplog divergence in a single schema. The rationale is that the platform's differentiating AI features (anomaly detection, cost forecasting, schedule optimisation) are inherently time-series problems, while the catalog (instances, policies, jobs) is relationship-heavy and benefits from relational integrity.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Engine / agent language | **Go 1.23+** | Every battle-tested tool in this space (WAL-G, Velero, pgBackRest interop, Percona operators) is Go or C. Go gives static single-binary agents that drop cleanly into containers/Kubernetes, native goroutine concurrency for parallel WAL streaming and S3 multipart upload, and first-class AWS/GCS/Azure SDKs. Matches the platform-engineering audience's expectations. |
| Control-plane API | **Go + `chi` router + `oapi-codegen`** | The API server shares the Go domain model with agents/operator. `oapi-codegen` generates handlers and clients from a hand-written **OpenAPI 3.1** spec (standards.md), giving an SDK-generatable, gateway-friendly contract. `chi` is stdlib-compatible and lightweight. |
| AI/ML sidecar | **Python 3.12 + FastAPI** | Anomaly detection, NL-PITR resolution, and cost forecasting need the Python ML ecosystem (scikit-learn, Prophet/statsmodels, an LLM SDK). Runs as a separate `ai-service` exposing a small internal REST API the Go control plane calls. Keeps the hot backup path free of a Python runtime. |
| Metadata + telemetry store | **PostgreSQL 17 + TimescaleDB 2.x** | Relational catalog (Suggestion 1/3) for entities and JSONB envelopes for engine-specific metadata; TimescaleDB hypertables (Suggestion 4) for write-rate samples, WAL-lag, backup/restore metrics, storage consumption. Single engine, single connection, joinable in one query. Continuous aggregates pre-compute the baselines the AI features consume. |
| Object storage SDKs | **AWS SDK for Go v2**, GCS, Azure Blob, `aws-sdk` S3-compatible (MinIO/Ceph) | S3 API is the de-facto backup target (standards.md). All four providers exposed behind one `BlobStore` interface. |
| Task queue / scheduling | **River (Postgres-backed job queue) + robfig/cron** | River persists jobs in the same Postgres instance (no extra infra for MVP), supports retries, uniqueness, and priorities — exactly what backup/restore/validation jobs need. `robfig/cron` parses policy cron expressions to enqueue scheduled jobs. |
| Frontend | **React 18 + TypeScript + Vite + TanStack Query + shadcn/ui + Tailwind** | Dashboard is a core MVP differentiator vs CLI-only incumbents. SPA consumes the OpenAPI-generated TS client. Recharts/visx for the PITR timeline and backup-health charts. |
| AuthN / AuthZ | **OIDC (Authorization Code + PKCE) via `coreos/go-oidc`; local password fallback; RBAC** | OAuth 2.0 / OIDC is the standard for enterprise SSO (standards.md). RBAC roles (`admin`/`operator`/`viewer`) gate every API action. JWT access tokens, refresh via OIDC. |
| Encryption | **AES-256-GCM (FIPS 197); TLS 1.3 (RFC 8446) in transit; envelope encryption via KMS** | Mandated by SOC 2 / HIPAA / GDPR (standards.md). Per-backup data key wrapped by a KMS-managed key (AWS KMS / GCP KMS / Vault Transit). |
| Secrets | **HashiCorp Vault or cloud Secrets Manager; never in DB** | DB stores only `credentials_ref` / `kms_key_ref` pointers. |
| Observability | **OpenTelemetry SDK → OTLP; Prometheus + Grafana** | OTel is a stated differentiator (standards.md: no incumbent emits OTel natively). Metrics: backup duration, WAL lag, restore latency, storage by tier. |
| Kubernetes integration | **`controller-runtime` operator; CRDs `Backup`, `Restore`, `Schedule`, `StorageLocation`** | Aligns with the Velero CRD model (standards.md) for cloud-native adoption (v1.1). |
| Containerisation | **Docker + docker-compose (dev/self-host); Helm chart (k8s)** | Self-hosted, k8s-native, and multi-cloud deployment targets (README). |
| Testing | **Go: `testing` + `testify` + `testcontainers-go`; Python: `pytest`; FE: `vitest` + Playwright** | `testcontainers-go` spins real Postgres/MySQL/MongoDB/MinIO for integration tests. |
| Lint / format / types | **`golangci-lint` + `gofmt`; `ruff` + `mypy`; `eslint` + `prettier` + `tsc`** | Standard per ecosystem; enforced in CI gates. |
| Migrations | **`golang-migrate`** (TimescaleDB-aware ordering) | Versioned SQL migrations; hypertable creation runs after extension enablement. |
| CI/CD | **GitHub Actions** | Lint → test → build images → publish. |

### Project Structure

```
database-backup-pitr-platform/
├── go.mod
├── go.sum
├── Makefile
├── docker-compose.yml                # postgres+timescale, minio, control-plane, ai-service, web
├── Dockerfile.controlplane
├── Dockerfile.agent
├── Dockerfile.operator
├── api/
│   └── openapi.yaml                   # OpenAPI 3.1 source of truth
├── cmd/
│   ├── controlplane/main.go           # API server + scheduler + job workers
│   ├── agent/main.go                  # backup agent (runs near the DB)
│   ├── operator/main.go               # Kubernetes operator (Phase 9)
│   └── dbpctl/main.go                 # CLI client
├── internal/
│   ├── domain/                        # core types: BackupJob, RestoreJob, Policy, etc.
│   ├── catalog/                       # repository layer over Postgres (sqlc-generated + hand-written)
│   ├── telemetry/                     # hypertable writers + queries
│   ├── api/                           # generated handlers (oapi-codegen) + middleware
│   ├── auth/                          # OIDC, JWT, RBAC enforcement
│   ├── scheduler/                     # cron → job enqueue
│   ├── jobs/                          # River worker definitions (backup, restore, validate, prune)
│   ├── engines/                       # per-engine drivers behind a common interface
│   │   ├── engine.go                  # BackupEngine interface
│   │   ├── postgres/                  # pg_basebackup + WAL archiving + PITR
│   │   ├── mysql/                     # xtrabackup/mysqldump + binlog
│   │   └── mongodb/                   # mongodump + oplog slices
│   ├── storage/                       # BlobStore interface + s3/gcs/azure/sftp impls
│   ├── crypto/                        # AES-256-GCM envelope encryption + KMS
│   ├── retention/                     # retention + pruning logic
│   ├── validation/                    # synthetic-restore + checksum + WAL-continuity
│   ├── aiclient/                      # client to the Python ai-service
│   ├── audit/                         # audit-log writer
│   └── observability/                 # OTel setup
├── ai-service/                        # Python FastAPI sidecar
│   ├── pyproject.toml
│   ├── app/
│   │   ├── main.py
│   │   ├── anomaly.py                 # write-rate + entropy anomaly models
│   │   ├── nlpitr.py                  # NL → timestamp resolution
│   │   ├── forecast.py                # cost forecasting / retention recommendation
│   │   └── schedule.py                # backup schedule optimisation
│   └── tests/
├── web/                               # React + Vite dashboard
│   ├── package.json
│   └── src/
│       ├── api/                       # generated TS client
│       ├── pages/                     # Dashboard, Databases, Backups, Restore, Alerts, Audit
│       └── components/
├── deploy/
│   ├── helm/                          # Helm chart
│   └── crds/                          # Backup/Restore/Schedule/StorageLocation CRDs
├── migrations/                        # golang-migrate SQL files
│   ├── 0001_extensions.up.sql
│   ├── 0002_catalog.up.sql
│   ├── 0003_hypertables.up.sql
│   └── ...
└── test/
    ├── fixtures/                      # sample WAL, binlog, oplog, NL-query corpora
    └── e2e/
```

The structure is additive: each phase fills in `internal/` packages and adds migrations without restructuring earlier code.

---

## Phase 1: Foundation & Metadata Store

### Purpose
Establish the repository, build tooling, the PostgreSQL+TimescaleDB schema, the domain model, and the catalog repository layer. After this phase the platform can persist and query database instances, storage locations, and policies — the configuration substrate everything else depends on. No backups happen yet.

### Tasks

#### 1.1 — Repository scaffolding and tooling

**What**: Initialise the Go module, Python project, web app, Docker Compose stack, and CI.

**Design**:
- `go.mod` module `github.com/wbsp/dbpitr`. `Makefile` targets: `make lint test build up migrate openapi-gen client-gen`.
- `docker-compose.yml` services: `db` (`timescale/timescaledb:2.x-pg17`), `minio`, `controlplane`, `ai-service`, `web`. Healthchecks on `db` and `minio`.
- `.github/workflows/ci.yml`: matrix job running `golangci-lint`, `go test ./...`, `ruff`+`mypy`+`pytest` in `ai-service/`, `tsc`+`eslint`+`vitest` in `web/`, then build all images.
- Config loaded via `envconfig` into a typed `Config` struct:
  ```go
  type Config struct {
      DatabaseURL   string `envconfig:"DATABASE_URL" required:"true"`
      ListenAddr    string `envconfig:"LISTEN_ADDR" default:":8080"`
      AIServiceURL  string `envconfig:"AI_SERVICE_URL" default:"http://ai-service:8000"`
      OIDCIssuer    string `envconfig:"OIDC_ISSUER"`
      JWTSecret     string `envconfig:"JWT_SECRET"`
      LogLevel      string `envconfig:"LOG_LEVEL" default:"info"`
  }
  ```

**Testing**:
- `Unit: Load() with all required env set → Config with defaults applied`.
- `Unit: Load() missing DATABASE_URL → error naming the field`.
- `Integration: docker-compose up → db and minio healthchecks pass within 60s` (CI smoke).

#### 1.2 — Database schema & migrations

**What**: Author `golang-migrate` SQL for extensions, relational catalog, and TimescaleDB hypertables.

**Design**:
- `0001_extensions.up.sql`: `CREATE EXTENSION IF NOT EXISTS timescaledb; CREATE EXTENSION IF NOT EXISTS pgcrypto;`
- `0002_catalog.up.sql`: relational tables from Suggestion 3 (JSONB-enveloped), at minimum:
  ```sql
  CREATE TABLE organizations (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), name TEXT NOT NULL, slug TEXT UNIQUE NOT NULL, settings JSONB NOT NULL DEFAULT '{}', created_at TIMESTAMPTZ NOT NULL DEFAULT now());
  CREATE TABLE users (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), organization_id UUID NOT NULL REFERENCES organizations(id), email TEXT UNIQUE NOT NULL, display_name TEXT NOT NULL, password_hash TEXT, auth_provider TEXT NOT NULL DEFAULT 'local', auth_config JSONB NOT NULL DEFAULT '{}', created_at TIMESTAMPTZ NOT NULL DEFAULT now());
  CREATE TABLE roles (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), organization_id UUID NOT NULL REFERENCES organizations(id), name TEXT NOT NULL, permissions JSONB NOT NULL DEFAULT '[]', UNIQUE(organization_id,name));
  CREATE TABLE user_roles (user_id UUID REFERENCES users(id) ON DELETE CASCADE, role_id UUID REFERENCES roles(id) ON DELETE CASCADE, PRIMARY KEY(user_id,role_id));
  CREATE TABLE database_instances (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), organization_id UUID NOT NULL REFERENCES organizations(id), name TEXT NOT NULL, engine TEXT NOT NULL CHECK (engine IN ('postgresql','mysql','mongodb')), engine_version TEXT, environment TEXT NOT NULL DEFAULT 'production', host TEXT NOT NULL, port INTEGER NOT NULL, connection_config JSONB NOT NULL DEFAULT '{}', credentials_ref TEXT, health_status TEXT DEFAULT 'unknown', tags JSONB NOT NULL DEFAULT '[]', created_at TIMESTAMPTZ NOT NULL DEFAULT now(), updated_at TIMESTAMPTZ NOT NULL DEFAULT now());
  CREATE TABLE storage_locations (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), organization_id UUID NOT NULL REFERENCES organizations(id), name TEXT NOT NULL, provider TEXT NOT NULL CHECK (provider IN ('s3','gcs','azure_blob','sftp','minio')), bucket TEXT NOT NULL, prefix TEXT DEFAULT '', region TEXT, provider_config JSONB NOT NULL DEFAULT '{}', credentials_ref TEXT NOT NULL, is_failover BOOLEAN NOT NULL DEFAULT false, created_at TIMESTAMPTZ NOT NULL DEFAULT now());
  CREATE TABLE backup_policies (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), database_id UUID NOT NULL REFERENCES database_instances(id), storage_location_id UUID NOT NULL REFERENCES storage_locations(id), failover_storage_id UUID REFERENCES storage_locations(id), name TEXT NOT NULL, is_active BOOLEAN NOT NULL DEFAULT true, full_backup_cron TEXT NOT NULL, incr_backup_cron TEXT, retention_days INTEGER NOT NULL DEFAULT 30 CHECK (retention_days >= 7), retention_full_count INTEGER, encryption_enabled BOOLEAN NOT NULL DEFAULT true, encryption_algorithm TEXT NOT NULL DEFAULT 'AES-256-GCM', kms_key_ref TEXT, engine_config JSONB NOT NULL DEFAULT '{}', ai_recommendations JSONB DEFAULT '{}', created_at TIMESTAMPTZ NOT NULL DEFAULT now(), updated_at TIMESTAMPTZ NOT NULL DEFAULT now());
  CREATE TABLE backup_jobs (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), policy_id UUID REFERENCES backup_policies(id), database_id UUID NOT NULL REFERENCES database_instances(id), backup_type TEXT NOT NULL CHECK (backup_type IN ('full','incremental','differential')), status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending','running','completed','failed','cancelled','expired')), triggered_by TEXT NOT NULL DEFAULT 'schedule', triggered_by_user_id UUID REFERENCES users(id), started_at TIMESTAMPTZ, completed_at TIMESTAMPTZ, size_bytes BIGINT, compressed_bytes BIGINT, storage_path TEXT, error_message TEXT, retry_count INTEGER NOT NULL DEFAULT 0, parent_backup_id UUID REFERENCES backup_jobs(id), engine_metadata JSONB NOT NULL DEFAULT '{}', validation_summary JSONB DEFAULT '{}', retention_expires_at TIMESTAMPTZ, created_at TIMESTAMPTZ NOT NULL DEFAULT now());
  CREATE TABLE log_segments (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), database_id UUID NOT NULL REFERENCES database_instances(id), segment_name TEXT NOT NULL, size_bytes BIGINT NOT NULL, compressed_bytes BIGINT, storage_path TEXT NOT NULL, checksum TEXT NOT NULL, archived_at TIMESTAMPTZ NOT NULL DEFAULT now(), segment_metadata JSONB NOT NULL DEFAULT '{}', UNIQUE(database_id, segment_name));
  CREATE TABLE restore_jobs (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), database_id UUID NOT NULL REFERENCES database_instances(id), backup_id UUID REFERENCES backup_jobs(id), requested_by UUID NOT NULL REFERENCES users(id), status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending','running','completed','failed','cancelled')), restore_mode TEXT NOT NULL DEFAULT 'new_instance' CHECK (restore_mode IN ('new_instance','in_place','branch')), target_time TIMESTAMPTZ, pitr_config JSONB NOT NULL DEFAULT '{}', nl_query JSONB DEFAULT '{}', target_host TEXT, started_at TIMESTAMPTZ, completed_at TIMESTAMPTZ, error_message TEXT, restore_details JSONB DEFAULT '{}', created_at TIMESTAMPTZ NOT NULL DEFAULT now());
  CREATE TABLE anomaly_alerts (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), database_id UUID NOT NULL REFERENCES database_instances(id), severity TEXT NOT NULL CHECK (severity IN ('info','warning','critical')), status TEXT NOT NULL DEFAULT 'open' CHECK (status IN ('open','acknowledged','resolved','false_positive')), alert_type TEXT NOT NULL, description TEXT NOT NULL, detected_at TIMESTAMPTZ NOT NULL DEFAULT now(), resolved_at TIMESTAMPTZ, resolved_by UUID REFERENCES users(id), snapshot_id UUID REFERENCES backup_jobs(id), anomaly_context JSONB NOT NULL DEFAULT '{}', created_at TIMESTAMPTZ NOT NULL DEFAULT now());
  CREATE TABLE audit_logs (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), organization_id UUID NOT NULL REFERENCES organizations(id), user_id UUID REFERENCES users(id), action TEXT NOT NULL, resource_type TEXT NOT NULL, resource_id UUID, ip_address INET, user_agent TEXT, context JSONB NOT NULL DEFAULT '{}', created_at TIMESTAMPTZ NOT NULL DEFAULT now());
  ```
  Indexes: `idx_backup_jobs_db_status`, `idx_backup_jobs_completed`, `idx_log_segments_db_time`, GIN on `engine_metadata`/`segment_metadata`/`tags`, `idx_audit_logs_org_time`.
- `0003_hypertables.up.sql`: hypertables from Suggestion 4 — `backup_metrics`, `wal_lag_measurements`, `write_rate_samples`, `storage_consumption`, `restore_performance`, `agent_heartbeats`, `anomaly_scores` — each `SELECT create_hypertable(...)`, plus the continuous aggregates and compression/retention policies (`write_rate_hourly`, `wal_lag_5min`, `backup_stats_daily`, `storage_daily`).

**Testing**:
- `Integration (testcontainers): migrate up then down → no error, all tables/hypertables created and dropped`.
- `Integration: insert backup_policy with retention_days=3 → CHECK violation`.
- `Integration: SELECT * FROM timescaledb_information.hypertables → 7 rows`.
- `Integration: insert backup_job with engine_metadata JSONB → round-trips intact`.

#### 1.3 — Domain model & catalog repository

**What**: Go structs for every entity and a `Catalog` repository with CRUD over the relational tables.

**Design**:
```go
type Engine string // "postgresql" | "mysql" | "mongodb"

type DatabaseInstance struct {
    ID, OrganizationID uuid.UUID
    Name string; Engine Engine; EngineVersion string
    Environment, Host string; Port int
    ConnectionConfig map[string]any // JSONB
    CredentialsRef string
    Tags []string
    CreatedAt, UpdatedAt time.Time
}

type BackupPolicy struct {
    ID, DatabaseID, StorageLocationID uuid.UUID
    FailoverStorageID *uuid.UUID
    Name string; IsActive bool
    FullBackupCron string; IncrBackupCron *string
    RetentionDays int; RetentionFullCount *int
    EncryptionEnabled bool; EncryptionAlgorithm string; KMSKeyRef *string
    EngineConfig map[string]any
    AIRecommendations map[string]any
}

type Catalog interface {
    CreateDatabaseInstance(ctx, *DatabaseInstance) error
    GetDatabaseInstance(ctx, uuid.UUID) (*DatabaseInstance, error)
    ListDatabaseInstances(ctx, orgID uuid.UUID, f InstanceFilter) ([]DatabaseInstance, error)
    UpdateDatabaseInstance(ctx, *DatabaseInstance) error
    DeleteDatabaseInstance(ctx, uuid.UUID) error
    // ...analogous for StorageLocation, BackupPolicy, BackupJob, RestoreJob, AnomalyAlert
    WithTx(ctx, func(Catalog) error) error
}
```
Implemented with `pgx` connection pool. JSONB columns marshalled via `encoding/json`. `WithTx` wraps a `pgx.Tx` for transactional job-status transitions.

**Testing**:
- `Integration: CreateDatabaseInstance then Get → equal`.
- `Integration: ListDatabaseInstances filtered by engine='postgresql' → only pg instances`.
- `Integration: WithTx that returns error → rolled back, no row persisted`.
- `Unit: ConnectionConfig with nested map → marshals/unmarshals losslessly`.

---

## Phase 2: Storage, Crypto & Engine Abstraction

### Purpose
Build the two lowest-level capabilities every backup depends on: a uniform object-storage interface across S3/GCS/Azure/SFTP with failover, AES-256-GCM envelope encryption, and the `BackupEngine` interface that later per-engine drivers implement. After this phase the platform can write encrypted, compressed blobs to any supported target and has a contract for engine drivers — but no real database is backed up yet.

### Tasks

#### 2.1 — BlobStore interface and providers

**What**: A storage abstraction with S3, MinIO (S3-compatible), GCS, Azure Blob, and SFTP implementations plus failover routing.

**Design**:
```go
type BlobStore interface {
    Put(ctx context.Context, key string, r io.Reader, opts PutOpts) (PutResult, error) // streaming, multipart
    Get(ctx context.Context, key string) (io.ReadCloser, error)
    List(ctx context.Context, prefix string) ([]ObjectInfo, error)
    Delete(ctx context.Context, key string) error
    Stat(ctx context.Context, key string) (ObjectInfo, error)
}
type PutOpts struct { StorageClass string; ContentLength int64; ChecksumSHA256 string }
type PutResult struct { ETag, VersionID string; Bytes int64 }
```
- `FailoverStore` wraps primary + failover `BlobStore`; on retryable error (timeout, 5xx) from primary it transparently writes to the failover and records the fallback in telemetry (WAL-G parity, a stated differentiator).
- S3 impl uses AWS SDK v2 multipart upload with concurrency = `min(4, NumCPU)`; honours `endpoint_url` for MinIO/Ceph.
- Credentials resolved from `credentials_ref` via the secrets provider, never from the DB.

**Testing**:
- `Integration (MinIO testcontainer): Put 100MB stream then Get → bytes and SHA-256 match`.
- `Integration: List(prefix) → returns only matching keys`.
- `Unit (mocked): FailoverStore primary returns 503 → write lands on failover, fallback metric emitted`.
- `Unit (mocked): FailoverStore primary returns 403 (non-retryable) → error propagated, no failover attempt`.

#### 2.2 — Envelope encryption & compression

**What**: AES-256-GCM encryption of backup streams with KMS-wrapped data keys, and pluggable compression.

**Design**:
```go
type Cryptor interface {
    EncryptStream(ctx, plaintext io.Reader, keyRef string) (ciphertext io.Reader, header EnvelopeHeader, err error)
    DecryptStream(ctx, ciphertext io.Reader, header EnvelopeHeader) (io.Reader, error)
}
type EnvelopeHeader struct {
    Algorithm string  // "AES-256-GCM"
    WrappedDataKey []byte // data key encrypted by KMS key
    KMSKeyRef string
    Nonce []byte
    ChunkSize int // 64 KiB chunks, each with its own GCM tag
}
```
- Per-backup random 256-bit data key, wrapped by the KMS key referenced in the policy (`AWS KMS`, `GCP KMS`, or `Vault Transit` behind a `KMS` interface). FIPS 197 / SOC 2 alignment.
- Streaming GCM in 64 KiB chunks so arbitrarily large backups never buffer fully in memory.
- Compression interface with `zstd` (default, level 3), `lz4`, `none`; compress-then-encrypt order. Pipeline: `db stream → compress → encrypt → BlobStore.Put`.

**Testing**:
- `Unit: encrypt 10MB random → decrypt → identical bytes`.
- `Unit: tamper one ciphertext byte → decrypt returns auth-tag failure`.
- `Unit: zstd round-trip preserves bytes; compression_ratio recorded`.
- `Unit (mocked KMS): wrap/unwrap data key → original key recovered`.

#### 2.3 — BackupEngine interface

**What**: Define the contract every database driver implements; supply a `noop` test engine.

**Design**:
```go
type BackupEngine interface {
    Engine() Engine
    TestConnection(ctx, *DatabaseInstance) error
    FullBackup(ctx, BackupRequest) (BackupStream, EngineMetadata, error)
    IncrementalBackup(ctx, BackupRequest, parent EngineMetadata) (BackupStream, EngineMetadata, error)
    StartLogArchiving(ctx, *DatabaseInstance, segCh chan<- LogSegment) error // WAL/binlog/oplog
    Restore(ctx, RestoreRequest) error
    ResolveTarget(ctx, *DatabaseInstance, PITRTarget) (ResolvedTarget, error) // time/LSN/name → engine coords
}
type BackupStream struct { Reader io.Reader; ExpectedBytes int64 }
type EngineMetadata map[string]any // stored in backup_jobs.engine_metadata
type PITRTarget struct { Time *time.Time; LSN, Name *string }
```
`EngineMetadata` carries the engine-specific JSONB documented in Suggestion 3 (pg: `wal_start_lsn`/`timeline_id`; mysql: `binlog_file`/`gtid_executed`; mongo: `oplog_start`/`oplog_end`).

**Testing**:
- `Unit: noop engine FullBackup → returns deterministic stream + metadata; satisfies interface`.
- `Unit: engine registry resolves 'postgresql' → postgres driver, unknown → error`.

---

## Phase 3: PostgreSQL Backup & WAL Archiving (Core Value)

### Purpose
Deliver the heart of the product for the primary engine: full/incremental PostgreSQL base backups and continuous WAL archiving to object storage, with the full pipeline (stream → compress → encrypt → store → catalog). After this phase a real PostgreSQL database can be backed up on a schedule with second-level WAL coverage.

### Tasks

#### 3.1 — PostgreSQL full & incremental base backup

**What**: Implement `FullBackup`/`IncrementalBackup` for PostgreSQL.

**Design**:
- Full: use the streaming base-backup protocol (`pg_basebackup --format=tar --wal-method=none` or libpq replication protocol) against a replication connection. Capture `START_WAL_LOCATION` (LSN), `timeline_id`, `system_identifier`, `pg_version` into `EngineMetadata`.
- Incremental: PostgreSQL 17 block-level incremental via `pg_basebackup --incremental=<manifest>`; store the backup manifest as a sibling object; `parent_backup_id` links the chain.
- Output streamed straight into the Phase 2 compress→encrypt→Put pipeline; `storage_path` = `{prefix}/{db_id}/base/{job_id}.tar.zst.enc`.
- Connection params resolved from `connection_config` + `credentials_ref`; TLS 1.3 enforced (`sslmode=verify-full`).

**Testing**:
- `Integration (postgres testcontainer): seed 50MB → FullBackup → object exists, engine_metadata has wal_start_lsn/timeline_id`.
- `Integration: FullBackup then IncrementalBackup after writes → incremental object smaller; parent linkage set`.
- `Integration: bad credentials → TestConnection error surfaced, no partial object`.

#### 3.2 — Continuous WAL archiving

**What**: Stream WAL segments to storage and register them in `log_segments` + emit `wal_lag_measurements`.

**Design**:
- Agent runs `pg_receivewal` (or the replication protocol directly) into a watched directory; each completed 16 MB segment is compressed, encrypted, and `Put` under `{prefix}/{db_id}/wal/{timeline}/{segment_name}.zst.enc`.
- On each archive: insert `log_segments` row (`segment_name`, `lsn_start`/`lsn_end` in `segment_metadata`, `checksum` SHA-256) and write a `wal_lag_measurements` sample (`lag_bytes`, `lag_seconds`, `current_position`, `archived_position`).
- Idempotent: re-archiving an already-present segment is a no-op (UNIQUE constraint guard).
- Replication slot created/managed to prevent the server from recycling un-archived WAL.

**Testing**:
- `Integration: generate WAL via writes → segments appear in log_segments with monotonic LSN ranges, objects in storage`.
- `Integration: re-archive same segment → single row, no duplicate object`.
- `Integration: kill and restart archiver → resumes from last archived segment, no gap`.
- `Unit: lag computed as live_lsn - archived_lsn correctly`.

#### 3.3 — Backup job worker & lifecycle

**What**: A River worker that executes backup jobs and drives the `pending→running→completed/failed` state machine, recording `backup_metrics`.

**Design**:
- States: `pending → running → completed | failed | cancelled`; `expired` set later by retention. Each transition is a transactional `UPDATE` + audit-log write.
- `BackupJobArgs{ JobID, DatabaseID, PolicyID, BackupType }`. Worker: load instance+policy → resolve engine → run backup via the pipeline → update `backup_jobs` (size, compressed_bytes, storage_path, engine_metadata) → insert `backup_metrics` (duration, compression_ratio, throughput).
- Retry policy: River exponential backoff, max 3 attempts; final failure sets `status='failed'` and `error_message`, emits a metric.

**Testing**:
- `Integration: enqueue full-backup job → completes, backup_jobs.status='completed', backup_metrics row present`.
- `Integration (mocked engine error): job fails after 3 retries → status='failed', error_message populated, audit entry written`.
- `Unit: illegal transition completed→running rejected`.

---

## Phase 4: Restore & Point-in-Time Recovery (Core Value)

### Purpose
Make backups useful: deterministic restore of a PostgreSQL base backup plus WAL replay to a precise target timestamp/LSN, into a new instance or an isolated branch. This is the capability the whole product exists to provide.

### Tasks

#### 4.1 — Target resolution & restore planning

**What**: Given a `PITRTarget`, select the base backup and the ordered WAL segment set needed to replay to the target.

**Design**:
- `ResolveTarget`: map a wall-clock time to an LSN/timeline by scanning `log_segments` (segment whose `[lsn_start,lsn_end]` and `archived_at` window bracket the target). Returns `ResolvedTarget{ BaseBackupID, Timeline, TargetLSN, WALSegments []string }`.
- Validate WAL continuity: the chosen base backup's `wal_start_lsn` through the target must have no gap in `log_segments` (else error `ErrWALGap` with the missing range).
- Restore plan persisted to `restore_jobs.pitr_config`.

**Testing**:
- `Integration: target time inside retention → plan selects correct base + contiguous WAL list`.
- `Integration: target before earliest backup → ErrTargetOutsideRetention`.
- `Integration: artificially delete one log_segments row → ErrWALGap names the missing LSN range`.

#### 4.2 — Restore execution & WAL replay

**What**: Execute the plan: download+decrypt base backup, stage it, configure recovery, replay WAL to target, promote.

**Design**:
- Worker downloads base via `BlobStore.Get` → decrypt → decompress → extract to target data dir. Stage required WAL segments to a `restore_command`-served directory.
- Write `recovery.signal` + `recovery_target_time`/`recovery_target_lsn`, `recovery_target_action='promote'` (PostgreSQL PITR protocol, standards.md). Start the server, wait for recovery completion, promote.
- `restore_mode`: `new_instance` (fresh data dir/container), `branch` (clone into an isolated shadow instance — zero-downtime restore differentiator), `in_place` (guarded, requires explicit confirm flag).
- Emit `restore_performance` rows per phase (`base_restore`, `wal_replay`, `validation`, `promotion`) including `time_accuracy_seconds` (actual vs target).

**Testing**:
- `Integration: backup → write row at T1 → write row at T2 → restore to T1 → row from T2 absent, T1 present`.
- `Integration: restore_mode='branch' → new isolated instance, original untouched`.
- `Integration: restore_performance rows emitted for each phase; time_accuracy within 1s`.
- `Integration: corrupted base object → restore fails cleanly, status='failed', no half-promoted server`.

#### 4.3 — Retention & pruning

**What**: Enforce retention policies, expiring base backups and WAL beyond the window while preserving PITR continuity.

**Design**:
- Scheduled prune job per policy: compute cutoff = `now - retention_days`; keep `retention_full_count` newest fulls regardless of age.
- Never delete WAL still needed to reach the oldest retained restorable point; never delete a base backup that a newer incremental chains from. Mark expired `backup_jobs.status='expired'`, then delete objects, then delete `log_segments` rows older than the oldest retained base.
- Minimum 7-day retention enforced at the policy layer (CHECK + API validation).

**Testing**:
- `Integration: 7-day retention, inject 10 days of backups → only ≥cutoff retained, older objects deleted`.
- `Integration: prune must not break PITR → after prune, oldest retained point still restorable end-to-end`.
- `Unit: retention_full_count=2 keeps 2 newest fulls even if older than cutoff`.

---

## Phase 5: Control-Plane API, Auth & Scheduler

### Purpose
Expose all capabilities through an OpenAPI 3.1 REST API protected by OIDC + RBAC, and add the cron scheduler that turns policies into jobs. After this phase the platform is operable programmatically and on autopilot.

### Tasks

#### 5.1 — OpenAPI 3.1 spec & generated server

**What**: Author `api/openapi.yaml` and generate handlers/clients with `oapi-codegen`.

**Design**:
- Resources: `/v1/database-instances`, `/storage-locations`, `/backup-policies`, `/backups`, `/backups/{id}`, `/restores`, `/restores/{id}`, `/alerts`, `/audit-logs`, `/wal-timeline/{databaseId}`.
- Standard verbs; cursor pagination (`?cursor=&limit=`); `application/json`; RFC 9457 problem+json error bodies. Spec served at `/openapi.yaml` and Swagger UI at `/docs`.
- Generated TS client for `web/` and Go client for `dbpctl`.

**Testing**:
- `Unit: spectral lint of openapi.yaml → no errors`.
- `Integration: POST /backups with valid body → 202 + job id; GET /backups/{id} → status`.
- `Integration: invalid body → 400 problem+json naming the field`.

#### 5.2 — OIDC authentication & RBAC

**What**: OIDC login + local fallback, JWT sessions, and per-route RBAC enforcement.

**Design**:
- OIDC Authorization Code + PKCE via `go-oidc`; issue platform JWT (access 15m, refresh 24h). Local users: argon2id password hashes.
- Permissions are `resource:action` strings (e.g. `backup:create`, `restore:execute`, `policy:update`, `audit:read`). Roles map to permission sets: `viewer`={`*:read`}, `operator`=viewer+{`backup:create`,`restore:execute`}, `admin`=`*`.
- Middleware extracts JWT → loads role permissions → checks the route's required permission; denies with 403.

**Testing**:
- `Integration (mock OIDC): valid code → JWT issued; expired token → 401`.
- `Integration: viewer calls POST /restores → 403; operator → 202`.
- `Unit: argon2id verify correct/incorrect password`.

#### 5.3 — Cron scheduler & audit logging

**What**: Translate active policies into scheduled jobs and write an audit record for every mutating action.

**Design**:
- Scheduler loop (leader-elected via Postgres advisory lock) parses `full_backup_cron`/`incr_backup_cron` with `robfig/cron`; on tick enqueues the appropriate River job with `triggered_by='schedule'`. Missed ticks during downtime are reconciled (catch-up at most one run).
- Every state-changing API handler writes an `audit_logs` row (`action`, `resource_type/id`, `user_id`, `ip_address`, before/after in `context`) — satisfies SOC 2 / HIPAA / GDPR audit requirements (standards.md).

**Testing**:
- `Integration: policy with '* * * * *' → job enqueued within the minute`.
- `Integration: two control-plane replicas → only the advisory-lock holder schedules (no double-enqueue)`.
- `Integration: POST /backup-policies → audit_logs row with new_state captured`.

---

## Phase 6: Web Dashboard

### Purpose
Deliver the polished restore/monitoring UX that the CLI-only incumbents lack — the headline differentiator from README. After this phase a DBA can monitor backup health and initiate a PITR restore from a timeline picker without touching a CLI.

### Tasks

#### 6.1 — App shell, auth flow & data layer

**What**: React+Vite SPA with OIDC login, routing, and the generated API client wired through TanStack Query.

**Design**:
- Pages: Dashboard, Databases, Backups, Restore, Alerts, Audit, Settings. shadcn/ui + Tailwind; layout with org/role-aware nav (hide actions the role can't perform).
- OIDC redirect flow; token stored in memory + refresh; 401 → re-auth.

**Testing**:
- `Component (vitest): nav hides 'Restore' action for viewer role`.
- `E2E (Playwright): login → dashboard loads instances`.

#### 6.2 — Backup health dashboard & PITR timeline

**What**: Health overview and an interactive point-in-time selector.

**Design**:
- Dashboard cards per database: last backup status/time, WAL lag, storage used, daily cost — sourced from the cross-model query (Suggestion 4) via `/wal-timeline` and summary endpoints.
- PITR timeline (Recharts/visx): visualises the restorable window (earliest retained point → latest WAL) with a draggable cursor; selecting a time calls target-resolution and shows the resolved LSN before the user confirms `POST /restores`.

**Testing**:
- `Component: timeline renders restorable window from API fixture; cursor outside window disabled`.
- `E2E (Playwright, seeded backend): pick a time → confirm restore → restore job appears running`.

---

## Phase 7: MySQL & MongoDB Engines

### Purpose
Fulfil the multi-database MVP requirement by implementing the `BackupEngine` for MySQL and MongoDB, reusing the entire Phase 2–5 pipeline. After this phase all three engines back up, archive logs, and restore to a point in time.

### Tasks

#### 7.1 — MySQL engine (binlog PITR)

**What**: Full/incremental backup and binlog streaming for MySQL/MariaDB.

**Design**:
- Full: `xtrabackup`/`mariabackup` (or `mysqldump --single-transaction` fallback) streamed into the pipeline; capture `binlog_file`, `binlog_position`, `gtid_executed`, `server_id` into `EngineMetadata` (MySQL Binary Log Replication Protocol, standards.md).
- Continuous: stream binlog via `mysqlbinlog --read-from-remote-server --stop-never` or `BINLOG DUMP GTID`; each rotated binlog → `log_segments` with `gtid_range` in `segment_metadata`.
- Restore/PITR: restore base, then `mysqlbinlog --stop-datetime=<target>` (or `--stop-position`/GTID) replayed against the restored instance.

**Testing**:
- `Integration (mysql testcontainer): backup → writes → restore to target datetime → post-target rows absent`.
- `Integration: GTID-based target resolution selects correct binlog range`.

#### 7.2 — MongoDB engine (oplog PITR)

**What**: Full backup and oplog-slice archiving for MongoDB replica sets.

**Design**:
- Full: `mongodump --oplog` against a secondary (`readPreference=secondary`); capture `oplog_start`/`oplog_end` (`{ts,t}`), `replica_set`, `cluster_time` (MongoDB oplog / PBM protocol, standards.md).
- Continuous: periodic oplog slicing (`oplog_span_min`, default 10) → `log_segments` with `ts_start`/`ts_end` in `segment_metadata`.
- Restore/PITR: `mongorestore --oplogReplay --oplogLimit=<ts>` to replay oplog up to the target timestamp.

**Testing**:
- `Integration (mongo replica-set testcontainer): backup → oplog writes → restore with oplogLimit → state matches target ts`.
- `Integration: oplog slices archived at the configured span; continuity validated`.

---

## Phase 8: Validation, Observability & Compliance

### Purpose
Make backups trustworthy and the platform operable in regulated environments: automated synthetic-restore validation, OpenTelemetry instrumentation, and compliance-grade audit exports. After this phase backups are continuously proven restorable and the platform emits standards-aligned telemetry.

### Tasks

#### 8.1 — Backup validation (synthetic restore to shadow instance)

**What**: Continuously verify checksums, WAL continuity, and a real restore to a throwaway shadow instance.

**Design**:
- Validation job per completed backup: (1) checksum verify object SHA-256; (2) WAL-continuity check over `log_segments`; (3) synthetic restore (reuse Phase 4 into an ephemeral container) + lightweight data-quality checks (row counts / `pg_catalog` sanity / `mongosh` ping).
- Results written to `backup_jobs.validation_summary` and a `backup_validations` outcome; failures raise a `critical` alert.

**Testing**:
- `Integration: valid backup → synthetic restore passes, validation_summary.synthetic_restore_passed=true`.
- `Integration: truncated object → checksum fails → critical alert raised`.

#### 8.2 — OpenTelemetry instrumentation & dashboards

**What**: Emit OTel metrics/traces/logs and ship Grafana dashboards.

**Design**:
- OTel SDK in control-plane and agent; metrics (OTel DB semantic conventions): `backup.duration`, `wal.lag.seconds`, `restore.latency`, `storage.bytes` (by tier), `backup.failures`. OTLP export; Prometheus scrape + Grafana dashboards committed under `deploy/`.

**Testing**:
- `Integration: run a backup → OTLP collector (testcontainer) receives backup.duration metric`.
- `Unit: WAL-lag gauge reflects telemetry sample`.

#### 8.3 — Compliance audit export

**What**: Exportable, tamper-evident audit trail and retention evidence.

**Design**:
- `GET /audit-logs/export?from=&to=&format=json|csv` (RBAC `audit:read`) producing a signed, hash-chained export (each record carries the prior record's hash) for non-repudiation (RFC 4810 / RFC 3227 spirit). Restore-test evidence (8.1 results) included to satisfy ISO 27001 8.13 / SOC 2 restore-test requirements.

**Testing**:
- `Integration: export range → records hash-chain verifies; tampering one record breaks the chain`.
- `Integration: viewer without audit:read → 403`.

---

## Phase 9: Kubernetes Operator (v1.1)

### Purpose
Serve the cloud-native deployment target with a Velero-aligned operator so backups, restores, schedules, and storage locations are declarative Kubernetes resources.

### Tasks

#### 9.1 — CRDs & controllers

**What**: Define `Backup`, `Restore`, `Schedule`, `StorageLocation` CRDs and reconcilers via `controller-runtime`.

**Design**:
- CRD shapes mirror the API resources and the Velero model (standards.md). Reconcilers translate CR specs into control-plane jobs and write status (`phase`, `lastBackupTime`, `restorableWindow`) back to the CR `.status`.
- Helm chart under `deploy/helm` installs CRDs, operator, control-plane, and ai-service.

**Testing**:
- `Integration (envtest): apply a Schedule CR → backup jobs created on cadence; status updated`.
- `Integration: apply Restore CR → restore job runs; CR status reaches Completed`.

---

## Phase 10: AI-Native Features

### Purpose
Deliver the differentiators no incumbent offers (research.md/features.md): anomaly-triggered protective snapshots, natural-language PITR, cost forecasting, and schedule optimisation — powered by the Python `ai-service` reading the TimescaleDB continuous aggregates.

### Tasks

#### 10.1 — Write-pattern anomaly detection & protective snapshots

**What**: Detect ransomware/bulk-delete/schema-corruption patterns and auto-trigger an out-of-schedule backup.

**Design**:
- Agents sample `write_rate_samples` (`writes_per_second`, `rows_deleted`, `ddl_operations`, `entropy_score`). `ai-service /anomaly/score` compares current rate to the 7-day baseline from `write_rate_hourly` (z-score) and entropy heuristics; writes `anomaly_scores`.
- On `is_triggered`, control plane creates an `anomaly_alerts` row and enqueues a protective backup (`triggered_by='anomaly'`), linking `snapshot_id`. Critical alerts can isolate the snapshot as immutable (Object Lock).

**Testing**:
- `Integration: inject 15x write spike + high entropy → critical alert + protective snapshot enqueued`.
- `Unit: z-score computed correctly against baseline; below threshold → no trigger`.

#### 10.2 — Natural-language PITR

**What**: Translate restore intent ("restore before the orders table truncation yesterday afternoon") into a precise timestamp/LSN.

**Design**:
- `POST /restores` accepts `nl_query`. `ai-service /nlpitr/resolve` combines an LLM with structured evidence (recent DDL/TRUNCATE/DELETE from WAL analysis + `log_segments` timeline) to propose `resolved_target_time`, `confidence`, and `candidate_events` (Suggestion 3 `nl_query` shape). The UI shows candidates for confirmation before executing.

**Testing**:
- `Integration (fixture corpus): NL query → resolves to expected timestamp within tolerance; confidence reported`.
- `Integration: ambiguous query → multiple candidate_events returned, no auto-execute`.

#### 10.3 — Cost forecasting & schedule optimisation

**What**: Forecast storage cost across retention windows and recommend backup cadence balancing RPO/RTO vs cost.

**Design**:
- `ai-service /forecast` runs linear regression on `storage_daily` growth (Suggestion 4 query) to project 30-day cost per retention option; `/schedule/optimize` analyses `write_rate_hourly` velocity to recommend `full_backup_cron`/`retention_days`, written to `backup_policies.ai_recommendations` with `confidence`. Surfaced as accept/dismiss suggestions in the UI.

**Testing**:
- `Integration: 30 days of storage_consumption → projected_cost within tolerance of linear extrapolation`.
- `Integration: high-velocity DB → recommends more frequent fulls than a low-velocity DB`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Metadata Store        ─── required by everything
    │
Phase 2: Storage, Crypto & Engine Abstraction ── requires 1
    │
Phase 3: PostgreSQL Backup & WAL (core)       ── requires 2
    │
Phase 4: Restore & PITR (core)                ── requires 3
    │
    ├── Phase 5: API, Auth & Scheduler         ── requires 4
    │       │
    │       ├── Phase 6: Web Dashboard         ── requires 5  (parallel with 7, 8)
    │       ├── Phase 7: MySQL & MongoDB        ── requires 4+5 (parallel with 6, 8)
    │       └── Phase 8: Validation/Obs/Compliance ─ requires 4+5 (parallel with 6, 7)
    │
    ├── Phase 9: Kubernetes Operator (v1.1)     ── requires 5 (parallel with 10)
    └── Phase 10: AI-Native Features            ── requires 4+5 (+7 for full multi-engine) (parallel with 9)
```

**Parallelism opportunities:**
- After Phase 5: **Phases 6, 7, and 8** can be developed concurrently by separate contributors.
- After Phase 5: **Phases 9 and 10** can proceed in parallel (operator vs AI), each independent of the other.

MVP scope (README "Must-have"): **Phases 1–8**. v1.1 (README "Should-have"): **Phases 9–10**.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests pass (`go test ./...`, `pytest`, `vitest`); `testcontainers` integration suites green in CI.
3. `golangci-lint`, `ruff`, `mypy`, `eslint`, `prettier --check`, and `tsc --noEmit` pass with no errors.
4. New/changed API operations are reflected in `api/openapi.yaml` and the spec passes `spectral` lint; generated Go/TS clients regenerated.
5. Database changes ship as forward + reverse `golang-migrate` migrations that apply and roll back cleanly against TimescaleDB.
6. `docker compose build` succeeds and the affected services start healthy.
7. The phase's headline capability works end-to-end (demonstrated by at least one integration or E2E test using a real dependency via testcontainers).
8. New configuration options (env vars / policy fields) are documented in the README/config reference with defaults.
9. New mutating actions emit audit-log entries and OTel metrics where applicable.
10. Security-sensitive code (crypto, auth, restore) has explicit negative-path tests (tamper, unauthorized, corrupted input).
