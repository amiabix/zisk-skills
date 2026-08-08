# Guest Programs

Use this reference when writing, reviewing, or explaining guest-side ZisK code.

## Current Guest Shape

A guest program is a Rust binary compiled to the ZisK RISC-V target and normally starts with:

```rust
#![no_main]
ziskos::entrypoint!(main);
```

The source of truth is `ziskos/entrypoint` plus the repo examples.

## Stable Mental Model

A good guest program does four things:

1. Read private input
2. Perform deterministic computation
3. Commit the minimum public output needed
4. Avoid unnecessary work because proving cost follows execution cost

## Guest-Side APIs To Verify First

Check `ziskos/entrypoint/src/io.rs` before suggesting code. The important functions are:

- `io::read::<T>()`
- `io::read_vec()`
- `io::read_input_slice()`
- `io::read_proof()`
- `io::commit(&value)`
- `io::write(&bytes)`
- `io::verify_zisk_proof(&proof)`

Do not assume external docs are current. Some guides use names like `commit_slice` or `read_bytes`; confirm whether the target repo actually exposes them.

## Designing Input

Prefer one of three patterns:

### Typed whole-input read

Use when the schema is stable and ergonomics matter more than streaming behavior.

```rust
let input: Input = ziskos::io::read();
```

### Primitive-by-primitive reads

Use when you want early validation, streaming logic, or lower peak memory.

```rust
let count: u64 = ziskos::io::read();
for _ in 0..count {
    let item: u32 = ziskos::io::read();
}
```

### Raw bytes / zero-copy

Use when the encoding is already defined externally or when raw byte access is the real contract.

## Designing Output

Prefer the smallest verifier-facing output that still makes the proof useful.

Recommended order:

1. Fixed-size digest or root
2. Small typed struct
3. Raw bytes only when exact layout control matters

If the verifier expects typed decoding, use `io::commit(&value)`.
If the verifier expects exact bytes, use the raw-write path actually present in the target repo.

## Performance Guidance

When optimizing guest code:

1. Start with clear Rust
2. Replace hot cryptographic/software paths with patched crates if available
3. Use `zisklib` wrappers next
4. Use raw syscalls only if necessary

Do not optimize blindly. Check examples, profiling support, or the existing project's hot paths first.

## Optimization Stack

Use this four-layer stack when reviewing or rewriting expensive guest code:

### Layer 1: Clear Rust

Start with obvious, typed Rust that makes the host/guest I/O contract and verifier-facing outputs unambiguous.

### Layer 2: Patched crates

Use a patched dependency first when one exists. This is the lowest-friction optimization path because the guest call sites usually stay unchanged.

External ZisK docs currently list these `[patch.crates-io]` candidates. Treat the list as docs-derived inventory and verify compatibility with the target tree before applying:

| Category | Crates currently listed by docs |
|---|---|
| Hashes | `sha2`, `sha3`, `tiny-keccak` |
| Curves / pairings / KZG | `k256`, `p256`, `substrate-bn`, `blst`, `bls12_381`, `c-kzg`, `kzg-rs`, `ark-algebra` |
| Big integers / modexp | `ruint`, `aurora-engine-modexp` |
| EVM / Ethereum | `revm`, `alloy-primitives`, `alloy-evm` |

Source: `zisk-docs/developer/writing-programs/libraries.mdx:286-333`.

### Layer 3: `zisklib` wrappers

If no patch exists, look for a wrapper in `ziskos/entrypoint/src/zisklib/lib/`. The current target tree exports wrapper families for:

- big integers and array arithmetic: `arith256`, `array_lib`
- hashes: `sha256`, `keccak256`, `blake2b`, `ripemd160`
- curves and signatures: `secp256k1`, `secp256r1`
- pairing systems and KZG: `bn254`, `bls12_381`

Source: `ziskos/entrypoint/src/zisklib/lib/mod.rs` and the module tree under `ziskos/entrypoint/src/zisklib/lib/`.

### Layer 4: Raw syscalls

Use raw syscalls only when neither a patch nor a wrapper covers the operation you need. Current syscall families in the target tree are:

- big integers: `add256`, `arith256`, `arith256_mod`, `arith384_mod`
- hashes / permutations: `keccakf`, `sha256f`, `poseidon2`, `blake2br`
- secp curves: `secp256k1_add`, `secp256k1_dbl`, `secp256r1_add`, `secp256r1_dbl`
- BN254: `bn254_curve_add`, `bn254_curve_dbl`, `bn254_complex_add`, `bn254_complex_sub`, `bn254_complex_mul`
- BLS12-381: `bls12_381_curve_add`, `bls12_381_curve_dbl`, `bls12_381_complex_add`, `bls12_381_complex_sub`, `bls12_381_complex_mul`

Source: `ziskos/entrypoint/src/syscalls/mod.rs` plus the individual files in `ziskos/entrypoint/src/syscalls/`.

When reviewing optimization choices, ask why the code is on the current layer and whether a lower-friction layer exists.

## Hints Vs Fcalls

Do not collapse these into one concept.

### Fcalls

`fcall` is a guest-side low-level mechanism exposed via CSR helpers and macros in `ziskos/entrypoint/src/fcall.rs:1-144`. It is used by `zisklib` internals and related support code for zkVM-side helper flows.

### Hints

The hints system is a native-side streaming/output pipeline gated behind cfgs and the `user-hints` feature:

- `ziskos/entrypoint/src/lib.rs:23-28` only exposes `pub mod hints` on non-zkvm builds when `feature = "user-hints"` and the relevant `zisk_hints*` cfgs are enabled
- `ziskos/entrypoint/src/hints/mod.rs` contains the socket/file writer pipeline and concrete hint handlers
- `ziskos-hints/` is a sibling wrapper crate around the same entrypoint source with hints-specific additions (`ziskos-hints/src/lib.rs`, `ziskos-hints/Cargo.toml`)

If a target repo depends on `ziskos-hints` rather than `ziskos`, that is an operational signal that the project cares about hint-enabled flows and preflight or hint-producer behavior. Do not simplify that distinction away.

## Common Review Questions

When reviewing a guest:

- Is the host/guest input order unambiguous?
- Is the public output minimal?
- Is the guest exposing unnecessary bytes?
- Is it using software crypto where a patch or `zisklib` path exists?
- Is the code depending on doc-only APIs that do not exist in the target repo?

## Host Build Wiring

In a normal ZisK project layout, the host crate's `build.rs` triggers a guest rebuild:

```rust
// host/build.rs
use zisk_sdk::build_program;

fn main() {
    build_program("../guest");
}
```

Check the pinned tree's nearest host `build.rs`; common examples also pre-serialize stdin files for constraint checks. The host then references the rebuilt ELF with:

```rust
pub const ELF: ElfBinary = include_elf!("guest-name");
```

The string passed to `include_elf!` is normally the guest crate's package name, not the file path. Verify the exact macro behavior in the pinned SDK before relying on it.

Guest crates usually do not need a `[[bin]]` declaration. They are normal Rust binary crates with `#![no_main]` + `ziskos::entrypoint!(main)`. Verify against the pinned examples.
