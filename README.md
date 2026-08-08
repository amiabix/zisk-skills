# zisk-skills

Agent skills for [ZisK zkVM](https://github.com/0xPolygonHermez/zisk) development, battle-tested working methods for writing, optimizing, understanding, and auditing ZisK guest programs and proving pipelines.

Built for [Claude Code](https://claude.com/claude-code) skills; the SKILL.md format is plain markdown and portable to any agent harness that loads skill files.

<img width="2560" height="1280" alt="zisk-skills-monument" src="https://github.com/user-attachments/assets/c18cf5ae-133e-4416-b05a-e8cd5f36f649" />


## The skills

| Skill | What it does | Use when |
| --- | --- | --- |
| [`zisk-developer`](skills/zisk-developer/SKILL.md) | Source-first guest/host development: pin the tree, start from a canonical example, fix the I/O contract, implement at the lowest friction level, validate with real runs | Writing or reviewing any ZisK guest or host code |
| [`zisk-optimizer`](skills/zisk-optimizer/SKILL.md) | Measured, theorem-preserving cost reduction: state the theorem, measure, prove acceleration by named op rows, climb the strategy ladder, re-measure on the same input | Reducing steps/cost/proof time, auditing precompile wiring |
| [`zisk-internals`](skills/zisk-internals/SKILL.md) | The mechanism layer: why cost jumps in discrete instances, why RAM scales, where every `-X` number comes from, how the recursion tree composes | Explaining/predicting proving cost or memory, debugging OOMs and step-function jumps |
| [`zisk-soundness`](skills/zisk-soundness/SKILL.md) | Soundness auditing: fcall hint verification, syscall preconditions, composition welds (VK/program/publics binding), verifier-side binding, negative tests | Reviewing aggregation/recursion, on-chain verifier wiring, anything whose proof someone else will trust |

## Design philosophy

These skills are **principle graphs, not documentation mirrors**. They teach the working method and point at where truth lives in the ZisK source tree; they deliberately avoid embedding version-specific facts (API signatures, cost constants, file:line references) that rot on every release. When a skill and the source disagree, the source wins, the skills tell you to check.

They were built by field-testing against real guests and measuring the outcome:

- The developer + optimizer skills were validated on fresh guest programs across four optimization strategy levels (precompile swaps, wire-format redesign, recompute-to-witness-verify, curve crypto) with wins from 8.5× to ~20,000× steps, every result gated by differential outputs and negative tests.
- In A/B trials (same task, same model, with vs. without skills), the skills' measured value was **reporting correctness and fail-closed testing discipline**, e.g. preventing a steps-only headline that overstated a real proving-area win by 16×.
- `zisk-internals` and `zisk-soundness` distill a line-level read of the ZisK source (execution core, planners, proofman recursion, guest runtime, SDK/verifier/contracts) and a full sweep of the official docs, capturing exactly the load-bearing knowledge the docs don't teach.

## Install (Claude Code)

Copy the skills into your user skills directory:

```bash
git clone https://github.com/amiabix/zisk-skills
cp -r zisk-skills/skills/* ~/.claude/skills/
```

Or per-project: copy into `<repo>/.claude/skills/`.

Claude Code auto-discovers them; invoke explicitly with `/zisk-developer`, `/zisk-optimizer`, `/zisk-internals`, `/zisk-soundness`, or let the model pick them up from their descriptions.

## Versioning caveat

Skills are validated against the ZisK 1.0.0-alpha line. Because they are principle-based they should survive releases well, but constants and APIs they route you toward live in *your* pinned ZisK tree, the skills' own rules (verify at the definition site, prove compatibility by build + run) are the upgrade path.

## Contributing

Improvements welcome, especially field reports where a skill's guidance was wrong or silent. The bar for adding content: it must be a principle or a pointer, verified against source, not a fact that a release bump can silently invalidate.

## License

MIT
