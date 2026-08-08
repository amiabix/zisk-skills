# Host And Prover

Use this reference when writing or reviewing host-side ZisK code.

## Rule

Do not copy host/prover API chains from memory or from this file. Pin the ZisK tree, then read the matching SDK definitions and examples.

## Source Order

1. Target repo host binaries and `build.rs`
2. ZisK `examples/`
3. `sdk/src/`
4. `cli/` and `<tool> --help`
5. Docs

## What To Establish

Before code:

- How the guest ELF is built or embedded.
- Which host binary or CLI command owns setup/execute/prove/verify.
- How stdin is framed and whether it is reusable or consumed.
- Which backend is selected: emulator, Assembly, distributed, GPU, SNARK wrapper.
- Which proof artifact is required: execution result, constraints, STARK, compressed proof, SNARK, or verifier calldata.

## API Verification Checklist

Verify these at definition sites in the pinned tree:

- ELF inclusion/build helper.
- Stdin type and write/clone/consume semantics.
- Client/builder construction and whether any singleton/process-global guard exists.
- Setup return type.
- Execute/verify-constraints/prove method names.
- Proof mode selectors and defaults.
- Proof result, public values, VK accessors, save/load helpers.
- SNARK/aggregation key loading behavior.
- Memory/proof options.

If an example and SDK disagree, trust the SDK definition and update the example-derived plan.

## Backend Rules

- Use emulator for correctness/debugging unless the task is specifically about performance, hints, Assembly, GPU, or proof generation.
- Use Assembly for performance-sensitive proof behavior; emulator success does not prove Assembly success.
- Show GPU usage from invocation/logs, not hardware presence.
- Treat SNARK wrapping and recursive aggregation as separate stages unless the target host binary explicitly combines them.

## Validation Path

Use the cheapest check that proves the claim:

- Build: API compatibility.
- Execute: guest semantics and stdin contract.
- Verify constraints: circuit/witness sanity when available.
- Prove: full proof generation.
- Verify: proof artifact and public input sanity.

Always report the exact level validated.

## Output

Report:

```text
Pinned SDK/source:
Host entry point:
Guest ELF path/build path:
Stdin contract:
Backend/proof mode:
Commands run:
Artifacts produced:
Unvalidated stages:
```
