---
name: zisk-developer
description: Use when working with ZisK zkVM guest programs, host/prover code, ziskos, zisk-sdk, cargo-zisk, ziskemu, ZiskStdin, hints, Assembly, precompiles, proof generation flows, recursive aggregation, verifier integration, or when ZisK docs/examples/API names disagree with the target repository.
license: MIT
metadata:
  version: "1.3.0"
  domain: zkvm
  triggers: ZisK, ziskos, zisk-sdk, cargo-zisk, ziskemu, guest program, host program, ZiskStdin, hints, Assembly, precompiles, recursive proof, aggregation, verifier
  role: specialist
  scope: implementation
  output-format: code
  related-skills: rust-engineer, zisk-build, zisk-remote-prover, zisk-ffi, zisk-optimizer, zisk-soundness, zisk-internals
---

# ZisK Developer

Senior ZisK engineer for guest programs, host/prover code, hints, precompiles, Assembly execution, distributed proving, and verifier integration. Treat the pinned ZisK source tree and the target repository as reality; docs and memory are orientation only.

## Core Workflow

1. **Pin the tree** - Identify the ZisK source/tag/toolchain, target repo revision, ELF, stdin, feature flags, executor kind, and proving mode.
2. **Find the canonical example** - Start from the nearest working example in the pinned ZisK tree or target repo. Do not design ZisK host/guest flow without checking source.
3. **Fix the I/O contract** - State what the host writes, what the guest reads, what is public, what stays private, and the byte/order format. Settle format questions by reading `io.rs` and hexdumping a real input. Remember that a saved input file is framed: the host writes each value as a length prefix followed by the payload padded to an alignment boundary, so the raw file is larger than the bytes the guest reads. Confirm the exact framing in source before hand-building an input file, because a comment that describes only the payload will produce a file the emulator mis-frames.
4. **Choose the lowest-friction path** - Plain Rust -> patched crate -> project wrapper -> `zisklib` wrapper -> raw syscall. Verify every API at its definition site before using it.
5. **Map the proof consumer** - Identify whether the proof is consumed by an off-chain service, a ZisKOS guest, a recurser, or `ZiskVerifier.sol`; write the artifact format and exact values that consumer must pin.
6. **Check upstream drift** - Record the verified upstream commit, compare it with the current tree, and re-read every changed proof, verifier, recursion, or calldata surface before reusing an integration pattern.
7. **Validate the exact path** - Build and run the same executor/prover path the project will use. Emulator success is not proof of Assembly, hints, GPU, recursion, or verifier success.

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
| Proof artifacts and off-chain verification | `common/src/proof.rs`, `sdk/src/verify.rs`, `cli/src/commands/user/verify.rs` | Loading a proof, calling `verify`, pinning program identity, or decoding publics |
| ZisKOS child-proof verification | `ziskos/entrypoint/src/zisklib/lib/zisk_verifier.rs`; `verifier/src/verifier.rs`; nearest verifier guest | Passing a proof through guest stdin or accepting another proof inside a guest |
| Recursive aggregation | `recurser/templates/aggregator.circom.tera`, `recurser/src/prove/`, `sdk/src/aggregate_proofs.rs` | Folding proofs, leaf allow-lists, `rootC`, free inputs, or recursive public outputs |
| EVM verification | `zisk-contracts/ZiskVerifier.sol`; `cli/src/commands/dev/export_solidity_calldata.rs` | Wrapping a PLONK proof, exporting calldata, or wiring an application contract |

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
ZiskStdin::write(value)      -> u64 byte length + bincode payload + zero padding to an 8-byte boundary
ZiskStdin::write_slice(buf)  -> u64 byte length + raw payload + zero padding to an 8-byte boundary
io::read<T>()                -> consumes one framed typed record
io::read_slice()             -> consumes one framed raw record
```

When building `.stdin` files by hand for `ziskemu -i`, do not write a bare payload unless the guest is written for that exact raw layout. Hexdump the file and verify every record has the 8-byte length prefix, payload bytes, and padding expected by the guest read order.

### Guest Runtime and Execution Modes

`entrypoint!` supports the upstream guest entrypoint forms; check its pinned definition
before choosing `fn()`, an integer-returning entrypoint, or a `Result`-returning one.
Keep guest termination, allocator choice, and feature gates explicit. A host build or
native test can hide guest-only allocation, `getrandom`, no-std, linker, and syscall
behavior.

Choose evidence by intent:

```text
execute-only: guest behavior; no witness/proof claim
execute: selected execution/prover path
verify-constraints: witness and constraint consistency; no proof artifact
prove: proof artifact, still requiring consumer verification
```

`verify-constraints` is embedded-client-only in the current SDK and is not a proof.
For toolchain/ELF/setup material load `zisk-build`; for a coordinator, worker, or live
stream load `zisk-remote-prover`; for C/C++/staticlib or raw ABI code load `zisk-ffi`.

### Host/Prover Shape

```rust
use zisk_sdk::{ExecutorKind, GuestProgram, ProverClient, ZiskStdin};

async fn run_guest(elf: &str, input: impl serde::Serialize) -> anyhow::Result<()> {
    let program = GuestProgram::from_uri(elf)?;
    let stdin = ZiskStdin::new();
    stdin.write(&input);

    let client = ProverClient::embedded()
        .executor(ExecutorKind::Assembly)
        .build()?;

    client.upload(&program).run()?;
    client.setup(&program).run()?.await?;
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

Current `cargo-zisk` supports the Assembly backend on Linux only; `--asm` is rejected on macOS.

### Upstream Drift Gate

Treat proof verification as release-specific code, not a stable recipe. Before writing
or reviewing a verifier integration, pin the current upstream revision and compare it
with the revision on which the integration was last validated:

```bash
git fetch origin
git log --oneline <last-verified-zisk-commit>..origin/main
git diff <last-verified-zisk-commit>..origin/main -- \
  common/src/proof.rs sdk/src/verify.rs verifier/src \
  ziskos/entrypoint/src/zisklib/lib/zisk_verifier.rs \
  recurser zisk-contracts cli/src/commands/user/verify.rs \
  cli/src/commands/dev/export_solidity_calldata.rs
```

If that diff changes a proof enum, public-vector layout, verification API, hash
family, recursive template, `rootC` selection, Solidity hash preimage, or calldata
ABI, repeat the entire consumer-path validation. Do not carry forward a previous
acceptance rule because a command name or proof kind still exists.

### Proof Consumer Matrix

```text
consumer                 cryptographic check                  application binding still required
off-chain CLI / SDK       Proof::verify()                      expected program, proof kind, full claim, rootC
ZisKOS guest              verify_zisk_proof_c                 stream layout, program, claim, final verifier key
recurser                  generated circuit / proofman         leaf allow-list, fold semantics, free values, rootC
Solidity                  ZiskVerifier.verifySnarkProof        contract-pinned program, rootC, claim and domain policy
```

The first column decides the wire format and verifier. The final column decides
whether your application can safely accept the proof. A valid proof is not an
application authorization until both columns are satisfied.

### Proof Artifact Contracts

`Proof::save()` is a bincode host artifact. It is suitable for host loading via
`Proof::load()`, not for guest proof verification. `Proof::get_proof_bytes()` emits
the ZisKOS VADCOP stream as little-endian u64 words:

```text
[minimal][n_publics][flag? | program_vk(4) | user_publics(64)][proof][vadcop_final_vk(4)]
```

| Proof shape | `minimal` | public count | flag | Intended use |
| --- | --- | --- | --- | --- |
| Leaf VADCOP | `0` | 69 | `1` | Raw ZisK final proof / recursion leaf |
| Recurser VADCOP | `0` | 69 | `0` | Output of an aggregation round |
| Minimal VADCOP | `1` | 68 | absent | Compressed standalone proof; require explicitly |
| PLONK | n/a | wrapped | n/a | EVM/off-chain SNARK verification |

The 64 user publics are native u64 field elements. `PublicValues` is a u32 view;
use it only where the exact source path proves truncation is intended. Recursive
proofs and PLONK `publicsHash` binding require the full-width values.

### Safe Consumer Sequence

```text
untrusted proof bytes/artifact
  -> require the expected proof family and supported hash mode
  -> parse the exact documented layout and reject wrong lengths/flags
  -> run the corresponding cryptographic verifier
  -> compare committed program VK, verifier key/rootC, and ordered claim
  -> decode the claim with the same field widths/order as its producer
  -> accept or commit only the checked claim
```

For a normal VADCOP proof, the committed values are in `ProofBody::Vadcop`:
`publics_full` is `[program_vk(4) | user_publics(64)]`, while `zisk_vk` is the
VADCOP final verifier key. For PLONK, use `ProofBody::Plonk.publics_full` and
`rootc`. Verify first, then compare those committed fields to expected constants.

Do not use a separate `Proof.program_vk` field from an untrusted serialized artifact
as a substitute for the program VK committed in the proof body. Do not treat
`with_program_vk(...)` as the PLONK program-binding step: current PLONK verification
derives its public hash from the body's `publics_full` and `rootc`.

### ZisKOS Child-Proof Verifier

`ziskos::zisklib::verify_zisk_proof_c` is a VADCOP verifier, not a generic proof
decoder or policy engine. It requires an 8-byte-aligned byte pointer and a byte length
divisible by eight, treats the last four words as the supplied VADCOP final verifier
key, and currently fixes the hash family to `Poseidon1`.

For a production child-proof guest:

1. Read one framed raw record with `io::read_slice()`; do not pass a host bincode artifact.
2. Reject unsupported proof flavor, word count, flag, and hash-family assumptions before decoding claim fields.
3. Call the verifier and reject `false`.
4. Compare the parsed program VK, every claim-bearing public/root, and trailing verifier key to guest constants or authenticated parent inputs.
5. Commit only a result derived from those checked values.

The built-in `agg_verify` example demonstrates only cryptographic VADCOP validation.
Do not use it as a production acceptance pattern without the binding checks above.

### Recursion And EVM Boundaries

The generated recurser circuit distinguishes leaves (`flag = 1`) from prior aggregate
outputs (`flag = 0`), can enforce a leaf program-VK allow-list, and selects root keys
according to that origin. Its `AggregatePublics` body is still the owner of range,
ordering, continuity, and merge semantics; write those constraints in the circuit and
test forged folds. `n_free` inputs are positional and exact-width: a leaf's values go
through normalization, while an aggregated proof's values are already `free_out`.

`ZiskVerifier.sol` verifies a PLONK proof for caller-supplied `programVK`,
`rootCVadcopFinal`, and `publicValues`. It hashes:

```text
sha256(programVK big-endian || user_publics little-endian || rootC big-endian)
```

The generic verifier contract does not encode your protocol policy. The application
contract must pin the accepted program/version/rootC, enforce its domain fields
(chain, range, input root, output root, nonce), and decode/check the public claim
before recording state or releasing value.

## Validation Commands

Use project-specific wrappers when present. Otherwise start with:

```bash
cargo-zisk build --release
ziskemu -e <guest.elf> -i <input.stdin> --steps
ziskemu -e <guest.elf> -i <input.stdin> -X --no-thousands-sep
cargo-zisk execute --elf <guest.elf> --inputs <input.stdin>
cargo-zisk prove --elf <guest.elf> --inputs <input.stdin>
cargo-zisk verify --proof <proof.bin>
```

For a verifier integration, also run its actual consumer path: a host acceptance test,
a ZisKOS guest proof, a recursive forged-fold test, or the Solidity fixture test. CLI
verification alone is not evidence that the downstream consumer pins the intended
claim.

For hints, validate the Assembly path, not only emulator:

```bash
RUSTFLAGS='--cfg zisk_hints' cargo build --release
cargo-zisk setup --elf <guest.elf> --asm --hints
cargo-zisk execute --elf <guest.elf> --asm --hints file:///abs/path/hints.bin
cargo-zisk prove --elf <guest.elf> --asm --hints file:///abs/path/hints.bin
```

Command names and flags can drift. Run `<tool> --help` and inspect the pinned source before treating these as exact.

## Constraints

### MUST DO

- Verify API names, signatures, features, and cfg gates at their definition site.
- Pin and report ZisK version, source revision, ELF path/hash, stdin path/hash, features, executor, and proving mode.
- Make host/guest I/O explicit before code changes.
- Confirm `[patch.crates-io]` entries actually apply by checking `Cargo.lock`.
- Keep public outputs minimal and deterministic.
- Pin the upstream ZisK commit and inspect verifier-surface changes before reusing a proof integration.
- State the proof artifact format, proof family, hash family, and consumer before writing the verifier path.
- Bind the committed full-width program VK, verifier key/rootC, and ordered public claim after cryptographic verification.
- Exercise the real consumer path with forged proof inputs, not only `cargo-zisk verify`.
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
- Pass `Proof::save()` bincode bytes to `verify_zisk_proof_c`, or assume it parses a PLONK proof.
- Treat a caller-supplied VADCOP key, PLONK calldata field, proof flag, or `rootC` as application-approved without an exact comparison.
- Accept a recursive fold because its input proofs verify while leaving ordering, continuity, free values, or output semantics unchecked.

## Output Templates

When implementing or reviewing ZisK code, provide:

1. Pinned tree/toolchain and target repo revision
2. Guest/public theorem and trust boundary
3. Host/guest I/O contract
4. Implementation path: plain Rust, patched crate, wrapper, `zisklib`, or syscall
5. Hints/Assembly/prover path if relevant
6. Proof consumer matrix: artifact, verifier, pinned values, decoded claim
7. Upstream-drift comparison and verifier surfaces re-read
8. Validation commands/results, including negative consumer tests
9. Unvalidated assumptions and residual risks

## Knowledge Reference

ZisK zkVM, ziskos, zisk-sdk, cargo-zisk, ziskemu, guest ELF, `entrypoint!`, `ZiskStdin`, `read`, `commit`, `commit_slice`, public outputs, precompiles, patched crates, `zisklib`, raw syscalls, hints stream, input hints, precompile hints, custom hints, pass-through hints, Assembly executor, ROM setup, distributed coordinator/worker proving, recursive proof aggregation, verifier contracts, program/VK binding, emulator profiling, `-X` cost accounting, GPU proving.
