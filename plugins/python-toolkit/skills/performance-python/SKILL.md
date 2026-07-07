---
name: performance-python
description: Use when optimizing Python performance — profiling a hot path with cProfile, py-spy, and timeit, finding allocations with tracemalloc, and applying the right fix (algorithm, caching, generators, numpy, or the GIL escape hatch) without guessing.
---

# Optimizing Python

## Overview

**Measure first — don't guess.** Find where the time actually goes, change one thing, re-measure. Python's cost is rarely where intuition says; profiling replaces a guess with a fact.

## 1. Profile — where does the time go?

```bash
python -m cProfile -o prof.out -m mymod       # whole run; then read it:
python -c "import pstats; pstats.Stats('prof.out').sort_stats('cumulative').print_stats(20)"
snakeviz prof.out                             # flame/icicle view in the browser
py-spy record -o flame.svg --pid 12345        # live process, NO code change, sampling flamegraph
py-spy top --pid 12345                         # live top-like view
```
Read **`tottime`** (time *in* the function itself — the hot leaf to optimize) vs **`cumtime`** (including callees — where a subtree costs). `py-spy` is the go-to for a running service because it attaches without modifying or stopping it.

Microbenchmark a specific line/expr:
```bash
python -m timeit -s "setup" "hot_expression"   # or timeit.repeat() for variance
```

Allocation-bound? `tracemalloc` (see debugging-python) or `memory_profiler`'s `@profile` line-by-line.

## 2. Fix — biggest levers first

- **Algorithm / data structure.** The usual win: an O(n²) `x in list` inside a loop → build a `set`/`dict` once for O(1) membership. This dwarfs micro-tuning.
- **Hoist work out of the loop.** Precompute invariants, bind hot attributes/methods to locals (`append = out.append`), compile regexes once, avoid re-creating objects per iteration.
- **`functools.lru_cache`** (or `cache`) for pure functions with repeated args.
- **Generators over lists** when you stream and don't need random access — constant memory, no build cost.
- **Build strings with `"".join(parts)`**, never `s += x` in a loop (O(n²)).
- **Vectorize numeric work with numpy** — replace Python-level element loops with array ops; it's often 10–100×.

```python
# before: O(n*m) membership + string concat
out = ""
for x in items:
    if x.id in valid_ids_list:       # list scan each iteration
        out += render(x)             # quadratic string build
# after
valid = set(valid_ids)               # O(1) membership
out = "".join(render(x) for x in items if x.id in valid)
```

## 3. When Python itself is the ceiling — the GIL

Profile shows CPU-bound pure-Python compute maxing one core:
- **CPU-bound** → `multiprocessing` / `concurrent.futures.ProcessPoolExecutor` (sidesteps the GIL across processes), or push the hot kernel to numpy / Cython / `numba` / a Rust/C extension.
- **I/O-bound** (waiting on network/disk) → threads or `asyncio` — the GIL is released during I/O, so concurrency helps here without processes.

Pick by the profile: threads/async do nothing for CPU-bound work; processes add IPC overhead that only pays off for real compute.

## 4. The discipline

Profile → change **one** thing → re-run the same benchmark and confirm it moved (and by how much). Keep the benchmark as a guard. Optimizing without a before/after number is how you make code uglier and no faster.

## Common mistakes

- Optimizing a spot the profiler never flagged — the bottleneck was elsewhere (often I/O, not CPU).
- `s += x` in a loop; `x in a_list` membership in a loop — both silently quadratic.
- Reaching for threads on a CPU-bound task — the GIL means no speedup; use processes.
- Rewriting in C/Cython before trying an algorithmic fix or numpy.
- Trusting one timing run over `timeit.repeat()` / multiple samples (variance is real).
