# Database Backup & PITR Platform — Feature & Functionality Survey

> Candidate #294 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| AWS Backup | Commercial SaaS | Proprietary; pay-per-GB stored + restore requests | https://aws.amazon.com/backup/ |
| Google Cloud Spanner PITR | Commercial SaaS | Proprietary; included in Spanner pricing | https://cloud.google.com/spanner/ |
| pgBackRest | Open source | MIT license | https://pgbackrest.org/ |
| WAL-G | Open source | Apache 2.0 license | https://github.com/wal-g/wal-g |
| Percona Everest | Open source + commercial | Commercial support available | https://www.percona.com/software/percona-everest |
| PlanetScale Postgres | Commercial SaaS | Proprietary; usage-based pricing | https://planetscale.com/postgres |
| Supabase | Commercial SaaS | Proprietary; free tier + paid plans | https://supabase.com/ |
| Litestream | Open source | Apache 2.0 license | https://litestream.io/ |
| Barman | Open source | GNU General Public License v3 | https://pgbarman.org/ |

## Feature Analysis by Solution

### AWS Backup

**Core features**
- Centralised backup management across AWS resources (RDS, Aurora, DynamoDB, Keyspaces) with PITR to 1-second precision
- Configurable retention windows from 1 to 35 days
- Automated backup scheduling with no impact on production performance
- Per-second granularity continuous backup for DynamoDB
- Transaction log replay mechanism to recover data to any point within retention window
- Vault locking for compliance (prevents accidental or malicious deletion)
- Cross-region backup replication for disaster recovery

**Differentiating features**
- Deep native integration with AWS services (no middleware required)
- Vault locking meets stringent regulatory requirements for immutable backups
- Consolidated billing across all AWS resources in one service
- Enterprise SLA guarantees

**UX patterns**
- AWS Management Console with straightforward PITR interface
- Backup rule templates for common scenarios
- Point-in-time selection via calendar UI with granular timestamp control

**Integration points**
- AWS SDK for programmatic backup and restore operations
- CloudFormation templates for infrastructure-as-code backup policies
- AWS Identity & Access Management (IAM) for role-based access control
- EventBridge for automated notifications

**Known gaps**
- Tightly coupled to AWS ecosystem; limited multi-cloud capability
- Data always stored in AWS (no hybrid or on-premises option)
- Per-GB storage costs can accumulate at scale for large databases
- Limited control over compression algorithms and backup scheduling granularity

**Licence / IP notes**
- Proprietary AWS service; no licensing concerns for adopters

---

### Google Cloud Spanner PITR

**Core features**
- Native PITR with microsecond-level granularity
- Version retention period configurable from 1 hour to 7 days (included in pricing)
- Two recovery modes: surgical (stale read + selective restore) and full database restore
- Schema and data protected within retention window
- No additional charges for PITR capability (costs scale with retention period and write frequency)
- Global availability across all Google managed regions

**Differentiating features**
- Microsecond precision (vs. second-level in most competitors)
- Surgical recovery via stale reads allows selective column/row restoration without full recovery
- Retention tuning based on write frequency; no fixed cost

**UX patterns**
- Cloud Console interface with timestamp picker
- Stale read queries use standard SQL with TIMESTAMP specifications
- Full restore creates new database from backup snapshot

**Integration points**
- Spanner SQL API with timestamp parameters
- gcloud CLI for programmatic backup and restore
- Cloud IAM for access control
- Cloud Logging integration

**Known gaps**
- Limited to Spanner database (not applicable to PostgreSQL, MySQL, or other DBMSs)
- 7-day retention maximum may be insufficient for regulatory compliance in some industries
- Longer retention periods increase storage and compute costs significantly
- No multi-cloud or on-premises deployment option

**Licence / IP notes**
- Proprietary Google Cloud service; no licensing concerns for adopters

---

### pgBackRest

**Core features**
- Full, incremental, and differential backups for PostgreSQL
- WAL archiving with asynchronous parallel processing to multiple object stores (S3, Azure, GCS)
- Block-level differential backups to minimise storage and backup time
- Configurable WAL retention policies per backup generation
- Point-in-time recovery via WAL replay to any point within retention window
- Compression options: lz4 (default), lzma, zstd, brotli
- Standby recovery support (non-exclusive backups do not impact production)

**Differentiating features**
- Highly configurable retention policies for granular cost control
- Block-level safety (no rsync-style time-resolution vulnerabilities)
- Asynchronous WAL archiving with parallelism for extreme write-volume databases
- Multi-storage destination support for redundancy

**UX patterns**
- CLI-first design with comprehensive configuration files
- Verbose operational logging for debugging
- Stanza-based organisation (one stanza = one database)

**Integration points**
- S3, Azure Blob, GCS, or SFTP repository targets
- PostgreSQL pg_basebackup protocol
- Streaming replication for WAL capture
- Archival command interface

**Known gaps**
- No web UI or dashboard; CLI-only operations
- Steep learning curve for complex configurations
- Limited to PostgreSQL; no MySQL, MongoDB, or other database support
- Minimal error recovery guidance; requires operational expertise
- Restore process requires detailed WAL configuration understanding

**Licence / IP notes**
- MIT license; permissive for commercial use and modification

---

### WAL-G

**Core features**
- Multi-database support: PostgreSQL, MySQL/MariaDB, MongoDB (beta), etcd, Redis (beta), MS SQL Server (beta)
- Cloud-native design with multiple compression algorithms: LZ4, LZMA, ZSTD, Brotli
- Parallel processing for fast backup and restoration
- Non-exclusive base backups (no production impact)
- Failover storage support (automatic fallback if primary storage fails)
- Written in Go for performance and reliability

**Differentiating features**
- Broadest multi-database support among open-source tools
- Built-in failover storage mechanism (unique feature)
- Fast, cloud-native architecture with minimal memory footprint
- Lightweight binary suitable for containerised environments

**UX patterns**
- CLI-first with environment variable configuration
- Minimal UI; focused on programmatic integration
- Docker-friendly design for Kubernetes deployments

**Integration points**
- Multiple cloud storage providers (S3, GCS, Azure Blob, SFTP)
- PostgreSQL native streaming replication
- MySQL/MariaDB mysqldump and binlog streaming
- MongoDB mongodump and oplog
- Kubernetes operator support

**Known gaps**
- No dashboard or monitoring UI
- Limited documentation compared to pgBackRest
- Beta support for MongoDB and Redis means production use requires caution
- Failover storage configuration requires manual setup
- Minimal error recovery guidance for failed restores

**Licence / IP notes**
- Apache 2.0 license; permissive for commercial use and modification

---

### Percona Everest

**Core features**
- Kubernetes-native database operations platform supporting MySQL, MongoDB, PostgreSQL
- Scheduled backups and on-demand snapshots with S3-compatible storage
- Point-in-time recovery (PITR) with WAL log replay
- Database provisioning from backups (create new cluster from backup snapshot)
- Automatic operator deployment and lifecycle management
- Multi-database management from single pane of glass
- High availability and failover management built-in

**Differentiating features**
- Purpose-built for Kubernetes-native deployments via operators
- Single platform manages both database operations and backup/recovery
- Automatic operator updates without downtime
- Multi-database support (MySQL, MongoDB, PostgreSQL) in one product

**UX patterns**
- Web console with drag-and-drop cluster provisioning
- Visual status dashboards showing backup history and PITR windows
- Backup schedule UI with calendar picker for granular control

**Integration points**
- Kubernetes API and custom resource definitions (CRDs)
- S3-compatible object storage for backups
- Helm charts for deployment automation
- REST API for programmatic access

**Known gaps**
- Kubernetes requirement limits on-premises or single-server deployments
- PITR upload interval means gap between latest backup and recovery point (not true continuous)
- Learning curve steep for non-Kubernetes operators
- Limited to three database engines; no Oracle, SQL Server, or MariaDB support
- Community edition support limited; enterprise support model less mature than competitors

**Licence / IP notes**
- Percona Community or commercial support model available; code availability and license terms require verification (vendor claims "open source" but specific license not explicitly confirmed in searches)

---

### PlanetScale Postgres

**Core features**
- Managed PostgreSQL with PITR support (new offering, built on MySQL legacy)
- PITR available from oldest backup to 5 minutes before current time
- Continuous WAL logging and base backup snapshots
- WAL replay to reconstruct exact database state at target timestamp
- Automatic backups with configurable retention
- Manual on-demand backups with deletion prevention option
- Branching for instant database cloning from any backup point

**Differentiating features**
- Frictionless restore UX with branching (creates new cluster without restore ceremony)
- Integration with PlanetScale's Postgres branching feature for CI/CD
- Zero-downtime restore (new branch created; old cluster remains available)

**UX patterns**
- Web dashboard with visual PITR timeline
- Point-in-time selector with simple date/time picker
- Restore as branching operation (mental model is Git-like, not traditional DB restore)

**Integration points**
- REST API for programmatic branch creation and PITR
- MySQL/Postgres driver compatibility (standard)
- GitHub integrations for CI/CD workflows

**Known gaps**
- Vendor lock-in (PlanetScale proprietary branching model)
- Restore performance depends on cluster size and WAL volume; no explicit SLA
- PITR limited to 5-minute resolution (not continuous to present moment)
- Limited retention control (tied to backup schedule, not configurable per business need)
- Limited documentation on PITR performance characteristics
- Newer offering (Postgres is recent for PlanetScale); less battle-tested than MySQL

**Licence / IP notes**
- Proprietary managed service; no licensing concerns for adopters

---

### Supabase

**Core features**
- Managed PostgreSQL with PITR available on Pro, Team, and Enterprise plans
- Daily automated backups (7 days on Pro, 14 days on Team, 30 days on Enterprise)
- PITR with second-level granularity on paid tiers (hourly billing)
- PITR requires Small compute add-on for performance
- WAL-based recovery mechanism (inherited from PostgreSQL)
- Web console for restore initiation

**Differentiating features**
- Developer-friendly platform with batteries-included Postgres
- Simple pricing for small projects (free tier has daily backups; PITR à la carte)
- Managed infrastructure removes operational burden

**UX patterns**
- Simple backup/restore UI in Supabase dashboard
- PITR toggle to enable/disable (substitutes for daily backups, not complementary)
- One-click restore initiation (creates new database branch)

**Integration points**
- Standard PostgreSQL wire protocol
- Supabase REST API for programmatic access
- GitHub integrations
- Webhook support for backup notifications

**Known gaps**
- PITR only on paid tiers (free tier not supported)
- PITR requires additional compute add-on (cost not transparent upfront)
- Hourly billing for PITR can be unpredictable for users testing recovery
- No custom retention policy control (fixed per tier)
- Vendor lock-in to Supabase ecosystem
- Limited transparency on WAL storage and recovery performance
- Enabling PITR disables daily backups (must choose one model)

**Licence / IP notes**
- Proprietary managed service; no licensing concerns for adopters

---

### Litestream

**Core features**
- Streaming replication for SQLite to cloud object storage (S3, Azure Blob, Google Cloud Storage, SFTP)
- Continuous incremental backup (no periodic snapshots; real-time replication)
- Point-in-time recovery via object storage history
- Multiple simultaneous replication targets for redundancy
- Runs as separate background process (no application modification needed)
- Low overhead and minimal latency impact

**Differentiating features**
- Only open-source solution for SQLite PITR/continuous backup
- True streaming replication (vs. periodic snapshots)
- Elegant separation-of-concerns design (runs as sidecar process)
- Extremely lightweight and embeddable

**UX patterns**
- Configuration-driven (YAML config file)
- CLI-first restore operations
- Minimal UI; designed for infrastructure integration

**Integration points**
- S3, Azure Blob, Google Cloud Storage, SFTP targets
- Standard SQLite API (no changes needed to application)
- REST API for programmatic restore
- Docker and Kubernetes friendly

**Known gaps**
- SQLite only (not applicable to PostgreSQL, MySQL, or other databases)
- Limited point-in-time precision documentation (depends on WAL page granularity)
- No managed service; self-hosted operation required
- Minimal observability; relies on application logs
- No dashboard or centralized management UI
- Community-driven; limited commercial support options
- Restore process requires operational expertise

**Licence / IP notes**
- Apache 2.0 license; permissive for commercial use and modification

---

### Barman

**Core features**
- PostgreSQL-specific backup and recovery manager
- WAL streaming via pg_receivewal for continuous archiving
- Archive_command support for traditional WAL archiving
- Configurable retention policies per backup
- Point-in-time recovery via WAL replay
- Multiple backup levels (full, incremental) on some configurations
- Remote backup capability (backups can be pulled from standby)

**Differentiating features**
- WAL streaming achieves near-zero RPO (Recovery Point Objective) with meticulous design
- Centralised backup management for multiple PostgreSQL servers
- Well-established, battle-tested tool used by enterprises

**UX patterns**
- CLI-first with config files per PostgreSQL server
- Verbose operational logging
- Recovery commands require understanding WAL file management

**Integration points**
- PostgreSQL native pg_receivewal and archive_command
- Streaming replication protocol
- File system or NFS for backup storage (cloud storage less documented)
- Custom scripts for pre/post-backup hooks

**Known gaps**
- No modern web UI or dashboard (CLI only)
- PostgreSQL-only (no MySQL, MongoDB, or other databases)
- Steep operational learning curve
- Complex WAL management for recovery operations
- Limited cloud-native deployment support (design predates containerisation)
- Minimal error recovery guidance; requires senior PostgreSQL expertise
- Modern competitors (pgBackRest, WAL-G) have better cloud integration

**Licence / IP notes**
- GNU General Public License v3; copyleft license requiring derivative works to be open-sourced if distributed

---

## Cross-Cutting Feature Themes

### Table-Stakes Features

Any production database backup and PITR platform must include:

- **Automated backup scheduling** with configurable frequency and retention policies
- **Point-in-time recovery** to any point within retention window (at minimum to second-level granularity)
- **Transaction log (WAL) archiving** for continuous backup capability
- **Multiple backup types** (full, differential, incremental) to balance cost and recovery time
- **Cloud object storage integration** (S3, GCS, Azure Blob) for scalable, durable storage
- **Encryption at rest and in transit** to meet regulatory compliance
- **Backup validation and integrity testing** to ensure restorability
- **Configurable retention policies** per backup generation or database
- **Logging and audit trails** for compliance and troubleshooting
- **Non-exclusive backups** (production impact minimised or eliminated)

### Differentiating Features

Capabilities that provide competitive advantage:

- **Dashboard or web console** for monitoring and restore initiation (vs. CLI-only)
- **Multi-database support** in single platform (Percona Everest, WAL-G) vs. single-database focus
- **Failover storage** (WAL-G) — automatic fallback if primary storage fails; rare in market
- **Surgical recovery** (Google Cloud Spanner) — selective row/column restore without full recovery
- **Branching for instant cloning** (PlanetScale) — creates isolated database from backup without restore ceremony
- **Streaming replication** (Litestream) — true continuous backup vs. periodic snapshots
- **Kubernetes-native operators** (Percona Everest) — designed for cloud-native deployments
- **Zero-downtime recovery** — new database created from backup; original remains available (PlanetScale)
- **Integrated database management** (Percona Everest, Supabase) — backup is part of larger platform, not standalone

### Underserved Areas / Opportunities

Gaps where existing solutions leave genuine room for innovation:

- **Intelligent backup scheduling** — use AI to analyse query patterns, write frequency, and data-change velocity to recommend optimal backup intervals, balancing cost and RPO/RTO without manual tuning
- **Anomaly-driven backup triggering** — automatically detect unusual write patterns (potential ransomware, bulk deletes, schema corruption) and trigger out-of-schedule snapshots
- **Natural-language restore requests** — translate user intent ("restore to just before the accidental table truncation yesterday at 3pm") to precise WAL replay timestamps without requiring timestamp lookup
- **Automated restore integrity testing** — continuously test recovery by replaying recent WAL to shadow instances and running data-quality checks without human intervention
- **Cost forecasting and optimisation** — predict backup storage costs across retention policies and storage tiers; recommend retention window minimizing both cost and compliance risk
- **Cross-cloud backup portability** — export/import backups between cloud providers to avoid vendor lock-in (most platforms store in proprietary formats)
- **Unified multi-cloud management** — manage PostgreSQL, MySQL, and MongoDB backups across AWS, GCP, and Azure with single UI and billing (Percona Everest is Kubernetes-only)
- **Ransomware detection and immutable snapshots** — automatically detect compromised backup and isolate immutable snapshots; alert on anomalous retention changes
- **Incremental restore** — restore only changed rows/objects since last restore point, reducing RTOs (not available in most solutions)

### AI-Augmentation Candidates

Features where AI could dramatically improve outcomes over manual/rule-based approaches:

- **Backup schedule optimisation** — analyse query patterns, transaction volume, and data velocity to recommend backup frequency balancing RPO/RTO and cost
- **Anomaly detection** — flag unusual write patterns (ransomware, bulk deletes, schema corruption) and trigger protective snapshots
- **Natural-language PITR** — translate business intent ("restore before the bad import") to exact WAL replay timestamp
- **Restore impact assessment** — predict query performance impact of restore to given timestamp based on schema changes and data patterns
- **Intelligent backup pruning** — recommend which older backups can be safely deleted while maintaining compliance and RPO/RTO targets
- **Root-cause analysis for restore failures** — classify failures (storage unavailable, WAL corrupted, schema incompatible) and suggest remediation

---

## Legal & IP Summary

All solutions reviewed have clear licence or commercial status:

- **AWS Backup, Google Cloud Spanner, PlanetScale, Supabase**: Proprietary cloud services with no licensing concerns for adopters.
- **pgBackRest**: MIT license (permissive).
- **WAL-G**: Apache 2.0 license (permissive).
- **Litestream**: Apache 2.0 license (permissive).
- **Barman**: GNU General Public License v3 (copyleft; requires derivative works to be open-sourced if distributed).
- **Percona Everest**: License terms claimed as "open source" but specific license type and terms **require verification before adoption**. Recommend legal review of license compatibility if planning derivative works or proprietary distribution.

No solutions reviewed appear encumbered by active software patents. Standard PITR techniques (WAL replay, differential backups, exponential backoff) are well-established industry practices not subject to known licensing restrictions.

---

## Recommended Feature Scope

### Must-have (MVP)

- Automated backup scheduling with configurable frequency and retention windows (minimum 7-day retention)
- Point-in-time recovery to any point within retention window (second-level granularity minimum)
- WAL-based continuous archiving to cloud object storage (S3, GCS, Azure Blob)
- Full and incremental backup types with intelligent scheduling
- Web dashboard for monitoring backup health, retention status, and restore initiation
- Encryption at rest and in transit meeting industry standards (AES-256)
- Multi-database support (PostgreSQL, MySQL, MongoDB)
- Backup validation and integrity testing with synthetic restores to shadow instances

### Should-have (v1.1)

- Kubernetes-native operator for cloud-native deployments
- Failover storage support (automatic fallback if primary storage fails)
- Natural-language PITR queries ("restore before the table truncation")
- Anomaly detection on write patterns triggering protective snapshots
- Cost forecasting and optimisation recommendations
- Zero-downtime restore via database branching (create isolated copy from backup)
- Role-based access control and audit logging for compliance
- Ransomware detection with immutable snapshot isolation

### Nice-to-have (backlog)

- Surgical recovery (selective row/column restore without full recovery)
- Cross-cloud backup portability (export/import between AWS/GCP/Azure)
- Incremental restore (restore only changed data since last restore)
- Integrated database monitoring and alerting
- Machine learning on backup patterns to predict future recovery needs
- Federated management of heterogeneous database platforms (managed and self-hosted)
- Automated performance impact assessment before restore

---

## Sources

- [AWS Backup PITR Documentation](https://docs.aws.amazon.com/aws-backup/latest/devguide/point-in-time-recovery.html)
- [AWS Backup PITR for Aurora Blog](https://aws.amazon.com/blogs/storage/streamlining-point-in-time-recovery-pitr-for-amazon-aurora-with-aws-backup/)
- [Google Cloud Spanner PITR Overview](https://cloud.google.com/spanner/docs/pitr)
- [Google Cloud Spanner PITR Usage Documentation](https://docs.cloud.google.com/spanner/docs/use-pitr)
- [pgBackRest Documentation](https://pgbackrest.org/)
- [pgBackRest User Guide](https://pgbackrest.org/user-guide.html)
- [pgBackRest GitHub Repository](https://github.com/pgbackrest/pgbackrest)
- [WAL-G GitHub Repository](https://github.com/wal-g/wal-g)
- [WAL-G Documentation](https://wal-g.readthedocs.io/)
- [Percona Everest Documentation](https://docs.percona.com/everest/)
- [Percona Everest Platform](https://www.percona.com/software/percona-everest)
- [PlanetScale Postgres PITR Documentation](https://planetscale.com/docs/postgres/backups/point-in-time-recovery)
- [Supabase Database Backups Documentation](https://supabase.com/docs/guides/platform/backups)
- [Supabase PITR Management Documentation](https://supabase.com/docs/guides/platform/manage-your-usage/point-in-time-recovery)
- [Litestream Documentation](https://litestream.io/)
- [Litestream GitHub Repository](https://github.com/benbjohnson/litestream)
- [Barman PostgreSQL Backup Manager](https://pgbarman.org/)
- [Barman Manual Documentation](https://docs.pgbarman.org/release/3.9.0/)
