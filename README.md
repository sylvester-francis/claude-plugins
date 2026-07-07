# claude-plugins

Claude Code plugins by Sylvester Francis, organized as per-language toolkits.

## Add the marketplace

```
/plugin marketplace add sylvester-francis/claude-plugins
```

## Toolkits

### go-toolkit

```
/plugin install go-toolkit@sylvester-plugins
```

Go developer-productivity skills:
- **scaffolding-go-projects** — layout and tooling for a new Go project
  (application or library) + annotated golang-standards/project-layout.
- **debugging-go** — races, hangs, panics, and leaks via the race detector,
  pprof, `GODEBUG`, and delve.
- **testing-go** — table-driven tests, fakes, `httptest`, fuzzing, coverage.
- **refactoring-go** — idioms, anti-patterns, and behavior-preserving cleanups.
- **performance-go** — measure-first profiling with benchmarks, pprof, benchstat.
- **security-go** — authz/IDOR, SQL injection, server timeouts, SSRF, `govulncheck`/`gosec`.
- **observability-go** — `slog` logging, Prometheus RED metrics, OpenTelemetry tracing.
- **tiered-model-orchestration** — plan (Opus/max) → execute (Sonnet) → cross-verify (Haiku/Sonnet).

### typescript-toolkit

```
/plugin install typescript-toolkit@sylvester-plugins
```

TypeScript developer-productivity skills:
- **scaffolding-typescript-projects** — layout and tooling for a Node/NestJS app
  or npm library (tsconfig strictness, module format, the exports map, NestJS
  conventions).
- **debugging-typescript** — source maps, event-loop blocks, unresolved promises,
  and memory leaks via the inspector, CPU/heap profiles, and diagnostic reports.
- **testing-typescript** — vitest/jest, fakes, `supertest`, Testcontainers, coverage.
- **refactoring-typescript** — killing `any`, discriminated unions, `satisfies`,
  narrowing, and strict compiler + type-aware lint.
- **performance-typescript** — measure-first tuning with load tests, clinic/0x,
  and `--cpu-prof`.
- **security-typescript** — authz/IDOR, input validation, injection, JWT, CORS, SSRF.
- **observability-typescript** — `pino` logging, `prom-client` RED metrics, OpenTelemetry.
- **tiered-model-orchestration** — plan (Opus/max) → execute (Sonnet) → cross-verify (Haiku/Sonnet).

### python-toolkit

```
/plugin install python-toolkit@sylvester-plugins
```

Python developer-productivity skills:
- **scaffolding-python-projects** — `src/` layout, `pyproject.toml`, uv, ruff, mypy, pytest.
- **debugging-python** — tracebacks, `pdb`, `py-spy`, `faulthandler`, `tracemalloc`, asyncio.
- **testing-python** — pytest, fixtures, parametrize, fakes, coverage, Hypothesis.
- **refactoring-python** — type hints + mypy, dataclasses, comprehensions, ruff.
- **performance-python** — `cProfile`/`py-spy` measure-first, the right fix, the GIL.
- **tiered-model-orchestration** — plan (Opus/max) → execute (Sonnet) → cross-verify (Haiku/Sonnet).

### model-orchestration

```
/plugin install model-orchestration@sylvester-plugins
```

Language-agnostic. Routes a substantial task across model tiers — **plan with
Opus (max effort) → execute with Sonnet → cross-verify with Haiku/Sonnet** via an
independent, adversarial pass.

## Repository layout

```
.claude-plugin/marketplace.json
plugins/
  go-toolkit/
    .claude-plugin/plugin.json
    skills/scaffolding-go-projects/...
  typescript-toolkit/
    .claude-plugin/plugin.json
    skills/scaffolding-typescript-projects/...
docs/superpowers/specs/        # design docs
```

Old standalone plugin names (`scaffolding-go-projects`,
`scaffolding-typescript-projects`) migrate to the toolkits automatically via the
marketplace `renames` map.

## License

[Apache-2.0](LICENSE).
