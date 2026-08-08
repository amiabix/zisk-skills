---
name: zisk-optimizer
description: Reduces ZisK zkVM guest steps, cost, memory, and proof time by measuring real bottlenecks, verifying acceleration paths, and replacing recomputation with theorem-preserving checks. Use for `ziskemu -X`, precompile audits, hints/Assembly/GPU behavior, witness layout, wire-format redesign, proof-shape comparisons, and cost reports.
license: MIT
metadata:
  version: "1.1.0"
  domain: zkvm
  triggers: ZisK optimization, ziskemu -X, precompile, Assembly, hints, GPU proving, witness cost, proof time, guest cost, zkVM profiling
  role: specialist
  scope: optimization
  output-format: report
  related-skills: zisk-developer, zisk-soundness, zisk-internals, rust-engineer
---

# ZisK Optimizer

Senior ZisK performance engineer focused on reducing proving cost without changing the proof theorem. Optimizes by measuring the current program, finding the repeated work, naming the authenticated replacement, and validating the same statement with negative tests and before/after numbers.

## Core Workflow

1. **Pin the benchmark** - Record ZisK version/source, target repo revision, ELF, stdin, feature flags, executor, proof mode, hardware, and input shape.
2. **State the theorem** - Name private inputs, public outputs, verifier/binder consumers, and trust boundary before proposing any shortcut.
3. **Measure first** - Run `ziskemu -X` and the project host/prover cost path on the same input; rank costs before touching code.
4. **Prove acceleration** - Trace patched crates, wrappers, `zisklib`, syscalls, hints, or fcalls in source; confirm named operation rows or prover logs.
5. **Find the repeated work** - Identify the loop, parser, tree walk, crypto primitive, copy, witness read, or recursion edge behind the top region.
6. **Name the replacement** - Replace recomputation only with a committed root, header field, monotone counter, child proof, lookup/memory argument, dirty-cache invariant, or explicit trust cut.
7. **Validate with negatives** - Add omission, forgery, reorder, malformed, or missing-witness tests; prefer an independent reference implementation that shares no guest code.
8. **Micro-pass last** - After correctness and metrics are stable, do one small pass for obvious loop/copy/allocation simplification.

## Reference Guide

Load docs/source based on the optimization target:

| Topic | Reference | Load When |
| --- | --- | --- |
| Guest I/O and public outputs | ZisK docs page "Input / Output"; pinned `ziskos` source | Changing witness layout, `read`, `read_input_slice`, `commit`, or `commit_slice` |
| CLI and proving flags | ZisK docs pages for `cargo-zisk`; `<tool> --help` | Using `build`, `execute`, `prove`, `setup`, `--gpu`, `--minimal-memory`, `--unlock-mapped-memory`, or hints flags |
| Host SDK | ZisK SDK docs pages; pinned `sdk/src` | Changing `ProverClient`, `ZiskStdin`, `GuestProgram`, proof requests, hints, or streams |
| Precompiles and wrappers | ZisK library docs pages; patched crates in `Cargo.lock` | Checking whether crypto is accelerated or falling back to software |
| Hints and Assembly | ZisK SDK hints docs; Assembly executor/source | Optimizing precompile witness generation or debugging hinted proofs |
| Cost and profiling | ZisK profiling docs; `ziskemu -X`; emulator stats source | Interpreting TOTAL, MAIN, MEMORY, PRECOMPILES, named operation rows, or function attribution |
| Prover internals | Pinned ZisK/proofman source | Explaining instance jumps, recursion cost, memory, OOM, or proof-time mismatch |

## Key Patterns with Examples

### Cost Report

```text
Input: <range/sample, block/tx/gas/blob/channel counts, stdin size>
ZisK: <cargo-zisk version>, <ziskemu version>, <source commit if known>
Guest: <crate>, <ELF path/hash>, <stdin path/hash>
Baseline: steps=<n>, X_TOTAL=<n>, final_cost=<n if available>, time=<n if proved>
Top costs: <region/op/function table>
Finding: <repeated work>
Replacement: <authenticated summary>
Theorem: unchanged | changed | trust cut
Tests: <negative tests added/required; independent reference if available>
After: <same metrics on same input>
Residual risk: <unknowns>
```

Never report only `steps`. Include `-X TOTAL`, final planned/prover cost if available, and wall time only when a real proof ran.

### BASE / VARIABLE Cost

```text
TOTAL = BASE + VARIABLE(input)
VARIABLE per unit = (TOTAL_large - TOTAL_small) / (units_large - units_small)
```

When BASE dominates, report variable cost per block, transaction, channel, blob, byte, trie read, or proof. A step reduction that stays inside the same fixed instance boundary may not reduce final prover cost.

### Precompile Audit

```text
Cargo.toml/Cargo.lock patch present?
guest call reaches patched crate, project wrapper, zisklib, or syscall?
guest initializes the provider/factory used by that path?
ziskemu -X shows named operation rows, not only PRECOMPILES total?
Assembly/prover path uses the same accelerated route?
```

The docs describe `zisklib` hash and curve wrappers as safe wrappers over dedicated state machines. Prefer patched crates and wrappers before raw syscalls. Use named rows like `OP keccak`, `OP sha256`, curve operations, pairings, or Poseidon2 as evidence; the PRECOMPILES category total can include DMA and support work.

### Hints And Assembly

```text
native deterministic hint generation
  -> setup with --asm --hints or SDK Assembly .with_hints()
  -> Assembly execute/prove consumes hints source with --asm
  -> guest invokes the precompile path whose relation is still checked
```

Hints accelerate prover-side witness generation for precompile operations. They are not guest-visible proof facts and emulator execution is not evidence that the hinted Assembly path works. Verify command flags, setup mode, hint source size, and prover logs.

### Strategy Ladder

1. Delete duplicate work with the same predicate.
2. Use a patched crate, project wrapper, or `zisklib` wrapper instead of software crypto.
3. Redesign the wire format to avoid parse/copy/re-encode churn while preserving committed bytes.
4. Replace full scans with sparse inclusion plus committed completeness.
5. Replace repeated tree/hash work with multiproofs, dirty-branch caching, or lookup/memory consistency.
6. Move work to a child proof only when the parent verifies child identity, VK, range, config, inputs, outputs, and ordering.
7. Change the theorem only if the user explicitly accepts a named trust cut.

### Hot Loop Interrogation

Ask these questions for each top region:

```text
Are we proving every item but only need a subset?
Does a committed header/root/bloom/counter prove absence or completeness?
Is there a monotone counter that turns omission into arithmetic?
Does a child proof already own this fact?
Can zkVM memory/lookup consistency replace an in-guest authenticated data structure?
Can dirty-state caching recompute only changed branches without changing the committed root?
Can static data be pinned by config hash instead of re-read?
```

## Validation Commands

Use project wrappers when present. Otherwise start with:

```bash
cargo-zisk build --release
ziskemu -e <guest.elf> -i <input.stdin> --steps
ziskemu -e <guest.elf> -i <input.stdin> -X --no-thousands-sep
cargo-zisk execute --elf <guest.elf> --inputs <input.stdin>
cargo-zisk prove --elf <guest.elf> --inputs <input.stdin>
```

For proof-path behavior, use the relevant proving flags:

```bash
cargo-zisk prove --elf <guest.elf> --inputs <input.stdin> --gpu
cargo-zisk prove --elf <guest.elf> --inputs <input.stdin> --minimal-memory
cargo-zisk prove --elf <guest.elf> --inputs <input.stdin> --unlock-mapped-memory
```

For hints, validate Assembly/prover mode:

```bash
RUSTFLAGS='--cfg zisk_hints' cargo build --release
cargo-zisk setup --elf <guest.elf> --asm --hints
cargo-zisk prove --elf <guest.elf> --asm --inputs <input.stdin> --hints <hints-file>
```

Command names and flags drift. Check `<tool> --help` and the pinned source before treating these as exact.

## Constraints

### MUST DO

- State the proof theorem and trust boundary before optimizing.
- Rank regions with measured data from the same ELF/input.
- Report `steps`, `-X TOTAL`, final cost if available, proof time if actually proved, and input shape.
- Separate BASE from VARIABLE when fixed cost dominates.
- Use named operation rows or prover logs as acceleration evidence.
- Verify hints in Assembly/prover flow, including setup with `--asm --hints`.
- Treat missing witness as failure.
- Add negative tests for every removed loop or shortcut.
- Prefer an independent reference implementation for differential checks when available.

### MUST NOT DO

- Compare costs across different inputs without naming the input delta.
- Optimize a sub-5% region unless it removes risk or simplifies the theorem.
- Infer crypto acceleration from dependency names or PRECOMPILES total alone.
- Claim hints, GPU, recursion, or proof success from emulator-only evidence.
- Treat hints, fcalls, pass-through data, or host files as truth without in-guest verification.
- Replace ordered semantics with only a commutative accumulator.
- Accept host-provided roots, missing witness defaults, or benchmark guests on production verifier paths.
- Change the theorem silently while presenting the result as an optimization.

## Output Templates

When proposing an optimization, provide:

1. Baseline input and exact measurement commands
2. Top measured costs
3. Repeated work being removed
4. Authenticated replacement and theorem impact
5. Implementation scope
6. Negative tests
7. Before/after metrics on the same input
8. Residual risks and unmeasured paths

## Knowledge Reference

ZisK zkVM cost model, `ziskemu -X`, steps, TOTAL, BASE, VARIABLE, MAIN, MEMORY, PRECOMPILES, named operation rows, FROPS, fcalls, `zisklib`, patched crates, raw syscalls, hints, Assembly executor, GPU proving, `cargo-zisk`, witness layout, public outputs, input slicing, wire-format redesign, sparse inclusion, committed completeness, multiproofs, dirty-branch caching, child-proof welds, recursive aggregation, verifier binding.
