# scaffolding-go-projects

A Claude Code plugin that guides scaffolding new Go projects toward idiomatic,
consistent structure and tooling.

## Install

```
/plugin marketplace add sylvester-francis/claude-plugins
/plugin install scaffolding-go-projects@sylvester-plugins
```

## What it does

The bundled skill activates when you create, scaffold, or restructure a Go
module, service, CLI, or library. It covers:

- **Application vs. library first** — the decision that drives the whole layout.
- **Concrete layouts** for a `cmd/` + `internal/` application and a flat-root
  importable library.
- **golang-standards/project-layout, annotated** — which directories to adopt,
  which to skip, and why `/pkg` is avoided for libraries.
- **Code conventions** — module path, pinned Go version, inward-pointing
  dependencies, error wrapping, functional options, `context`, `log/slog`,
  table tests.
- **Tooling defaults** — `Makefile`, `gofmt` + pinned `staticcheck`, a
  `doc-check`, `-race` tests, CI, GoReleaser with SBOM/cosign/SLSA.
- **A scaffold checklist** and a **common-mistakes** list.

## License

[Apache-2.0](../../LICENSE).
