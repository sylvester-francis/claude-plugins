# Go & TypeScript productivity toolkits — design

Date: 2026-07-06
Status: approved
Repo: `sylvester-francis/claude-plugins` (marketplace `sylvester-plugins`)

## Goal

Grow the marketplace from two standalone scaffolding plugins into two
**per-language toolkits**, each bundling a coherent set of developer-productivity
skills for Go and TypeScript.

## Catalog

Two plugins, five skills each (two already exist):

| `go-toolkit` | `typescript-toolkit` |
|---|---|
| scaffolding-go-projects *(exists)* | scaffolding-typescript-projects *(exists)* |
| debugging-go | debugging-typescript |
| testing-go | testing-typescript |
| refactoring-go | refactoring-typescript |
| performance-go | performance-typescript |

Each skill is a language-specific **reference/technique** skill: a *method* plus
concrete tooling, not a bare command list.

### Per-skill scope

- **debugging-**: systematic root-cause method + tooling. Go: `dlv`, `-race`,
  `GODEBUG`, pprof goroutine/heap dumps, reading panics, classic bug shapes
  (nil, goroutine leaks, deadlocks, data races). TS: `node --inspect`, source
  maps, async stack traces, heap/`--cpu-prof`, unhandled rejections; NestJS notes.
- **testing-**: Go: table-driven + subtests, `t.Parallel`/`t.Cleanup`, fakes over
  mocks, golden files, fuzzing, coverage. TS: vitest/jest, AAA, mocking seams,
  Testcontainers/supertest, coverage gates, snapshot caution.
- **refactoring-**: idioms, anti-patterns, simplification, reviewing a diff. Go:
  accept-interfaces/return-structs, `%w` wrapping, small interfaces. TS:
  narrowing, discriminated unions, `satisfies`, killing `any`, module boundaries.
- **performance-**: measure-first. Go: `-bench -benchmem`, pprof
  (cpu/mem/block/mutex), escape analysis, allocation reduction. TS: clinic/0x
  flame graphs, event-loop lag, async concurrency, bundle-size budgets.

## Structure & migration

Repo layout:

```
plugins/go-toolkit/
  .claude-plugin/plugin.json          # name: go-toolkit
  README.md
  skills/{scaffolding-go-projects, debugging-go, testing-go,
          refactoring-go, performance-go}/
plugins/typescript-toolkit/
  .claude-plugin/plugin.json          # name: typescript-toolkit
  README.md
  skills/{scaffolding-typescript-projects, debugging-typescript, ...}/
```

`marketplace.json` lists the two toolkits and adds `renames` so anyone who
installed the old standalone plugins auto-migrates:

```json
"renames": {
  "scaffolding-go-projects": "go-toolkit",
  "scaffolding-typescript-projects": "typescript-toolkit"
}
```

Install: `/plugin install go-toolkit@sylvester-plugins` (one install = all Go
skills), likewise `typescript-toolkit`.

## Build process (per new skill)

The proven RED→GREEN loop:
1. **Baseline** — dispatch subagent(s) to perform the task with no skill; capture
   real gaps/divergences.
2. **Author** — write `SKILL.md` (+ a supporting reference file where heavy)
   targeting those gaps.
3. **Verify** — dispatch a subagent that loads the skill and performs the task;
   confirm it drives correct behavior. Refactor to close any gap it reveals.
4. **Package** — copy into the toolkit's `skills/` dir.

## Phasing

Each phase is validated (`claude plugin validate`), committed, pushed, and tagged.
No Claude attribution anywhere; Apache-2.0.

- **Phase 0** — restructure the two existing plugins into `go-toolkit` +
  `typescript-toolkit`, add `renames`. Tag `v0.3.0`.
- **Phase 1** — author + verify the 4 new Go skills. Tag `v0.4.0`.
- **Phase 2** — author + verify the 4 new TypeScript skills. Tag `v0.5.0`.

## Conventions

- Each skill Go/TS-specific, concrete commands, a method not just tools.
- `SKILL.md` concise (aim < ~500 words) + a supporting reference file when the
  material is heavy (e.g. a tool cheatsheet).
- Description follows SDO: "Use when …", triggering conditions only.
- Apache-2.0; no AI attribution in commits, releases, or repo content.

## Success criteria

- `claude plugin validate` passes at every phase.
- Each new skill passes its GREEN verification (a fresh agent loading it produces
  correct, idiomatic output that avoids the baseline's gaps).
- `go-toolkit` and `typescript-toolkit` install and expose all bundled skills;
  old plugin names migrate via `renames`.
