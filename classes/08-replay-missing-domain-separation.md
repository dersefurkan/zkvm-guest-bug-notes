# 8. Replay & missing domain separation at verification

**Description.** A proof, receipt, or signature that is valid in one context must not be valid in
another. The failures are mechanical: no nonce or uniqueness tag in the journal, the same typed
digest signed by two different roles, zero/sentinel digests accepted as real commitments, and
accumulator verifiers that cannot tell a leaf from an intermediate node.

**Why it matters.** In the Boundless market, the absence of domain separation between client and
prover signatures let a `clientSignature` be passed as a `proverSignature` (self-locking griefing),
and the set verifier accepted an intermediate MMR node — even the root — as if it were a proven
`claimDigest`. In Steel, a crafted commitment with a zero digest was accepted as valid even though
a zero digest corresponds to no block at all. Each of these is a valid proof or signature doing
work it was never bound to do.

**Detection hints.**

- Ask of every accepted proof/signature: what stops it from being replayed — across requests,
  across chains, across contract upgrades, across roles? Look for nonces, request IDs, chain IDs,
  and consumed/nullifier tracking.
- Sentinel hygiene: wherever `bytes32(0)` / `Digest::ZERO` can appear, is it explicitly rejected
  where a real commitment is required? Is the same zero value overloaded for "none", "empty", and
  "valid" (a collision of meanings)?
- Accumulator/set verifiers: are leaves tagged or hashed differently from internal nodes? Try
  submitting an intermediate node (or the root) with a partial path.
- If two roles sign the same message shape, test cross-role replay directly.

**Public references.**

- [Veridise — Boundless Market Beta audit](https://github.com/risc0/rz-security/blob/main/audits/boundless/veridise-boundless-250404.pdf) —
  V-RISC0-VUL-004 (Medium): `RiscZeroSetVerifier` does not distinguish MMR leaves from intermediate
  nodes; V-RISC0-VUL-005 (Medium): no domain separation between client and prover signatures.
- [GHSA-gjv3-89hh-9xq2 / CVE-2025-52884](https://github.com/advisories/GHSA-gjv3-89hh-9xq2) —
  `Steel.validateCommitment` returned `true` for a crafted commitment whose digest is zero.
- [Veridise — RISC Zero zkVM Round 2](https://github.com/risc0/rz-security/blob/main/audits/zkVM/veridise_zkVM_20250224.pdf) —
  V-RISC0-VUL-023: `Digest::ZERO` simultaneously encodes `None`, empty arrays, and empty tagged
  iterations; `tagged_iter` produces the same digest regardless of tag.
- [Veridise — Steel audits](https://github.com/risc0/rz-security/blob/main/audits/steel/veridise_steel_250414.pdf) —
  commitment validation in the Steel coprocessor library (see also the
  [Steel README](https://github.com/risc0/risc0-ethereum/blob/main/crates/steel/README.md)).
