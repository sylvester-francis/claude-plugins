# go-toolkit

A bundle of Go developer-productivity skills for Claude Code.

## Install

```
/plugin marketplace add sylvester-francis/claude-plugins
/plugin install go-toolkit@sylvester-plugins
```

## Skills

- **scaffolding-go-projects** — best-practice layout and tooling for a new Go
  project (application or library), with annotated golang-standards/project-layout
  guidance.
- **debugging-go** — diagnose panics, data races, hangs, and goroutine/memory
  leaks with the race detector, pprof, `GODEBUG`, and delve; read panics and
  goroutine dumps.
- **testing-go** — table-driven tests, fakes over mocks, `httptest` handlers,
  `errors.Is` assertions, golden files, fuzzing, and the `go test` workflow.
- **refactoring-go** — Go idioms (accept interfaces/return structs, `%w` wrapping,
  small interfaces), anti-patterns, behavior-preserving cleanups, and reviewing a
  diff.
- **performance-go** — measure-first optimization with benchmarks, pprof
  (cpu/mem/block/mutex), escape analysis, and benchstat.
- **security-go** — review/harden a Go HTTP service: authz/IDOR, SQL injection,
  server timeouts, TLS, SSRF, secrets, and `govulncheck`/`gosec` CI gating.
- **observability-go** — structured logging (`slog`), Prometheus RED metrics, and
  OpenTelemetry tracing wired as middleware without coupling business logic.
- **tiered-model-orchestration** — route a substantial task across model tiers:
  plan with Opus (max effort), execute with Sonnet, cross-verify with
  Haiku/Sonnet via an independent, adversarial pass. (Also available standalone
  as the `model-orchestration` plugin.)

## License

[Apache-2.0](../../LICENSE).
