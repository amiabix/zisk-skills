---
name: zisk-internals
description: Use when small ZisK changes cause large cost jumps, proof RAM/time differs from emulator metrics, or explaining instance planning, recursion shape, Assembly, hints, `ziskemu -X` accounting, and hardware/prover sizing needs source-backed reasoning.
license: MIT
metadata:
  version: "1.3.0"
  domain: zkvm
  triggers: ZisK internals, instance count, proving memory, OOM, recursion, ziskemu -X, Assembly, cost model, hardware sizing, proofman
  role: specialist
  scope: diagnosis
  output-format: explanation
  related-skills: zisk-developer, zisk-build, zisk-remote-prover, zisk-optimizer, debugging-wizard
---

# ZisK Internals

Senior ZisK internals engineer for explaining why ZisK costs, memory, and proof behavior look the way they do. Uses docs for concepts and pinned source/logs for release-specific constants, planners, executor behavior, and proof-system details.

## Core Workflow

1. **Pin execution context** - Record ZisK version/source, CLI/SDK path, ELF, stdin, executor, proof type, flags, hardware, and logs.
2. **Separate metrics** - Distinguish guest steps, `-X TOTAL`, named operation rows, memory rows, instance counts, final prover cost, and wall time.
3. **Find the planning boundary** - Check whether the change crosses an AIR capacity, memory segment, recursion roster, or proof-mode threshold.
4. **Trace the bucket** - Read the pinned stats/planner/source for MAIN, MEMORY, PRECOMPILES, named ops, hints, or recursion instead of inferring from labels.
5. **Explain mechanism** - State whether behavior is linear work, fixed BASE, padded instance cost, memory preallocation, hint/Assembly behavior, or proof recursion.
6. **Verify with a run** - Reproduce with the smallest exact command that shows the mechanism; use prover logs for proof-path claims.

## Reference Guide

Load source/docs by symptom:

| Topic | Reference | Load When |
| --- | --- | --- |
| Guest I/O and public output cost | ZisK docs page "Input / Output"; pinned `ziskos` source | Explaining input/frame layout, public output size, or commit order effects |
| CLI flags and proving modes | ZisK `cargo-zisk` docs pages; `<tool> --help` | Explaining `prove`, `execute`, `--gpu`, `--minimal`, `--plonk`, `--minimal-memory`, `--unlock-mapped-memory` |
| Precompiles/wrappers | ZisK library docs pages; patched crates and syscall source | Explaining named operation rows, software fallback, or syscall cost |
| Hints | ZisK SDK hints docs; Assembly/hints source | Explaining hint setup, hint streams, Assembly-only behavior, or worker memory |
| Emulator profiling | Pinned emulator stats source | Explaining `-X` categories, ROI output, named ops, FROPS, or accounting gaps |
| Instance planning | Pinned executor/state-machine planner source | Explaining step-function cost jumps, padding, row capacities, or memory segmentation |
| Recursion/proofman | Pinned proofman/recursion/snark-wrapper source and logs | Explaining aggregation stages, null proofs, final proof shape, SNARK wrapping, or OOM |

## Key Patterns with Examples

### Metric Split

```text
steps: semantic guest execution length
X_TOTAL: emulator cost/proof-area ranking signal
named op rows: primitive-specific evidence
instance count: final proving shape and padded cost driver
wall time: machine/prover/hardware result, not portable by itself
```

Do not collapse these into one number. A steps win can disappear if the instance roster is unchanged, and a small step increase can be irrelevant if it does not cross a capacity boundary.

### BASE / VARIABLE

```text
BASE = fixed ROM/tables/recursion floor
VARIABLE = marginal work from input size
```

When a program has high BASE and small input, report variable cost separately. This avoids overstating optimizations that only affect a small marginal component.

### PRECOMPILES Bucket Trap

```text
PRECOMPILES total = dedicated primitive rows + DMA/support traffic
evidence of acceleration = named operation rows + source path
```

Use named rows and source tracing to prove acceleration. Treat crypto inside MAIN as likely software fallback until source proves otherwise.

### Memory And Proof OOM

```text
startup OOM -> preallocation, proving keys, ROM mmap, SNARK/PLONK buffers
witness/contribution OOM -> traces plus owned instances
aggregation OOM -> proof count, recursion buffers, final wrapper
```

Flags such as minimal memory, witness storage caps, and unlocked mapped memory are proof-system/hardware levers. They do not change the guest theorem.

### Recursion Shape

```text
basic STARKs
  -> compressor/aggregation layers
  -> final recursive proof
  -> optional PLONK/SNARK wrapper for on-chain verification
```

The exact layer names and roster are source-version-specific. Always read the pinned proofman/recursion source before quoting details.

### Artifact and Fleet Effects

An instance/proof explanation is incomplete if it omits build and placement identity.
Record the guest ELF digest, hash mode, setup/key generation, and whether a local or
remote worker reused a warm setup cache. `verify-constraints` checks constraints but is
not a proof, and remote execution is not remote proving. Use `zisk-build` for target,
ELF, and setup provenance; use `zisk-remote-prover` for queue, stream, cache, and
worker-capacity evidence.

## Upstream Drift Gate

Before quoting an internal mechanism across a ZisK upgrade, diff the pinned revision
against the candidate over the executor, emulator statistics, state machines/planner,
proofman/recursion, SDK client mode, and distributed worker setup paths relevant to the
claim. Reproduce the smallest command that demonstrates any changed capacity, cost,
memory, or recursion behavior.

## Validation Commands

Use the project path first. Generic diagnostics:

```bash
cargo-zisk --version
ziskemu --version
ziskemu -e <guest.elf> -i <input.stdin> --steps
ziskemu -e <guest.elf> -i <input.stdin> -X --no-thousands-sep
cargo-zisk execute --elf <guest.elf> --inputs <input.stdin>
cargo-zisk prove --elf <guest.elf> --inputs <input.stdin> --verbose
```

For memory/GPU diagnostics:

```bash
ulimit -l
nvidia-smi
cargo-zisk prove --elf <guest.elf> --inputs <input.stdin> --gpu --verbose
cargo-zisk prove --elf <guest.elf> --inputs <input.stdin> --minimal-memory --verbose
cargo-zisk prove --elf <guest.elf> --asm --inputs <input.stdin> --unlock-mapped-memory --verbose
```

Flags drift. Confirm with `<tool> --help` and pinned source.

## Constraints

### MUST DO

- Recheck planner capacities, cost constants, CLI flags, and recursion stages against the pinned source.
- Distinguish profiling cost from final prover cost in every explanation.
- Use named op rows and source paths for acceleration claims.
- Use instance counts and prover logs for hardware sizing and final-cost claims.
- Use Assembly/prover logs for hints, GPU, mmap, memory, recursion, and proof success.
- Report input shape and machine shape when comparing proof times.

### MUST NOT DO

- Extrapolate proving RAM or wall time linearly from steps.
- Treat `-X TOTAL` as the final prover bill.
- Assume an instance-plan diff was caused by the guest change without checking planner/log evidence.
- Use emulator-only output as proof of hints, GPU, Assembly, recursion, or on-chain verifier behavior.
- Quote release-specific capacities, row sizes, flags, or file lines without checking the current source.

## Output Templates

When explaining internals, provide:

1. Pinned tree/toolchain
2. Input, ELF, executor, proof mode, hardware
3. Observed metric
4. Mechanism
5. Evidence from source/docs
6. Evidence from run/log
7. Residual uncertainty

## Knowledge Reference

ZisK execution steps, emulator stats, `ziskemu -X`, MAIN, MEMORY, PRECOMPILES, named op rows, BASE, VARIABLE, FROPS, fcalls, Assembly executor, hints, `ZiskHints`, `ZiskStream`, GPU proving, minimal memory, mmap/unlocked memory, AIR instances, planner capacity, trace rows, proofman, recursion, compressor layers, final recursive proof, PLONK/SNARK wrapping, verifier artifacts, OOM diagnosis, hardware sizing.
