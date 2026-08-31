# 9. Upgrade-path ELF↔vk mismatch

**Description.** Rebuilding or changing a guest program changes its identity — the ELF changes, so
the image ID / program verification key changes. An upgrade path is safe only if the onchain
allowlist rotates *with* the guest and deprecated verifier versions are retired; otherwise proofs
of the old program keep passing after the logic has moved on, or a verifier with a known flaw stays
callable through a router.

**Why it matters.** This class has bitten in production twice in public: RISC Zero's official
verifier router had to explicitly disable the vulnerable 2.0 verifier after the rv32im
underconstraint advisory, and chains running SP1 verifiers pushed emergency verifier upgrades (v4 →
v5) when the underlying Plonky3 vulnerability was disclosed — L2BEAT's upgrade history for Kroma
shows the rotation happening as a governance-visible event. Application-side, the quieter failure
is the stale allowlist entry: the guest was fixed, the old key was never removed, and the old
semantics remain provable.

**Detection hints.**

- Inventory every place a program key lives: contract immutables, allowlist mappings, router
  registrations, off-chain verifier configs. For each, ask: what is the rotation procedure, and who
  can trigger it?
- After any guest upgrade, diff the deployed ELF/image ID against every pinned key — build-system
  nondeterminism alone causes mismatches (this is also an availability bug: valid proofs rejected).
- Router/selector architectures: are old verifier versions tombstoned (calls revert), or merely
  deprecated in documentation?
- Invariant to test: after an upgrade, proofs generated under the previous guest version must fail
  verification everywhere the new semantics are live.

**Public references.**

- [GHSA-g3qg-6746-3mg9 / CVE-2025-52484](https://github.com/advisories/GHSA-g3qg-6746-3mg9) —
  rv32im underconstraint; remediation included disabling the affected verifier generation in the
  official [risc0-ethereum](https://github.com/risc0/risc0-ethereum) verifier router.
- [L2BEAT — Kroma project page](https://l2beat.com/scaling/projects/kroma) — upgrade history shows
  the SP1 verifier rotated to v5 in response to the Plonky3 vulnerability disclosure.
- [SP1 — contract addresses / gateway docs](https://docs.succinct.xyz/docs/sp1/verification/contract-addresses) —
  how the `SP1VerifierGateway` routes proof verification across verifier versions.
- [GHSA-6248-228x-mmvh](https://github.com/succinctlabs/sp1/security/advisories/GHSA-6248-228x-mmvh) —
  advisory whose fix required freezing/replacing affected verifiers — the operational template for
  verifier rotation.
