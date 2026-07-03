# Hanzo Compliance

Internal. How Hanzo answers regulated-industry requirements — federal, state,
defense, healthcare — so anyone can build on Hanzo safely. Engineering ground
truth lives in [hanzoai/security](https://github.com/hanzoai/security)
(`POSTURE.md` / `STANDARDS.md`); this repo maps that truth onto the
frameworks auditors and customers ask about, and tracks what each framework
still requires of us.

- [`FRAMEWORKS.md`](./FRAMEWORKS.md) — per-framework position and path:
  SOC 2, FedRAMP, StateRAMP, CMMC / DoD impact levels, HIPAA/HITECH, and the
  post-quantum (CNSA 2.0 / FIPS 203-205) transition.
- [`RECORDS.md`](./RECORDS.md) — audit trail, records retention, and backup
  posture: what is recorded, where it lives, how long it is kept, how it is
  restored.

Rule of use: never claim a certification or authorization in a customer
answer that is not listed as **held** in `FRAMEWORKS.md`. "In progress" and
"designed to align" are the honest phrasings until then.
