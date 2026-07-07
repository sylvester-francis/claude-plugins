---
name: performance-typescript
description: Use when optimizing Node.js/TypeScript performance — load-testing, classifying CPU-bound vs event-loop-blocked vs I/O-bound vs GC pressure, and profiling with clinic, 0x, --cpu-prof, and perf_hooks.
---

# Optimizing Node.js / TypeScript

## Overview

**Never optimize from a hunch.** Reproduce under load → baseline → let a profile point at the hot frame → change **one** thing → re-measure against the same harness → confirm the wide frame shrank. Profilers *locate*; a clean load run *quantifies*. Baseline in a prod-like setup: `NODE_ENV=production`, built `dist/` (not ts-node), real payloads, a warm-up pass so V8 has JITed.

## 1. Reproduce under load

```bash
autocannon -c 100 -d 30 http://localhost:3000/slow   # read Latency p99 + Req/Sec
```
For scenarios/thresholds use k6 (checked-in, gated). Record p50/p99/throughput — the tuple every change is judged against.

## 2. Triage before deep profiling

Two cheap signals: **event-loop delay** (`perf_hooks.monitorEventLoopDelay()`, near-zero cost, works in prod) and **`clinic doctor`** (one-shot classifier that recommends CPU / Event Loop / I/O / Memory).

| Class | CPU | Loop delay p99 | Signature | Fix |
|---|---|---|---|---|
| **CPU-bound** | ~100% one core, steady | moderate | wide genuine-compute frames; caps at 1 core | better algorithm, cache, worker pool |
| **Event-loop-blocked** | spiky | **high + bursty** | a `*Sync`/`JSON.parse`/regex frame is wide; *all* requests stall in bursts | make async, chunk, offload |
| **I/O-bound** | **low**, idle loop | low | time in `await`s to DB/HTTP | parallelize, batch, cache, pool |
| **GC pressure** | periodic spikes | periodic spikes | sawtooth heap; `--trace-gc` frequent pauses | cut allocation rate, reuse, stream |

The confusable pair is CPU-bound vs loop-blocked (both burn CPU): loop-blocked shows **bursty p99 loop delay** + a *synchronous API that shouldn't be there*; CPU-bound is steady compute you must make cheaper. I/O-bound is the tell: CPU low while latency high.

## 3. Deep profile + how to read it

```bash
clinic flame -- node dist/server.js     # flamegraph (0x underneath); width = time share, not chronological
node --prof dist/server.js && node --prof-process isolate-*.log > prof.txt  # [Summary]: JS/C++/GC ticks
node --cpu-prof --cpu-prof-dir ./prof dist/server.js   # .cpuprofile -> DevTools
clinic bubbleprof -- node dist/server.js # I/O async flow: long spans, sequential chains, N+1
node --trace-gc dist/server.js           # frequent Scavenge = churn; rising old-space = retention
```
Flamegraph: scan for the **widest frame at any level**, especially a wide plateau near the top, or something with no business being wide (`JSON.stringify`, a regex, a deopt). `--prof` `[Summary]`: high GC% → pressure, high C++ → native (crypto/zlib/JSON), high JS → your code; then `[Bottom up (heavy)]` names the function by self-time.

## 4. Fixes by class

```js
// event-loop-blocked: sync KDF blocks everyone -> async (threadpool); pure CPU -> piscina worker pool
const key = await promisify(crypto.pbkdf2)(pw, salt, 1_000_000, 64, 'sha512');
// sync-in-hot-path: read+parse once at startup, cache the object (not per request)
// accidental O(n^2): index once
const byId = new Map(users.map(u => [u.id, u]));      // was users.find(...) in a loop
// I/O-bound: parallelize + batch
const [a, b] = await Promise.all([getA(), getB()]);   // was serial awaits; N+1 -> WHERE id IN / DataLoader
// GC pressure: single pass, reuse buffers, stream large responses
// JSON APIs: fast-json-stringify with a schema on hot payloads (often 2-5x)
```

## 5. The loop & discipline

After each change, re-run the *exact same* autocannon/k6 command, compare the p99/throughput tuple, and re-open the flamegraph to confirm the targeted frame shrank. One variable per iteration; multiple runs to see variance, not one lucky number. Stop when p99 hits target or the profile flattens into many small frames — further tuning stops paying and adds risk.

## Common mistakes

- Profiling in dev (ts-node, dev middleware, cold V8) — numbers don't transfer.
- Skipping triage and flame-graphing a problem that was I/O-bound (CPU was idle the whole time).
- Confusing CPU-bound with event-loop-blocked — check the loop-delay histogram.
- Batching several changes so you can't attribute the win; trusting one run over repeated measurement.
- `--max-semi-space-size` tuning before fixing the allocation rate the heap profile shows.
