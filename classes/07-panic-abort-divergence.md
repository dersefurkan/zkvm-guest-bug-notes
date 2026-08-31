# 7. Panic/abort divergence between native and proven execution

**Description.** A panic in a guest program is not a crash — it is a state for which *no valid
proof exists*. Any panic path an attacker can reach from onchain state (an empty epoch, a zero
denominator, a crafted event) becomes a liveness failure: the honest prover cannot produce the
proof the protocol needs to progress. The mirror-image bug lives onchain: verifiers that accept a
proof without checking its exit code treat *aborted* executions as successful ones.

**Why it matters.** In RISC Zero's PoVW system, an attacker could force a division-by-zero panic in
the `mint-calculator` guest; because commit roots form a chain, one poisoned `workLogId` blocked
every future proof and froze the epoch for all users — and a separate unbounded loop over the
journal's `updates` array let an attacker inflate the journal past the block gas limit, making
epoch payouts impossible to finalize onchain. No cryptography was broken; the arithmetic edge case
was enough.

**Detection hints.**

- List every panic point in the guest (`unwrap`, `expect`, `assert!`, division, indexing,
  `unwrap_or_else(|| panic!(...))`). For each one ask: can an attacker create this condition via
  chain state or inputs? Pay special attention to division denominators and empty-collection edge
  cases.
- In chained/aggregated designs (commit-root chains, epoch sequences): does one poisoned element
  halt the entire chain?
- Onchain: does the verifier path check the exit code / success flag? Are loops over proof-supplied
  arrays (journal length, update lists) bounded, and who bounds them?
- Prover-side DoS belongs to the same family: a guest or ELF that inflates proving time/memory
  (see class 1's `ecall_sha` example) is a cost attack on proving services.

**Public references.**

- [Hexens — Boundless PoVW audit (Aug 2025)](https://github.com/risc0/rz-security/blob/main/audits/povw/hexens-250827.pdf) —
  RISC6-3 (High): division-by-zero panic in `mint-calculator` guest blocks the epoch's proof chain;
  RISC6-4 (High): unbounded loop over the journal `updates` array in `PovwMint.mint` enables
  gas-limit DoS of epoch payouts.
- [Code4rena — Succinct audit (Sep 2025)](https://code4rena.com/reports/2025-09-succinct) —
  truncated `public_values` causing panics/DoS; `.unwrap()`-based DoS in the PLONK verifier path.
- [Hexens — RISC Zero zkVM audit (Oct 2023)](https://github.com/risc0/rz-security/blob/main/audits/zkVM/hexens_zkVM_20231031.pdf) —
  RSC0-3: `ecall_sha` invoked directly from assembly skips the block-count check; `count=10000`
  inflates memory by 130GB+, a one-shot DoS against proving services.
- [Sigma Prime — SP1 and zkVMs: A Security Auditor's Guide](https://sigmaprime.io/blog/sp1-zkvm-security-guide/) —
  panic discipline: in the guest a panic is a security property of the program, in the host a
  liveness bug; both sides must be reviewed.
