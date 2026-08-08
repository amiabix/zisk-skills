---
name: zisk-soundness
description: Audit ZisK zkVM guests, proof compositions, and verifier integrations for soundness holes - unverified fcall hints, unbound child proofs in aggregation, missing verifier-side binding, syscall precondition violations, and trust cuts hidden as optimizations. Use when reviewing proof aggregation or recursion, in-guest proof verification, on-chain verifier wiring, direct fcall/syscall usage, split-pipeline designs, or before shipping any guest whose proof someone else will trust.
---

# ZisK Soundness Auditor

A proof is a claim about a theorem. This skill audits whether the theorem actually binds everything it appears to. Slow bugs cost money; soundness bugs cost everything, audit before shipping, not after.

## The Core Question

For every value the guest uses: **who proved it?** Every input is one of: (a) committed input the theorem quantifies over, (b) recomputed in-guest, (c) a hint verified in-guest, or (d) trusted. Category (d) must be an explicit, named trust cut, never an accident.

## The Four Audit Surfaces

### 1. Hints (fcalls)

Fcall results are unconstrained free inputs: the VM proves the guest *received* the words, nothing about correctness. Soundness = the in-guest check after the call (which is then in-circuit). The stock zisklib wrappers verify every hint (inverses re-multiplied, sqrts squared including the non-residue branch, division via Euclid's lemma, decompositions recomposed). **Direct `fcall_*` use without an equivalent check is silent unsoundness.** Audit rule: for each fcall call site, name the verifying computation; if you can't, it's a hole.

### 2. Syscall preconditions

Precompile circuits do not validate your inputs for you. Curve-add syscalls assume finite, on-curve, canonical points and distinct non-inverse operands; alignment is mandatory; some results are canonical only if documented so. Audit rule: at each raw syscall site, list the preconditions from the syscall's doc/source and show where each is established, by prior check, by construction, or not at all.

### 3. Composition (the welds)

In-guest proof verification checks internal consistency of a proof blob **against the VK material the blob itself carries**. A sound aggregator must add the welds itself:

- **VK weld**: the blob's aggregation VK equals a known-good constant compiled into the parent, not read from the blob.
- **Program weld**: the child's program VK equals the expected child program, not whatever the host supplied.
- **Publics weld**: child publics are read, checked, and bound into the parent's own committed outputs; unread child publics are unproven claims.
- **Ordering/range/config welds**: sequence, block range, chain/config identity, anything the parent's theorem quantifies over must be checked, not inferred from input order.
- **Key-family weld**: in-guest verification is pinned to a specific hash family/proving-key generation; a mismatch fails closed, but plan key management around it.

Treat official aggregation examples as mechanics demos, not templates, verify which welds they actually make before copying.

### 4. Verifier-side binding

The proof system commits to: program identity (ROM root), a small fixed budget of public outputs, and the recursion VK. Everything else binds only if a public output binds it. Audit rules: the on-chain/consumer contract must pin the program VK itself (the verifier checks whatever it is handed); outputs are a hard small budget, so commit digests/roots, never truncate silently; any host-side "publics decode" must match the guest's commit order and types exactly.

## Unsafe Classes (reject on sight)

- Host-provided post-states or roots accepted without in-guest derivation or authenticated verification.
- Missing witness interpreted as absence ("no data = no violation").
- Skipping a binding (signature, KZG, receipt/log, blockhash, state root) because another stage "probably" checks it.
- Ordered semantics replaced by a commutative accumulator alone.
- Benchmark-only guests reachable from a production verifier path.
- A trust cut introduced by an optimization without a name, an owner, and the user's explicit acceptance.

## Method

1. Write the theorem: private inputs, public outputs, who consumes them, and the trust boundary.
2. Classify every guest input into (a)-(d) above; demand a named check for every (c) and a named acceptance for every (d).
3. Walk the welds table for every proof-in-proof edge.
4. For each check, write the smallest **negative test**: forge, omit, reorder, or default the value and confirm the guest aborts / the verifier rejects. A check without a failing test is a hypothesis.
5. Report: theorem, holes found (with the forged input that exploits each), welds verified, trust cuts accepted, tests added.

## MUST

- Verify claims at the definition site of the verifier/wrapper actually in use, trust models change between releases.
- Fail closed: every audit finding gets a negative test that currently fails (or a written trust-cut acceptance).
- Audit the composition even when each component is individually sound, the welds are where systems break.

## MUST NOT

- Accept "the example does it this way" as soundness evidence.
- Let an optimization change the theorem silently.
- Sign off on a pipeline whose child-proof VKs, program identities, or publics are supplied by the host without in-guest comparison against expected constants.
