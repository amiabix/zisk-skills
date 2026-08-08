---
name: zisk-internals
description: Reason about ZisK zkVM proving mechanics from source - what a step costs, why proving cost jumps in discrete increments, why RAM scales, how the recursion tree composes, and where each -X number comes from. Use when explaining or predicting proving cost/memory behavior, debugging OOMs or step-function cost jumps, interpreting instance counts, sizing hardware, or when ziskemu profiling numbers and real prover behavior disagree.
---

# ZisK Internals

Explains *why* ZisK costs behave the way they do, so optimization and debugging decisions rest on mechanism, not folklore. Everything here is a model to re-verify against the pinned source tree — constants drift between releases; the mechanisms drift slowly.

## The Three Mental Models

**1. Cost is quantized by instances, not accumulated smoothly.**
Execution splits into chunks; op counts pack into fixed-capacity AIR instances (each AIR's row count is hardwired in the generated trace definitions, `pil/src/pil_helpers/traces.rs`). One op past a capacity boundary materializes an entire padded instance *plus* its recursion legs. Profiling cost (`-X TOTAL`) moves linearly; real prover cost moves in steps. Predict real cost with an execute-only run's instance counts, not with TOTAL deltas. Beware planner nonlinearities: some state machines toggle on/off per run based on layout-size comparisons — instance-set diffs are not always caused by your code change.

**2. RAM scales through three mechanisms, all front-loaded.**
(a) Minimal traces (∝ total steps) stay resident for the whole proof; (b) witness for all owned instances materializes up front in the contributions phase; (c) prover buffers — and SNARK/Plonk buffers if preloaded — are preallocated at construction, before any proving. OOMs at startup implicate (c); OOMs during witness implicate (a)+(b); `minimal_memory` and per-block splitting attack (b).

**3. The recursion tree has a fixed shape.**
Basic STARKs → per-AIR Compressor → Recursive1 (embeds the aggregation VK) → k-ary Recursive2 tree (padded with null proofs) → VadcopFinal → optional compressed/SNARK wrap. Absent state machines are proven as null proofs, so the final circuit never changes shape — that fixed roster is part of the BASE floor you pay even for a trivial guest.

## Cost Accounting — where -X numbers come from

- A **step is one ZisK op**, not one RISC-V instruction (atomics and CSR ops expand; some patterns fuse to free ops). MAIN = steps × a per-step constant. OPCODES = per-op constants from the op table (`core/src/zisk_ops.rs` + cost files). PRECOMPILES = ops with input payloads **plus DMA traffic** — libc memcpy/memcmp/memset are globally replaced with DMA precompile ops, so this bucket is polluted by ordinary memory work; judge crypto acceleration only by named op rows. MEMORY prices 8-byte-aligned access cheap and everything else expensive; sub-8-byte access is never cheap.
- BASE is a fixed ROM+tables floor. When it dominates, report VARIABLE (= TOTAL − BASE) or your win/regression is distorted.
- FROPS: op instances whose operands fall in precomputed table ranges are discharged as lookups and **excluded from TOTAL** — constant-heavy code is cheaper than the raw op costs imply.
- Costs are proof-area units, not time. Fcalls cost zero at the op level; their price is the in-guest verification that follows.
- The stats collector has known accounting gaps (e.g., some precompiles' memory traffic uncounted, unaligned writes billed at read prices in some releases) — treat `-X` as a ranking signal, verified against the pinned emulator source when a bucket looks wrong.

## Reasoning Moves

1. **Predict a jump before making a change**: estimate the op-count delta for the dominating primitive against its AIR capacity; crossing a multiple = a new instance.
2. **Attribute an OOM**: at startup → preallocation (SNARK preload, prover buffers); during contributions/witness → resident traces + upfront witness; during aggregation → payload count × per-proof buffers.
3. **Explain "steps dropped but cost didn't"**: BASE floor, instance padding, or the win landed in an op that FROPS already absorbed.
4. **Get real cost cheaply**: run the execute-only/instance-plan path (no proving keys) and count instances per AIR.

## Where Truth Lives

| Question | Look in |
| --- | --- |
| Op table, opcodes, per-op costs, precompile params | `core/src/zisk_ops.rs`, cost constant files in `core/`/`emulator/` |
| Stat bucket computation, FROPS accounting | `emulator/src/stats/` |
| AIR row capacities (instance sizes) | `pil/src/pil_helpers/traces.rs` |
| Planners (packing, toggles, memory segmentation) | `state-machines/*/src/*_planner.rs`, `common/src/planner_helpers.rs` |
| Chunking, witness replay, execute-only mode | `executor/src/` |
| Recursion stages, null proofs, preallocation | the pinned pil2-proofman rev (`proofman.rs`, `recursion.rs`, `snark_wrapper.rs`) |
| Memory map, publics collection | `core/src/mem.rs`, `executor/src/adapters.rs` |

## MUST

- Re-verify capacities and cost constants against the pinned tree before quoting them — they are release-specific.
- Distinguish profiling cost (linear, best-case) from final prover cost (instance-quantized) in every report.
- Use named op rows, never the PRECOMPILES bucket total, as acceleration evidence.
- Use instance counts for hardware sizing and final-cost claims.

## MUST NOT

- Extrapolate proving RAM or time linearly from steps.
- Treat -X TOTAL as the prover's bill.
- Assume an instance-plan diff was caused by the guest change under test.
- Quote this file's mechanisms against a different ZisK generation without re-checking the source.
