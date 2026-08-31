# 5. Precompile boundary mismatch

**Description.** zkVM precompiles (SHA-256, keccak, bigint, elliptic-curve ops, uint256) are
circuits the guest invokes via syscall. A boundary mismatch exists when the precompile *accepts* a
wider input range than its circuit *constrains*: the classic division gadget that enforces
`numer = quot*denom + rem` but forgets `rem < denom`, sign-correction terms that are one modulus
short in EC arithmetic, or missing modular reduction in uint256 operations.

**Why it matters.** Every guest that calls the precompile inherits the gap: a malicious prover can
satisfy the constraints with quotient/remainder (or curve-operation) witnesses that do not
correspond to the real computation, and the guest's downstream logic — signatures, hashes,
financial math — is then "proven" on wrong values. Because precompiles are marketed as safe
building blocks, application auditors routinely skip them; the boundary contract is exactly where
they should look.

**Detection hints.**

- For every "guess the answer, then multiply-check" gadget ask two questions: is there a bound on
  the remainder/result (`rem < denom`, canonical range)? What happens in the `denom = 0` arm — is
  it separately constrained?
- In bigint/ECC precompiles, trace every expression feeding a nondeterministic quotient/remainder
  op: can the expression go negative in the worst case, and is the added correction term (how many
  multiples of the prime) sufficient?
- Diff the *documented* safe-usage domain against the *constrained* domain — the vendor's own
  precompile-usage documentation is the spec to diff against.
- Edge-case test set per precompile: `0`, `1`, `-1` (signed), `p-1`, `p`, `2^256-1`, and aliasing
  inputs (`x_ptr == y_ptr`, unaligned pointers).

**Public references.**

- [SP1 — Safe Usage of SP1 Precompiles](https://docs.succinct.xyz/docs/sp1/security/safe-precompile-usage) —
  the vendor's own boundary contract for precompile inputs.
- [GHSA-f6rc-24x4-ppxp / CVE-2025-54873](https://github.com/advisories/GHSA-f6rc-24x4-ppxp) —
  signed division underconstrained; division-by-zero results underconstrained.
- [Veridise — RISC Zero zkVM Round 2](https://github.com/risc0/rz-security/blob/main/audits/zkVM/veridise_zkVM_20250224.pdf) —
  V-RISC0-VUL-003 (Critical): `DoDiv` constrains `numer = quot*denom + rem` but omits
  `rem < denom`; two valid `(quot, rem)` pairs exist for `numer=2, denom=1`.
- [Veridise — BigInt2 precompile audit](https://github.com/risc0/rz-security/blob/main/audits/precompiles/veridise_bigint2_240324.pdf) —
  V-RISC0_R2-VUL-002: positivity correction for nondeterministic quot/rem ops added `prime` where
  `prime*prime` was required.
- [Cantina — SP1 competition audit](https://github.com/succinctlabs/sp1/blob/main/audits/cantina.pdf)
  and [KALOS — SP1 Recursion VM audit](https://github.com/succinctlabs/sp1/blob/main/audits/kalos.md) —
  uint256 precompile underconstrained (missing mod reduction; `x_ptr == y_ptr` aliasing); recursion
  ALU division allowed `0/0` to produce arbitrary results.
