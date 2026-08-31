# 10. Wrapper / trusted-setup binding gaps (Groth16/PLONK)

**Description.** Most zkVM pipelines end by wrapping a STARK in a Groth16 or PLONK proof for cheap
onchain verification. That last mile re-introduces its own assumptions: public inputs must be
range-checked against the *scalar* field (not the base field), the trusted setup (SRS) actually
used must be the production one, and commitment extensions must bind what they claim to bind.

**Why it matters.** RISC Zero's snarkjs-generated verifier checked public inputs against the wrong
field, leaving roughly `q - r` (≈ 1.48 × 10³⁸) alias values accepted as valid public inputs — a
public-input aliasing hole in the contract everyone routes proofs through. SP1's Go verifier
builder silently selected an unsafe SRS whenever the data-directory path contained the substring
"dev". And gnark's Groth16 commitment extension was independently broken twice (zero-knowledge and
soundness), showing that wrapper conveniences are new cryptography, not free features.

**Detection hints.**

- In generated or hand-written verifier contracts, check every public-input range check: which
  field modulus is it compared against? (For BN254 Groth16: scalar field `r`, not base field `q`.)
- Inventory how the SRS / proving key is selected: string matches on paths, environment variables,
  feature flags. Can a production deployment trip the dev/test branch?
- Commitment/translation layers between the STARK and the SNARK (compressed proof digests, public
  value commitments): verify the binding is complete — every public value accounted for, no free
  words.
- Cross-check the wrapper's verification key provenance: is the vk baked into the contract/verifier
  tied to the ceremony/setup the deployment claims to use?

**Public references.**

- [Hexens — RISC Zero SNARK Verifier Contract audit (Jun 2024)](https://github.com/risc0/rz-security/blob/main/audits/contracts/hexens_verifiercontract_20240605.pdf) —
  SCRI-3 (High): `Groth16Verifier.checkField` checked public inputs against the base field instead
  of the scalar field, accepting ≈ 1.48 × 10³⁸ alias values.
- [Zellic — Two Vulnerabilities in gnark's Groth16 Proofs with Commitments](https://www.zellic.io/blog/gnark-bug-groth16-commitments)
  and [gnark GHSA-q3hw-3gm4-w5cr](https://github.com/Consensys/gnark/security/advisories/GHSA-q3hw-3gm4-w5cr) —
  commitment extension unsound with more than one commitment.
- [Veridise — SP1 audit](https://github.com/succinctlabs/sp1/blob/main/audits/veridise.pdf) —
  V-SP1-VUL-003 (High): unsafe SRS selected whenever the data-directory path contains "dev";
  V-SP1-VUL-006: shard indexes not range-checked in `verify_shard`.
- [zkSecurity — RISC Zero Groth16 verifier audit](https://github.com/risc0/rz-security/blob/main/audits/groth16/zksecurity_groth16.pdf) —
  audit of the STARK→Groth16 wrapper used for onchain verification.
- [LambdaClass — Responsible disclosure of an exploit in Succinct's SP1 zkVM](https://blog.lambdaclass.com/responsible-disclosure-of-an-exploit-in-succincts-sp1-zkvm-found-in-partnership-with-3mi-labs-and-aligned-which-arises-from-the-interaction-of-two-distinct-security-vulnerabilities/)
  and the [Succinct security update](https://blog.succinct.xyz/sp1-security-update-1-27-25/) —
  two interacting bugs (missing verifier checks; underconstrained `is_complete` in the recursion
  layer) composing into universal compressed-proof forgery, documented in
  [GHSA-c873-wfhp-wx5m](https://github.com/succinctlabs/sp1/security/advisories/GHSA-c873-wfhp-wx5m).
