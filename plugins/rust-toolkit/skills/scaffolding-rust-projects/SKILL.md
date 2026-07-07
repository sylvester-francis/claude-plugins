---
name: scaffolding-rust-projects
description: Use when creating, scaffolding, or initializing a new Rust project — a binary, library crate, or Cargo workspace — deciding module layout, error handling (thiserror/anyhow), clippy + rustfmt, features, edition, testing, and publishing to crates.io.
---

# Scaffolding Rust projects

## Overview

Let Cargo do the layout; spend your day-one effort on the two things that shape a Rust codebase: **binary vs. library** (and whether it's a workspace), and **error handling** (`thiserror` for libraries, `anyhow` for applications). Turn clippy into a gate immediately — it's the idiom teacher.

## First decision: binary, library, or workspace?

| | Binary (app/CLI) | Library crate | Workspace |
|---|---|---|---|
| `cargo new` | `--bin` (default) | `--lib` | multiple crates under one `[workspace]` |
| Entry | `src/main.rs` | `src/lib.rs` (the public API) | a `Cargo.toml` with `members` |
| Errors | **`anyhow`** (`anyhow::Result`, context) | **`thiserror`** (typed enums consumers match on) | per-crate |
| Extra binaries | `src/bin/*.rs` | — | one crate per concern |

A library with a reference CLI is common: a `lib.rs` crate plus a thin `src/main.rs` (or a `bin` crate in a workspace) that calls into it.

## Layout

```
myproj/
  Cargo.toml            # [package], edition = "2021", deps; or [workspace] members
  Cargo.lock            # commit for binaries; libraries commit it too (dev/CI)
  src/
    main.rs / lib.rs
    <module>.rs  or  <module>/mod.rs   # modules via `mod`, public API via `pub`
  tests/                # integration tests (each file is its own crate)
  benches/              # criterion benchmarks
  rustfmt.toml  .clippy.toml   # if you customize; defaults are good
```

Prefer flat `foo.rs` + `foo/` submodules over `mod.rs` (2018+ style). Keep `lib.rs`/`main.rs` thin — declare modules and wire, don't implement.

## Error handling (the load-bearing choice)

- **Library** → `thiserror`: a typed error enum per crate so callers can `match`/`?` on specific variants.
  ```rust
  #[derive(thiserror::Error, Debug)]
  pub enum Error {
      #[error("not found: {0}")] NotFound(String),
      #[error(transparent)] Io(#[from] std::io::Error),
  }
  ```
- **Application** → `anyhow`: `anyhow::Result<T>` + `.context("loading config")` for a readable chain; you don't need to type every error at the top level.
- Never `unwrap()`/`expect()` in library code paths that can fail — return `Result`. `expect("invariant: ...")` is fine for genuine invariants.

## Toolchain (gate all of these in CI)

```bash
cargo fmt --all --check          # rustfmt: formatting is not a debate
cargo clippy --all-targets --all-features -- -D warnings   # clippy as an error gate
cargo test --all-features        # unit + integration + doc tests
cargo build --release
```

Pin the edition (`edition = "2021"`), optionally a `rust-toolchain.toml` for a reproducible compiler. Use `[features]` for optional functionality (keep default features minimal); libraries stay `no_std`-friendly only if they need to.

## Publishing a library

Fill `description`, `license` (SPDX, e.g. `Apache-2.0`), `repository`, `keywords`, `categories` in `Cargo.toml`; write `//!` crate docs and `///` item docs (they become docs.rs and run as doc tests). `cargo publish --dry-run` + `cargo package` before release; SemVer strictly (a public API change is a major bump).

## Scaffold checklist

1. `cargo new --bin`/`--lib` (or a workspace `Cargo.toml`).
2. Choose errors: `thiserror` (lib) or `anyhow` (app).
3. Module layout; keep `main.rs`/`lib.rs` thin.
4. `.gitignore` (`/target`, keep `Cargo.lock` for bins); `rustfmt`/`clippy` config only if customizing.
5. CI: `fmt --check` → `clippy -D warnings` → `test` → `build --release`.
6. First test + a doc test on the public API; commit.

## Common mistakes

- `anyhow` in a **library** → consumers can't match error variants; use `thiserror`.
- `unwrap()`/`expect()` scattered through fallible library code.
- `main.rs`/`lib.rs` doing real work instead of wiring modules.
- Clippy warnings left un-gated → idiomatic drift (run `clippy -- -D warnings`).
- `mod.rs` everywhere (pre-2018 style) instead of `foo.rs` + `foo/`.
- Not committing `Cargo.lock` for a binary (non-reproducible builds).
