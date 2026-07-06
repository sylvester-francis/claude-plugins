# scaffolding-typescript-projects

A Claude Code plugin that guides scaffolding new TypeScript projects toward
strict, idiomatic, consistent structure and tooling — with a dedicated NestJS
track.

## Install

```
/plugin marketplace add sylvester-francis/claude-plugins
/plugin install scaffolding-typescript-projects@sylvester-plugins
```

## What it does

The bundled skill activates when you create, scaffold, or restructure a
TypeScript project. It covers:

- **Application vs. library first** — the decision that drives module format,
  build, and whether you publish.
- **A max-strict TypeScript baseline** — `tsconfig` flags, pinned toolchain
  (Node + package manager), ESLint 9 flat config + type-aware rules, Prettier.
- **Library publishing** — the `package.json` `exports` map (types-first,
  per-format `.d.ts`/`.d.cts`), ESM-first vs dual, `tsup`/`tsdown`, `vitest`,
  `publint` + `attw` gating, changesets, and npm provenance.
- **NestJS conventions** (`nestjs.md`) — feature-first vertical slices, thin
  controllers / fat services, DTO validation + a global `ValidationPipe`, typed
  and validated config, migrations (never `synchronize` in prod), and the
  unit + e2e (Testcontainers) testing tiers.
- **A scaffold checklist** and a **common-mistakes** list.

## License

[Apache-2.0](../../LICENSE).
