# 11. Native ↔ recursive verifier divergence

**Description.** A zkVM stack ships more than one verifier: the native (Rust) verifier, the
recursive in-circuit verifier that compresses shard proofs, and the STARK→SNARK wrapper your
on-chain contract ultimately trusts. Each re-implements the acceptance criteria — exit codes,
chip ordering, row counts, cumulative sums, verification-key roots. Where the recursive or
wrapper implementation constrains *less* than the native one checks, a proof can be **valid for
the on-chain path and invalid for the native verifier** — up to universal forgery of compressed
proofs.

**Why it matters.** Integrators consume the wrapped proof, not the native one. A divergence means
the contract's security is defined by whichever check the *weaker* verifier skipped — and the
gap is invisible from the guest program's side. For reviewers this is a diff problem: two
implementations of the same policy, written in different languages (Rust vs. circuit IR vs. Go /
gnark), that must agree exactly.

**Detection hints.**

- Enumerate every acceptance check in the native verifier, then find where each lives in the
  recursive verifier and the wrapper circuit. Anything missing on one side is finding-shaped:
  `exit_code` handling, `is_complete`-style flags, `vk_root` binding, chip/row-count consistency
  between commitment and evaluation sides.
- Boundary to diff: the prover-facing structs that cross into the circuit (shard shapes, Merkle
  paths, deferred digests). Ask "what constrains this field in-circuit?" for each one.
- Test method: differential acceptance — craft edge-case proofs (panicking guest, empty or
  maximal shards, reordered chips) and feed them to both verifiers; they must accept and reject
  identically.
- Watch fix neighborhoods: when an advisory lands, the diff that closes it marks the exact
  invariant family that was previously unenforced — adjacent rows of the same table often share
  the gap.

**Public references.**

- [LambdaClass / 3MI Labs / Aligned responsible disclosure](https://blog.lambdaclass.com/responsible-disclosure-of-an-exploit-in-succincts-sp1-zkvm-found-in-partnership-with-3mi-labs-and-aligned-which-arises-from-the-interaction-of-two-distinct-security-vulnerabilities/)
  ([PoC](https://github.com/lambdaclass/PoC-exploit-SP1),
  [Succinct security update](https://blog.succinct.xyz/sp1-security-update-1-27-25/)) — two
  combined bugs produced universal forgery of SP1 compressed proofs; the recursive verifier
  accepted what the native path rejected.
- [GHSA-c873-wfhp-wx5m](https://github.com/succinctlabs/sp1/security/advisories/GHSA-c873-wfhp-wx5m) —
  missing verifier checks: STARK `chip_ordering` validation absent, `is_complete` flag
  underconstrained in recursion, Fiat-Shamir observation gaps.
- [GHSA-63x8-x938-vx33 / CVE-2026-40323](https://github.com/succinctlabs/sp1/security/advisories/GHSA-63x8-x938-vx33) —
  recursion jagged PCS verifier: commitment-side row counts not bound to evaluation-side prefix
  sums; the native verifier was unaffected. Found via the SP1 bug bounty, fixed in v6.1.0.
- [GHSA-6248-228x-mmvh](https://github.com/succinctlabs/sp1/security/advisories/GHSA-6248-228x-mmvh) —
  Rust verifier `verify_compressed` / `verify_shrink` / `verify_deferred_proof` missing `vk_root`
  checks (Zellic), plus the embedded-allocator hazards fixed alongside.
