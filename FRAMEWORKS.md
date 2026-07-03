# Framework Positions

Status vocabulary: **held** (certificate/authorization in hand) ·
**in progress** (engagement underway) · **aligned** (controls exist, not yet
assessed) · **gap** (work required). Nothing here is *held* today — Hanzo has
no completed third-party attestation yet. That is the first thing to fix and
the reason this file exists.

## SOC 2 (Type I → Type II)

**Status: aligned (unassessed).** The control substrate exists:

- CC6 (logical access): Hanzo IAM everywhere, gateway-minted org scoping,
  no local auth, seeded-superuser model — no shared/built-in admins.
- CC7 (system operations): tamper-evident audit trail in `cloud/audit/`
  (hash-chained, append-only, `/v1/admin/audit`), o11y stack via
  platform.hanzo.ai.
- CC8 (change management): single CI path (`hanzoai/ci`), GitOps deploys via
  operator/platform — no manual production access; every change is a commit.

**Path**: pick auditor → 3-month observation window → Type I, then Type II
over the following period. Engineering pre-work: finish the evidence
automation in `RECORDS.md` so the observation window collects itself.

## FedRAMP (federal)

**Status: gap, with real foundations.** The audit recorder is already
annotated to AU-2/AU-9; KMS/IAM/registry map cleanly onto AC/IA/CM/SC
families. What FedRAMP additionally demands:

- FIPS 140-3 validated crypto modules in the data path (see PQC section —
  we solve both at once).
- A dedicated authorization boundary: a `gov` deployment of the cloud stack
  (platform.hanzo.ai profile) with US-persons ops and its own KMS root.
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
  (§164.312(b) maps directly onto `cloud/audit/`), TLS 1.3 transport,
  encrypted storage, Guard redacting PII/PHI from LLM traffic — the
  differentiator for AI workloads: PHI does not leak into prompts or logs.
- Required work: BAA template (counsel), documented risk analysis
  (§164.308(a)(1)), breach-notification runbook, workforce training records,
  and a PHI data-flow inventory per product so we can answer "where does
  PHI live" in one page.
- Sub-processor chain: BAAs with any upstream model/API providers a
  deployment uses, or the deployment pins to self-hosted models only
  (the Zen engine path) — that "no PHI leaves the boundary" story is the
  strongest healthcare answer we have.

## Post-quantum — CNSA 2.0 / FIPS 203-205

**Status: aligned in direction, gap in the data path.** Federal timelines
(CNSA 2.0) require PQ readiness for new national-security systems now and
broadly by 2030-2033. Position:

- **Signatures**: ML-DSA (FIPS 204) for release/image signing in the
  registry and CI provenance — first concrete PQ deliverable, no protocol
  changes required.
- **Key establishment**: hybrid X25519+ML-KEM-768 (FIPS 203) at the
  gateway/ingress TLS edge as library support lands; hybrid is the
  defensible interim answer to "harvest-now-decrypt-later".
- **KMS**: root-of-trust algorithm agility — KMS must be able to wrap keys
  with ML-KEM and sign with ML-DSA before any gov boundary ships; FIPS
  140-3 module selection happens here and satisfies the FedRAMP crypto
  requirement in the same move.
- SLH-DSA (FIPS 205) reserved for firmware/long-horizon signing; not on the
  critical path.

## The standing answer to "are you compliant?"

Until attestations are held: *"Hanzo is built on a single audited control
substrate — centralized KMS, IAM SSO, hash-chained audit logging, GitOps
change control, isolated builds — designed to align with SOC 2 and NIST
800-53. SOC 2 attestation is in progress; FedRAMP/CMMC roadmaps available
under NDA; HIPAA deployments supported with a BAA."* Every clause of that
sentence must remain literally true against `POSTURE.md`.
