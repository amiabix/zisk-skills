---
name: zisk-optimizer
description: Optimize ZisK zkVM guests and proving pipelines by measuring real cost, verifying precompile/Assembly/hints wiring, and proposing theorem-preserving reductions. Use when asked to reduce ZisK steps/cost/proof time, explain `ziskemu -X` output, audit whether crypto is precompile-accelerated or falling back to software, debug Assembly/GPU proving behavior, compare guest designs, or find smarter proof shapes without weakening soundness.
---

# ZisK Optimizer

## Overview

Use this skill to optimize a ZisK program without guessing. The workflow is: state the proof theorem, measure current cost, prove which paths are accelerated, then replace recomputation with cheaper authenticated verification only when the same statement still holds.

## Core Workflow

1. **Pin the tree and input** — Record ZisK version, source revision, ELF, stdin, feature flags, host command, and input shape.
2. **State the theorem** — Name private inputs, public outputs, verifier/binder consumers, and trust boundary before proposing optimizations.
3. **Measure first** — Run emulator/profile and project host cost commands on the same input. Rank regions before touching code.
4. **Prove acceleration** — Trace patches/wrappers/syscalls in source and confirm named operation rows in `ziskemu -X`.
5. **Find the loop** — Identify the repeated primitive, parser, tree walk, copy, or witness read behind the top region.
6. **Name the authenticated replacement** — Replace work only with a committed root, header field, monotone counter, child proof, lookup/memory argument, dirty-cache invariant, or explicit trust cut.
7. **Validate** — Add the smallest negative test for omission/forgery/reorder/default, prefer a reference implementation that shares no code with the guest when available, rerun the same benchmark, and report before/after.
8. **Micro-pass last** — After the theorem, tests, and metrics are correct, do one small pass for obvious loop/copy/allocation simplification. Do not trade clarity or soundness for tiny step wins.

## Non-Negotiables

- Treat the target repository and installed ZisK source as truth; docs are secondary.
- Never optimize before naming the exact public theorem and trust boundary.
- Never claim a path is accelerated unless source wiring and `ziskemu -X`/prover output confirm it.
- Never compare numbers from different inputs without naming the input delta.
- Never report only `steps`; report `steps`, `-X TOTAL`, final planned/prover cost if available, and wall time if a proof was run.
- Missing witness is failure, never default/zero/empty.
- Every removed loop needs an authenticated replacement: a header field, state root, commitment, child proof, monotone counter, lookup/memory argument, or explicit trust cut.

## First Pass

Before changing code, collect:

- ZisK version: `cargo-zisk --version`, `ziskemu --version`.
- Target ELF path, stdin path, guest crate, host command, feature flags, and git status.
- Input shape: block/range/channel count, tx count, gas, witness size, blob count, or equivalent domain units.
- Current theorem: what private inputs are read, what public outputs are committed, and which verifier/binder consumes them.

Use these baseline commands when the project has normal ZisK artifacts:

```bash
ziskemu -e <guest.elf> -i <input.stdin> --steps
ziskemu -e <guest.elf> -i <input.stdin> -X --no-thousands-sep
```

If the project has its own host/prover binary, prefer that for final planned cost because project host code may report instance planning and proof-mode cost that `ziskemu -X` does not.

## Cost Model

Use ZisK cost terms precisely:

- `STEPS`: guest instruction count/cycles. Useful, but not sufficient.
- `BASE`: fixed cost paid even for tiny inputs. If this dominates, report variable cost separately.
- `VARIABLE`: marginal cost per domain unit: block, transaction, channel, blob, byte, trie read, proof, or application-specific unit.
- `TOTAL` from `ziskemu -X`: proof-area proxy and usually the best local ranking signal.
- `MAIN`: normal RISC-V guest execution; often hides parsing, tree walking, RLP, allocation, decompression, and interpreter work.
- `PRECOMPILES`: mixed bucket containing dedicated operation rows and DMA/copy effects. Do not use this category total alone to decide whether crypto is accelerated.
- `MEMORY`: zkVM memory argument and copying pressure; large witnesses and DMA show here.
- Final prover/instance cost: can jump at capacity boundaries, so small source changes may have zero or step-function impact.

Do not optimize a region below roughly 5% unless it also removes a soundness risk or simplifies the theorem.

## Precompile Audit

Verify acceleration in this order:

1. Check dependency patches/features in `Cargo.toml`: patched hash/curve/EVM crates, `native-keccak`, guest-only features, and target-specific cfgs.
2. Trace guest call sites: patched crate call, `ziskos::zisklib::*`, `zkvm_interface::*`, or raw syscall.
3. Confirm initialization: crypto provider/precompile factory must actually be installed in the guest path before execution.
4. Run `ziskemu -X`; key on named operation rows like `OP keccak`, `OP sha256`, `OP secp256k1_*`, `OP bn254_*`, `OP bls12_381_*`, `OP arith*_mod`, `OP poseidon2`, or the relevant primitive.
5. If an expensive crypto operation appears only inside `MAIN`, treat it as software fallback until source proves otherwise.

Do not infer acceleration from the `PRECOMPILES` category total. DMA and support machinery can pollute that bucket; named operation rows are the evidence.

Prefer the lowest-friction acceleration layer that works:

1. Existing patched crate.
2. Existing project wrapper.
3. `zisklib` wrapper.
4. Raw syscall.

Do not drop to raw syscalls unless wrappers are absent or measured insufficient.

## Hints And Assembly

Keep these distinct:

- Precompiles are circuit/state-machine acceleration visible in `ziskemu -X`.
- Hints are prover-side/native assistance and require the Assembly flow; do not claim hints from emulator-only tests.
- Fcalls are guest/native helper mechanisms that may not appear as named `-X` precompile rows; their results still need in-guest verification against the theorem.
- Assembly is the production-performance path and is Linux/x86_64-oriented; macOS emulator success does not prove Assembly success.
- GPU use must be shown by the worker/prover invocation and logs; do not infer GPU from hardware presence.

For Assembly issues:

- Record exact command, version, ELF hash/path, stdin hash/path, flags, machine RAM/GPU, and `ulimit -l`.
- If shared-memory mmap fails with locked-memory errors, try the project/CLI equivalent of unlocked mapped memory rather than changing guest logic.
- If the ZisK team asks for `legacy_mem_count_and_plan`, treat it as a diagnostic build of ZisK/proof machinery, not an application optimization.
- Reproduce with `execute` first, then `prove`; a guest that executes but fails in recursion/proving is usually not a guest semantic failure.

## Strategy Ladder

Stop at the first theorem-preserving rung that matters in the profile:

1. Delete duplicate work with the same predicate.
2. Use an existing patched crate/wrapper/syscall instead of software crypto.
3. Redesign wire format to avoid parse/copy/re-encode churn while preserving the same committed bytes.
4. Replace full scans with sparse inclusion plus committed completeness.
5. Replace repeated tree/hash work with multiproofs, dirty-branch caching, or lookup/memory consistency.
6. Move work to a child proof only when the parent verifies child identity and all welds.
7. Change the theorem only if the user explicitly accepts a trust cut.

Ask these questions for each hot loop:

- Are we proving every item but only need a subset?
- Does a committed summary prove absence or completeness?
- Is there a monotone counter that turns omission into arithmetic?
- Does a child proof already own this fact?
- Can zkVM memory/lookup consistency replace an in-guest authenticated data structure?
- Can dirty-state caching recompute only changed branches without changing the committed root?
- Can fixed or static data be pinned by config hash instead of re-read?

Safe classes:

- Pure deduplication with the same predicate.
- Obvious loop/copy/allocation simplification after correctness is proven.
- Negative proofs from committed blooms, counters, roots, or headers.
- Sparse inclusion plus committed completeness.
- Multiproofs/read-set manifests where missing reads fail closed.
- Incremental dirty-branch recomputation where unchanged branch hashes are authenticated.
- Moving work to a child proof only if the parent/binder checks child VK, range, config, input root, output root, and ordering.

Unsafe classes:

- Host-provided post roots.
- Missing witness interpreted as absence.
- Skipping signatures, KZG, receipt/log binding, blockhash binding, or state-root binding because another stage “probably” catches it.
- Replacing ordered semantics with only a commutative accumulator.
- Benchmark-only guests accepted on a production verifier path.

## Function-Level Profiling

Diagnose profiling failures from the exact error line, not a single rule. Empty function tables can mean stripped ELF, missing debug info, unsupported symbol format, wrong ELF, or a tool/version mismatch.

Fallbacks:

- Rebuild or ask for an unstripped profiling ELF.
- Run `ziskemu -X` without source/function attribution and use named operation rows.
- Add narrow inline profile tags such as `profile_report_start!(ident)` / matching stop macro around suspected regions when the project already supports them.
- If profile tags are not already in use, prefer a one-region temporary patch over broad instrumentation.

Useful profiling shape:

```bash
ziskemu -e <guest.elf> -i <input.stdin> -X -S -D -T 30 -C 20 \
  --roi-filter '<crate|function|primitive regex>' \
  --top-roi-filter --no-thousands-sep
```

Use narrow filters for suspected hot families: trie/MPT/RLP, decompression, hashing, signature recovery, serialization/deserialization, witness reads, alloc/copy, and recursive verification.

## Reporting Format

Report optimization findings as:

```text
Input: <exact range/sample, size, gas/tx/blob/channel counts>
ZisK: <cargo-zisk>, <ziskemu>, <source commit if known>
Guest: <crate>, <ELF path/hash>, <stdin path/hash>
Baseline: steps=<n>, X_TOTAL=<n>, final_cost=<n if available>, time=<n if proved>
Top costs: <region/op/function table>
Finding: <what repeats>
Replacement: <authenticated summary>
Theorem: unchanged | changed | trust cut
Tests: <negative tests required/added; independent reference used if available>
After: same metrics on same input
Residual risk: <unknowns>
```

If exact metrics are unavailable, say so and stop at a measurement plan. Do not fill gaps with vibes.
