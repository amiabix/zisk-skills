# Project Patterns

Use this reference when explaining what ZisK projects look like or how to structure one.

## Common Shapes

### 1. Single guest + single host

Use for normal application proofs.

Typical shape:

- `guest/`, RISC-V guest binary crate (`#![no_main]`, `ziskos::entrypoint!(main)`)
- `host/`, host binary crate(s) plus `build.rs` calling `build_program("../guest")`
- optional `common/`, shared types between host and guest
- host `build.rs`

Bin layout convention. The canonical examples in `~/zisk/examples/` put host bin entry points at `host/bin/<name>.rs` (not the Cargo-default `src/bin/`) and declare them explicitly via `[[bin]]` blocks in `Cargo.toml`. See `examples/sha-hasher/host/Cargo.toml:20-42`. Newcomers expecting the Cargo default will be confused. The target repo may follow either convention, always check `Cargo.toml`.

This is the default pattern for new application work.

### 2. One host driving multiple guest programs

Use when the application has distinct provable programs with separate proving keys or separate execution roles.

The host usually:

- embeds more than one ELF
- calls `setup` for each
- executes/proves each independently

### 3. Aggregation / recursive verification

Use when one guest verifies proofs produced by other guests. Verified concrete wire-up:

Host side (`examples/aggregation/host/src/main.rs:49-66`):

```rust
let proof_bytes = client.prepare_send_proof(
    vadcop_result.get_proof(),
    vadcop_result.get_publics(),
    &vkey,
)?;

let stdin_aggregation = ZiskStdin::new();
stdin_aggregation.write_proof(&proof_bytes);  // not .write()
// ...repeat for each base proof...

let agg_result = client.prove(&pk_agg, stdin_aggregation)
    .with_proof_options(ProofOpts::default().minimal_memory())
    .run()?;
client.verify(agg_result.get_proof(), agg_result.get_publics(), &vkey_agg)?;
```

Guest side (`examples/aggregation/guest_agg/src/main.rs:8-21`):

```rust
let proof1 = ziskos::io::read_proof();
let proof2 = ziskos::io::read_proof();

if !ziskos::io::verify_zisk_proof(&proof1) { panic!("Proof 1 verification failed"); }
if !ziskos::io::verify_zisk_proof(&proof2) { panic!("Proof 2 verification failed"); }
```

Key APIs (verify against current source before quoting):

- `client.prepare_send_proof(&proof, &publics, &vk) -> Result<Vec<u8>>`, host serializer
- `stdin.write_proof(&bytes)`, distinct from `stdin.write(&value)`
- `ziskos::io::read_proof() -> &[u8]` (zkvm) / `Box<[u8]>` (non-zkvm), `ziskos/entrypoint/src/io.rs:50-59`
- `ziskos::io::verify_zisk_proof(&bytes) -> bool`, `ziskos/entrypoint/src/io.rs:83-86`

Recursive verification of external Groth16/PLONK proofs typically lands on the BN254 curve and complex-operation precompiles at Layer 4 of the optimization stack; check whether a `substrate-bn` patch or higher-level wrapper path exists before dropping to raw syscalls.

## Architecture Boundaries

Keep these boundaries clear:

- guest logic: provable application logic
- host logic: orchestration, setup, input prep, proof lifecycle
- shared types: only when they genuinely reduce host/guest contract errors
- proving internals: avoid coupling app code directly to low-level prover machinery unless needed

## Good Explanations

When explaining a ZisK project, answer these explicitly:

- What is the guest proving?
- What prepares the input?
- What is public at the end?
- What proof artifact is produced?
- Does another verifier, contract, or guest consume it?
