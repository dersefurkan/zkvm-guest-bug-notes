# 12. Unbound prover hints and witness values

**Description.** Verifier-side circuits delegate expensive operations to untrusted prover hints:
field inversions, range-check limb decompositions, quotient/witness values. If the hint's output
is not range-checked or otherwise bound to the value it claims to represent, a malicious prover
chooses the circuit's effective inputs. This is the underconstrained-guest-input class (class 1)
one layer down — inside the verifier and wrapper stack that a guest developer or integrator
inherits unchanged.

**Why it matters.** Guest teams audit *their* program; the verifier circuit arrives as
infrastructure. But the on-chain contract's trust is exactly as strong as the weakest unbound
hint in that stack. The pattern repeats across codebases and gets rediscovered publicly roughly
once per audit cycle — range-check gadgets whose limbs are not bound to the checked value, field
inverses taken on faith, truncated public-values parsing.

**Detection hints.**

- For every hint / auxiliary / witness value in a verifier or wrapper circuit, ask: "what
  constrains this to equal the function of the committed values?" If the answer is a comment, it
  is finding-shaped.
- Grep targets: `InvF` / `InvE`-style inversion hints, `RangeCheck` gadgets, `from_limbs` /
  decomposition helpers, and any `hint` callback in gnark or Plonky3-based stacks.
- Boundary to diff: Go (gnark) verifier code against the circuit it mirrors — decimal parsing,
  length prefixes, and modulus conformance are classic drift points.
- Test method: mutate one hint value at a time toward a false statement; a correctly constrained
  circuit rejects, an unbound one accepts.

**Public references.**

- [GHSA-f77q-r5qm-w4m8](https://github.com/succinctlabs/sp1/security/advisories/GHSA-f77q-r5qm-w4m8) —
  gnark recursion circuit: `InvF` / `InvE` hint values never range-checked against the BabyBear
  modulus.
- [Code4rena — Succinct audit (2025-09)](https://code4rena.com/reports/2025-09-succinct) — High:
  `KoalaBearRangeCheck()` hint limbs not bound to the value being checked; plus wrapper-verifier
  `vk_root` binding and public-values truncation findings in the same report.
- [KALOS — SP1 recursion VM audit](https://github.com/succinctlabs/sp1/blob/main/audits/kalos.md) —
  poseidon2 chip underconstrained hash and arbitrary-memory-write class findings in recursion
  AIRs: witness values accepted without constraint.
- [Plonky3 GHSA-f69f-5fx9-w9r9](https://github.com/Plonky3/Plonky3/security/advisories/GHSA-f69f-5fx9-w9r9) —
  insufficient checks in the FRI verifier: proof-supplied values trusted past their constrained
  domain.
