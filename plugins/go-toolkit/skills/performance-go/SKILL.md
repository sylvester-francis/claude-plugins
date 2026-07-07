---
name: performance-go
description: Use when optimizing Go performance — profiling and benchmarking a hot path with go test -bench, pprof (cpu/mem/block/mutex), benchstat, and escape analysis to cut time and allocations without guessing.
---

# Optimizing Go

## Overview

**Measure first — don't touch code until a profiler or benchmark names the cost.** Intuition about Go performance is wrong often enough that guessing wastes time and adds risk. The loop: benchmark → profile → explain the cost (escape analysis) → make one change → prove it with benchstat → repeat until the profile flattens.

## 1. Pin the function with a benchmark

The benchmark is both the instrument and the regression gate.

```go
func BenchmarkParse(b *testing.B) {
    input := loadFixture(b)   // realistic, representative input
    b.ReportAllocs()
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        sink, _ = Parse(input) // assign to a package var to defeat dead-code elimination
    }
}
```

```bash
go test -run '^$' -bench BenchmarkParse -benchmem -count 10 \
  -cpuprofile cpu.out -memprofile mem.out | tee old.txt
```

Read three numbers: **ns/op** (time), **B/op** (bytes), **allocs/op** (count). On an allocation-heavy path `allocs/op` is usually the lever — each allocation is amortized GC pressure, not just malloc cost.

## 2. Profile — find the real cost

```bash
go tool pprof -http=:8080 cpu.out      # flame graph; top; list Parse (line-by-line)
go tool pprof -http=:8081 mem.out      # default to -sample_index=alloc_space / alloc_objects
```

Heavy time in `runtime.mallocgc`/`growslice`/`mapassign`/GC workers confirms allocations are the bottleneck → back to the mem profile. Check `block`/`mutex` profiles too — "slow" can mean *waiting*, not computing; an empty block/mutex profile is a result (rules out contention).

## 3. Explain the cost before changing it

```bash
go build -gcflags='-m -m' ./pkg 2>&1 | grep -E 'escapes to heap|moved to heap'
```

`escapes to heap` on a pprof-flagged line tells you *why* it allocates — returned pointer, interface boxing, closure capture, slice outliving the frame — so the fix is precise, not flailing.

## 4. One change, then prove it

```bash
# make exactly one change, then:
go test -run '^$' -bench BenchmarkParse -benchmem -count 10 | tee new.txt
benchstat old.txt new.txt
```

benchstat reports the delta with a **p-value** and `~` when the change is statistical noise. If it says `~`, the optimization did nothing — **revert it.** This is the guardrail against fooling yourself.

## The changes that pay off (alloc-heavy paths)

- **Preallocate** with known capacity: `make([]T, 0, n)` kills repeated `growslice` — often the biggest single win.
- **Reuse buffers** via `sync.Pool` for transient per-call objects (measure — it's not free).
- **Stop boxing into `interface{}`** and avoid `fmt.Sprintf` in hot code; use `strconv.AppendInt`, `strings.Builder`/`bytes.Buffer` with `Grow`.
- **Append-style APIs** (`AppendFoo(dst []byte, …)`) so the caller controls allocation; reuse `buf[:0]`.
- Avoid needless `[]byte`↔`string` copies; hoist allocations out of loops.

```go
// 3 allocs/op → 1: Builder+Grow, AppendInt, no boxing
var b strings.Builder
b.Grow(len(ids) * 4)
for i, id := range ids {
    if i > 0 { b.WriteByte(',') }
    b.Write(strconv.AppendInt(nil, int64(id), 10))
}
```

## The discipline

Benchmark → profile (cpu/mem/block/mutex) → escape analysis → **one** change → benchstat proves it (revert on `~`) → repeat. Keep the benchmark in the tree as a regression guard.

## Common mistakes

- Optimizing without a profile — fixing a cost that wasn't the bottleneck.
- A benchmark the compiler optimizes away (no `sink`, unrealistic input).
- Batching several changes so you can't tell which helped; trusting one run over benchstat's p-value.
- `sync.Pool` everywhere — it adds complexity and can hurt if objects aren't actually hot/transient.
