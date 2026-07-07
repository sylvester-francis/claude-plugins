# typescript-toolkit

A bundle of TypeScript developer-productivity skills for Claude Code.

## Install

```
/plugin marketplace add sylvester-francis/claude-plugins
/plugin install typescript-toolkit@sylvester-plugins
```

## Skills

- **scaffolding-typescript-projects** — best-practice layout and tooling for a new
  TypeScript project (Node service, NestJS app, CLI, or npm library): tsconfig
  strictness, module format, the package.json exports map, and NestJS conventions.
- **debugging-typescript** — wrong stack traces (source maps), event-loop blocks,
  unresolved promises, and memory leaks via the inspector, CPU/heap profiles,
  event-loop-delay, and diagnostic reports.
- **testing-typescript** — vitest vs jest, fakes over mocks, `supertest` HTTP tests,
  Testcontainers integration, the test pyramid, and coverage.
- **refactoring-typescript** — killing `any`, discriminated unions, `satisfies`,
  narrowing over `as`, `readonly`, and strict compiler + type-aware lint.
- **performance-typescript** — measure-first tuning: load tests, classifying
  CPU-bound vs event-loop-blocked vs I/O-bound vs GC, and clinic/0x/`--cpu-prof`.

## License

[Apache-2.0](../../LICENSE).
