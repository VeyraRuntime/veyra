# Veyra Design (draft)

**Status:** planning. No implementation exists. Implementation starts when ts-rust publishes its first stable crates.io release with a documented typed IR and query API (see section 3).

## 1. Purpose

Veyra compiles type-checked TypeScript into native binaries that run on a Rust runtime. It is open source, maintained as a GitHub organization, and independent of ts-rust.

## 2. Non-goals

- Not a JavaScript engine or a Node, Bun, or Deno replacement.
- No promise of full JavaScript semantics. v0 targets a defined typed subset (section 7).
- Not a second type checker or module resolver. Those belong to ts-rust and Oxc.
- Not a commercial product.

## 3. Dependency on ts-rust

ts-rust is a separate project under its own account and release schedule. Veyra depends on it as a versioned library crate, not by running its binary.

**Start trigger.** Veyra implementation begins when ts-rust has:

1. a first stable crates.io release,
2. cross-file checking and module resolution complete,
3. a documented, versioned typed IR,
4. a documented query API with stable ids,
5. an agreed compatibility bar on real projects (not only its own fixtures).

**Contract Veyra needs from ts-rust:**

- Typed IR with a version field, a text dump format, and explicit "dynamic/unknown" markers where the checker cannot determine a type.
- Stable ids for modules, declarations, and nodes, usable as cache keys.
- A resolved module graph with per-module export tables, including package `exports`/`imports` conditions.
- Diagnostics with severity, so Veyra can refuse to compile on errors.

Veyra starts at the typed IR stage. Where the IR crate lives (inside ts-rust or a small shared crate) is an open question (section 19).

## 4. Architecture

```text
TypeScript source
  -> ts-rust (parse, check, resolve)        [external]
  -> typed IR + query results               [contract]
  -> Veyra lowering
  -> Veyra IR
  -> Cranelift (dev) or Clang (release)
  -> object code + Veyra Rust runtime
  -> native executable
```

Development and release backends share one runtime ABI.

## 5. Veyra IR

Lower than TypeScript semantics, higher than machine code. Independent of Cranelift and Clang. It represents values, functions, calls, control flow, objects, arrays, strings, modules, closures, exceptions, async operations, and runtime calls. Ownership and mutation state are optional annotations, not core requirements.

## 6. Checker-gap policy

ts-rust is still incomplete (advanced generics and other features pending). Veyra must not trust the checker beyond what it covers:

- Codegen must be correct without type information. Types enable specialization only behind guards or inside verified subsets.
- Where the IR marks a type as dynamic/unknown, codegen uses generic dispatch or the fallback path. It never guesses.
- Checker false positives block compilation; checker gaps do not. Each gap is reported.

## 7. Supported subset (v0)

v0 supports programs that use TypeScript features through ts-rust's Generics milestone, ESM only, no `eval`, no runtime `require`, no `Proxy`. The exact list is maintained in `docs/subset.md` and grows as ts-rust coverage grows. Anything outside it is a compile-time error, or an explicit opt-in to the fallback.

## 8. Runtime

A Rust runtime linked into the final binary. It provides values, memory, objects, arrays, strings, closures, modules, errors, promises, timers, builtins, filesystem, and networking as they are implemented. Normal execution needs no Node.js or JavaScript engine.

### Event loop

Node's event-loop model, implemented independently on Tokio. Decide and document early:

- whether Veyra matches Node's phase ordering (timers, I/O, `setImmediate`, microtask drain) or only the API shape,
- how microtasks and promise jobs interleave with Tokio tasks.

Ordering-sensitive behavior is covered by differential tests against Node.

## 9. JavaScript fallback (temporary)

QuickJS is the planned temporary fallback, so unsupported code can run while native Rust implementations are built.

- **Opt-in and visible.** Builds report which modules use the fallback (for example `veyra build --report-fallback`).
- **Isolated.** It lives in its own crate or feature (for example `veyra-fallback-quickjs`) and is not part of the ABI, so removing it later is a deletion.
- **Defined boundary.** Value conversion between native code and QuickJS (copy vs handle) and promise integration with the Tokio loop are specified before implementation.
- **Honest accounting.** Tests record whether they ran native or on the fallback. Native-only builds exclude QuickJS entirely.

## 10. npm compatibility

Static imports are resolved by ts-rust and Oxc before lowering. Veyra does not repeat Node resolution. Supported packages are those inside the v0 subset, plus packages served by the fallback. The claim "npm compatible" is scoped to that list, which is published and tested.

## 11. Code generation

Cranelift is the development backend (fast compiles, fast iteration). Clang is the release backend (optimization, linking). Start with one backend (Cranelift) and add Clang after the ABI is proven. Backend details must not leak into the frontend, IR, or runtime design.

## 12. Ownership and immutability

An optional research track, not a v0 gate. Ownership, moves, borrows, and stronger immutability are designed and tested as semantics before any syntax is committed. Existing TypeScript remains the compatibility target.

## 13. Memory model

Experimental and isolated. Different strategies must be testable without changing ts-rust or the Veyra IR. Ordinary TypeScript may need managed runtime values.

## 14. Testing

- Differential tests: Veyra output versus Node (or tsx) per fixture, in the style of ts-rust's `compare3.js` and `tsc-drift`. CI fails on regressions in mismatches.
- Hand-written IR fixtures so the backend and runtime can be tested before ts-rust emits real IR.
- Layered tests: lowering, Cranelift, Clang, runtime, npm subset, event-loop ordering, integration.

## 15. Performance and measurement

Veyra's front end inherits ts-rust's measurements. Quote them only with the workload stated.

| Metric | ts-rust value | Workload | Machine | Cold/warm |
| --- | --- | --- | --- | --- |
| tsc diagnostic compatibility | about 92% | _fill in_ | _fill in_ | _fill in_ |
| Peak heap | about 6 MB | _fill in_ | _fill in_ | _fill in_ |
| Cache hit rate | about 97% | _fill in_ | _fill in_ | _fill in_ |

Also state what counts as a "match" for compatibility (exact diagnostics, per-line, per-file) and what counts as a "hit" for cache rate. Report false positives separately from gaps. Veyra's own benchmarks (compile time, binary size, startup, throughput vs Node and Bun) are added once there is code.

## 16. Implementation order (after the start trigger)

1. Typed IR consumer and Veyra IR definition, with text dump.
2. Runtime value model, ABI, and Tokio event loop.
3. Cranelift backend over hand-written IR.
4. Lowering from ts-rust IR for single files, then multi-file.
5. Functions, control flow, objects, arrays, modules.
6. Differential test harness against Node.
7. npm subset, then the QuickJS fallback.
8. Clang release backend.
9. Ownership and immutability research track.

## 17. v0 success criteria

Compile and run a defined set of typed programs natively through Cranelift, with a working event loop, differential tests passing against Node, and a published list of the supported subset and fallback usage. Release builds, ownership, and broad npm support are later milestones.

## 18. Design principles

- Reuse ts-rust and Oxc; do not rebuild them.
- Keep frontend, IR, backends, and runtime independent.
- Be correct without types; use types only to specialize.
- Prefer native Rust over JavaScript runtime dependencies; treat the fallback as temporary.
- Keep the scope small enough to change.
- Test every layer alone and together.

## 19. Open questions

- Where does the typed IR crate live: ts-rust, or a small shared crate?
- Does Veyra match Node's event-loop phase ordering exactly?
- How do values cross the native/QuickJS boundary?
- What is the exact v0 subset, and what compatibility bar on real projects counts as "stable" for ts-rust?
- Which wedge is the headline: single-binary deploys, startup time, incremental builds, or the safety track?

## 20. Repository structure (planned)

```text
crates/
  veyra  veyra-ir  veyra-lower  veyra-codegen
  veyra-codegen-cranelift  veyra-codegen-clang
  veyra-runtime  veyra-builtins  veyra-link
  veyra-cli  veyra-npm  veyra-fallback-quickjs
tests/  examples/  benches/  docs/  scripts/
```
