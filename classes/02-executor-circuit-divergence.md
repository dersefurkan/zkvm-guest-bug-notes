# 2. Executor ↔ circuit divergence

**Description.** A zkVM has two definitions of "what happened": the executor (native/RV32IM
semantics) and the constraint system that proves the execution. Where the two disagree — ignored
ELF permission flags, reserved-memory conventions, division-by-zero behavior, register `x0`
handling — an execution can be *provable* that could never occur on real hardware, or a real
execution can become *unprovable*.

**Why it matters.** Divergence is a soundness hazard in one direction (a malicious prover proves an
"infeasible" execution — e.g. a write to a read-only segment succeeds in the zkVM but faults on
real RISC-V) and a completeness/liveness hazard in the other (honest executions cannot be proven).
For guest developers it also breaks the mental model "if it runs correctly in tests, the proof
means the same thing": unit tests exercise the executor, not the constraints.

**Detection hints.**

- Boundary to diff: platform semantics versus real ISA behavior. Ask for every documented (or
  undocumented) deviation: "is the execution the zkVM proves feasible on actual hardware?" If the
  difference is undocumented, it is finding-shaped.
- Check the ELF/image loader as the place where initial state is defined: are segment `vaddr`s
  confined to user memory? Is word alignment enforced? Are `p_flags` (PF_X/PF_W/PF_R) applied or
  ignored? Are all offset sums overflow-checked *in release builds* (where overflow does not panic)?
- Arithmetic edge cases where executor and circuit conventions must match: division/remainder by
  zero (RV32IM returns `q = -1, r = n` rather than faulting), signed overflow, sign extension.
- Test method: differential execution — run the same guest natively (or in a reference emulator)
  and under the zkVM executor on adversarial inputs, and diff results, faults, and committed
  outputs.

**Public references.**

- [Veridise — RISC Zero zkVM Round 2](https://github.com/risc0/rz-security/blob/main/audits/zkVM/veridise_zkVM_20250224.pdf) —
  V-RISC0-VUL-022 (ELF `PT_LOAD` permission flags ignored: read-only segments are writable in the
  zkVM while Linux would fault — "infeasible executions" become provable) and V-RISC0-VUL-024
  (release-build offset overflow in `load_elf` wraps to the start of the file).
- [GHSA-f6rc-24x4-ppxp / CVE-2025-54873](https://github.com/advisories/GHSA-f6rc-24x4-ppxp) —
  signed integer division underconstrained: multiple outputs valid for some inputs;
  division-by-zero results underconstrained.
- [LambdaClass — The future of ZK is in RISC-V zkVMs, but the industry must be careful](https://blog.lambdaclass.com/the-future-of-zk-is-in-risc-v-zkvms-but-the-industry-must-be-careful-how-succincts-sp1s-departure-from-standards-causes-bugs/) —
  how departures from the RISC-V standard (register `x0` / memory-register handling) produce bugs.
- [Zellic — SP1 Design Review](https://github.com/succinctlabs/sp1/blob/main/audits/zellic.pdf) —
  RISC-V standards-compliance review, including reserved memory regions and deviation risk.
