# zisk-skills

A set of four agent skills for building on the [ZisK zkVM](https://github.com/0xPolygonHermez/zisk). Each skill is a compact working method that an AI coding agent loads on demand: one for writing guest and host code, one for reducing proving cost, one for reasoning about the prover's internals, and one for auditing soundness. The skills were written for [Claude Code](https://claude.com/claude-code), but a `SKILL.md` file is ordinary Markdown, so they work in any agent that can load skill files.

<img width="2560" height="1280" alt="zisk-skills-monument" src="https://github.com/user-attachments/assets/c18cf5ae-133e-4416-b05a-e8cd5f36f649" />

## Why these exist

The official ZisK documentation is good at showing you the happy path, but an agent working on real ZisK code keeps running into questions the documentation does not answer. Why did a one-line change make the proof cost jump by a fixed amount? Why is the crypto in a guest running as slow software when the zkVM has a precompile for it? Is an aggregation guest actually binding the child proofs it claims to verify, or is it trusting whatever the host hands it? These skills capture the answers as reusable method, so an agent solves the problem instead of rediscovering it each time.

Every skill follows the same design rule: it teaches a way of working and points at where the truth lives in the ZisK source tree, and it deliberately avoids hard-coding facts that go stale between releases, such as exact function signatures, cost constants, or file-and-line references. When a skill and the source tree disagree, the source tree wins, and each skill says so explicitly. This keeps the skills useful across ZisK versions instead of rotting the moment the toolchain moves.

## The four skills

| Skill | What it does | Reach for it when |
| --- | --- | --- |
| [`zisk-developer`](skills/zisk-developer/SKILL.md) | Writes and reviews guest and host code the source-first way: pin the exact tree, start from a working example, make the input and output contract explicit, choose the cheapest implementation that works, and validate with a real run. | You are writing or reviewing any ZisK guest, host, or prover code. |
| [`zisk-optimizer`](skills/zisk-optimizer/SKILL.md) | Reduces proving cost without changing what the proof asserts: state the theorem, measure the real bottleneck, prove that a path is accelerated by reading the named operation rows rather than a summary total, and re-measure on the same input. | You need to cut steps, cost, or proof time, or you want to confirm whether crypto is genuinely accelerated. |
| [`zisk-internals`](skills/zisk-internals/SKILL.md) | Explains and predicts the prover's behavior from source: why cost rises in discrete jumps rather than smoothly, why memory scales the way it does, where each number in the profiler comes from, and how the recursion tree is shaped. | A small change caused a large cost jump, proving memory does not match the emulator numbers, or you need to size hardware. |
| [`zisk-soundness`](skills/zisk-soundness/SKILL.md) | Audits whether a proof actually binds the theorem it claims to: checks that every host value, hint, syscall result, and child proof has an owner, and that composition, ordering, counts, and verifier binding are all enforced. | You are reviewing aggregation, recursion, an on-chain verifier, direct hint or syscall use, or anything whose proof another party will trust. |

## How they were validated

The skills were not written from intuition and shipped. Each one was tested against fresh guest programs that the skill had never seen, and the results were checked against ground truth measured directly from the emulator and the source tree.

The developer and optimizer skills were exercised across four distinct optimization strategies: swapping a software primitive for a precompile, redesigning the input wire format, replacing in-guest recomputation with witness verification, and accelerating elliptic-curve cryptography. The measured wins ranged from roughly eight times fewer steps to several thousand times fewer, and every result was gated by comparing the committed output byte-for-byte against a baseline and by running negative tests.

The skills were also put through controlled A/B trials: the same task, given to the same model, once with the skill loaded and once without. The consistent finding is that a capable model finds the answer either way, so the skill does not add raw problem-solving power. What the skill adds is discipline and structure. In one optimization trial the skill prevented a headline that reported a large improvement in raw step count while hiding that the improvement in actual proving-area cost was sixteen times smaller. In a soundness trial the skill produced a reusable audit artifact, an input-ownership map with a named negative test for every value, rather than a wall of prose. The token cost of loading a skill is close to neutral, and in several trials the added structure made the run slightly cheaper because the agent explored less before converging.

The internals and soundness skills were distilled from a line-by-line reading of the ZisK source, covering the execution core, the instance planners, the recursion and aggregation machinery, the guest runtime, and the SDK and verifier, together with a complete pass over the official documentation. That reading is what lets these two skills explain the mechanisms the documentation never states.

## Installing in Claude Code

Clone the repository and copy the skills into your user skills directory:

```bash
git clone https://github.com/amiabix/zisk-skills
cp -r zisk-skills/skills/* ~/.claude/skills/
```

To scope the skills to a single project instead, copy them into that project's `.claude/skills/` directory. Claude Code discovers the skills automatically once they are in place. You can invoke a skill explicitly by name, for example `/zisk-optimizer`, or you can let the model select the right one based on the descriptions in each skill's front matter.

## A note on versions

The skills were validated against the ZisK 1.0.0-alpha line. Because they are built from principles rather than pinned facts, they should carry forward across releases, but the constants and APIs they point you toward live in your own pinned ZisK tree, not in the skill text. The skills are written to account for this: they instruct the agent to verify every API at its definition site and to prove version compatibility by building and running rather than by trusting a version label. That habit is the upgrade path.

## Contributing

Contributions are welcome, and the most valuable ones are field reports: a case where a skill's guidance was wrong, out of date, or silent on something it should have covered. The bar for adding content to a skill is that it must be a principle or a pointer to where truth lives, verified against the source tree, rather than a fact that a release bump could quietly invalidate.

## License

MIT. See [LICENSE](LICENSE).
