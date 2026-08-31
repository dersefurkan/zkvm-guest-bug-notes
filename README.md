# zkvm-guest-bug-notes

A living, curated map of **zkVM guest-program and off-chain → on-chain binding bug classes**, with
public references for every entry. Teaching notes, not a disclosure dump and not a zkVM-internal
soundness tracker.

The zk ecosystem has excellent catalogs for circuit-level and proof-system-level bugs
([0xPARC/zk-bug-tracker](https://github.com/0xPARC/zk-bug-tracker),
[zksecurity/zkbugs](https://github.com/zksecurity/zkbugs),
[Trail of Bits ZKDocs](https://www.zkdocs.com/), and the academic
[SoK on SNARK vulnerabilities](https://arxiv.org/abs/2402.15293)). This collection deliberately
covers a different, thinner-documented layer: **the application boundary of general-purpose zkVMs**
(RISC Zero, SP1, and similar RISC-V/ISA-based systems) — the Rust guest program, the
journal/public-values it commits to, and the onchain verifier contract that consumes the proof.

## Scope

**In scope:**

- Guest-program bugs: untrusted host input, panic discipline, memory-model and executor-semantics
  assumptions, precompile misuse boundaries.
- Binding bugs: journal/public-values not committed to the application context, missing or stale
  program-key (image ID / verification key) allowlists onchain, replay and domain-separation gaps,
  ELF↔key mismatch across upgrades.
- Wrapper/integration bugs at the last mile: STARK→SNARK (Groth16/PLONK) verifier contracts and
  their public-input handling.

**Out of scope:** this is *not* a zkVM-internal soundness tracker. Core-VM circuit
under-constraints, Fiat-Shamir hygiene, and proof-system cryptography are covered better elsewhere
— see the datasets linked above. Core-VM findings appear here only where they change what a
**guest developer or integrator** must do (e.g. a precompile that accepts a wider input range than
its circuit constrains is an application-level hazard, because every guest that calls it inherits it).

Every class page lists detection hints (what to grep, which invariant to test, which boundary to
diff) and **only already-public references**: audit reports, security advisories, vendor
documentation, and published writeups. No undisclosed material, no private findings.

## Bug classes

| # | Class | One-liner | Refs |
|---|-------|-----------|------|
| 1 | [Underconstrained guest input](classes/01-underconstrained-guest-input.md) | Host-supplied data is fully untrusted; anything the guest does not re-validate is a value a malicious prover chooses for you. | 5 |
| 2 | [Executor ↔ circuit divergence](classes/02-executor-circuit-divergence.md) | What runs natively and what gets constrained disagree — executions become provable that could never happen on real hardware. | 4 |
| 3 | [Journal / public-values binding gap](classes/03-journal-public-values-binding.md) | The proof is valid, but it does not commit to the context the application thinks it does. | 5 |
| 4 | [Missing vk/ELF allowlist onchain](classes/04-missing-vk-elf-allowlist-onchain.md) | The onchain verifier does not pin the expected program key, so a proof of *any* program passes. | 5 |
| 5 | [Precompile boundary mismatch](classes/05-precompile-boundary-mismatch.md) | The precompile's accepted input range is wider than the range its circuit actually constrains. | 5 |
| 6 | [Memory & timestamp argument assumptions](classes/06-memory-timestamp-assumptions.md) | Raw syscalls, custom allocators and inline ecalls inherit memory-model assumptions the circuits may not enforce. | 4 |
| 7 | [Panic/abort divergence](classes/07-panic-abort-divergence.md) | A guest panic is a liveness bug (the state becomes unprovable) — and, if exit codes go unchecked onchain, a soundness bug. | 4 |
| 8 | [Replay & missing domain separation](classes/08-replay-missing-domain-separation.md) | A valid proof or signature is reusable across roles, chains or contexts it was never bound to. | 4 |
| 9 | [Upgrade-path ELF↔vk mismatch](classes/09-upgrade-path-elf-vk-mismatch.md) | The guest program was upgraded, but the onchain key allowlist — or the deprecated verifier — was not rotated with it. | 4 |
| 10 | [Wrapper / trusted-setup binding gaps](classes/10-wrapper-trusted-setup-binding.md) | The STARK→SNARK wrapper (Groth16/PLONK) re-introduces field-aliasing and setup-trust assumptions at the last mile. | 5 |

See [SECURITY.md](SECURITY.md) for disclosure rules.

## Who maintains this

Maintained by **Furkan Derse** — **dersefurkan32@gmail.com** · Telegram:
[@FURY_Fn](https://t.me/FURY_Fn).

This is a curated notes collection, not an exhaustive dataset. Entries are added or amended as new
public reports and advisories appear.

## Contributing

Corrections and additions are welcome — open an issue or a pull request. Ground rules:

1. **Public material only.** Every referenced finding must already be disclosed (published audit
   report, advisory, or vendor-approved writeup). Link the primary source.
2. **No undisclosed findings.** Do not file your own unpatched or in-scope-bounty findings here.
3. Keep entries class-level: patterns, invariants, and detection hints — not one-off bug writeups.
4. Prefer references with stable URLs (vendor security advisories, archived audit PDFs, official
   docs).

## Related public work

- [0xPARC/zk-bug-tracker](https://github.com/0xPARC/zk-bug-tracker) — the original public ZK bug
  catalog.
- [zksecurity/zkbugs](https://github.com/zksecurity/zkbugs) — reproducible PoCs of ZKP
  vulnerabilities.
- [Trail of Bits ZKDocs](https://www.zkdocs.com/) ([repo](https://github.com/trailofbits/zkdocs)) —
  reference documentation for ZK proof systems.
- [SoK: What don't we know? Understanding Security Vulnerabilities in SNARKs](https://arxiv.org/abs/2402.15293)
  — academic taxonomy.
- [Sigma Prime: SP1 and zkVMs — A Security Auditor's Guide](https://sigmaprime.io/blog/sp1-zkvm-security-guide/) —
  the first public guest-program audit checklist.
- [RISC Zero security audit log](https://github.com/risc0/rz-security/blob/main/audits/README.md)
  and [SP1 audits directory](https://github.com/succinctlabs/sp1/tree/main/audits) — primary
  sources for most classes here.

## License

[MIT](LICENSE).
