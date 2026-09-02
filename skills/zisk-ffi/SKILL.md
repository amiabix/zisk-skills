---
name: zisk-ffi
description: Use when building ZisK guest programs with C or C++, integrating ziskos-staticlib, ziskclib, lib-c, raw ziskos ABI functions, custom allocators, callbacks, or cross-language proof verification buffers.
license: MIT
metadata:
  version: "1.0.0"
  domain: zkvm
  triggers: ZisK C, ZisK C++, ziskos-staticlib, ziskclib, lib-c, staticlib, FFI, extern C, custom allocator, linker script, callback, verify_zisk_proof_c
  role: specialist
  scope: ffi
  output-format: implementation-report
  related-skills: zisk-build, zisk-developer, zisk-soundness, rust-engineer
---

# ZisK FFI and Static Library

Use this skill whenever a value crosses the guest language or ABI boundary. The ABI
boundary is a theorem boundary: wrong ownership, alignment, framing, lifetime, or
return-code handling can turn a correct Rust routine into an unsound or crashing guest.

## Scope and Routing

Own C/C++ guest construction, `ziskos-staticlib`/`ziskclib`/`lib-c` integration,
export signatures, linker/runtime/allocator compatibility, and raw buffer handling.
Use `zisk-build` for the wider toolchain/setup chain; use `zisk-soundness` for proof
acceptance and public-claim binding; use `zisk-developer` for normal guest APIs.

## ABI Contract First

For every imported or exported function, write a contract before coding:

```text
symbol + exact upstream revision/signature
argument encoding, pointer alignment, length unit, mutability and nullability
caller allocation/ownership and lifetime; callee allocation/free behavior
return/status convention and error handling
guest/public effect; validation and negative cases
```

Copy neither a Rust signature nor a stale header into C. Read the pinned declaration,
wrapper/export macro, and the implementation's preconditions together. Regenerate or
review bindings after an upstream ABI change.

## Staticlib Runtime Model

`ziskos-staticlib` deliberately isolates ziskos's system allocator. Its exported
wrappers reset that private bump allocator after each host call. Therefore:

- A pointer returned from a ziskos call is valid only for the lifetime documented by
  that call; do not retain it across a later exported call or allocator reset.
- Host/C/C++ allocations must use the host program's persistent allocator. Do not free
  memory through a different allocator family, and do not let C++ static objects depend
  on the resettable ziskos heap.
- If `alloc-stats` is enabled, treat its peak as diagnostic output; it does not make
  retained pointers safe.
- Add a staticlib export by following the upstream wrapper pattern, including the exact
  signature and isolation behavior. Do not export the internal `#[no_mangle]` symbol in
  a way that clashes with the wrapper.

## C/C++ Guest Build Contract

The pinned staticlib README currently requires the ZisK linker script and a matching
RISC-V ISA/ABI; C++ examples use `rv64ima`, `lp64`, and `-mcmodel=medany`, with
freestanding/no-startfiles linking. Treat flags as source-verified examples, not an
evergreen copy-paste recipe.

When using C++:

- Link ZisK's linker script. It keeps code/read-only data and writable data in the
  guest memory ranges and supplies init/fini array bounds the staticlib runtime needs.
- C++ static constructors run before `main` and destructors after it, but the current
  runtime has concrete limits: no TLS/PT_TLS, no `.preinit_array`, legacy `.ctors`/
  `.dtors` are not a substitute, a fixed destructor-registration capacity, and no
  provided exception/RTTI runtime. Build freestanding with exceptions/RTTI disabled
  unless the project deliberately supplies and validates its own runtime.
- Do not use a custom linker script until it preserves the ZisK load windows and
  initializer-array semantics. A successfully linked ELF can still be rejected by the
  interpreter.

Load `zisk-build` for artifact identity and inspect the generated ELF after every
non-Rust build-path change.

## Buffers, Framing, and Callbacks

- Translate `size_t`/word counts/byte counts explicitly. Never infer a proof length
  from a sentinel or C string terminator.
- Validate alignment before casting a byte buffer to words. The current C proof verifier
  rejects misaligned/non-whole-word input; this must be a normal error path, not an
  assertion after an unsafe cast.
- Preserve endianness and frame boundaries. A ZisKOS proof stream is not a serialized
  host `Proof` object, and a raw unframed input is not a `ZiskStdin` record.
- Check every status code and output length before using output. Convert ABI failures
  into deterministic guest rejection/explicit host errors, never default success.
- Make callbacks deterministic, reentrancy-aware, and lifetime-safe. Do not pass a
  pointer to stack state into a callback that can outlive the call.

For `verify_zisk_proof_c`, separate three questions: whether the raw VADCOP stream is
well formed and cryptographically verifies, whether its trailing verifier key is the
application's expected key, and whether its program VK/public claim match policy. The
C call addresses only the first question; program/claim pinning belongs in the consumer.

## Unsafe Review Matrix

| Boundary | Required review |
| --- | --- |
| Rust ↔ C/C++ | ABI layout, `repr(C)`/header match, status code, ownership, lifetime |
| bytes ↔ u64/field words | length, alignment, endian conversion, canonical encoding |
| host allocator ↔ ziskos allocator | allocation/free family, no retained resettable pointers |
| C++ runtime ↔ guest entrypoint | linker script, init/fini arrays, no unsupported TLS/exception assumptions |
| proof bytes ↔ consumer | artifact format, exact lengths, cryptographic check, pinned key/VK/claim |

## Upstream Drift Gate

Before merging an FFI change or upgrading ZisK:

```bash
git fetch origin
git diff <last-verified-zisk-commit>..origin/main -- \
  ziskos-staticlib ziskclib lib-c ziskos/entrypoint \
  ziskbuild/zisk_linker_script.ld verifier/src \
  common/src/proof.rs
```

Rebuild every language binding and rerun ABI fixtures when exports, interface types,
allocator/reset behavior, linker windows, proof-stream parsing, or C/C++ runtime code
change. Do not accept a link-only check as ABI validation.

## Validation

Test a smallest representative guest plus deliberate boundary failures:

```text
valid buffer/callback -> malformed length -> misalignment -> wrong ownership/lifetime
-> wrong status handling -> C++ constructor/destructor path (if used)
-> actual guest execute/prove path
```

Verify exported symbols with the repository's isolated-staticlib build check and inspect
the final ELF's segments. For a proof verifier, feed malformed, wrong-key, wrong-VK,
wrong-claim, and valid artifacts through the actual C/C++ consumer.

## Must / Must Not

**Must:** source-check each signature; document ownership; use the correct allocator;
validate length/alignment/endian/status; retain a cross-language negative test; inspect
the final guest ELF.

**Must not:** retain pointers from resettable ziskos allocations; mix `new`/`free` or
`malloc`/`delete`; pass host bincode as a ZisKOS proof stream; ignore integer conversion
or return codes; rely on generic proof verification as application authorization.
