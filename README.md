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

Go developer-productivity skills. Currently:
- **scaffolding-go-projects** — layout and tooling for a new Go project
  (application or library) + annotated golang-standards/project-layout.

_Coming: `debugging-go`, `testing-go`, `refactoring-go`, `performance-go`._

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
