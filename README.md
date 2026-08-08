# zisk-skills

Four agent skills for building on the [ZisK zkVM](https://github.com/0xPolygonHermez/zisk): one for writing guest and host code, one for reducing proving cost, one for reasoning about the prover's internals, and one for auditing soundness. Written for [Claude Code](https://claude.com/claude-code), but a `SKILL.md` file is plain Markdown, so they work in any agent that loads skill files.

<img width="2560" height="1280" alt="zisk-skills-monument" src="https://github.com/user-attachments/assets/c18cf5ae-133e-4416-b05a-e8cd5f36f649" />

Each skill teaches a working method and points at where the truth lives in the ZisK source tree, rather than hard-coding facts that go stale between releases. When a skill and the source disagree, the source wins.

## The skills

| Skill | Reach for it when |
| --- | --- |
| [`zisk-developer`](skills/zisk-developer/SKILL.md) | Writing or reviewing any ZisK guest, host, or prover code. |
| [`zisk-optimizer`](skills/zisk-optimizer/SKILL.md) | Cutting steps, cost, or proof time, or checking whether crypto is genuinely accelerated. |
| [`zisk-internals`](skills/zisk-internals/SKILL.md) | A small change caused a large cost jump, proving memory surprises you, or you need to size hardware. |
| [`zisk-soundness`](skills/zisk-soundness/SKILL.md) | Reviewing aggregation, recursion, a verifier, or anything whose proof another party will trust. |

## Install

```bash
git clone https://github.com/amiabix/zisk-skills
cp -r zisk-skills/skills/* ~/.claude/skills/
```

Or copy into a project's `.claude/skills/` to scope them to that repo. Claude Code discovers them automatically; invoke a skill by name (for example `/zisk-optimizer`) or let the model pick one from its description.

## License

MIT. See [LICENSE](LICENSE).
