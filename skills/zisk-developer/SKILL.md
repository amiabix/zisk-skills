---
name: zisk-developer
description: Use when working with ZisK zkVM guest programs, host/prover code, ziskos, zisk-sdk, cargo-zisk, ziskemu, ZiskStdin, hints, Assembly, precompiles, proof generation flows, recursive aggregation, verifier integration, or when ZisK docs/examples/API names disagree with the target repository.
license: MIT
metadata:
  version: "1.1.0"
  domain: zkvm
  triggers: ZisK, ziskos, zisk-sdk, cargo-zisk, ziskemu, guest program, host program, ZiskStdin, hints, Assembly, precompiles, recursive proof, aggregation, verifier
  role: specialist
  scope: implementation
  output-format: code
  related-skills: rust-engineer, zisk-optimizer, zisk-soundness, zisk-internals
---

# ZisK Developer

Senior ZisK engineer for guest programs, host/prover code, hints, precompiles, Assembly execution, distributed proving, and verifier integration. Treat the pinned ZisK source tree and the target repository as reality; docs and memory are orientation only.

## Core Workflow

1. **Pin the tree** - Identify the ZisK source/tag/toolchain, target repo revision, ELF, stdin, feature flags, executor kind, and proving mode.
2. **Find the canonical example** - Start from the nearest working example in the pinned ZisK tree or target repo. Do not design ZisK host/guest flow without checking source.
3. **Fix the I/O contract** - State what the host writes, what the guest reads, what is public, what stays private, and the byte/order format. Settle format questions by reading `io.rs` and hexdumping a real input. Remember that a saved input file is framed: the host writes each value as a length prefix followed by the payload padded to an alignment boundary, so the raw file is larger than the bytes the guest reads. Confirm the exact framing in source before hand-building an input file, because a comment that describes only the payload will produce a file the emulator mis-frames.
4. **Choose the lowest-friction path** - Plain Rust -> patched crate -> project wrapper -> `zisklib` wrapper -> raw syscall. Verify every API at its definition site before using it.
5. **Validate the exact path** - Build and run the same executor/prover path the project will use. Emulator success is not proof of Assembly, hints, GPU, recursion, or verifier success.

## Reference Guide

Use docs for orientation, then verify in source:

| Topic | Docs / source to check | Load when |
| --- | --- | --- |
| Guest shape and I/O | ZisK docs quickstart; `ziskos/entrypoint/src/`, especially `io.rs` | Writing `entrypoint!`, `read`, `commit`, `commit_slice`, or public output code |
| Host/prover SDK | ZisK docs quickstart and hints examples; `sdk/src/`; target repo host binaries | Writing `execute`, `prove`, `setup`, `verify-constraints`, or recursive host code |
| CLI behavior | `<tool> --help`, `cli/`, target repo scripts | Using `cargo-zisk`, `ziskemu`, coordinator, worker, or project wrappers |
| Precompiles | ZisK precompiles docs; `ziskos/entrypoint/src/zisklib/lib/`, `precompiles/`, patched crates in `Cargo.lock` | Accelerating crypto or checking software fallback |
| Hints | ZisK hints-stream docs; `ziskos/entrypoint/src/hints/`, `ziskos-hints/`, `precompiles-hints/`, `sdk/src/hints.rs` | Using input hints, precompile hints, custom hints, or distributed hint streaming |
| Assembly/proving | ZisK proving docs; `executor/src/execution/asm/`, emulator-asm, and ROM setup source | Debugging Assembly, setup, shared memory, GPU proving, or `legacy_mem_count_and_plan` |
| Distributed proving | ZisK prover docs; coordinator/worker crates, remote SDK paths, deployment scripts | Running coordinator/worker proofs or interpreting cluster behavior |
| Proof composition | target repo bind/aggregate guests; SDK verifier paths; verifier contracts | Verifying child proofs, VKs, publics, recursion, or on-chain outputs |

Do not preload stale local notes. If this skill, docs, and source disagree, the pinned source wins.

## Key Patterns with Examples

### Guest Program Shape

```rust
#![no_main]

ziskos::entrypoint!(main);

fn main() {
    let input: Vec<u8> = ziskos::io::read();
    let digest = ziskos::zisklib::sha256(&input);
    ziskos::io::commit_slice(&digest);
}
```

Use `entrypoint!` for normal Rust guests. Keep public outputs compact: field elements, fixed bytes, digest, root, or claim hash. Avoid committing large witness data.

### Input Framing

```text
ZiskStdin::write(value)      -> u64 byte length + bincode payload + 8-byte padding
ZiskStdin::write_slice(buf)  -> u64 byte length + raw payload + 8-byte padding
io::read<T>()                -> consumes one framed typed record
io::read_input_slice()       -> consumes one framed raw record
```

When building `.stdin` files by hand for `ziskemu -i`, do not write a bare payload unless the guest is written for that exact raw layout. Hexdump the file and verify every record has the 8-byte length prefix, payload bytes, and padding expected by the guest read order.

### Host/Prover Shape

```rust
use zisk_sdk::{ExecutorKind, GuestProgram, ProverClient, ZiskStdin};

async fn run_guest(elf: &str, input: impl serde::Serialize) -> anyhow::Result<()> {
    let program = GuestProgram::from_uri(elf)?;
    let mut stdin = ZiskStdin::new();
    stdin.write(&input)?;

    let client = ProverClient::embedded()
        .executor(ExecutorKind::Assembly)
        .build()?;

    client.upload(&program).run()?;
    let result = client.execute(&program, stdin).run()?.await?;
    println!("steps={}", result.get_execution_steps());
    Ok(())
}
```

Treat this as a shape, not an API contract. Recheck method names and builder behavior in the pinned `sdk/src/` before editing production code.

### Hints Lifecycle

```text
native deterministic run with --cfg zisk_hints
  -> hint stream/file produced in exact guest order
  -> setup generated with --asm --hints or SDK Assembly .with_hints()
  -> Assembly execute/prove consumes the same hint source with --asm
  -> guest/circuit verifies the hinted result
```

Hints are not truth. Input hints move private data delivery; precompile hints move expensive helper work; custom/pass-through hints move user-defined data. Soundness comes only from the guest/circuit check that consumes the hinted value.

Determinism requirements:
- The native hint-generation path and zkVM guest path must emit/consume hints in the same order.
- Do not generate hints from nondeterministic iteration such as unordered `HashMap` loops.
- Do not generate hints from parallel code unless ordering is explicitly fixed and tested.

### Precompile / Acceleration Audit

```text
Cargo.toml/Cargo.lock patch present?
guest call reaches patched crate, wrapper, zisklib, or syscall?
guest initializes the provider/factory used by that path?
ziskemu -X shows named operation rows, not only PRECOMPILES total?
Assembly/prover path uses the same accelerated route?
```

Never infer acceleration from dependency names alone. Confirm source wiring and a measured run.

### Assembly vs Emulator

```text
Emulator: fast semantic check and -X profiling signal
Assembly: production-performance path, hints path, shared memory, prover behavior
Proof: final evidence only after execute/prove/verify path succeeds
```

A guest that executes in emulator but fails in Assembly/proving may still be semantically correct; capture exact command, versions, ELF hash, stdin hash, executor, flags, machine, GPU, RAM, and `ulimit -l`.

## Validation Commands

Use project-specific wrappers when present. Otherwise start with:

```bash
cargo-zisk build --release
ziskemu -e <guest.elf> -i <input.stdin> --steps
ziskemu -e <guest.elf> -i <input.stdin> -X --no-thousands-sep
cargo-zisk execute --elf <guest.elf> --inputs <input.stdin>
cargo-zisk prove --elf <guest.elf> --inputs <input.stdin>
```

For hints, validate the Assembly path, not only emulator:

```bash
RUSTFLAGS='--cfg zisk_hints' cargo build --release
cargo-zisk setup --elf <guest.elf> --asm --hints
cargo-zisk execute --elf <guest.elf> --asm --inputs <input.stdin> --hints file:///abs/path/hints.bin
cargo-zisk prove --elf <guest.elf> --asm --inputs <input.stdin> --hints file:///abs/path/hints.bin
```

Command names and flags can drift. Run `<tool> --help` and inspect the pinned source before treating these as exact.

## Constraints

### MUST DO

- Verify API names, signatures, features, and cfg gates at their definition site.
- Pin and report ZisK version, source revision, ELF path/hash, stdin path/hash, features, executor, and proving mode.
- Make host/guest I/O explicit before code changes.
- Confirm `[patch.crates-io]` entries actually apply by checking `Cargo.lock`.
- Keep public outputs minimal and deterministic.
- Use emulator for fast correctness/profile loops and Assembly for production-performance, hints, and prover behavior.
- For hints, verify generation, ordering, setup with `--asm --hints`, consumption, and in-guest/circuit checking.
- Report commands actually run and what remains unvalidated.

### MUST NOT DO

- Use an API, CLI flag, executor mode, or builder method without checking the pinned docs/source or `<tool> --help`.
- Mix normal input and hint input modes unless the CLI/SDK path explicitly supports that combination.
- Run a hinted proof without both an Assembly client/path and setup material generated with `--asm --hints` or SDK Assembly `.with_hints()`.
- Use emulator execution as evidence that hints, Assembly, GPU proving, recursion, or verifier integration works.
- Emit hints from nondeterministic native code paths such as parallel execution or unordered map iteration.
- Treat input hints, pass-through hints, host files, or missing witness as authenticated truth unless the guest verifies them.
- Bypass patched crates or `zisklib` wrappers for raw syscalls without checking syscall preconditions at the source.
- Publish large or private witness values with `commit`/`commit_slice`; committed outputs are public and order-sensitive.
- Accept benchmark/demo guests on a production verifier path without explicit program/VK binding.

## Output Templates

When implementing or reviewing ZisK code, provide:

1. Pinned tree/toolchain and target repo revision
2. Guest/public theorem and trust boundary
3. Host/guest I/O contract
4. Implementation path: plain Rust, patched crate, wrapper, `zisklib`, or syscall
5. Hints/Assembly/prover path if relevant
6. Validation commands and results
7. Unvalidated assumptions and residual risks

## Knowledge Reference

ZisK zkVM, ziskos, zisk-sdk, cargo-zisk, ziskemu, guest ELF, `entrypoint!`, `ZiskStdin`, `read`, `commit`, `commit_slice`, public outputs, precompiles, patched crates, `zisklib`, raw syscalls, hints stream, input hints, precompile hints, custom hints, pass-through hints, Assembly executor, ROM setup, distributed coordinator/worker proving, recursive proof aggregation, verifier contracts, program/VK binding, emulator profiling, `-X` cost accounting, GPU proving.
