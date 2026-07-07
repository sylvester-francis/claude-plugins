---
name: refactoring-typescript
description: Use when refactoring or reviewing TypeScript for type safety and idioms — killing any, modeling with discriminated unions, satisfies, narrowing over as, readonly, and enabling strict compiler + type-aware lint.
---

# Refactoring & reviewing TypeScript

## Overview

Refactoring is **behavior-preserving** — keep tests green, change one thing at a time. In TypeScript the biggest wins are making **illegal states unrepresentable** and letting the compiler prove correctness. Turn strictness up first and fix against the error list rather than eyeballing.

## Step 0: let the compiler find the bugs

Enable `strict` + `noUncheckedIndexedAccess` + `exactOptionalPropertyTypes` and read the errors — that surfaces the real `any`/null bugs faster than reading code.

## What to scan for

`any` (explicit *and* implicit — untyped params, `JSON.parse`, `catch (e)`, `as any`, `object`, `{}`); `as` casts and `!` (each disables checking); flag+optional soup (`ok: boolean` + `data?`/`error?`); `enum`; `?.`/`??` masking an invariant that should be `| undefined`; module-level `let` / mutable arrays; `arr[i]` assumed defined.

## High-value fixes

**Kill `any` — `unknown` at boundaries, generics for passthrough**
```ts
function parse(s: string): unknown { return JSON.parse(s); }        // forces a narrow at the call site
function first<T>(a: readonly T[]): T | undefined { return a[0]; }
```
`useUnknownInCatchVariables` makes `catch (e: unknown)` → narrow with `e instanceof Error`.

**Discriminated union instead of flags + optionals**
```ts
type Result = { ok: true; data: User } | { ok: false; error: string };
function handle(r: Result) { if (r.ok) r.data; else r.error; } // auto-narrows, no data!/?.
```
Add an exhaustiveness guard so a new variant is a compile error:
```ts
const assertNever = (x: never): never => { throw new Error(`unhandled: ${JSON.stringify(x)}`); };
```

**`satisfies` to validate shape without widening literals**
```ts
const routes = { home: '/', profile: '/profile' } satisfies Record<string, string>;
type Route = keyof typeof routes; // 'home' | 'profile', not string
```

**Narrowing / type guards over `as`**
```ts
const el = document.getElementById('app');
if (el instanceof HTMLCanvasElement) el.getContext('2d');   // proven, not asserted
function isUser(v: unknown): v is User { return typeof v === 'object' && v !== null && 'id' in v; }
```
At real I/O boundaries reach for a schema validator (zod/valibot) and `infer` the type, so the runtime check and static type can't drift.

**`readonly` + `as const`; `enum` → string-literal union**
```ts
const DIRECTIONS = ['north', 'south'] as const;
type Direction = (typeof DIRECTIONS)[number];
type Status = 'active' | 'inactive';   // no runtime code, structural, serializes cleanly
```

## tsconfig & lint to rely on

`strict`, plus the two that catch the most real bugs and aren't in `strict`: **`noUncheckedIndexedAccess`** and **`exactOptionalPropertyTypes`** (also `noImplicitOverride`, `noFallthroughCasesInSwitch`, `verbatimModuleSyntax`). Type-aware `@typescript-eslint` (needs `parserOptions.projectService`): `no-explicit-any`, the **`no-unsafe-*` family** (catches `any` leaking implicitly), `no-non-null-assertion`, `consistent-type-assertions`, `switch-exhaustiveness-check`, `no-floating-promises`/`no-misused-promises`.

## Order of operations

Flip on strict flags → work the compiler error list → convert flag+optional shapes to discriminated unions → replace `as`/`!` with guards/narrowing → add `readonly`/`as const` → enable type-aware ESLint last to catch the residue. Each step behavior-preserving, the strict typechecker as the regression net.

## Common mistakes

- `as`/`!` to silence an error instead of narrowing — the bug is still there, just hidden.
- `boolean` + optionals where a discriminated union makes illegal states unrepresentable.
- `enum` for what a string-literal union does better (no runtime code, structural).
- Casting `JSON.parse`/HTTP bodies to a type instead of validating at the boundary (zod) — the type is a lie.
- Refactoring types with no tests to prove behavior is unchanged.
