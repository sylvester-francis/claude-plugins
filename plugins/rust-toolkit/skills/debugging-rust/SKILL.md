---
name: debugging-rust
description: Use when a Rust program panics, hangs, deadlocks, overflows, or an async task is stuck — reading panics and backtraces, dbg!, rust-gdb/lldb, the tracing crate, tokio-console for async, and understanding borrow-checker errors.
---

# Debugging Rust

## Overview

Rust moves most bugs to compile time — the borrow checker *is* debugging, so read its errors as help, not obstacles. For runtime issues, the panic message + backtrace usually names the cause; for async hangs, `tokio-console` is the live task view. **Reproduce, read the message the tool gives you, then fix — don't guess.**

## Panics & backtraces

```bash
RUST_BACKTRACE=1 cargo run      # backtrace of the panic; =full for every frame
```
The panic message names the *what* and the source location (`src/x.rs:42`); the **first non-`std`/non-`core` frame** in the backtrace is your crash site. Common panic sources: `unwrap()`/`expect()` on `None`/`Err`, index out of bounds, and **integer overflow** (panics in debug builds, wraps in `--release` — make intent explicit with `checked_`/`wrapping_`/`saturating_` ops rather than relying on the mode).

Inspect a value inline with `dbg!` (prints `file:line = value` and returns the value, so it drops into an expression) instead of `println!` scattering.

## Interactive debugging

```bash
rust-gdb target/debug/app        # or rust-lldb — wrappers that load Rust pretty-printers
```
`break file.rs:42`, `run`, `bt`, `print var`, `frame`. The Rust pretty-printers render `Vec`/`String`/`Option` readably (plain gdb/lldb won't).

## Structured logging — the `tracing` crate

Use `tracing` (not `log`) for anything async or concurrent: `#[instrument]` on functions creates spans, `info!`/`error!` are span-aware, and `tracing-subscriber` formats/filters (`RUST_LOG=debug`). Spans give you the causal path through concurrent code that a flat log can't.

## Async hangs — `tokio-console`

A stuck async task is the hardest Rust runtime bug. Wire in `console-subscriber` and run `tokio-console` for a **live view of every task**: which are idle vs busy, poll durations, and tasks that have been polled-but-never-completed (the analog of a goroutine dump). Two classic causes:
- **Blocking the runtime**: a sync call (`std::fs`, `std::thread::sleep`, a big CPU loop, a blocking DB driver) on an async worker starves *all* tasks. Move it to `tokio::task::spawn_blocking`. `tokio-console` shows the worker with a huge poll time.
- **A future that never resolves**: awaiting a channel/lock/response nothing completes. `tokio-console` shows the task parked; a `Mutex` held **across an `.await`** is a frequent deadlock — don't hold a lock over await points.

## Reading borrow-checker errors

The compiler names the conflict precisely: "cannot borrow `x` as mutable because it is also borrowed as immutable", with both spans and lifetimes. Don't fight it with `clone()`/`Rc<RefCell>` reflexively — that's often hiding a design issue. Restructure ownership: split borrows, narrow scopes, or pass owned values. `E0499`/`E0502` (aliasing), `E0382` (use-after-move), `E0597` (borrow outlives value) each map to a specific ownership fix.

## Common bug shapes

- `unwrap()`/`expect()` panic on `None`/`Err` → handle with `?`/`match`/`unwrap_or`.
- Integer overflow (debug panic / release wrap) → `checked_`/`saturating_`.
- Deadlock: `Mutex` locked in inconsistent order, or held across `.await` → restructure.
- Memory growth from an `Rc`/`Arc` **cycle** → break it with `Weak`.
- Async stall from a blocking call on the runtime → `spawn_blocking`.

## Quick reference

```bash
RUST_BACKTRACE=full cargo run    # panic backtrace
RUST_LOG=debug cargo run         # tracing output
rust-lldb target/debug/app       # interactive, with Rust pretty-printers
tokio-console                    # live async task inspector (needs console-subscriber)
```

## Common mistakes

- `println!` debugging where `dbg!` (value + location) or `tracing` spans are cleaner.
- Reaching for `clone()`/`Rc<RefCell>` to silence the borrow checker instead of fixing ownership.
- Holding a `Mutex` guard across an `.await` (deadlock/blocked runtime).
- Benchmarking or judging a hang in a debug build (integer overflow panics; everything is slow).
- Ignoring `tokio-console` for an async hang and guessing at the stuck task.
