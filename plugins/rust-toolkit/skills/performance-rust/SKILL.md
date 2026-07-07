---
name: performance-rust
description: Use when optimizing Rust performance — benchmarking with criterion, profiling with cargo flamegraph/perf, the release profile (lto/codegen-units), cutting allocations and clones, iterators, and data parallelism with rayon.
---

# Optimizing Rust

## Overview

**Measure first, in `--release`.** A debug build is 10–100× slower and its numbers are meaningless — never benchmark or judge perf from `cargo run` without `--release`. Then benchmark → profile → change one thing → re-benchmark.

## 1. Benchmark — criterion

Put statistical benchmarks in `benches/` with **criterion**:
```rust
fn bench(c: &mut Criterion) {
    let input = realistic_fixture();
    c.bench_function("parse", |b| b.iter(|| parse(black_box(&input))));
}
```
`black_box` stops the optimizer from eliding the work. `cargo bench` reports mean, variance, and **change vs. the last run with a significance verdict** — that's your before/after truth.

## 2. Profile — find the hot frame

```bash
cargo flamegraph            # perf-based flamegraph of a release binary
cargo flamegraph --bench parse    # or profile a specific benchmark
samply record ./target/release/app   # cross-platform sampling profiler
# macOS: cargo instruments -t "Time Profiler"
```
Read the flamegraph for the **widest frame** — real compute vs. something that shouldn't be hot (`memcpy`/`drop`/`realloc` = allocation churn; a hash function; bounds checks). Build with debug symbols in release for readable frames (`[profile.release] debug = true` temporarily).

## 3. The release profile

```toml
[profile.release]
lto = "thin"          # link-time optimization (or "fat" for max, slower build)
codegen-units = 1     # better optimization, slower compile
panic = "abort"       # smaller/faster if you don't need unwinding
```
`opt-level = 3` is already the release default.

## 4. The fixes that pay off

- **Cut allocations** — the usual Rust win. Reuse buffers (`Vec::clear` + refill), `Vec::with_capacity(n)` when the size is known, `&str`/`&[u8]` over owned `String`/`Vec` in hot paths, `SmallVec`/`arrayvec` for small collections, avoid `collect()` into an intermediate you immediately re-iterate.
- **Iterators** — lazy, fused, and often **bounds-check-free** (the compiler proves the index is valid), so an iterator chain frequently beats a manual `for i in 0..len` with `xs[i]`.
- **Kill clones in hot loops** — a `.clone()` per iteration is pure allocation; borrow or move.
- **`#[inline]`** on tiny hot functions across crate boundaries (LTO handles within-crate).
- **Data parallelism — `rayon`**: turn `iter()` into `par_iter()` for fearless CPU parallelism on independent work. Measure — parallelism has overhead and only pays for real per-item compute.

## 5. The discipline

Benchmark → profile → **one** change → `cargo bench` again and read criterion's change verdict (revert if it says "no change"). Keep the benchmark in `benches/` as a regression guard. An optimization without a before/after number doesn't count.

## Common mistakes

- Benchmarking a **debug** build → 10–100× off and misleading.
- Optimizing a spot the flamegraph never flagged (often it's allocation/`Drop`, not your logic).
- `.clone()` / `.to_vec()` in a hot loop; `String` concatenation where a reused buffer or `&str` works.
- Manual index loops (bounds checks) where an iterator chain elides them.
- Reaching for `rayon` on trivial per-item work → thread overhead exceeds the gain; measure.
- Forgetting `black_box`, so criterion measures nothing (the optimizer removed the call).
