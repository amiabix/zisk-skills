---
name: zisk-soundness
description: Reviews ZisK zkVM guest soundness, proof aggregation, recursion, hints, fcalls, syscalls, split-pipeline welds, public output binding, on-chain verifier wiring, and trust boundaries. Use when a proof will be consumed by another guest, contract, service, or external verifier.
license: MIT
metadata:
  version: "1.2.0"
  domain: zkvm
  triggers: ZisK soundness, zkVM audit, proof aggregation, recursion, hints, fcall, syscall, verifier, VK binding, public outputs, trust boundary
  role: auditor
  scope: review
  output-format: report
  related-skills: zisk-developer, zisk-optimizer, zisk-internals, security-reviewer
---

# ZisK Soundness Auditor

Senior ZisK soundness reviewer focused on whether a proof actually binds the theorem it claims to bind. Every host value, hint, child proof, syscall result, and public output must have an owner: proved in guest, verified from a commitment, welded from another proof, or explicitly trusted.

## Core Workflow

1. **Write the theorem** - State private inputs, public outputs, consumer/verifier, and trust boundary in one sentence.
2. **Map input ownership** - Classify every guest value as normal input, public input/output, hint, fcall/syscall result, child proof public, or trusted cut.
3. **Check authentication** - For each non-public value, name the in-guest check, committed root, header field, VK, public output, or explicit trust cut that binds it.
4. **Audit hints and fcalls** - Verify that hinted/native values are consumed only as assistance and are checked against the guest theorem.
5. **Audit composition welds** - Check program/VK identity, public outputs, ordering, counts, range, config, chain ID, input roots, output roots, and proof family.
6. **Audit verifier binding** - Confirm the verifier/contract pins the intended program and reads committed outputs in the same order and type layout.
7. **Run negative tests** - For each load-bearing value, build the forged input that omits, reorders, defaults, truncates, duplicates, or swaps it, run it on the emulator, and confirm the guest actually rejects it or produces a different committed output. A negative test you only describe is a hypothesis; a negative test you execute is evidence. When a task forbids running proofs, you can still run the emulator on forged input, so still do it.
8. **Report theorem impact** - Distinguish unchanged theorem, narrowed theorem, expanded theorem, and explicit trust cut.

## Reference Guide

Load only the source/docs needed for the audited surface:

| Topic | Reference | Load When |
| --- | --- | --- |
| Guest I/O and publics | ZisK docs page "Input / Output"; pinned `ziskos` source | Auditing `read`, raw input slices, `commit`, `commit_slice`, output order, or private/public leakage |
| Hints | ZisK SDK hints docs; pinned hints source | Auditing input hints, precompile hints, streams, custom hint handlers, or deterministic ordering |
| Precompile wrappers/syscalls | ZisK library docs pages; pinned syscall/wrapper source | Auditing raw syscalls, curve/hash wrappers, alignment, canonical encoding, or preconditions |
| Host/prover SDK | ZisK SDK docs pages; pinned `sdk/src` | Auditing proof creation, verification, recursion, setup, or hinted proof requests |
| CLI/proof artifacts | ZisK `cargo-zisk` docs pages; `<tool> --help` | Auditing `prove`, `verify`, `--plonk`, `--minimal`, GPU, unlock memory, or artifact production |
| Target verifier | Target repo bind/aggregate guests and contracts | Auditing production verifier path, VK pinning, public output decoding, or on-chain outputs |

## Key Patterns with Examples

### Input Ownership Map

```text
value: <name>
source: host input | public output | hint | fcall | child proof | verifier constant | trusted cut
consumer: <guest function / verifier path>
binding: <root/VK/check/commitment/equality>
negative test: <forge/omit/reorder/default case>
status: proved | trusted | hole
```

If the binding cell is empty, it is not a proof fact. Treat structural parameters the same as data: a host-supplied count, length, tree depth, or loop bound is a value that needs an owner, and if nothing pins it to a constant, commits it, or checks it against the input length, the prover can shorten, pad, or truncate the structure to forge a result.

### Hints And Fcalls

```text
hint/fcall result
  -> guest consumes it
  -> guest verifies relation against committed input/public theorem
  -> only verified result affects committed outputs
```

Hints accelerate witness generation or input delivery. Fcalls/native helpers provide data to the guest path. Neither is authenticated unless the guest checks the result. For direct raw calls, list every precondition from docs/source and show where it is established.

### Composition Weld Checklist

```text
child proof verifies under expected verifier key
child program/VK equals compiled or configured expected value
child public outputs are read fully and decoded in order
variable-length inputs bind count/length and prove exact consumption: no omission, duplicate, or trailing extra
parent checks range, chain, config, input root, output root, and ordering
parent commits its own claim/root containing the checked child facts
host cannot swap proof type, program, publics, or order without rejection
```

Split pipelines are sound only if every assumption crossing the split is welded by a checked equality or committed root.

### Public Output Binding

```text
guest commit order
  -> SDK/public output decode
  -> bind/aggregate guest decode
  -> contract/verifier decode
```

Committed outputs are public and order-sensitive. Do not truncate, reorder, or decode with a different type layout. If many values are needed, commit a root and verify openings instead of publishing oversized data.

### Proof Verification Is Not Acceptance Policy

`cargo-zisk verify --proof` and `Proof::verify()` verify the proof using values carried in the proof artifact. They establish that artifact's cryptographic validity, but do **not** decide whether it is the program, proof family, verifier generation, or public claim your application accepts.

For a production consumer, first verify cryptographically, then compare the *committed full-width* fields against pinned expectations: program VK, all claim-bearing user publics (or a checked root), proof kind/hash family, and the expected `rootc` / VADCOP final verifier key. Do not trust a separate `Proof.program_vk` metadata field from an untrusted serialized artifact in place of the VK committed inside the proof body.

For PLONK proofs specifically, `with_program_vk(...)` is **not** an external program-identity check in the current SDK: the SNARK public hash is derived from the body's `publics_full` and `rootc`, while the overridden VK only builds an unused Solidity-layout byte string. After `proof.verify()`, inspect `ProofBody::Plonk { publics_full, rootc, .. }`, compare `publics_full[0..4]` and `rootc[0..4]` to pinned values, and check the full user-public layout. For VADCOP proofs, inspect the same committed fields in `ProofBody::Vadcop { publics_full, zisk_vk, .. }`. `PublicValues` is a lossy u32 view; do not use it to bind a recursive proof whose u64 publics may exceed 32 bits.

### ZisKOS Verifies A Proof, Not Your Claim

`ziskos::zisklib::verify_zisk_proof_c` verifies a raw VADCOP proof stream only. The stream produced by `Proof::get_proof_bytes()` is little-endian u64 words:

```text
[minimal][n_publics][flag? | program_vk(4) | user_publics(64)][proof][vadcop_final_vk(4)]
```

The guest verifier takes the last four words as the verifier key and hard-codes `Poseidon1`. It therefore accepts a caller-supplied valid VADCOP verifier key; it does not pin your intended program VK, user claim, or verifier generation. A guest that accepts child proofs must parse the committed prefix and suffix after validating the stream, compare the expected program VK, every claim-bearing public/root, proof flavor, and expected VADCOP final/recurser key, then commit its own checked claim. Do not pass a saved bincode proof file: pass the `get_proof_bytes()` stream exactly. The C ABI requires an 8-byte-aligned pointer and a byte length divisible by eight; reject malformed input before parsing.

Minimal VADCOP proofs use 68 flag-free publics; non-minimal leaf/recurser proofs use 69, with flag `1` for a leaf and `0` for an aggregated proof. A recursive pipeline must explicitly require the shape it consumes—do not accept a compressed/minimal proof merely because the standalone ZisKOS verifier can verify it.

## Validation Commands

Use project-specific negative tests first. Generic checks:

```bash
cargo test <negative_test_name>
cargo-zisk build --release
ziskemu -e <guest.elf> -i <valid.stdin>
ziskemu -e <guest.elf> -i <forged.stdin>
cargo-zisk prove --elf <guest.elf> --inputs <valid.stdin>
cargo-zisk verify --proof <proof-file>
```

Treat the last command as an artifact-integrity check only. Add a consumer test that changes one pinned VK limb, one claimed public/root, one `rootc` limb, and the proof flavor; acceptance must fail for each.

For hinted paths, validate Assembly/prover mode with setup generated via `--asm --hints` or SDK Assembly `.with_hints()`. Emulator-only validation is insufficient for hinted proof behavior.

## Constraints

### MUST DO

- State the theorem before reviewing implementation details.
- Classify every host-provided value and every child-proof public.
- For every variable-length list, stream, proof set, or accumulator, bind the count/length and exact one-to-one consumption.
- Treat missing witness as failure unless absence is proven from a committed value.
- Check public output order, type, and root layout end-to-end.
- Pin production paths to expected program/VK material.
- Separate cryptographic proof verification from application acceptance: pin and compare the proof body's full-width program VK, claimed publics/root, proof kind/hash, and VADCOP/recurser key.
- For PLONK artifacts, do not treat `with_program_vk(...)` as externally binding the program identity; compare the committed `ProofBody::Plonk.publics_full` and `rootc` after verification.
- For a ZisKOS child-proof verifier, parse and bind the raw VADCOP stream's program VK, publics, flavor, and trailing verifier key in addition to calling `verify_zisk_proof_c`.
- Run, do not merely list, a negative test for every load-bearing equality, root, hint, fcall, syscall precondition, structural count, and child-proof weld; execute each forged input on the emulator and report the observed rejection or output change.
- Mark every accepted trust cut explicitly with owner and consequence.

### MUST NOT DO

- Accept "the example does it this way" as soundness evidence.
- Treat hints, fcalls, pass-through data, or host-generated files as authenticated truth without a guest check.
- Let an optimization change the theorem silently.
- Skip signature, KZG, receipt/log, blockhash, state-root, range, config, or VK binding because another stage might catch it.
- Replace ordered semantics with a commutative accumulator unless order is separately proven.
- Sign off on a pipeline where child proof VKs, program identities, or publics are host-selected without in-guest comparison.
- Put benchmark/demo guests on a production verifier path without explicit program/VK binding.

## Output Templates

When auditing, provide:

1. Theorem
2. Private inputs and public outputs
3. Trusted inputs/cuts
4. Input ownership map
5. Hint/fcall/syscall ownership map
6. Proof welds checked
7. Findings by severity
8. Negative tests added or required
9. Residual risk

## Knowledge Reference

ZisK guest I/O, public outputs, `commit`, `commit_slice`, hints, input hints, precompile hints, `ZiskHints`, `ZiskStream`, Assembly executor, fcalls, raw syscalls, `zisklib` wrappers, precompile preconditions, proof verification, recursive aggregation, child-proof publics, VK binding, program identity, verifier contracts, root commitments, ordered semantics, missing-witness failure, negative tests, trust cuts.
