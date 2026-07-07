---
name: refactoring-rust
description: Use when refactoring or reviewing Rust — applying clippy, borrowing idioms (accept &str/&[T], avoid needless clone), error handling with ? and thiserror, iterators over index loops, enums to make illegal states unrepresentable, and reducing Rc<RefCell>.
---

# Refactoring & reviewing Rust

## Overview

Refactoring is **behavior-preserving** — keep tests green, change one thing at a time. In Rust the biggest wins are **borrowing the caller's data instead of owning copies**, **iterators/combinators over manual loops**, and **types that make illegal states unrepresentable**. Run clippy first — it teaches the idioms and auto-suggests most of them.

## Run clippy first

`cargo clippy --all-targets --all-features -- -D warnings` catches needless clones, manual `map`/`unwrap_or`, redundant closures, and dozens of idiom fixes — many with `cargo clippy --fix`. Gate it in CI so idiom drift can't reland. `cargo fmt` for formatting.

## High-value fixes

**Borrow, don't own, in function signatures.** Accept the most general borrow so callers don't allocate:
```rust
fn greet(name: &String)      ->  fn greet(name: &str)          // &str takes String, &str, literals
fn sum(xs: &Vec<i32>)        ->  fn sum(xs: &[i32])            // &[T] takes Vec, array, slice
fn open(p: String)           ->  fn open(p: impl AsRef<Path>)  // flexible, zero-copy
```

**Kill needless clones.** `x.clone()`/`.to_owned()`/`.to_vec()` in a hot path or to dodge the borrow checker is usually a restructure signal (clippy flags many). Borrow, split the borrow, or narrow the scope instead.

**Errors: `?` + typed/`anyhow`, not `unwrap` chains.**
```rust
let s = std::fs::read_to_string(p).unwrap();   ->   let s = std::fs::read_to_string(p)?;
```
Libraries: `thiserror` enums; apps: `anyhow` + `.context(...)`. `ok_or`/`ok_or_else` to turn `Option` into `Result`.

**Iterators & combinators over index loops.**
```rust
let mut out = Vec::new();
for i in 0..xs.len() { if xs[i].active { out.push(xs[i].name.clone()); } }
// ->
let out: Vec<_> = xs.iter().filter(|x| x.active).map(|x| x.name.clone()).collect();
```
`map`/`and_then`/`unwrap_or_default`/`filter_map` read better than a verbose `match` when the intent is a transform.

**Make illegal states unrepresentable.** Replace bool-pairs / stringly-typed state with an `enum`; derive `Debug`/`Clone`/`PartialEq` instead of hand-writing; use the **newtype** pattern (`struct UserId(u64)`) so IDs don't mix.
```rust
struct Conn { connected: bool, error: Option<String> } // both true? illegal
enum Conn { Connected(Socket), Failed(Error) }          // illegal states gone
```

**Avoid `Rc<RefCell<T>>` / `Arc<Mutex<T>>` unless you truly share mutation.** Reflexive shared-interior-mutability is often a design smell — pass ownership or borrow instead; reach for it only for genuine shared state. Elide lifetimes where the compiler allows; annotate only where required.

## Reviewing a diff

Read for: a `&String`/`&Vec<T>` parameter that should be `&str`/`&[T]`; a `.clone()` dodging the borrow checker; an `unwrap()`/`expect()` in fallible library code; an index loop that's an iterator; a bool-pair that's an enum; an `Rc<RefCell>` where ownership would do. Confirm `clippy -D warnings` + tests pass.

## The discipline

Tests green before/after · one behavior-preserving change per commit · `clippy -D warnings` + `fmt` clean · prefer deleting code. A change that alters behavior is a feature/fix with its own test.

## Common mistakes

- `&String`/`&Vec<T>` parameters instead of `&str`/`&[T]` → callers must own/allocate.
- `.clone()` to silence the borrow checker instead of restructuring ownership.
- `unwrap()` chains in library code instead of `?` + a typed error.
- Manual index loops where an iterator chain is clearer and bounds-check-free.
- `bool` fields / string tags where an `enum` makes illegal states impossible.
