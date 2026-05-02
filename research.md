# Database Backup & PITR Platform

> Candidate #294 · Researched: 2026-05-02

## Existing Products and Software Packages

| Tool | Description | Type | Pricing | Strengths / Weaknesses |
|------|-------------|------|---------|------------------------|
| AWS Backup | Centralised backup service across AWS resources with PITR at 1-second precision for up to 35 days | Commercial SaaS | Pay-per-GB stored + restore requests | Deep AWS integration; tightly coupled to AWS ecosystem |
| Google Cloud Spanner PITR | Native PITR for Spanner with up to 7-day retention | Commercial SaaS | Included in Spanner pricing | Low ops overhead; limited to Spanner; 7-day window may be insufficient |
| pgBackRest | PostgreSQL-native backup and WAL archiving tool supporting full, incremental, and differential backups with PITR | Open source | Free | Highly configurable; complex to operate; Postgres-only |
| WAL-G | Lightweight Postgres/MySQL/MongoDB WAL archiver and base backup tool designed for cloud object storage | Open source | Free | Fast, cloud-native; minimal UI; requires operational expertise |
| Percona Everest | Kubernetes-native database operations platform with PITR via WAL log replay for MySQL, MongoDB, and PostgreSQL | Open source / Commercial | Free community; paid support | Multi-database; Kubernetes-first; steep learning curve |
| PlanetScale Postgres | Managed Postgres service with PITR to any point within the retention window | Commercial SaaS | Usage-based | Frictionless restore UX; vendor lock-in; limited retention control |
| Supabase | Managed Postgres platform with automated backups and PITR on higher tiers | Commercial SaaS | Free tier; Pro with PITR from $25/mo | Developer-friendly; PITR limited to Pro+ tiers |
| Litestream | Continuous SQLite replication to cloud object storage for near-PITR recovery | Open source | Free | Solves SQLite replication elegantly; SQLite only |
| Barman (pgBarman) | PostgreSQL Backup and Recovery Manager with WAL streaming and PITR | Open source | Free | Battle-tested; Postgres-only; CLI-focused, no modern UI |

## Relevant Industry Standards or Protocols

- **WAL (Write-Ahead Log) Archiving** — the foundational mechanism for continuous Postgres (and similar) PITR; events are streamed to durable storage and replayed to a target timestamp on restore
- **Recovery Point Objective (RPO) / Recovery Time Objective (RTO)** — industry-standard SLA metrics that drive backup frequency and infrastructure design decisions
- **AWS S3 / GCS / Azure Blob Storage APIs** — de-facto object storage targets for backup archives; all major open-source tools support these
- **GDPR / SOC 2 / HIPAA** — data protection regulations that mandate backup retention minimums, encryption at rest and in transit, and access audit logging for backup operations

## Available Research Materials

1. AWS Docs (2026). *Continuous backups and point-in-time recovery (PITR) – AWS Backup*. https://docs.aws.amazon.com/aws-backup/latest/devguide/point-in-time-recovery.html
2. PostgreSQL Global Development Group (2026). *25.3. Continuous Archiving and Point-in-Time Recovery (PITR)*. PostgreSQL Documentation 18. https://www.postgresql.org/docs/current/continuous-archiving.html
3. Pawale (2026). *Why you don't need PITR and incremental backups for most PostgreSQL databases in 2026*. Medium. https://medium.com/@pawale7663/why-you-dont-need-pitr-and-incremental-backups-for-most-postgresql-databases-in-2026-b2a1f3ec6833
4. Percona Docs (2026). *Percona Everest – Enable Point-in-time recovery (PITR)*. https://docs.percona.com/everest/backups_and_restore/createBackups/EnablePITR.html
5. PlanetScale Docs (2025). *Point-in-time recovery – PlanetScale*. https://planetscale.com/docs/postgres/backups/point-in-time-recovery
6. Google Cloud Docs (2026). *Point-in-time recovery (PITR) overview – Spanner*. https://docs.cloud.google.com/spanner/docs/pitr
7. Adyson, P. (2026). *MySQL backup and restore — Complete guide to MySQL database backup strategies in 2026*. DEV Community. https://dev.to/piteradyson/mysql-backup-and-restore-complete-guide-to-mysql-database-backup-strategies-in-2026-4cdk

## Market Research

**Market Size:** The database backup and recovery market is estimated at over $5 billion globally in 2025, growing at approximately 10–13% CAGR. The shift to cloud-managed databases is reducing demand for self-managed backup tools while increasing demand for cross-cloud and multi-database backup orchestration.

**Funding:** Percona is private-equity backed. Most pure-play backup tools (pgBackRest, WAL-G, Barman, Litestream) are open-source community projects. Cloud vendor offerings are bundled products. There is limited venture investment in standalone database backup SaaS.

**Pricing Landscape:** Cloud-native services bundle backup costs into database pricing or charge per GB stored (typically $0.02–$0.10/GB/month). Self-hosted open-source tools are free but require significant operational investment. Managed offerings like Supabase include PITR from $25/month.

**Key Buyer Personas:** Database administrators and platform engineers managing self-hosted or multi-cloud databases; startup CTOs seeking automated backup without operational overhead; regulated-industry engineers needing auditable recovery evidence for compliance.

**Notable Trends:** Managed cloud databases are absorbing the PITR problem, narrowing the market for standalone backup tooling. The remaining opportunity lies in multi-cloud and multi-database backup orchestration, unified restore UX, and compliance-focused audit trails. Kubernetes-native databases (via operators) are creating new demand for operator-aware backup tooling.

## AI-Native Opportunity

- Automated RPO/RTO optimisation: analyse query patterns and data-change velocity to recommend backup schedules that minimise both cost and recovery time without manual tuning
- Anomaly-based backup triggering: detect unusual write patterns (potential ransomware, bulk deletes, schema corruption) and trigger an out-of-schedule snapshot automatically
- AI-assisted restore: accept natural-language restore requests ("restore to just before the accidental orders table truncation yesterday afternoon") and translate to a precise WAL replay timestamp
- Intelligent backup validation: continuously test restore integrity by replaying recent WAL segments to a shadow instance and running automated data-quality checks without human intervention
- Cost forecasting: predict backup storage costs across retention policies and object storage tiers, recommending the optimal retention window for a given budget and compliance requirement
