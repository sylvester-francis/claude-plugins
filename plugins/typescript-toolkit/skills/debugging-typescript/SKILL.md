---
name: debugging-typescript
description: Use when a Node.js/TypeScript program throws with the wrong (compiled .js) stack traces, hangs, blocks the event loop, has a promise that never settles, or leaks memory — diagnosing with source maps, the inspector, CPU/heap profiles, event-loop-delay, and diagnostic reports.
---

# Debugging TypeScript / Node

## Overview

**One discriminating measurement picks the branch, then a tool emits a named artifact — a frame, a retainer path, a pending handle — and you confirm by moving that signal, not by editing code you assume is guilty.**

## Stacks point at `.js`, not `.ts`

A source-map plumbing gap, not a code bug. Maps must be *emitted*, *shipped*, and *consumed*:

1. Emit: `tsconfig` `"sourceMap": true`; check `dist/x.js.map` exists and `x.js` ends with `//# sourceMappingURL=`.
2. Consume: run with `node --enable-source-maps` (set `NODE_OPTIONS=--enable-source-maps` so children inherit it). Running TS directly via `tsx`/`ts-node`? the loader registers maps itself.
3. If the flag doesn't help, the map is broken — open the `.js.map`, verify non-empty `mappings` and correct `sources`.

`--async-stack-traces` (default on) keeps `await` frames; raise `Error.stackTraceLimit` if truncated.

## Hangs — check CPU first (routes you to different tools)

`top`/`htop` on the pid while it's hung:

- **One core pegged ~100%** → event loop blocked by sync work. Capture a CPU profile: `node --cpu-prof app.js`, or `clinic doctor -- node app.js` (flags "event loop blocked"), or `clinic flame`/`0x`. The **widest bar on the main thread is the culprit** — catastrophic regex, huge `JSON.parse`, `*Sync` fs/crypto/zlib, O(n²) loop. Quantify with `perf_hooks.monitorEventLoopDelay()` (`.percentile(99)`); the fix should collapse it.
- **CPU idle but requests hang** → a promise that never settles. Surface cheaply: `process.on('unhandledRejection')` + `--unhandled-rejections=strict`, `--trace-warnings`. Find the stuck handle: diagnostic report (`node --report-on-signal` then `kill -SIGUSR2`) lists open sockets/timers; `why-is-node-running` prints active handles; `clinic bubbleprof` visualizes async flow. Prove it by wrapping the suspect `await` in `Promise.race` with a timeout — the silent hang becomes a loud rejection at the exact site. Usual root: a network/DB call with no timeout, or a `resolve()` never reached on some branch.

## Memory leak — confirm the pool, then snapshot

Log `process.memoryUsage()` under steady load (`--expose-gc`, force `global.gc()`):
- `heapUsed` climbs and stays up after GC → JS-heap leak → heap snapshots.
- `external`/`arrayBuffers` grows → Buffer/native leak (snapshots won't find it).
- `rss` grows, heap flat → fragmentation/native addon.

**3-snapshot technique** (`--inspect` → DevTools → Memory): snapshot after warmup → drive N identical ops → snapshot → drive N again → snapshot. In snapshot 3, view "objects allocated between 1 and 2" still alive; sort by **Retained Size**; the constructor whose **Delta grows ~N per batch** is leaking. The **Retainers** pane names the reference chain to a GC root — that chain *is* the proof (a module-level `Map` you never clear, a per-request listener never removed, a closure in a cache). Headless: `node --heapsnapshot-signal=SIGUSR2` or `--heapsnapshot-near-heap-limit=N`.

## Quick reference

```bash
node --enable-source-maps app.js         # .ts frames
node --cpu-prof --cpu-prof-dir ./prof app.js   # .cpuprofile -> DevTools
clinic doctor -- node app.js             # one-shot classifier
node --report-on-signal app.js           # then kill -SIGUSR2 -> handle report
node --trace-gc app.js                   # GC pauses + heap
```

## Common mistakes

- "Fixing" a `.js` stack trace by reading compiled output instead of enabling source maps.
- Reaching for a CPU profiler on an *idle*-CPU hang (it's an unsettled promise — get the handle report).
- Taking one heap snapshot instead of the 3-snapshot diff; guessing the retainer instead of reading the Retainers pane.
- Snapshotting a `heapUsed`-flat process whose `external` is the thing growing.
