# 4. Missing vk/ELF allowlist onchain

**Description.** A zkVM proof says "some program ran correctly"; the program's identity — RISC
Zero's image ID, SP1's program verification key (`programVKey`) — is a separate value the onchain
verifier must pin explicitly. When the contract (or the off-chain verifier in front of it) does not
enforce an allowlist of expected program keys, or accepts keys supplied in calldata, a proof of
*any* program the attacker wrote will pass.

**Why it matters.** This turns every downstream check into theater: the attacker compiles a guest
that commits the desired output, proves it honestly, and the contract accepts it because nothing
bound the proof to the intended program. The same gap exists one layer down: SP1's Rust verifier
shipped versions that did not check `vk_root`, and SP1's recursive verifier initially failed to
load the Merkle root of valid verification keys as a constant, letting an attacker supply an
arbitrary root.

**Detection hints.**

- Grep the contract for where the program key comes from: `imageId` / `programVKey` should be an
  immutable, a constant, or an allowlist lookup — never unvalidated calldata. Patterns like
  `verify(seal, imageId, journalDigest)` with caller-supplied `imageId` are red flags unless the
  caller is itself a trusted, pinned contract.
- Check the aggregation/recursion boundary: is the set of valid vks committed (Merkle root,
  constant) inside the recursive verifier, or provided by the prover?
- Invariant to test: proof verifies ⇒ the proven program's key ∈ the pinned set *at verification
  time* (including after upgrades — see
  [class 9](09-upgrade-path-elf-vk-mismatch.md)).
- Boundary to diff: the key the SDK computes for the deployed guest binary versus the key the
  contract has pinned — build drift alone creates outages; unenforced drift creates holes.

**Public references.**

- [RISC Zero — Verifier Contracts documentation](https://dev.risczero.com/api/blockchain-integration/contracts/verifier) —
  the image ID is the application-pinned identifier of the guest program in onchain verification.
- [SP1 — Solidity SDK / onchain verification docs](https://docs.succinct.xyz/docs/sp1/verification/solidity-sdk)
  and the [sp1-contracts repo](https://github.com/succinctlabs/sp1-contracts) — the
  `ISP1Verifier`/`SP1VerifierGateway` workflow and the `programVKey` the application must fix.
- [GHSA-6248-228x-mmvh](https://github.com/succinctlabs/sp1/security/advisories/GHSA-6248-228x-mmvh) —
  missing `vk_root` checks in `verify_compressed` / `verify_shrink` / `verify_deferred_proof`
  (Zellic audit finding), fixed in v5.0.0.
- [rkm0959 — SP1 V3 audit](https://github.com/succinctlabs/sp1/blob/main/audits/rkm0959.md) —
  finding #4 (High): Merkle root of valid vks not loaded as a constant in the recursive verifier.
- [Code4rena — Succinct audit (Sep 2025)](https://code4rena.com/reports/2025-09-succinct) —
  M-01: PLONK/Groth16 host verifiers did not bind `vk_root` to a trusted root.
