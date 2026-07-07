# rust-toolkit

A bundle of Rust developer-productivity skills for Claude Code.

## Install

```
/plugin marketplace add sylvester-francis/claude-plugins
/plugin install rust-toolkit@sylvester-plugins
```

## Skills

- **scaffolding-rust-projects** — binary / library / workspace layout, error
  handling (`thiserror` vs `anyhow`), modules, clippy + rustfmt gates, publishing.
- **debugging-rust** — panics + backtraces, `dbg!`, `rust-gdb`/`lldb`, the
  `tracing` crate, `tokio-console` for async, and reading borrow-checker errors.
- **testing-rust** — unit/integration/doc tests, `Result`-returning tests, fakes
  vs `mockall`, `proptest`, and `criterion` benchmarks.
- **refactoring-rust** — clippy, borrowing idioms (`&str`/`&[T]`, no needless
  clone), `?` + `thiserror`, iterators, and enums that make illegal states
  unrepresentable.
- **performance-rust** — measure-first with `criterion` + `cargo flamegraph`, the
  release profile, cutting allocations/clones, iterators, and `rayon`.
- **security-rust** — review/harden an axum/actix service: authz/IDOR, `sqlx`
  injection, unwrap/panic-as-DoS, auth, SSRF, `unsafe` auditing, and `cargo audit`/`deny`.
- **observability-rust** — the `tracing` crate + `tracing-opentelemetry`, Prometheus
  RED metrics, and correlation IDs wired as tower layers.
- **tiered-model-orchestration** — route a substantial task across model tiers:
  plan with Opus (max effort), execute with Sonnet, cross-verify with
  Haiku/Sonnet via an independent, adversarial pass.

## License

[Apache-2.0](../../LICENSE).
