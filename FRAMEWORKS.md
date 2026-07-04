# Framework Positions

Status vocabulary: **held** (certificate/authorization in hand) ·
**readiness** (engagement being prepared, no auditor engaged yet) ·
**aligned** (controls exist, not yet assessed) · **gap** (work required).
Nothing here is *held* today — Hanzo has no completed third-party attestation
and no auditor is currently engaged. That is the first thing to fix and the
reason this file exists. The evidenced control inventory and the single
prioritized remediation list live in hanzoai/security `POSTURE.md`
("Compliance control-family posture" + "What's LEFT"); this file maps that
truth onto the frameworks and does not duplicate the list.

## SOC 2 (Type I → Type II)

**Status: aligned (unassessed).** The control substrate exists and is
evidenced in `POSTURE.md`:

- CC6 (logical access): Hanzo IAM everywhere; the one SuperAdmin predicate
  `owner == AdminOrg` enforced at a single forward-auth gate
  (`gateway/cmd/admin-guard`); spoof-proof identity
  (`cloud/middleware_identity.go` `SanitizeIdentity`); provision-don't-promote
  regression-pinned (`iam/object/privesc_security_test.go`); no shared/built-in
  admins. **MFA methods are shipped but not enforced** — the top CC6 gap.
- CC7 (system operations): tamper-evident hash-chained audit trail
  (`cloud/audit/`, `/v1/admin/audit`) with a fail-closed coverage predicate
  (`cloud/audit_middleware.go`), o11y via platform.hanzo.ai.
- CC8 (change management): single CI path (`hanzoai/ci`), GitOps deploys via
  operator/platform — no manual production access; every change is a commit.

**Path**: engage an auditor → 3-month observation window → Type I, then
Type II over the following period. Engineering pre-work: the eight-item SOC 2
critical path in `POSTURE.md` "What's LEFT" (MFA enforcement, mesh mTLS,
retention lifecycle, backup/restore drills, access review, evidence
automation, pen test) — finish evidence automation so the observation window
collects itself. **No auditor is engaged yet; do not describe the audit as
"in progress."**

## FedRAMP (federal)

**Status: gap, with real foundations.** The audit recorder is annotated to
AU-2/AU-9; KMS/IAM/registry map cleanly onto AC/IA/CM/SC families; hybrid PQ is
partly in the data path already (see PQC section). What FedRAMP additionally
demands:

- **FIPS 140-3 validated crypto modules** in the data path. Hanzo already runs
  hybrid PQ (ML-KEM-768 + X25519 at the ZAP edge; ML-DSA-65 hybrid signing in
  KMS) but on non-validated modules (`luxfi/pq`); FedRAMP requires the
  *validated* module. This is now a validation/packaging task, not a
  build-the-crypto task.
- A dedicated authorization boundary: a `gov` deployment of the cloud stack
  (platform.hanzo.ai profile) with US-persons ops and its own KMS root.
- **Mesh mTLS is mandatory (SC-8).** The in-cluster ZAP :9653 plane is
  plaintext today (`cloud/serve.go:200`); FedRAMP will not accept
  network-isolation-only for east-west.
- SSP + continuous monitoring (ConMon) — POA&M discipline, monthly scans.
  The native security module (security repo, plan of record) is the scanner
  substrate for ConMon; without it we would be buying a third-party scanner
  anyway.
- Route: FedRAMP 20x / agency sponsorship at Moderate first; High only with
  a sponsoring agency in hand.

## StateRAMP (state & local)

**Status: follows FedRAMP.** StateRAMP accepts FedRAMP artifacts with light
deltas; do not run a separate program. Answer state RFPs today with the
SOC 2 position plus the FedRAMP roadmap.

## DoD / military — CMMC 2.0 & impact levels

**Status: gap.** Two distinct asks arrive under "military":

- **CMMC Level 2** (contractors handling CUI): 110 controls of NIST
  800-171. Most map to existing standards (KMS, IAM, audit, build
  isolation); the deltas are documented incident response, media
  sanitization, and personnel/physical controls — organizational, not code.
- **IL4/IL5 hosting**: requires the gov boundary above deployed on approved
  infrastructure. Sequence strictly after FedRAMP Moderate.

## Healthcare — HIPAA / HITECH

**Status: aligned (unassessed), shortest path to revenue.** HIPAA has no
certification — it is a posture plus a signed BAA:

- Technical safeguards mostly exist: IAM access control, audit trail
  (§164.312(b) maps directly onto `cloud/audit/`), TLS 1.3 transport at the
  edge, encrypted storage (per-tenant AES-256-GCM / SQLCipher / PQ age),
  Guard redacting PII/PHI from LLM traffic — the differentiator for AI
  workloads: PHI does not leak into prompts or logs. Note the internal
  plaintext-ZAP gap (SC-8) applies to §164.312(e) transmission security and is
  on the remediation list.
- Required work: BAA template (counsel), documented risk analysis
  (§164.308(a)(1)), breach-notification runbook, workforce training records,
  and a PHI data-flow inventory per product so we can answer "where does
  PHI live" in one page.
- Sub-processor chain: BAAs with any upstream model/API providers a
  deployment uses, or the deployment pins to self-hosted models only
  (the Zen engine path) — that "no PHI leaves the boundary" story is the
  strongest healthcare answer we have.

## Post-quantum — CNSA 2.0 / FIPS 203-205

**Status: partially implemented in the data path; validation + edge-coverage
gaps remain.** Federal timelines (CNSA 2.0) require PQ readiness for new
national-security systems now and broadly by 2030-2033. What is already live:

- **Key establishment (IMPLEMENTED):** hybrid **X25519 + ML-KEM-768** at the
  gateway inbound ZAP TLS listener (`gateway/zap_listener.go:51-55`,
  `tls.X25519MLKEM768`). This is the concrete answer to
  harvest-now-decrypt-later at that edge, live today — no longer roadmap.
- **KMS key wrapping (IMPLEMENTED):** content-encryption keys wrapped with
  ML-KEM-768 + X25519 hybrid HPKE (`kms/sdk/go/cek.go`, `members.go`).
- **Signatures (IMPLEMENTED in KMS):** secp256k1 + **ML-DSA-65** hybrid signing
  (`kms/go.mod`, `luxfi/keys`). Remaining: wire ML-DSA into release/image
  signing in `registry` + CI provenance — the first *product-facing* PQ
  deliverable, no protocol changes required.
- **Object encryption at rest (IMPLEMENTED):** VFS uses the PQ-capable
  `luxfi/age` (`vfs`, `luxfi/pq`).
- **Remaining gaps:** (1) extend hybrid `X25519MLKEM768` from the ZAP edge to
  the browser/API HTTPS ingress; (2) adopt a **FIPS 140-3 validated** ML-KEM /
  ML-DSA module (this also satisfies the FedRAMP crypto requirement in the same
  move); (3) make the internal ZAP transport hybrid once mesh mTLS lands.
- SLH-DSA (FIPS 205) reserved for firmware/long-horizon signing; not on the
  critical path.

## The standing answer to "are you compliant?"

Until attestations are held: *"Hanzo is built on a single audited control
substrate — centralized KMS, IAM SSO with a single SuperAdmin predicate,
hash-chained tamper-evident audit logging, per-tenant encryption at rest, and
GitOps change control — designed to align with SOC 2 and NIST 800-53, and
already running hybrid post-quantum key establishment and signing (ML-KEM-768 /
ML-DSA-65) in the data path. SOC 2 readiness is underway (controls aligned;
formal attestation not yet engaged); FedRAMP/CMMC roadmaps available under NDA;
HIPAA deployments supported with a BAA."* Every clause of that sentence must
remain literally true against `POSTURE.md` — in particular, do **not** claim a
SOC 2 audit is "in progress" until an auditor is engaged, and do **not** imply
MFA is enforced until the admin-org enforcement gap is closed.
