# 1. Underconstrained guest input

**Description.** In a zkVM, the host (or prover) that executes the guest program is fully
untrusted: every byte delivered via `env::read()` / `sp1_zkvm::io::read()`, every syscall return
value, and every hinted length is chosen by the prover. The proof only attests that *some*
execution of the guest was valid — if the guest consumes host data without re-validating it, the
prover effectively chooses the program's inputs and, through them, its outputs.

**Why it matters.** A malicious prover can feed crafted inputs that steer the guest into committing
wrong results to the journal — and the resulting proof still verifies onchain. At the VM boundary
this goes further: RISC Zero's `sys_read` channel allowed a crafted host response to write into
arbitrary guest memory (CVE-2025-61588), compromising every guest program that used it. For proving
services (Bonsai, Boundless, prover networks), unchecked `count`/`len` arguments are also a direct
denial-of-service and cost-amplification vector against the prover.

**Detection hints.**

- Enumerate every host→guest channel: grep the guest for `env::read`, `io::read`, `read_slice`,
  `read_vec`, `stdin`, and any raw `ecall`. For each one ask: what does the guest *recompute or
  re-check* versus trust?
- Check syscall implementations at the executor level (not just the library level — guests can call
  ecalls directly, bypassing library checks): are all guest-supplied pointers inside the user
  memory region? Is `ptr + len` computed with checked arithmetic? Are `count` parameters capped
  (e.g. `MAX_SHA_COMPRESS_BLOCKS`)?
- Invariant to test: for every input `x` the host supplies, there exists a guest-side predicate
  `validate(x)` whose failure aborts execution *before* `x` influences committed output.
- Boundary to diff: what the host *could* send (arbitrary bytes) versus what the guest *accepts*
  (the validated subset). The gap is the attack surface.

**Public references.**

- [GHSA-jqq4-c7wq-36h7 / CVE-2025-61588](https://github.com/advisories/GHSA-jqq4-c7wq-36h7) —
  crafted host `sys_read` response writes to arbitrary guest memory; arbitrary code execution in
  the guest.
- [Hexens — RISC Zero zkVM audit (Oct 2023)](https://github.com/risc0/rz-security/blob/main/audits/zkVM/hexens_zkVM_20231031.pdf) —
  RSC0-5 (ecall pointer parameters escape user-space memory checks), RSC0-10 (unchecked
  `buf_ptr + buf_len`), RSC0-3 (direct-assembly `ecall_sha` with `count=10000` → 130GB+ memory
  inflation, prover DoS).
- [Veridise — RISC Zero zkVM Round 2 (Feb 2025)](https://github.com/risc0/rz-security/blob/main/audits/zkVM/veridise_zkVM_20250224.pdf) —
  V-RISC0-VUL-010: `ecall_software` allocates the `to_guest` vector *before* checking its address,
  with a non-overflow-safe `into_guest_ptr + into_guest_len` sum.
- [Sigma Prime — SP1 and zkVMs: A Security Auditor's Guide](https://sigmaprime.io/blog/sp1-zkvm-security-guide/) —
  host/guest trust boundary: "the proof only proves the guest", so critical logic must live in the
  guest and validate its inputs.
- [SP1 Security Model](https://docs.succinct.xyz/docs/sp1/security/security-model) — vendor
  documentation of which user programs count as safe and what the guest must enforce itself.
