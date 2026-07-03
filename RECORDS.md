# Records, Audit & Backups

What is recorded, where it lives, how long we keep it, and how it comes
back. Auditors ask these four questions in this order; keep the answers in
this file current.

## Audit trail

- **Mechanism**: the append-only, hash-chained recorder in `hanzoai/cloud`
  (`audit/`, `audit_middleware.go`), wired into every subsystem through
  `Deps.Audit` and read back at `/v1/admin/audit`. Tamper-evidence comes
  from the hash chain — deletion or mutation breaks verification.
- **Coverage**: admin actions, authn events (IAM), per-subsystem writes.
  Every new `cloud` subsystem must emit to the recorder (standard §5 in
  hanzoai/security `STANDARDS.md`) — coverage grows with the platform by
  construction, not by checklist.
- **Retention target**: 1 year hot / 6 years cold (SOC 2 needs the
  observation window; HIPAA §164.316(b)(2) requires 6 years for policies
  and records; FedRAMP AU-11 ≥ 90 days online + 1 year total — 6y cold
  covers all three).

## Records retention

| Record class | System of record | Retention |
|---|---|---|
| Audit events | cloud audit store → S3 cold tier | 1y hot / 6y cold |
| Access & authn logs | IAM + o11y stack | 1y |
| Build & deploy provenance | hanzoai/ci runs, registry.hanzo.ai manifests, Git history | life of the artifact |
| Billing/metering | `Deps.Metering` store | 7y (tax) |
| Customer content | per-tenant `DataDir` stores / Postgres | customer-controlled; delete on offboarding per contract |
| Policies & compliance docs | this repo + hanzoai/security (Git) | permanent (versioned) |

Git is the system of record for anything that is a document: history is the
retention mechanism, and the private repos are the boundary.

## Backups

- **Object storage** (registry images, audit cold tier, artifacts): S3
  (hanzoai/s3) with versioning; cross-region replication for the gov/regulated
  profiles.
- **Per-tenant embedded stores** (`DataDir` SQLite/ZapDB): nightly snapshot
  to S3 with per-tenant prefixes — restore granularity is a single tenant,
  which is also the offboarding-deletion granularity.
- **PostgreSQL** (production multi-instance): WAL archiving + nightly base
  backup; point-in-time recovery.
- **KMS**: root keys escrowed offline (sealed, split custody); everything
  else is derivable or restorable from S3 + Git + KMS, in that dependency
  order.

## Restore discipline

A backup that has never been restored is a hope, not a control. Quarterly:
restore one tenant store and one Postgres PITR into a scratch namespace via
platform.hanzo.ai, verify app-level reads, record the drill (time-to-restore,
gaps) — the drill record is itself audit evidence (CP-9/CP-10, SOC 2 A1.2).

## Evidence automation (pre-work for every framework)

The observation window should collect itself: a small exporter that snapshots
control evidence (IAM config, audit-chain verification result, backup drill
records, CI provenance) into this repo on a schedule. This is the cheapest
engineering investment with the highest audit leverage, and it is the same
plumbing ConMon needs later.
