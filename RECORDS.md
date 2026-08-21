# Records, Audit & Backups

What is recorded, where it lives, how long we keep it, and how it comes
back. Auditors ask these four questions in this order; keep the answers in
this file current. Each row is tagged **[implemented]** (citable now) or
**[target]** (prescribed, not yet built) — do not let a target masquerade as a
control.

## Audit trail

- **Mechanism [implemented]**: the append-only, hash-chained recorder in
  `hanzoai/cloud` (`audit/record.go` seals `SHA-256(canonical-JSON ‖ prevHash)`
  from a genesis link, under lock; `audit/store.go` exposes `Head() → (count,
  headHash)` for tail-truncation detection and mirrors to a ClickHouse OLAP
  tier). Wired into every subsystem through the `AuditTrail` middleware
  (`audit_middleware.go`) and read back at `/v1/admin/audit`. Tamper-evidence
  comes from the hash chain — deletion or mutation breaks `Verify`.
- **Coverage [implemented]**: the coverage predicate lives in one place
  (`audit_middleware.go` `isSecurityRelevant`) — every mutating request, every
  `/v1/admin/*` request (including reads), and every 401/403 denial; the actor
  is taken from the validated identity, never a client header. **AU-5**: an
  audit-store write failure fails the mutation closed (503, read-only) rather
  than mutating unaudited. Coverage grows with the platform by construction.
- **Retention [target]**: 1 year hot / 6 years cold (SOC 2 needs the
  observation window; HIPAA §164.316(b)(2) requires 6 years; FedRAMP AU-11
  ≥ 90 days online + 1 year total — 6y cold covers all three). **Not yet
  enforced**: the S3 cold-tier lifecycle and WORM / object-lock immutability
  are on the remediation list (POSTURE "What's LEFT" #3). Today the hot
  ClickHouse mirror is the durable copy.

## Records retention

| Record class | System of record | Retention | Status |
|---|---|---|---|
| Audit events | cloud audit store → ClickHouse; S3 cold tier | 1y hot / 6y cold | hot [implemented]; cold-tier lifecycle + WORM [target] |
| Access & authn logs | IAM (`object/record.go`) + o11y stack | 1y | [implemented] |
| Build & deploy provenance | hanzoai/ci runs, oci.hanzo.ai manifests, Git history | life of the artifact | [implemented] |
| Billing/metering | `Deps.Metering` store | 7y (tax) | [implemented] |
| Customer content | per-tenant `DataDir` stores / Postgres | customer-controlled; delete on offboarding per contract | [implemented] |
| Policies & compliance docs | this repo + hanzoai/security (Git) | permanent (versioned) | [implemented] |

Git is the system of record for anything that is a document: history is the
retention mechanism, and the private repos are the boundary.

## Backups

- **Per-tenant embedded stores [implemented, HA path]**: each org's SQLite is
  snapshotted (WAL-checkpointed) and replicated to the VFS/SeaweedFS object
  store by the `Replicator` (`cloud/internal/org/replica.go`); snapshots are
  encrypted per-org with an AES-256-GCM key derived from the KMS master
  (`cloud/internal/org/cipher.go`), AAD-bound to the orgID. Restore granularity
  is a single tenant — which is also the offboarding-deletion granularity.
- **Scheduled nightly snapshot to S3 with per-tenant prefixes [target]**: the
  replica mechanism provides the snapshot/restore primitives; a *scheduled*
  backup job with retained history is not yet wired (POSTURE "What's LEFT" #4).
- **Object storage** (registry images, audit cold tier, artifacts): S3
  (hanzoai/s3) with versioning; cross-region replication for the gov/regulated
  profiles [target for the gov profile].
- **PostgreSQL** (production multi-instance): WAL archiving + nightly base
  backup; point-in-time recovery [implemented where Postgres is deployed].
- **KMS**: root keys escrowed offline (sealed, split custody); everything
  else is derivable or restorable from S3 + Git + KMS, in that dependency
  order.

## Restore discipline

A backup that has never been restored is a hope, not a control. **[target]**
Quarterly: restore one tenant store and one Postgres PITR into a scratch
namespace via platform.hanzo.ai, verify app-level reads, record the drill
(time-to-restore, gaps) — the drill record is itself audit evidence
(CP-9/CP-10, SOC 2 A1.2). No drill has been run and recorded yet; standing up
this cadence + a runbook is POSTURE "What's LEFT" #4.

## Evidence automation (pre-work for every framework) — [target]

The observation window should collect itself: a small exporter that snapshots
control evidence (IAM config, audit-chain `Verify` result, backup drill
records, CI provenance) into this repo on a schedule. This is the cheapest
engineering investment with the highest audit leverage, and it is the same
plumbing ConMon needs later (POSTURE "What's LEFT" #6).
