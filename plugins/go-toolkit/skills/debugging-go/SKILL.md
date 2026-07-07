---
name: debugging-go
description: Use when a Go program panics, crashes, hangs, deadlocks, leaks goroutines or memory, or fails intermittently — diagnosing with the race detector, goroutine/heap/CPU profiles, GODEBUG, delve, and by reading panics and stack traces.
---

# Debugging Go

## Overview

**Evidence over guessing.** Go gives you tools that name the exact conflicting line or blocked frame — a race report, a goroutine dump, a profile. Reach for those before reading code and theorizing. Reproduce the failure first (intermittent + concurrent almost always means a data race, so run under `-race` from the start), then let a tool point at the cause, then prove the fix survives stress.

## Reading a panic

- The **first non-`runtime.` frame** is the crash site — open that exact line.
- `SIGSEGV ... addr=0x0` → method/field on a nil pointer; `addr=0x18` → nil base + field at offset 0x18 (a nil *field* of a real struct). The nil argument shows in the frame: `(*T).m(0x0, …)`.
- `GOTRACEBACK=all` shows every goroutine; `GOTRACEBACK=crash` + `ulimit -c unlimited` drops a core for post-mortem.

## Data races & intermittent nil

Settle race-vs-logic with the detector — don't guess:

```bash
go build -race ./... && ./app-race          # or: go test -race -count=200 ./...
go test -race -c -o pkg.test ./... && stress ./pkg.test   # x/tools/cmd/stress until it fails
```

A `DATA RACE` report prints the conflicting **read and write stacks plus both goroutines' creation sites** — those two lines are the root cause. A deterministic nil (returns `(nil, err)` used without checking, map miss, failed assertion) reproduces every time with the same input; confirm the producer in `dlv`.

## Hangs & goroutine leaks

The runtime only reports `all goroutines are asleep - deadlock!` when *every* goroutine is blocked; a partial deadlock silently stops serving. Dump the goroutines yourself:

```bash
curl -s 'localhost:6060/debug/pprof/goroutine?debug=2'   # full stacks (needs net/http/pprof)
kill -QUIT <pid>                                          # any Go binary, no code change
```

Each header shows state + age: `goroutine 77 [sync.Mutex.Lock, 14 minutes]`. **Take two snapshots ~30s apart** — same goroutines pinned = deadlock; growing count = leak. Quantify the primitive with block/mutex profiles (`runtime.SetBlockProfileRate(1)` / `SetMutexProfileFraction(1)` → `go tool pprof .../debug/pprof/block|mutex`).

## Memory & scheduler

```bash
go tool pprof -base h1 h2 .../debug/pprof/heap   # diff two heap snapshots for a leak
GODEBUG=gctrace=1 ./app                          # back-to-back GCs w/ rising heap = pressure
GODEBUG=schedtrace=1000,scheddetail=1 ./app      # runnable goroutines not progressing
```

## Delve (interactive / post-mortem)

```bash
dlv debug ./cmd/app    dlv attach <pid>    dlv core ./app-race ./core
(dlv) break file.go:88 ; continue ; print theReceiver ; bt ; goroutines
```

## The discipline

1. Reproduce (ideally under `-race`). 2. Get a tool to name the cause (race report / goroutine dump / profile) — not a theory. 3. Fix the invariant. 4. **Prove it:** the same case survives `-race -count=1000` or `stress`. A fix that "seems to work" once is a guess.

## Common mistakes

- Reading code and speculating instead of running `-race`.
- Trusting one green run of an intermittent failure — stress it.
- Assuming a hang is a deadlock without a goroutine dump (it may be GC/OOM — check the heap).
- Debugging with print statements where a profile or `dlv` would give a direct answer.
