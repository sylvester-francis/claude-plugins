---
name: refactoring-go
description: Use when refactoring, reviewing, or cleaning up Go code — applying idioms (accept interfaces/return structs, small interfaces, error wrapping), removing anti-patterns, and simplifying without changing behavior.
---

# Refactoring & reviewing Go

## Overview

Refactoring is **behavior-preserving** — keep tests green (or write them first), change one thing at a time. The goal is idiomatic Go: small interfaces defined where they're used, concrete return types, errors that carry context, and shallow control flow. Let the tools catch the mechanical issues so review attention goes to design.

## Run the tools first

`gofmt`/`goimports`, `go vet`, and `staticcheck` (or `golangci-lint`) catch formatting, shadowing, ineffective assignments, and misused stdlib before a human looks. Anything they can enforce shouldn't cost review time.

## The idioms (with fixes)

**Accept interfaces, return structs.** Take the narrowest interface a function needs; return concrete types so callers keep full access and you don't over-abstract.
```go
func NewCache(s Store) *Cache        // param: small interface
func (c *Cache) Stats() Stats        // return: concrete struct, not interface{}
```

**Define interfaces at the consumer, keep them small.** The package that *uses* a dependency declares the 1–3 methods it needs — not the package that implements it. A 10-method interface is a smell.

**Wrap errors; branch on identity, not strings.** Add context with `%w`; expose sentinels/typed errors; check with `errors.Is`/`errors.As`.
```go
if err != nil { return fmt.Errorf("load user %s: %w", id, err) }   // not fmt.Errorf("...: %v")
if errors.Is(err, ErrNotFound) { ... }                             // not strings.Contains(err.Error(), "not found")
```

**Handle every error.** No `_ =` on calls that can fail meaningfully; `errcheck`/vet flags them. Return or log with context — never swallow.

**Reduce nesting with early returns.** Guard-clause the error/edge cases and return; keep the happy path un-indented.
```go
if err != nil { return err }
// happy path continues at column 1, not nested in an else
```

**Pointers only when needed.** Use values for small immutable structs; pointers for mutation or large structs. A `*string`/`*int` field usually wants a value + an `ok`/zero-value convention instead.

**Naming.** No stutter (`user.User` → `user.Account`); short receiver names (`c *Cache`, not `this`); exported symbols documented starting with the name. Prefer `zero value useful` types (a usable `var b bytes.Buffer`).

**Prefer the standard library.** `log/slog`, `errors`, `net/http`, `slices`/`maps` before a dependency. Fewer deps age better.

## Reviewing a diff

Read for: an error path that's swallowed or loses context; an interface returned where a struct would do; a wide interface that should be narrowed; nondeterminism (maps ranged for ordered output, time/rand not injected); a goroutine with no lifecycle/`context`; a mutex held across I/O. Confirm tests cover the new behavior and still pass under `-race`.

## The discipline

Tests green before and after · one behavior-preserving change per commit · run `gofmt`+`vet`+`staticcheck` · prefer deleting code over adding. If a change alters behavior, it's a feature/fix with its own test — not a refactor.

## Common mistakes

- "Refactoring" that quietly changes behavior with no test to prove otherwise.
- Introducing an interface with a single implementation "for testability" when a concrete type + a consumer-side interface at the *test* would do.
- `fmt.Errorf("...: %v", err)` — loses the wrap; use `%w`.
- Premature abstraction/generics before there are two real call sites.
