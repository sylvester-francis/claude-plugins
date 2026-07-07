---
name: testing-rust
description: Use when writing or improving Rust tests — unit tests in-module, integration tests in tests/, doc tests, Result-returning tests, fakes vs mockall, property tests with proptest, criterion benchmarks, and cargo test/nextest.
---

# Testing Rust

## Overview

Rust bakes testing into the toolchain: unit tests live **next to the code** (and can reach private items), integration tests live in `tests/` and exercise only the **public API**, and your doc examples run as tests. Default to small traits + hand-written fakes; reach for `mockall` only when you need generated mocks.

## The three test locations

- **Unit** — same file, `#[cfg(test)] mod tests { … }`; can test private functions.
  ```rust
  #[cfg(test)]
  mod tests {
      use super::*;
      #[test]
      fn normalizes_email() { assert_eq!(normalize(" A@B.CO "), "a@b.co"); }
  }
  ```
- **Integration** — `tests/foo.rs`; each file is its own crate that `use`s your library, so it tests the public surface exactly as a consumer would.
- **Doc tests** — ```` ```rust ```` blocks in `///`/`//!` docs compile and run under `cargo test`. They keep your documented examples honest.

## Assertions & shape

`assert!`, `assert_eq!`, `assert_ne!` (with an optional message). Return `Result` from a test and use `?` so setup errors fail cleanly:
```rust
#[test]
fn parses() -> Result<(), Box<dyn std::error::Error>> {
    let cfg = Config::parse("port = 8080")?;
    assert_eq!(cfg.port, 8080);
    Ok(())
}
```
`#[should_panic(expected = "...")]` for panic paths; `#[tokio::test]` (or `#[async_std::test]`) for async.

## Dependencies: small traits + fakes

Define the dependency as a **trait** the code depends on; hand-write a fake struct for tests and assert on its state. Use **`mockall`** (`#[automock]`) when you need generated mocks with call/argument expectations. Inject via generics (`fn new(store: impl Store)`) or `Box<dyn Trait>`. `tempfile` for filesystem tests.

## Property-based & snapshots

- **`proptest`** (or `quickcheck`) for invariants — generate inputs, shrink failures:
  ```rust
  proptest! {
      #[test]
      fn roundtrips(s in ".*") { prop_assert_eq!(decode(&encode(&s)), s); }
  }
  ```
- **`insta`** for snapshot tests of complex output (review + accept diffs).

## Benchmarks (separate from tests)

Put statistical benchmarks in `benches/` with **criterion** (`cargo bench`) — it reports mean, variance, and regression vs. the last run. Don't use `#[bench]` (unstable) or judge perf from a debug build.

## Running

```bash
cargo test                     # unit + integration + doc tests
cargo test parses -- --nocapture   # filter by name; show println output
cargo nextest run              # faster runner, better output (optional)
cargo test --doc               # just the doc tests
```

## Common mistakes

- Putting logic-only tests in `tests/` (can't reach private items) — unit-test in-module.
- Mocking call sequences with `mockall` where a hand fake + state assertion is simpler and refactor-proof.
- Letting doc examples rot — they're compiled tests; keep them passing (or mark `no_run`/`ignore` deliberately).
- Benchmarking in debug mode, or with `#[bench]` — use criterion in `benches/` on `--release`.
- `unwrap()` in test setup instead of a `Result`-returning test with `?` (worse failure messages).
