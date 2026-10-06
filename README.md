# Veyra

Open-source tooling for compiling type-checked TypeScript into native binaries with a Rust runtime.

> **Status: planning.** Nothing here is implemented yet. Implementation starts once [ts-rust](https://github.com/arun-murali0/ts-rust) ships its first stable crates.io release with a documented typed IR and query API.

## What Veyra is

- A planned native compiler and runtime for type-checked TypeScript.
- A Rust runtime with a Node-style event loop implemented on Tokio.
- Built on existing infrastructure: ts-rust for type checking, Oxc for parsing and resolution, Cranelift for development builds, Clang for release builds.

## What Veyra is not

- Not a JavaScript engine, and not a Node, Bun, or Deno replacement.
- Not a full implementation of the JavaScript language. v0 targets a defined, typed subset.
- Not a commercial product. Veyra is open source only.

## Relationship to ts-rust

ts-rust is a separate, independent project with its own roadmap and releases. Veyra will depend on it as an external crate (pinned by version) and will not duplicate its checker or module resolver.

## Repositories

| Repo | Purpose | Status |
| --- | --- | --- |
| `veyra` | Compiler, runtime, and design docs | Planning (docs only) |

## License

MIT 
