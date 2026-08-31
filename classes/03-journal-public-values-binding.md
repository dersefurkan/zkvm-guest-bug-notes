# 3. Journal / public-values binding gap

**Description.** The journal (RISC Zero) or public values (SP1) are the only channel through which
a guest's output reaches the onchain verifier. A binding gap exists when the proof is valid but the
values the application acts on are not fully committed to the claim being verified — fields
consumed from calldata instead of the journal, digest comparisons that check only the first word,
or batch commitments built from order-insensitive (commutative) hashes.

**Why it matters.** This is the highest-impact class at the application boundary: the cryptography
is fine, the guest ran correctly, and yet an attacker swaps *context*. In the Boundless market
audit, an unverified `RequestId`↔`requestDigest` link let a PoC swap two IDs in a batch and charge
another party's account; a commutative batch root let an attacker permute `imageId`/`journal`
fields and deliver a bogus fulfillment. In recursive verifiers, checking only the first word of a
`deferred_proofs_digest` leaves the remaining words free for the prover to choose.

**Detection hints.**

- For every value the onchain contract consumes, trace it into the proof's commitment: is the
  journal hash part of the verified claim (RISC Zero) or the public-values digest (SP1)? Is any
  security-relevant field taken from calldata instead?
- Grep the verifier for comparison patterns: `for` loops over digest words versus single-element
  checks (`digest[0]`, `first_word`), early-exit comparisons, truncated byte comparisons.
- In batch/aggregation structures, ask whether leaf *order* is meaningful; if yes, the commitment
  scheme must be order-sensitive (commutative hashes are not).
- Invariant to test: changing any bit of any field the application consumes must change the claim
  being verified — write the test that flips each field and expects verification to fail.
- Boundary to diff: the guest's journal encoding (Rust struct → bytes) versus the contract's
  decoding (`abi.decode` offsets). Length-prefix, padding, and endianness mismatches are binding
  bugs too.

**Public references.**

- [Veridise — Boundless Market Beta audit (Apr 2025)](https://github.com/risc0/rz-security/blob/main/audits/boundless/veridise-boundless-250404.pdf) —
  V-RISC0-VUL-001 (Critical: `RequestId` not validated against `requestDigest` in fulfillment
  functions), V-RISC0-VUL-002 (High: commutative batch root permits permuting
  `imageId`/`journal` leaves).
- [Veridise — SP1 audit](https://github.com/succinctlabs/sp1/blob/main/audits/veridise.pdf) —
  V-SP1-VUL-004 (High): recursive verifier checks only the first word of `deferred_proofs_digest`
  across shards.
- [rkm0959 — SP1 V3 audit](https://github.com/succinctlabs/sp1/blob/main/audits/rkm0959.md) —
  finding #7: exit code checked on the first shard only instead of "0 on all shards".
- [Code4rena — Succinct audit (Sep 2025)](https://code4rena.com/reports/2025-09-succinct) —
  public-values handling findings, including truncated `public_values` behavior.
- [zkSecurity — R0VM Helios audit](https://reports.zksecurity.xyz/reports/risc0-helios/) — the
  positive pattern: genesis root and fork IDs are passed into the guest so the committed output is
  bound to the correct chain context.
