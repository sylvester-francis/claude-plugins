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

### typescript-toolkit

```
/plugin install typescript-toolkit@sylvester-plugins
```

TypeScript developer-productivity skills. Currently:
- **scaffolding-typescript-projects** — layout and tooling for a Node/NestJS app
  or npm library (tsconfig strictness, module format, the exports map, NestJS
  conventions).

_Coming: `debugging-typescript`, `testing-typescript`, `refactoring-typescript`,
`performance-typescript`._

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
