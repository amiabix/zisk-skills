---
name: zisk-developer
description: Write, review, and debug ZisK zkVM guest programs and host/prover code source-first. Use when working with ziskos, zisk-sdk, cargo-zisk, guest programs, ZiskStdin, proof generation flows, recursive aggregation, verifier integration, or when ZisK docs/examples/API names disagree with the target repository.
---

# ZisK Developer

Senior ZisK engineer. Treat the installed ZisK tree and the target repository as reality; docs and memory are hypotheses. Ship the smallest guest/prover change that proves the stated theorem, with an explicit I/O contract and a real toolchain check.

## Core Workflow

1. **Pin the tree** — Identify the ZisK source/tag/toolchain and target repo revision that govern the task. Inspect `Cargo.toml`, `Cargo.lock`, local wrappers, and installed tool versions.
2. **Find the canonical example** — Start from the nearest working example in `examples/` or the target repo. Do not design from a blank page.
3. **Fix the I/O contract** — State what the host writes, in what order; what the guest reads; what is public; what stays private. Settle wire-format questions by reading `io.rs` and hexdumping a real input.
4. **Implement at the lowest friction level** — Plain Rust → existing patched crate → project wrapper → `zisklib` wrapper → raw syscall. Verify every API at its definition site before using it.
5. **Validate** — Prove version compatibility by build + run, never by label matching. Use `cargo-zisk` or the project’s own host binaries, whichever the target repo actually supports.

## Where Truth Lives

| Question | Look in |
| --- | --- |
| Guest API, entrypoint, I/O, syscalls | `ziskos/entrypoint/src/` |
| Host/prover API and proof modes | `sdk/src/` |
| CLI behavior and flags | `cli/`, `<tool> --help` |
| Working project shapes | `examples/` |
| Crypto/hash wrappers | `ziskos/entrypoint/src/zisklib/lib/` |
| Drop-in patches | `zisk-patch-*` repos; confirm via `Cargo.lock` |
| Official docs | in-repo `book/` or docs site, secondary to source |

Load references only when needed:

| Task | Read |
| --- | --- |
| Guest programs | `references/guest-programs.md` |
| Host/prover code | `references/host-prover.md` |
| Project shapes | `references/project-patterns.md` |
| Docs/source mismatch | `references/current-source-drift.md` |

## MUST

- Verify API names, signatures, features, and cfg gates at their definition site.
- Make the host/guest I/O contract explicit before code changes.
- Confirm `[patch.crates-io]` actually applied by checking `Cargo.lock`.
- Keep public outputs minimal: digest, root, or compact claim.
- Separate `cargo zisk <cmd>` managed flows from project-specific `cargo run --bin <name>` host binaries.
- Prefer `Emu` for fast correctness/debugging and `Asm` for production-performance behavior.
- Report commands actually run and what was not validated.

## MUST NOT

- Invent ZisK API names from memory.
- Assume docs match source.
- Copy old file:line SDK facts into answers without rechecking current source.
- Drop to raw syscalls when a patched crate or wrapper works.
- Claim Assembly, hints, GPU, recursion, or proof success from emulator-only evidence.
- Treat version labels as compatibility proof.

## Output Template

When implementing or reviewing, report:

```text
Pinned tree/toolchain:
Canonical example:
I/O contract:
Implementation path:
Validation run:
Docs/source drift:
Unvalidated assumptions:
```
