# Current Source Drift

Use this reference when public docs, examples, memory, and the target repo disagree.

## Rule

Prefer the target repository's pinned source over external docs unless the task explicitly targets a different ZisK tag, branch, or SDK generation.

## Drift Pattern

ZisK moves quickly. API names and helper chains seen in old docs or examples may be absent, renamed, or moved in the installed tree.

Common names to verify before using:

- `GuestProgram`
- `register_program!`
- `ProverClient::default()`
- chained calls such as `client.prove(...).stark().submit()?`
- guest I/O helpers such as `read_slice`, `read_vec`, `read_bytes`, `commit`, `commit_slice`, and `write`

Do not assert any of these exists or does not exist from memory. Check the pinned tree.

## Where To Check

| Question | Source |
| --- | --- |
| Guest I/O and commit semantics | `ziskos/entrypoint/src/io.rs` |
| Entrypoint and guest runtime exports | `ziskos/entrypoint/src/` |
| Host/prover builders and proof modes | `sdk/src/` |
| CLI subcommands and flags | `cli/`, `<tool> --help` |
| Working examples | `examples/` |
| Host stdin framing | `common/src/io/` and the target repo's host code |

For wire-format questions, inspect both sides:

1. Host writes: stdin builder / project host command.
2. Guest reads: `io.rs` plus guest call order.
3. Actual bytes: hexdump a real stdin if framing is unclear.

## How To Handle Drift

When drift exists:

1. Say the docs/source differ.
2. Name the exact files that establish current behavior.
3. Recommend one surface for the task.
4. Prove compatibility with build + run, not version-label matching.

## Senior-Engineer Standard

Do not say "ZisK supports X" unless you can point to implementation, a current example, or the target repo's wrapper code. If you cannot validate it, say it is unverified.
