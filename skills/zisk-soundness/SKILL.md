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

1. **Pin upstream and diff it** - Record the ZisK revision, prior verified revision, proof mode, and verifier consumer; inspect every change in the proof/verifier/recursion/calldata surfaces before relying on an existing pattern.
2. **Write the theorem** - State private inputs, public outputs, consumer/verifier, and trust boundary in one sentence.
3. **Map input ownership** - Classify every guest value as normal input, public input/output, hint, fcall/syscall result, child proof public, or trusted cut.
4. **Check authentication** - For each non-public value, name the in-guest check, committed root, header field, VK, public output, or explicit trust cut that binds it.
5. **Audit hints and fcalls** - Verify that hinted/native values are consumed only as assistance and are checked against the guest theorem.
6. **Audit proof acceptance** - Separate artifact cryptographic verification from the application policy that pins program identity, proof family, verifier generation, public claim, and domain fields.
7. **Audit composition welds** - Check program/VK identity, public outputs, ordering, counts, range, config, chain ID, input roots, output roots, and proof family.
8. **Run negative tests** - For each load-bearing value, build the forged input that omits, reorders, defaults, truncates, duplicates, or swaps it, run it on the emulator, and confirm the guest actually rejects it or produces a different committed output. A negative test you only describe is a hypothesis; a negative test you execute is evidence. When a task forbids running proofs, you can still run the emulator on forged input, so still do it.
9. **Report theorem impact** - Distinguish unchanged theorem, narrowed theorem, expanded theorem, and explicit trust cut.

## Reference Guide

Load only the source/docs needed for the audited surface:

| Topic | Reference | Load When |
| --- | --- | --- |
| Guest I/O and publics | ZisK docs page "Input / Output"; pinned `ziskos` source | Auditing `read`, raw input slices, `commit`, `commit_slice`, output order, or private/public leakage |
| Hints | ZisK SDK hints docs; pinned hints source | Auditing input hints, precompile hints, streams, custom hint handlers, or deterministic ordering |
| Precompile wrappers/syscalls | ZisK library docs pages; pinned syscall/wrapper source | Auditing raw syscalls, curve/hash wrappers, alignment, canonical encoding, or preconditions |
| Host/prover SDK | ZisK SDK docs pages; pinned `sdk/src` | Auditing proof creation, verification, recursion, setup, or hinted proof requests |
| CLI/proof artifacts | ZisK `cargo-zisk` docs pages; `<tool> --help` | Auditing `prove`, `verify`, `--plonk`, `--minimal`, GPU, unlock memory, or artifact production |
| Off-chain proof verification | `common/src/proof.rs`; `sdk/src/verify.rs`; `cli/src/commands/user/verify.rs` | Auditing `Proof::verify`, verifier overrides, serialized artifacts, or full-width publics |
| ZisKOS proof verification | `ziskos/entrypoint/src/zisklib/lib/zisk_verifier.rs`; `verifier/src/verifier.rs`; child-proof guests | Auditing raw proof streams, guest proof acceptance, hash support, layout, or key binding |
| Recursive aggregation | `recurser/templates/aggregator.circom.tera`; `recurser/src/prove`; `sdk/src/aggregate_proofs.rs` | Auditing leaf allow-lists, flags, rootC selection, free values, fold order, or merged output semantics |
| EVM verification | `zisk-contracts/ZiskVerifier.sol`; `cli/src/commands/dev/export_solidity_calldata.rs` | Auditing wrapped PLONK proofs, Solidity calldata bytes, verifier upgrades, or application-contract binding |
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

### Upstream Drift Gate

Proof acceptance is tied to the exact upstream implementation. Record the revision
that the current audit covers, then compare it before every upgrade:

```bash
git fetch origin
git log --oneline <last-verified-zisk-commit>..origin/main
git diff <last-verified-zisk-commit>..origin/main -- \
  common/src/proof.rs sdk/src/verify.rs verifier/src \
  ziskos/entrypoint/src/zisklib/lib/zisk_verifier.rs \
  recurser zisk-contracts cli/src/commands/user/verify.rs \
  cli/src/commands/dev/export_solidity_calldata.rs
```

Re-audit the consumer if that diff changes `Proof`/`ProofBody`, proof-kind or public
layout, hash-mode handling, VADCOP verification, recursive templates/rootC rules,
PLONK public-hash construction, or Solidity ABI/calldata encoding. Report both the
previous and current revision; “latest version” without a source comparison is not
evidence.

### Verification Boundary Matrix

| Consumer | Cryptographic statement | Required application weld |
| --- | --- | --- |
| `cargo-zisk verify` / `Proof::verify()` | This artifact verifies against commitments it carries | Expected program, proof family, full claim, hash mode, verifier key/rootC |
| ZisKOS `verify_zisk_proof_c` | This raw VADCOP stream verifies under its trailing key and `Poseidon1` | Expected stream shape, program VK, claim, final/recurser key, intended proof flavor |
| Recurser | Two child proofs satisfy the generated fold circuit | Leaf allow-list, free-value meaning, order/continuity/range and resulting claim |
| `ZiskVerifier.sol` | PLONK proof matches caller-supplied digest inputs | Contract-pinned program/rootC, decoded claim, chain/config/domain policy |

The left column proves a cryptographic relation. The right column proves that relation
is the one the application intended to accept. A missing right-column equality is a
host-selected trust cut.

### Artifact Commitment Map

```text
VADCOP final/recurser: [flag | program_vk(4) | user_publics(64)]  // 69 u64s
VADCOP minimal:        [program_vk(4) | user_publics(64)]         // 68 u64s
ZisKOS stream:         [minimal][n_publics][publics][proof][vadcop_final_vk(4)]
PLONK public hash:     program_vk(BE) || user_publics(LE) || rootc(BE)
```

`flag = 1` identifies a raw leaf and `flag = 0` an aggregated recurser proof.
`rootc` is the VADCOP final verifier key stamped into the wrapper: it is the final
VADCOP key for a plain proof and the recurser's key for an aggregated proof. It is
not generally the program VK. The `PublicValues` API is a u32 view; recursive and
PLONK binding must use committed full-width u64 values.

`Proof::save()` produces a bincode host artifact. It is not the ZisKOS proof stream.
Only `Proof::get_proof_bytes()` supplies the LE-u64 VADCOP layout expected by the
guest verifier.

### Off-Chain Acceptance Audit

For an untrusted artifact, audit this sequence in order:

```text
load / parse
  -> require expected proof kind and hash family
  -> cryptographically verify
  -> read proof-body committed full-width values
  -> compare pinned program VK, rootc/final key, and ordered claim
  -> decode claim and enforce protocol domain rules
  -> accept
```

For VADCOP, compare `ProofBody::Vadcop.publics_full` and `zisk_vk`. For PLONK,
compare `ProofBody::Plonk.publics_full` and `rootc` after `proof.verify()`.
Do not rely on a separate `Proof.program_vk` metadata field from an untrusted bincode
artifact as the committed identity.

For current PLONK verification, `with_program_vk(...)` is not an external
program-identity weld: the SNARK public hash is constructed from the proof body's
`publics_full` and `rootc`. A sound consumer compares the body’s committed values
itself, rather than treating an override call as authorization.

### ZisKOS Child-Proof Audit

`verify_zisk_proof_c` verifies VADCOP only. It takes the last four words of the
provided stream as the verifier key, fixes the hash family to `Poseidon1`, and checks
alignment/whole-word input at the C boundary. It does not parse an application claim
or choose the expected key for you.

Audit an in-guest verifier as:

```text
framed raw stdin record
  -> exact byte/word/header/flag check
  -> VADCOP verification
  -> expected trailing key equality
  -> expected program VK and public-claim equality
  -> guest commits checked parent claim
```

Require the proof flavor the guest actually consumes. In particular, a standalone
minimal proof’s 68-public layout is not interchangeable with a 69-public leaf or
recurser layout merely because both can be cryptographically valid.

### Recursive And On-Chain Welds

The generated recurser circuit authenticates child proofs and can enforce a leaf
program-VK allow-list, but `AggregatePublics` owns the protocol theorem. Review its
constraints for sequence/order, continuity, range, state root, configuration, and
every output slot; review `NormalizePublics` and positional `n_free` values as part of
the same theorem. Verify that the output program VK/rootC selection is compatible
with the next recursive round and the final consumer.

The generic Solidity verifier receives `programVK`, `rootCVadcopFinal`,
`publicValues`, and proof bytes from calldata. It validates their PLONK relation; it
does not know which program, rootC generation, or public claim the application allows.
The consuming contract must pin those values and validate every domain field before
mutating state or releasing funds.

### Negative Proof-Acceptance Matrix

| Forge | Required result |
| --- | --- |
| Replace one program-VK limb | Rejected by host, guest, recurser, or contract policy |
| Replace one VADCOP final/recurser-key (`rootc`) limb | Rejected |
| Change public claim/root, field width, or field order | Rejected or yields a distinct committed parent claim |
| Swap child proofs or insert a foreign leaf | Fold rejected unless the theorem explicitly permits it |
| Supply wrong flag/minimal shape/hash family | Rejected before acceptance |
| Use bincode bytes where a ZisKOS stream is required | Rejected |
| Submit valid PLONK calldata for an unapproved program/rootC | Application contract rejects |

Run these on the actual acceptance path. A successful standalone CLI verification is
not evidence that a production host, guest, aggregation circuit, or contract rejects
the forged variant.

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

Treat the last command as an artifact-integrity check only. Add a consumer test for
every row of the negative proof-acceptance matrix and run the real host, guest,
recurser, or contract acceptance path.

For hinted paths, validate Assembly/prover mode with setup generated via `--asm --hints` or SDK Assembly `.with_hints()`. Emulator-only validation is insufficient for hinted proof behavior.

## Constraints

### MUST DO

- State the theorem before reviewing implementation details.
- Classify every host-provided value and every child-proof public.
- For every variable-length list, stream, proof set, or accumulator, bind the count/length and exact one-to-one consumption.
- Treat missing witness as failure unless absence is proven from a committed value.
- Check public output order, type, and root layout end-to-end.
- Pin production paths to expected program/VK material.
- Compare the upstream verifier surfaces with the last audited revision before accepting an upgrade.
- Bind application policy to the proof body’s full-width committed program VK, publics/root, kind/hash, and VADCOP/recurser key after cryptographic verification.
- Treat `with_program_vk(...)` as insufficient PLONK application binding unless the body’s committed values are also compared.
- For a ZisKOS child-proof verifier, bind the raw stream layout, proof flavor, program VK, publics, and trailing verifier key in addition to calling `verify_zisk_proof_c`.
- Run forged-artifact tests on the actual final consumer, not only a CLI verifier.
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
7. Upstream revision/diff and changed verifier surfaces
8. Findings by severity
9. Negative tests added or required, with observed consumer result
10. Residual risk

## Knowledge Reference

ZisK guest I/O, public outputs, `commit`, `commit_slice`, hints, input hints, precompile hints, `ZiskHints`, `ZiskStream`, Assembly executor, fcalls, raw syscalls, `zisklib` wrappers, precompile preconditions, proof verification, recursive aggregation, child-proof publics, VK binding, program identity, verifier contracts, root commitments, ordered semantics, missing-witness failure, negative tests, trust cuts.
