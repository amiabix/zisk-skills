---
name: zisk-build
description: Use when installing or upgrading ZisK, building a guest ELF, configuring the ZisK Rust target/linker, generating setup artifacts or keys, diagnosing setup/cache mismatches, or preparing Assembly/proving material.
license: MIT
metadata:
  version: "1.0.0"
  domain: zkvm
  triggers: ZisK build, ziskup, cargo-zisk build, ziskbuild, guest ELF, RISC-V target, Rustflags, linker, rom-setup, proving key, verifier key, setup, check-setup, verify-constraints
  role: specialist
  scope: build-and-setup
  output-format: implementation-report
  related-skills: zisk-developer, zisk-remote-prover, zisk-internals, zisk-soundness
---

# ZisK Build and Setup

Use this skill to preserve the build-to-proof identity chain. A ZisK proof is about
the exact guest ELF and setup material, not the source directory name, Cargo package,
or a locally remembered tool version.

## Scope and Routing

Own toolchain installation, target and linker selection, guest ELF inspection,
setup/key material, artifact caching, and proof-mode selection. Hand guest semantics,
I/O, hints, and precompiles to `zisk-developer`; cluster transport to
`zisk-remote-prover`; and any accepting verifier to `zisk-soundness`.

Use this skill before changing a Rust target, linker flag, `RUSTFLAGS`, ziskup bundle,
setup directory, proving key, hash mode, Assembly material, or an artifact cache.

## Build Identity Record

Before troubleshooting or generating a production artifact, record:

```text
upstream revision / release and cargo-zisk, ziskup, rustc versions
guest source revision and Cargo.lock
guest package, features, profile, target, linker and effective Rust flags
ELF path + SHA-256/BLAKE3 as the project convention requires
hash mode; executor/proof mode; setup/proving-key/verifier-key locations and hashes
```

Do not identify a program by a filename. A feature, profile, linker, toolchain, or
dependency patch change can make a different ELF with the same name.

## Correct Build Path

1. Pin a ZisK release or source commit. Obtain its matching tooling and setup bundle
   through the upstream-supported installer/commands; inspect `ziskup --help` at that
   revision instead of copying an installer recipe from another release.
2. Build through the ZisK guest build path. Its target is
   `riscv64ima-zisk-zkvm-elf`; `ziskbuild` owns the target features, linker script,
   and effective Rust-flag resolution. Do not replace it with a generic RISC-V target
   just because the program links.
3. Inspect the resulting ELF before setup. Confirm it is the requested release ELF,
   has loadable segments within the ZisK memory windows, and has not silently used a
   host linker or host-only configuration.
4. Generate or locate setup material for *that exact ELF and hash mode*. `rom-setup`
   content-addresses program material by ELF and hash mode; reuse is valid only when
   both match.
5. Select the operation deliberately:

   | Need | Appropriate evidence |
   | --- | --- |
   | Fast program behavior | emulator / execute-only run |
   | Witness/prover compatibility | `verify-constraints` or execute with the intended backend |
   | Proof artifact | prove, then verify the output with the intended consumer |
   | EVM consumption | proving flow with PLONK wrapping; minimal proof mode is not that path |

`verify-constraints` validates witness/constraint consistency. It does **not** produce
or independently verify a cryptographic proof, and the current SDK exposes it only on
the embedded client. Never write “proof verified” based on that command.

## Flag and Linker Discipline

Treat the final command line as an input artifact. In particular:

- Check project `.cargo/config*`, build scripts, environment, and CI for flags that
  leak from a host build into the guest build.
- Prefer the ZisK build wrapper over manually assembling `--target`, `-C linker`,
  `-C link-arg`, or `target-feature` switches. If an override is unavoidable, compare
  it against the pinned `ziskbuild/src/lib.rs` behavior and preserve the linker script.
- Do not mix setup generated for a non-Assembly path with an Assembly/hints proof
  request. Hinted Assembly requires corresponding Assembly/hint setup material.
- When a build succeeds but execution fails, inspect target, `file`/`readelf` output,
  segment ranges, and effective flags before changing guest logic.

For non-Rust guests, use the upstream linker script and match ISA/ABI/code model.
The static-library C/C++ flow has additional runtime/allocator requirements; load
`zisk-ffi` rather than copying flags into a Rust workflow.

## Setup and Cache Lifecycle

Keep distinct names and locations for these artifacts:

```text
ELF -> ROM/setup material -> verifier material -> proving material -> proof artifact
                                      \-> optional SNARK/PLONK wrapping material
```

The exact directory layout and subcommands evolve. The current upstream has dev
surfaces such as `check-setup`, `program-setup`, `proofman-setup`,
`rebuild-witness-libs`, `setup-snark`, and compressed-final setup; inspect the pinned
CLI source and `--help` before automating any one command.

Invalidate or regenerate rather than patching an old cache after changing any of:

- ELF bytes, target, release/debug profile, Rust version, dependencies, or features;
- hash mode or executor/Assembly/hints mode;
- ZisK/proving backend revision, proving-key bundle, circuit/witness library, or
  recursion/PLONK setup generation.

An idempotent setup response is evidence that the system recognizes the same identity,
not proof that the intended source code was rebuilt. Compare the ELF digest first.

## Failure Triage

| Symptom | Check first |
| --- | --- |
| Host binary appears instead of guest ELF | target, `ziskbuild` invocation, linker, profile output path |
| Setup missing or wrong program | ELF digest, hash mode, setup root, installed bundle version |
| Witness/constraint failure after a harmless source change | actual ELF changed; regenerate setup and inspect target/features |
| Assembly/hints only fails | setup generated with Assembly+hints, hint cfg/features, exact hint source |
| PLONK/wrap failure | proof family, required SNARK setup, full committed public layout, consumer requirements |
| Cache “works” but proof is for old code | compare cached identity to new ELF digest; rebuild rather than trusting timestamps |

## Upstream Drift Gate

Before an upgrade, compare the previous verified revision with the candidate:

```bash
git fetch origin
git diff <last-verified-zisk-commit>..origin/main -- \
  ziskbuild rom-setup toolchain cli/src/commands/dev \
  cli/src/commands/user sdk/src/embedded sdk/src/setup
```

Re-read the changed command implementation when target triples, rustflag precedence,
linker scripts, artifact paths, hash mode, setup commands, witness libraries, or key
formats change. Record both revisions and whether every artifact was regenerated.

## Validation

Run the target repository's wrapper when it exists. Otherwise use the pinned command
help to form equivalents of:

```bash
cargo-zisk build --release
file <guest.elf>
readelf -lW <guest.elf>
cargo-zisk-dev check-setup --help
cargo-zisk-dev verify-constraints --help
cargo-zisk prove --help
```

For a release handoff, include the build identity record, ELF digest, setup/key
identity, exact command output, and the distinction between execution, constraints,
proof generation, and downstream acceptance.

## Must / Must Not

**Must:** pin the source and toolchain; use the supported guest target/build path;
bind setup to the ELF plus hash mode; regenerate dependent artifacts on identity
changes; validate the intended executor and proof consumer.

**Must not:** call a host build a guest ELF; hand-edit proving caches; treat
`verify-constraints` as a proof; reuse key/setup output solely because filenames or
timestamps match; assume current CLI flags from a different release.
