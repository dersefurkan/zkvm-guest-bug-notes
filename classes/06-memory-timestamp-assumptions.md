# 6. Memory & timestamp argument assumptions

**Description.** zkVM memory consistency rests on arguments the guest never sees: timestamp/address
ordering, initialization tables, and allocator invariants. Guest programs that step outside the
safe Rust surface — raw syscalls, inline `ecall`s, custom or embedded allocators, hand-rolled
memory regions — inherit assumptions about this machinery that the circuits may not actually
enforce.

**Why it matters.** When the memory argument breaks, the prover fabricates memory contents and the
entire execution trace becomes fiction — the deepest soundness failure a VM has. The same
assumption gaps bite applications directly: SP1's embedded allocator had a pointer-overflow and a
heap-overlap flaw reachable from ordinary guest allocation patterns, and RISC Zero's
`ecall_software` could be driven into billions of unintended stores by an overflowed
`ptr + len` sum.

**Detection hints.**

- Treat every use of `unsafe`, custom `#[global_allocator]`, inline assembly, and direct
  `ecall`/`syscall` in a guest as an audit boundary: which memory-model invariant does the code
  rely on, and where is that invariant enforced (guest, executor, or circuit)?
- Allocator review: pointer arithmetic overflow (`ptr + len`, `align_up`), heap bounds versus
  reserved regions (`_end` versus reserved input/stack areas), initialization state.
- At the VM layer, the recurring shapes are: double-initialization of global memory tables,
  unconstrained "is this a memory op" selector bits, and interaction kinds that fail to distinguish
  memory from syscall events.
- Invariant to test: every byte the guest reads was written by this execution (or is committed
  initial image), and timestamps/addresses are monotone where the argument requires it.

**Public references.**

- [KALOS — SP1 Recursion VM audit](https://github.com/succinctlabs/sp1/blob/main/audits/kalos.md) —
  poseidon2-chip arbitrary memory write with underconstrained hash (Critical); `MemoryGlobalChip`
  double-initialization breaking the memory argument (High).
- [rkm0959 — SP1 V4 audit](https://github.com/succinctlabs/sp1/blob/main/audits/sp1-v4.md) — CPU
  `is_memory` selector underconstrained; `GlobalChip` `InteractionKind` not included in the digest
  (memory ↔ syscall event confusion).
- [GHSA-6248-228x-mmvh](https://github.com/succinctlabs/sp1/security/advisories/GHSA-6248-228x-mmvh) —
  embedded allocator: `read_vec_raw` pointer overflow; heap `_end` allowed past the reserved input
  region.
- [Veridise — RISC Zero zkVM Round 2](https://github.com/risc0/rz-security/blob/main/audits/zkVM/veridise_zkVM_20250224.pdf) —
  V-RISC0-VUL-010: `into_guest_ptr + into_guest_len` without safe math; overflow bypasses the check
  and triggers billions of stores (proving-service DoS).
