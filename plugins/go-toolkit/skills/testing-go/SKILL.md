---
name: testing-go
description: Use when writing or improving Go tests — table-driven tests, subtests and parallelism, fakes vs mocks, testing HTTP handlers, error assertions, golden files, coverage, fuzzing, and the go test workflow.
---

# Testing Go

## Overview

Test through the **exported API** with the standard library. Default to **black-box** test packages, **hand-written fakes** over mock frameworks, and **table-driven** cases that assert on outcomes and state (so tests survive refactors). Keep dependencies behind small, consumer-defined interfaces so faking is trivial.

## Layout & package

`foo_test.go` next to `foo.go`. Use `package foo_test` (black-box) so tests exercise only the public surface; drop to white-box `package foo` or an `export_test.go` seam only to reach a specific unexported helper worth testing directly.

## Dependencies: fakes over mocks

Define the interface on the **consumer** side, as narrow as the code actually uses — a small interface makes a fake trivial. Prefer a hand-written in-memory fake with error-injection fields over a generated/`testify` mock: assert on results/state, not call sequences. Reach for a mock framework only when you genuinely need strict call-order/argument assertions. Inject nondeterminism (clock, id gen) through the constructor.

```go
type Store interface {                 // consumer-defined, minimal
    GetByEmail(ctx context.Context, email string) (User, error)
    Create(ctx context.Context, u User) error
}
type fakeStore struct{ byEmail map[string]User; getErr, createErr error } // + hooks to force failures
```

## Table-driven + subtests + parallel

One row per behavior; compare errors with **`errors.Is`** (wrapped errors still match; sentinels stay part of the contract). Construct dependencies **inside** each subtest so `t.Parallel()` never shares mutable state — the race detector confirms it.

```go
func TestRegister(t *testing.T) {
    t.Parallel()
    tests := []struct{ name string; in User; seed *User; storeErr, wantErr error }{
        {name: "ok", in: User{Email: "a@b.co"}},
        {name: "duplicate", in: User{Email: "a@b.co"}, seed: &User{Email: "a@b.co"}, wantErr: ErrEmailTaken},
        {name: "store down", in: User{Email: "a@b.co"}, storeErr: errDB, wantErr: errDB},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            store := newFakeStore()          // fresh deps per subtest
            // ... seed, inject errors, build svc, call, assert with errors.Is ...
        })
    }
}
```

## HTTP handlers

`net/http/httptest`: build a request, drive it with a `ResponseRecorder`, assert on recorded status/body — no real server. Test through the `*http.ServeMux` when routing (method/path) is part of the behavior. Cover the status mapping (2xx/4xx/5xx) plus malformed input the service never sees.

## Toolkit

- `t.Helper()` on assertion helpers; `t.Cleanup()` for teardown; `testdata/` for fixtures/golden files (the go tool ignores that dir).
- `google/go-cmp` (`cmp.Diff(want, got)`) for struct/slice comparisons — the one dependency worth adding.
- **Fuzz** parsers/decoders: `func FuzzX(f *testing.F)` + `go test -fuzz=FuzzX -fuzztime=30s`.
- Keep integration tests (real DB) behind `testing.Short()` or a build tag so the default run stays fast and hermetic.

## Running

```bash
go test -race -shuffle=on -cover ./...            # the default bar (also CI)
go test -run TestRegister/duplicate ./pkg/        # one subtest, regex on name
go test -v -count=1 ./pkg/                         # verbose, bypass the test cache
go test -coverprofile=cover.out ./... && go tool cover -html=cover.out
```

## Common mistakes

- Mocking call sequences instead of asserting outcomes → brittle tests.
- Sharing deps across parallel subtests → flakes the race detector catches.
- Wide interfaces that force heavyweight mocks → narrow the interface at the consumer.
- Comparing errors with `==`/string match instead of `errors.Is`/`As`.
- Snapshot/golden tests as the primary assertion for logic that a table would pin precisely.
