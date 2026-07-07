---
name: scaffolding-typescript-projects
description: Use when creating, scaffolding, initializing, or restructuring a TypeScript project — a Node service, NestJS app, CLI, or npm library — deciding tsconfig strictness, module format (ESM/CJS/dual), package manager, build/test/lint tooling, the package.json exports map, or NestJS module structure and conventions.
---

# Scaffolding TypeScript Projects

## Overview

Stand up a TypeScript project that is **strict by default** and honest about its **module boundaries**. Two decisions drive everything: **application vs. library** (which sets module format, build, and whether you publish), and **how strict** the compiler is (set it to the max on day one — strictness is far cheaper now than retrofitted).

NestJS is a specific *application* shape with its own conventions — see [nestjs.md](nestjs.md).

**Core rule: max strictness, explicit `exports`, ESM-first, pin the toolchain, keep `dist/` out of git.**

## First decision: application or library?

| | Application (service / NestJS / CLI) | Library (published to npm) |
|---|---|---|
| Consumed by | run as a process | `import` by other projects |
| Module format | ESM (or CJS for classic NestJS) | **ESM-first**; dual ESM+CJS only for max reach |
| `package.json` | `"private": true` | `exports` map + `types` + `files` |
| Build | `tsc` / framework CLI / `tsx` | bundler (`tsup`/`tsdown`) emitting JS **and** `.d.ts` |
| Tests | jest (NestJS) or vitest | vitest |
| Extra gates | — | `publint` + `@arethetypeswrong/cli`, changesets, provenance |

## Universal TypeScript baseline (both)

- **Max-strict `tsconfig`.** `strict: true` plus `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noImplicitOverride`, `noFallthroughCasesInSwitch`, `verbatimModuleSyntax`, `isolatedModules`, `forceConsistentCasingInFileNames`, `skipLibCheck`. Set `moduleResolution` to `"nodenext"` (Node apps) or `"bundler"` (libraries/bundled apps); `target` ES2022+.
- **Pin the toolchain.** `engines.node`, an `.nvmrc`, and `"packageManager": "pnpm@x.y.z"` (enable Corepack). Default to **pnpm** (fast, strict about phantom deps, reproducible with `--frozen-lockfile`). Commit the lockfile.
- **Lint = ESLint 9 flat config + type-aware `typescript-eslint` + Prettier.** Use `eslint.config.mjs` (not legacy `.eslintrc`), `recommendedTypeChecked` with `projectService: true`. `@typescript-eslint/no-floating-promises` is the highest-value rule — it catches un-awaited async. Prettier owns formatting only.
- **Scripts** (consistent names): `build`, `typecheck` (`tsc --noEmit`), `lint`, `format`, `test`, plus `dev`/`start` as fits.
- **Never commit `dist/`**, `coverage/`, `*.tsbuildinfo`, or `.env`. Commit `.env.example`.

## Application layout

```
myapp/
  src/
    main.ts            # thin composition root: build the app, wire config, start
    config/            # typed, validated config; the ONLY place that reads process.env
    <feature>/         # vertical slices (see NestJS below), or domain + adapters
  test/                # e2e / integration
  package.json  tsconfig.json  eslint.config.mjs  .nvmrc  .env.example  Dockerfile
```

Keep `main`/bootstrap thin; put logic in modules that are unit-testable. Read env in exactly one config layer, validated at startup (fail fast on a missing var). For **NestJS**, follow [nestjs.md](nestjs.md).

## Library layout & publishing

```
mylib/
  src/index.ts         # the public barrel — everything here is semver API
  src/*.ts             # internals, surfaced only through the barrel
  package.json  tsconfig.json  tsup.config.ts  vitest.config.ts  eslint.config.mjs
```

The `package.json` is the contract — get the **`exports` map** right:

```jsonc
{
  "type": "module",
  "files": ["dist"],
  "exports": {
    ".": {
      "import": { "types": "./dist/index.d.ts",  "default": "./dist/index.js"  },
      "require":{ "types": "./dist/index.d.cts", "default": "./dist/index.cjs" }
    },
    "./package.json": "./package.json"
  },
  "publishConfig": { "access": "public", "provenance": true }
}
```

- **`types` first** in each condition block (resolution is top-to-bottom).
- **Per-format declarations** — `.d.ts` for `import`, `.d.cts` for `require`. Reusing one `.d.ts` for both is the #1 cause of "works in ESM, breaks in CJS" type errors.
- **ESM-first**; add the `require` conditions + a `.cjs` build only when you need to reach older CJS consumers (dual publishing). Node 22+ can `require()` ESM, so **ESM-only is a legitimate choice** — simpler, no dual-package hazard.
- **`files: ["dist"]`** is an allowlist (safer than an `.npmignore` denylist).
- Build with **`tsup`** (esbuild; emits JS both formats + `.d.ts` in one config); **`tsdown`** is the modern successor. Test with **vitest**.
- **Gate the package in CI** with `publint --strict` and `attw --pack .` — they catch broken `exports`/type wiring before users do.
- Optional deps are **`peerDependencies`** (+ `peerDependenciesMeta.optional`), never hard deps.
- Version with **changesets**; publish from CI with **npm provenance** (`id-token: write` + `publishConfig.provenance`). Stay `0.x` until the API is proven, then commit to semver.

## NestJS (application)

The idiomatic shape — full detail in [nestjs.md](nestjs.md):
- **Feature-first vertical slices**: each feature is a module owning its controller, service, DTOs, entity, and tests. `nest g resource <name>` is the only ceremony to add one.
- **Thin controllers, logic in services.** A module **`exports` its service**, so siblings depend on the service, never on another module's repository.
- **DTOs + `class-validator` + a single global `ValidationPipe`** (`whitelist`, `forbidNonWhitelisted`, `transform`).
- **Typed, validated config** via `@nestjs/config` + `registerAs` namespaces; validate `process.env` at boot.
- **Migrations, never `synchronize: true`** in real environments.
- **Two test tiers**: unit (`Test.createTestingModule`, deps mocked) + e2e (`supertest`, a real DB via Testcontainers).

## Scaffold checklist

1. Decide **application vs. library** (and if an app, is it **NestJS**? → [nestjs.md](nestjs.md)).
2. `pnpm init`; set `packageManager`, `engines`, `.nvmrc`; `"type": "module"`.
3. Write the max-strict `tsconfig.json`.
4. Add `eslint.config.mjs` (flat, type-aware) + Prettier.
5. App: `src/main.ts` + a validated config layer. Library: `src/index.ts` + the `exports` map + `tsup`/`vitest`.
6. Add `.gitignore` (`dist`, `node_modules`, `coverage`, `*.tsbuildinfo`, `.env`), `.env.example`, `README`, `LICENSE`.
7. CI: install `--frozen-lockfile` → `typecheck` → `lint` → `test` → `build`; libraries also run `publint` + `attw`.
8. First test, then commit.

## Common mistakes

- **Not maxing strictness** (missing `noUncheckedIndexedAccess`/`exactOptionalPropertyTypes`) → set it day one.
- **One `.d.ts` for a dual package** → ship `.d.ts` + `.d.cts`; let `attw` verify.
- **Committing `dist/`** → build in CI; gitignore it.
- **Legacy `.eslintrc`** → ESLint 9 flat config (`eslint.config.mjs`).
- **NestJS `synchronize: true`** in prod → migrations only.
- **Business logic in controllers** → controllers map HTTP; services hold logic.
- **`process.env` read all over** → one validated config layer.
- **Unpinned Node / package manager** → `engines` + `.nvmrc` + `packageManager`.
- **Hard-depending on React/peer libs in a library** → optional `peerDependencies`.
