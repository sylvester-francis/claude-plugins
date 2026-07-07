---
name: scaffolding-go-projects
description: Use when creating, scaffolding, initializing, or restructuring a Go project, module, service, CLI, or library — deciding directory layout (cmd/, internal/, pkg/), applying golang-standards/project-layout, choosing a module path and Go version, or setting up go.mod, tooling, linting, tests, and CI for fresh Go code.
---

# Scaffolding Go Projects

## Overview

Lay out a new Go project so its structure matches what it *is*. **The first decision is application vs. library** — everything else (where the public API lives, whether there's a `cmd/`, whether `/pkg` is even a question) follows from it.

Adopt the conventions from [golang-standards/project-layout](https://github.com/golang-standards/project-layout) that fit, and **skip the ones that don't**. That repo is useful but: it is *not* an official Go standard, it is application-oriented, and several of its directories are wrong for libraries. The full directory-by-directory verdict is in [project-layout.md](project-layout.md).

**Core rule: layout follows the artifact. Keep the root clean, keep the public surface small, keep dependencies minimal and pointing inward.**

## First decision: application or library?

|  | Application (binary / service / CLI) | Library (imported by others) |
|---|---|---|
| Public API | none — it's a program | the module **root package** |
| Entry point | `cmd/<name>/main.go` (thin shim) | none |
| Private code | `internal/` (most of it lives here) | `internal/` (impl details only) |
| `/pkg`? | no | **no** — the root package *is* the API |
| Goal | testable logic out of `main` | `import "github.com/you/lib"` → `lib.Do()` |

If it produces a binary, it's an application. If someone runs `go get` and imports it, it's a library. A repo can be both (a library with a reference CLI under `cmd/`).

## Application layout

```
myapp/
  cmd/myapp/main.go        # thin: signals -> internal, map error to exit code. NO logic.
  internal/                # everything private; compiler-enforced non-importable
    cli/                   # command wiring, flag parsing, output formatting
    <domain>/              # domain types + pure rules (no I/O), the core
    store/ or api/         # adapters implementing interfaces the core defines
    config/                # typed config + Load() (flags > env > file > defaults)
  deployments/             # Dockerfile, docker-compose.yml, k8s manifests
  docs/  (+ docs/adr/)     # design docs + Architecture Decision Records
  testdata/                # fixtures (special-cased by the go tool; not compiled)
  .github/workflows/       # ci.yml, security.yml, release.yml
  go.mod  Makefile  .gitignore  README.md  CHANGELOG.md  LICENSE  SECURITY.md
```

`main` is a shim so real behavior sits in importable, testable `internal/` packages, and so a second binary (`cmd/myappd`) can be added later without restructuring.

## Library layout

```
mylib/
  go.mod
  doc.go                   # package overview -> gives pkg.go.dev a real landing page
  mylib.go                 # constructor + top-level entry points (package mylib)
  <type>.go  options.go  errors.go
  mylib_test.go  example_test.go   # black-box (package mylib_test) + testable Examples
  sub/                     # OPTIONAL importable sub-packages, kept dependency-light
  internal/                # private impl (idgen, etc.) — free to change, never breaks callers
  examples/quickstart/     # runnable programs, built by CI
```

The public API lives at the **module root**: `import "github.com/you/mylib"` then `mylib.New(...)`. No `/pkg`, no `/src`. Optional features go in **separate importable sub-packages** so importing the core never drags in a database driver or renderer a consumer doesn't use — advertise a zero-`require` `go.mod` and keep it that way.

## golang-standards/project-layout — what to adopt vs skip

Full annotated table in [project-layout.md](project-layout.md). The short version:

- **Adopt:** `cmd/` (apps), `internal/`, `docs/`, `examples/`, `tools/`, `deployments/`, `testdata/` (Go's own convention, preferred over `/test` for fixtures).
- **Skip by default / conditional:** `/api` (only with OpenAPI/proto), `/configs` `/init` (only if you ship config/service files), `/web` `/website` (a GitHub Pages site can live in `docs/` or `site/`), `/scripts` `/build` (a `Makefile` + `.github/workflows` cover these; Actions *must* live in `.github/workflows`, goreleaser config at the repo root), `/vendor` (use modules; vendor only for hermetic/air-gapped builds), `/assets` `/third_party` `/githooks`.
- **Avoid:** **`/pkg`.** It is contested even by the layout's own README, and for a library it pushes your real API down to `mylib/pkg/mylib` and breaks the clean import path. Use the root package + `internal/`. Only consider `/pkg` for a large app deliberately publishing a few shared packages — and never move a *published* library into it (that breaks every importer).
- **Never create an empty golang-standards directory as ceremony.** A directory exists when it has real contents.

## Code conventions (idiomatic Go)

- **Module path = repo URL** (`module github.com/you/name`); **pin the Go version** explicitly in `go.mod` (e.g. `go 1.25.0`).
- **Dependencies point inward.** Domain/core imports nothing from cli/http/db. **Define interfaces where they're consumed; return concrete types.**
- **Errors:** wrap with `%w`; sentinel errors via `errors.New`; branch with `errors.Is`/`errors.As`, never on error strings. (If a value crosses a boundary that flattens error *types* — e.g. a durable/replay journal, an RPC — carry the decision as a *value*, and branch on error presence.)
- **Constructors:** functional options (`WithX(...) Option`) for anything with growable config, so adding a knob never breaks callers. Unexported struct fields + accessor methods keep the surface small.
- **`context.Context` is the first parameter** of anything doing I/O or blocking.
- **`log/slog`** for structured logging (stdlib — no dependency).
- **Tests:** table-driven, run with `-race`; `testdata/` for fixtures; testable `Example` functions for libraries (they run under `go test` and show on pkg.go.dev).
- **Doc every exported symbol** with a comment that starts with its name; enforce it (see doc-check below).
- **Versioning:** `v0.x` while unstable; for `v2+` the module path carries the suffix (`/v2`). Don't hardcode a version constant — stamp it via `-ldflags` at build time.

## Recommended defaults (house conventions)

Proven across sibling repos; adjust per project, but start here.

- **Stdlib-first, minimal dependencies.** Prefer `flag` over cobra, `net/http` over web frameworks, `log/slog` over log libs. Every third-party dep must earn its place; a small core is auditable. Prefer pure-Go deps (e.g. `modernc.org/sqlite`) to keep `CGO_ENABLED=0` static single-binary builds.
- **`Makefile`** with: `all build vet lint test race cover mutate tidy fmt clean` (see the sibling repos for the canonical file).
- **Lint = `gofmt` clean + `staticcheck`**, pinned and run via `go run honnef.co/go/tools/cmd/staticcheck@<version>` (a tool, not a module dep — no install).
- **`doc-check`:** a tiny stdlib AST walker (`tools/doccheck`) that fails on any undocumented exported symbol.
- **Mutation testing** with `gremlins` (advisory, non-blocking) over the packages where logic lives.
- **Optional `ascii-check`:** enforce ASCII-only in `.go`/`.md` *only* if the project commits to it; skip it if docs use Unicode (em dashes, mermaid).
- **CI (`.github/workflows/`):** `ci` (build + `go vet` + `go test` + `go test -race` + `gofmt` on ubuntu **and** macos), `security` (`govulncheck`, nightly), optional `mutation` (advisory) and `pages`.
- **Release with GoReleaser:** `CGO_ENABLED=0` static binaries, linux/darwin/windows × amd64/arm64, plus a CycloneDX **SBOM** (syft), **keyless cosign** signature over the checksums (GitHub OIDC → Fulcio/Rekor), and **SLSA build-provenance** attestation.
- **Repo hygiene:** `LICENSE` (Apache-2.0 + a `NOTICE` if Apache), `CHANGELOG.md` (Keep a Changelog + SemVer), `CONTRIBUTING.md`, `SECURITY.md`, `docs/adr/` for decisions.
- **`.gitignore`:** binaries, `dist/`, `coverage.out`, `*.db`/`*.db-wal`/`*.db-shm`, `.DS_Store`, editor dirs, and any local scratch/build-prompt files. **Never commit build artifacts or databases.**

## Scaffold checklist

1. Decide **application vs. library** (table above).
2. `go mod init github.com/you/name`; pin the Go version.
3. Create the layout for that type (trees above) — only dirs that will have contents.
4. Write the thin `cmd/<name>/main.go` (app) **or** `doc.go` + root package (library).
5. Add `Makefile`, `.gitignore`, `LICENSE` (+`NOTICE`), `README.md`, `CHANGELOG.md`, `SECURITY.md`, `CONTRIBUTING.md`.
6. Add `.github/workflows/ci.yml` (+ `security.yml`); add `.goreleaser.yaml` if it ships binaries; put any `Dockerfile`/compose under `deployments/`.
7. Write the first `-race` test and wire `doc-check`.
8. `go mod tidy`, `make`, commit.

## Common mistakes

- **Library API under `/pkg` or a nested package** → breaks the clean import path. Root package = API.
- **All logic in `main.go`** → move it to `internal/` so it's testable; `main` is a shim.
- **Reaching for cobra/viper/gin by reflex** → start stdlib; add a framework only when justified.
- **Empty golang-standards dirs as ceremony** (`/api`, `/web`, `/build`, `/configs`) → create a dir only when it has real contents.
- **Committing binaries / `*.db` / coverage files** → gitignore them; keep the root clean.
- **No `internal/`** → everything is accidentally public API you can never refactor.
- **Hardcoded version constant** → stamp via `-ldflags`; a constant only drifts.
- **Blindly "applying" golang-standards/project-layout** to a library → it's app-oriented; the flat root layout is correct for libs.
