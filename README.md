# claude-plugins

Claude Code plugins by Sylvester Francis.

## Add the marketplace

```
/plugin marketplace add sylvester-francis/claude-plugins
```

## Plugins

### scaffolding-go-projects

Best-practice layout and tooling for scaffolding a new Go project — deciding
application vs. library structure, applying the parts of
[golang-standards/project-layout](https://github.com/golang-standards/project-layout)
that fit (and skipping the ones that don't), and setting up `go.mod`, tooling,
linting, tests, and CI.

Install:

```
/plugin install scaffolding-go-projects@sylvester-plugins
```

Once installed, the bundled skill triggers when you scaffold or restructure a Go
project. It ships:

- **`SKILL.md`** — the application-vs-library decision, concrete layouts for
  each, code conventions, tooling defaults, a scaffold checklist, and common
  mistakes.
- **`project-layout.md`** — the full golang-standards/project-layout directory
  table, each directory marked **adopt / conditional / skip / avoid**.

## Repository layout

```
.claude-plugin/marketplace.json          # marketplace definition
plugins/
  scaffolding-go-projects/
    .claude-plugin/plugin.json           # plugin manifest
    skills/scaffolding-go-projects/
      SKILL.md
      project-layout.md
```

## License

[Apache-2.0](LICENSE).
