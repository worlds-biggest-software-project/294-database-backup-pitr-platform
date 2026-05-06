# Standards & API Reference

> Project: Database Backup & PITR Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 27001:2022 — Annex A Control 8.13: Information Backup**
- URL: https://www.iso.org/standard/82875.html
- The primary ISO standard governing backup requirements for information security management systems. Control 8.13 mandates that organisations maintain backup copies of information, software, and systems and test them regularly against agreed recovery objectives. Requires encryption of backup data commensurate with its risk classification, off-site or separate-region storage, and demonstrated RTO/RPO alignment through periodic restore tests. Any platform targeting enterprise or regulated-industry buyers must produce audit evidence compatible with ISO 27001 assessments.

**ISO/IEC 27002:2022 — Implementation Guidance for Control 8.13**
- URL: https://www.iso.org/standard/75652.html
- Companion guidance document to ISO 27001 that expands on implementation specifics for backup: frequency, media rotation, labelling, transport, secure disposal, and restoration testing procedures. Relevant to any platform producing compliance-ready backup-event audit trails.

### W3C & IETF Standards

**RFC 3227 — Guidelines for Evidence Collection and Archiving (IETF BCP 55)**
- URL: https://www.rfc-editor.org/rfc/rfc3227.html
- Best Current Practice for the ordering and archiving of evidence from systems. Its principle of "volatile-to-less-volatile" data capture and bit-level copy requirements is directly applicable to forensic backup integrity and chain-of-custody guarantees expected by regulated customers.

**RFC 4810 — Long-Term Archive Service Requirements (IETF)**
- URL: https://www.rfc-editor.org/rfc/rfc4810
- Defines requirements for systems that must demonstrably preserve data integrity over long archival periods. Relevant to backup platforms that must provide non-repudiation evidence—i.e., proving a backup existed at a specific point in time and was not subsequently altered. Applicable to WORM-style backup vault features.

**RFC 9000 — QUIC: A UDP-Based Multiplexed and Secure Transport**
- URL: https://datatracker.ietf.org/doc/html/rfc9000
- QUIC is increasingly relevant as a high-throughput, low-latency transport for streaming WAL segments and base-backup data to cloud object storage at scale. Some next-generation database replication and backup systems are evaluating QUIC as an alternative to TCP for WAL streaming in high-latency WAN environments.

### Data Model & API Specifications

**PostgreSQL WAL Archiving and PITR Protocol**
- URL: https://www.postgresql.org/docs/current/continuous-archiving.html
- The foundational technical specification for WAL-based continuous archiving and point-in-time recovery in PostgreSQL. Defines the `archive_command`, `archive_library`, WAL segment format, `recovery_target_time`, and `restore_command` configuration interfaces. All open-source PostgreSQL backup tools (pgBackRest, WAL-G, Barman) implement this protocol. Essential reading for any platform targeting PostgreSQL PITR.

**MySQL Binary Log Replication Protocol**
- URL: https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_replication.html
- Official MySQL internals specification for the binary log (binlog) format and streaming replication protocol used for PITR in MySQL and MariaDB. Defines event types, log structure, and the network protocol for streaming binlog events to backup agents. WAL-G and Percona tools implement this protocol for MySQL PITR.

**MongoDB Oplog Specification and Percona Backup for MongoDB (PBM) Protocol**
- URL: https://docs.percona.com/percona-backup-mongodb/features/point-in-time-recovery.html
- The MongoDB oplog is the de facto transaction log mechanism for PITR in MongoDB. The Percona Backup for MongoDB open-source tool defines a widely-adopted configuration schema (oplogSpanMin, compression algorithms, priority weighting) for oplog-slice collection. Any multi-database backup platform targeting MongoDB must implement oplog-compatible archiving.

**OpenAPI Specification 3.1 (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.1.0
- The industry-standard format for describing REST APIs. Backup platforms exposing programmatic access to backup plans, restore operations, and audit logs should publish an OAS 3.1 schema to enable SDK auto-generation, API gateway integration, and third-party tooling. AWS Backup and Percona Everest both expose REST APIs; adopting OAS 3.1 ensures interoperability.

**Kubernetes Custom Resource Definition (CRD) API**
- URL: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
- The Kubernetes CRD extension mechanism is the de facto standard for declaring backup, restore, and schedule operations in cloud-native database platforms. Velero (the leading Kubernetes backup tool) defines `Backup`, `Restore`, and `Schedule` CRD types that have become informal standards for cloud-native backup orchestration. Platforms targeting Kubernetes deployments should align with or extend this model.

**Velero CRD Specification**
- URL: https://velero.io/docs/
- Velero has positioned itself as the Kubernetes backup standard, using CRDs to represent backup lifecycle declaratively. Its CRD schema (`BackupStorageLocation`, `VolumeSnapshotLocation`, `Schedule`) is adopted by multiple cloud-native database backup solutions. A PITR platform should evaluate compatibility with or extension of the Velero CRD model.

**AWS S3 API / S3-Compatible Object Storage Protocol**
- URL: https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html
- The de facto standard protocol for backup archive storage, implemented by AWS S3, Google Cloud Storage (via interoperability endpoint), Azure Blob Storage (S3-compatible mode), MinIO, Ceph, and others. All major open-source backup tools (pgBackRest, WAL-G, Barman Cloud, Litestream) target the S3 API. A backup platform must implement S3-compatible object storage as its primary storage backend interface.

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) and OpenID Connect 1.0**
- URLs: https://datatracker.ietf.org/doc/html/rfc6749 · https://openid.net/specs/openid-connect-core-1_0.html
- The standard authorisation and authentication protocols for securing backup platform APIs and web dashboards. Required for enterprise SSO integration (Okta, Azure AD, Google Workspace) and for delegated access control when backup agents authenticate against the platform.

**NIST SP 800-34 Rev. 1 — Contingency Planning Guide for Federal Information Systems**
- URL: https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final
- Defines Maximum Tolerable Downtime (MTD), Recovery Time Objective (RTO), and Recovery Point Objective (RPO) as canonical metrics for backup and recovery planning. Specifies backup frequency requirements by system impact level (low/moderate/high). These metrics are the universal language for backup SLA design and must be surfaced in any platform targeting US federal, government-adjacent, or compliance-driven enterprise customers.

**SOC 2 Type II — Trust Services Criteria (AICPA)**
- URL: https://www.aicpa-cima.com/resources/landing/system-and-organization-controls-soc-suite-of-services
- SOC 2 availability and confidentiality criteria mandate encrypted backups at rest (AES-256) and in transit (TLS 1.2+), RBAC with MFA for administrative access, centralized audit logging, regular restore-test evidence, and incident response procedures. A backup platform serving SaaS vendors or regulated enterprises must be SOC 2 Type II certified and provide audit-trail exports supporting customer audits.

**HIPAA Security Rule — 45 CFR § 164.308(a)(7) — Contingency Plan**
- URL: https://www.hhs.gov/hipaa/for-professionals/security/laws-regulations/index.html
- Requires covered entities and business associates to maintain a data backup plan, a disaster recovery plan, and an emergency mode operations plan. Mandates six-year retention for access logs and backup records. Any backup platform handling healthcare data (PHI) must support HIPAA-compliant retention windows and business associate agreement (BAA) workflows.

**GDPR Article 5 and Article 32 — Data Protection Principles and Security of Processing**
- URL: https://gdpr-info.eu/art-5-gdpr/ · https://gdpr-info.eu/art-32-gdpr/
- Article 5 requires that personal data not be retained longer than necessary for its purpose (storage limitation). Article 32 requires appropriate technical measures to ensure data security, including the ability to restore availability and access to personal data in a timely manner. These provisions directly affect backup retention policy design (retention limits, right-to-erasure from backups) and restore-capability testing requirements.

**AES-256 Encryption Standard (FIPS 197)**
- URL: https://csrc.nist.gov/publications/detail/fips/197/final
- The universally required encryption algorithm for data at rest in backup archives. All major cloud backup services (AWS Backup, Google Cloud SQL, Supabase) and open-source tools (pgBackRest, WAL-G) support AES-256 encryption. A backup platform must implement AES-256 for archive encryption and document key management procedures.

**TLS 1.3 (RFC 8446) — Transport Layer Security**
- URL: https://datatracker.ietf.org/doc/html/rfc8446
- The standard encryption protocol for data in transit. WAL segment streaming, object storage uploads, and platform API calls must all use TLS 1.3 (minimum TLS 1.2 for legacy compatibility). Required by SOC 2, HIPAA, and GDPR technical controls.

### Observability & Monitoring Standards

**OpenTelemetry (CNCF) — Metrics, Traces, and Logs**
- URL: https://opentelemetry.io/docs/
- The CNCF standard for vendor-neutral observability instrumentation. Database backup platforms should emit OTel-compatible metrics (backup duration, WAL lag, storage consumption, restore latency) and structured logs to support integration with Prometheus, Grafana, Datadog, and other observability stacks. The OTel semantic conventions for databases are directly applicable.

---

## Similar Products — Developer Documentation & APIs

### AWS Backup

- **Description:** Centralised AWS backup service supporting RDS, Aurora, DynamoDB, EFS, and other AWS services with PITR at 1-second precision and up to 35-day retention windows.
- **API Documentation:** https://docs.aws.amazon.com/aws-backup/latest/devguide/api-reference.html
- **SDKs/Libraries:** AWS SDK for Go (v2): https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/service/backup · AWS SDK for Python (Boto3): https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/backup.html · Additional SDKs for Java, JavaScript, .NET available at https://aws.amazon.com/developer/tools/
- **Developer Guide:** https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html
- **Standards:** REST/JSON; AWS-proprietary API shape (not OAS 3.1 published); CloudFormation IaC integration
- **Authentication:** AWS IAM (SigV4 request signing); IAM roles for service access control

---

### Google Cloud SQL — PITR

- **Description:** Managed PITR for Cloud SQL instances running MySQL, PostgreSQL, and SQL Server. Enterprise Plus edition supports 1–35 day retention; Enterprise edition supports 1–7 days.
- **API Documentation:** https://docs.cloud.google.com/sql/docs/postgres/backup-recovery/pitr · https://docs.cloud.google.com/sql/docs/mysql/backup-recovery/pitr
- **SDKs/Libraries:** Google Cloud Client Libraries for Python, Go, Java, Node.js, Ruby, PHP: https://cloud.google.com/apis/docs/cloud-client-libraries · gcloud CLI: https://cloud.google.com/sdk/gcloud
- **Developer Guide:** https://docs.cloud.google.com/sql/docs/postgres/backup-recovery/configure-pitr
- **Standards:** REST/JSON (Google Discovery Document format); Terraform provider support
- **Authentication:** Google Cloud IAM; OAuth 2.0 service accounts

---

### Percona Everest

- **Description:** Kubernetes-native open-source database operations platform with PITR support for MySQL, MongoDB, and PostgreSQL via REST API and custom resource definitions.
- **API Documentation:** https://docs.percona.com/everest/API.html
- **SDKs/Libraries:** No official SDK; REST API consumed directly · Kubernetes CRD-based management via kubectl · GitHub: https://github.com/percona/everest-operator
- **Developer Guide:** https://docs.percona.com/percona-everest/ · Blog: https://www.percona.com/blog/managing-postgresql-on-kubernetes-with-percona-everests-rest-api/
- **Standards:** REST/JSON; Kubernetes CRD API; Helm charts for deployment
- **Authentication:** Kubernetes RBAC; bearer token authentication for REST API

---

### pgBackRest

- **Description:** Open-source PostgreSQL backup and restore utility with WAL archiving, PITR, full/incremental/differential backups, and multi-cloud object storage support. CLI-based; no REST API.
- **API Documentation:** https://pgbackrest.org/command.html · https://pgbackrest.org/configuration.html
- **SDKs/Libraries:** No REST API or SDK; CLI binary only · GitHub: https://github.com/pgbackrest/pgbackrest
- **Developer Guide:** https://pgbackrest.org/user-guide.html · EDB integration docs: https://www.enterprisedb.com/docs/supported-open-source/pgbackrest/
- **Standards:** Integrates via PostgreSQL `archive_command` / `restore_command` protocol; S3/GCS/Azure Blob object storage; JSON output for repository info
- **Authentication:** Cloud provider IAM credentials (instance roles or access keys); SSH for remote repository access

---

### WAL-G

- **Description:** Lightweight open-source WAL archiving and base backup tool for PostgreSQL, MySQL, MongoDB, Redis, and etcd, designed for cloud object storage with parallel processing.
- **API Documentation:** https://wal-g.readthedocs.io/ · https://github.com/wal-g/wal-g
- **SDKs/Libraries:** No REST API; CLI binary only. Written in Go; source available for embedding · Kubernetes integration via environment variable configuration in pod specs
- **Developer Guide:** https://github.com/wal-g/wal-g/blob/master/README.md
- **Standards:** PostgreSQL `archive_command` / MySQL binlog / MongoDB oplog protocols; S3-compatible object storage API; multiple compression standards (LZ4, ZSTD, LZMA, Brotli)
- **Authentication:** Environment-variable-based AWS/GCS/Azure credentials; supports IAM instance profiles

---

### Barman / barman-cloud

- **Description:** Open-source PostgreSQL backup and recovery manager with WAL streaming via pg_receivewal. The `barman-cloud` CLI variant supports direct cloud object storage without a central Barman server.
- **API Documentation:** https://docs.pgbarman.org/release/3.17.0/user_guide/barman_cloud.html · Barman Cloud CNPG-I plugin API: https://cloudnative-pg.io/plugin-barman-cloud/docs/0.6.0/plugin-barman-cloud.v1/
- **SDKs/Libraries:** Python package (barman-cli-cloud); Boto3 for S3 integration · CNPG-I Kubernetes plugin: https://cloudnative-pg.io/plugin-barman-cloud/docs/usage/
- **Developer Guide:** https://docs.pgbarman.org/ · CNPG integration: https://cloudnative-pg.io/plugin-barman-cloud/
- **Standards:** PostgreSQL streaming replication protocol; S3/Azure Blob/GCS object storage; GPLv3 licence
- **Authentication:** Cloud provider credentials via environment variables or instance metadata

---

### Percona Backup for MongoDB (PBM)

- **Description:** Open-source backup and PITR tool for MongoDB deployments, supporting oplog-slice-based continuous archiving to S3-compatible object storage.
- **API Documentation:** https://docs.percona.com/percona-backup-mongodb/features/point-in-time-recovery.html · PITR options: https://docs.percona.com/percona-backup-mongodb/reference/pitr-options.html
- **SDKs/Libraries:** CLI tool (pbm); no official REST API or SDK · GitHub: https://github.com/percona/percona-backup-mongodb
- **Developer Guide:** https://docs.percona.com/percona-backup-mongodb/usage/pitr-tutorial.html
- **Standards:** MongoDB oplog protocol; S3-compatible object storage; compression via gzip/snappy/lz4/zstd
- **Authentication:** MongoDB URI authentication; cloud provider credentials for object storage

---

### Supabase Management API (Backup Operations)

- **Description:** Managed PostgreSQL platform with PITR on paid tiers. Exposes a Management API for programmatic access to platform operations including backup configuration.
- **API Documentation:** https://supabase.com/docs/reference/api/introduction
- **SDKs/Libraries:** Supabase JS SDK: https://supabase.com/docs/reference/javascript · Python, Go, Dart, Swift client libraries: https://supabase.com/docs/guides/api/rest/client-libs
- **Developer Guide:** https://supabase.com/docs/guides/platform/backups
- **Standards:** REST/JSON; PostgREST for data API; OAS 3.0-compatible schema
- **Authentication:** API keys (anon key, service role key); OAuth 2.0 for Management API

---

### Velero (Kubernetes Backup Standard)

- **Description:** Open-source Kubernetes backup and restore tool using CRDs to declaratively define backup, restore, and schedule operations. The closest thing to a Kubernetes-native backup standard.
- **API Documentation:** https://velero.io/docs/ · CRD specifications: https://github.com/vmware-tanzu/velero/tree/main/config/crd
- **SDKs/Libraries:** Velero Go client library: https://pkg.go.dev/github.com/vmware-tanzu/velero · CLI: https://velero.io/docs/main/basic-install/
- **Developer Guide:** https://velero.io/docs/v1.8/how-velero-works/
- **Standards:** Kubernetes CRD API; S3-compatible object storage (BackupStorageLocation); VolumeSnapshot API for PVC snapshots
- **Authentication:** Kubernetes RBAC; cloud provider credentials via Kubernetes secrets

---

## Notes

**Absence of a unified inter-tool standard:** There is no published IETF, ISO, or W3C standard defining a vendor-neutral protocol for database backup metadata exchange, cross-platform restore orchestration, or backup catalogue discovery. Each tool (pgBackRest, WAL-G, Barman, PBM) uses its own proprietary metadata format and CLI conventions. This is a genuine gap: an open schema for backup catalogue metadata (backup type, timestamps, WAL ranges, compression, encryption parameters) would enable cross-tool interoperability and is a candidate differentiator for an AI-native platform.

**Emerging Kubernetes backup standardisation:** Velero is actively pursuing recognition as the Kubernetes backup standard via its CRD model. A new platform should monitor this effort and consider CRD compatibility as a first-class design constraint.

**Encryption key management:** None of the open-source tools reviewed provide built-in key management service (KMS) integration beyond environment-variable credentials. Integrating with AWS KMS, Google Cloud KMS, or HashiCorp Vault for backup encryption key lifecycle management is a gap in the current ecosystem.

**OpenTelemetry adoption is nascent:** No reviewed tool emits OTel-compatible metrics natively. Providing built-in OTel instrumentation (backup duration, WAL replication lag, restore latency, storage consumption by tier) would be a meaningful differentiator for platform-engineering teams operating modern observability stacks.
