---
name: testing-typescript
description: Use when writing or improving TypeScript/Node tests — choosing vitest vs jest, structuring unit/integration/e2e, fakes vs mocks, testing HTTP with supertest, Testcontainers, and coverage.
---

# Testing TypeScript / Node

## Overview

Test through the public surface with a fast runner, **fakes over mocks**, and a real **test pyramid**: many fake-backed unit tests, a few real-DB adapter tests, fewer end-to-end. Assert on outcomes and state so tests survive refactors.

## Runner

**Vitest** by default — ESM/TS-native via esbuild (no `ts-jest`/babel transform to babysit), instant watch, Jest-compatible API, built-in v8 coverage. **Stay on Jest for an established NestJS app** (its templates ship Jest + `@nestjs/testing`); the structure below is identical, only config/imports differ.

## Layout

Colocate fast unit tests; isolate slow container-backed tests:

```
src/users/user.service.ts
src/users/user.service.spec.ts    # unit — fake store, no I/O
src/users/user.controller.spec.ts # HTTP via supertest
test/users.e2e-spec.ts            # full app + Testcontainers
vitest.config.ts  vitest.integration.config.ts
```

`*.spec.ts` = fast unit (default run); `*.e2e-spec.ts` / `*.int.spec.ts` = container-backed (separate config, excluded from the fast loop).

## Dependencies: fakes over mocks

The dependency sits behind an interface, injected via the constructor. Default to a hand-written in-memory **fake** (a real `Map`-backed implementation) — state-based assertions, no brittle call-count coupling, ~90% of service tests. Use a **mock** (`vi.fn`) only when the interaction *is* the behavior ("notify called once") or to inject a failure; a **stub** to force one return/throw.

```ts
class InMemoryUserStore implements UserStore {
  private rows = new Map<string, User>();
  findByEmail = async (e: string) => this.rows.get(e) ?? null;
  insert = async (u: User) => { this.rows.set(u.email, u); };
}

describe('UserService.register', () => {
  let service: UserService;
  beforeEach(() => { service = new UserService(new InMemoryUserStore()); }); // fresh per test
  it('rejects a duplicate email', async () => {
    await service.register('a@b.com', 'pw');
    await expect(service.register('a@b.com', 'pw')).rejects.toThrow(/already exists/);
  });
});
```

## HTTP layer — supertest

Build the app from a factory `createApp(deps)` (Express) or `Test.createTestingModule().overrideProvider(Store).useValue(fake).compile()` (Nest); hit `supertest(app)` with no port bound. Wire the **real service + fake store** so you test the wire: routing, status codes, DTO validation (`400`), error→status mapping (`404/409`), auth (`401/403`). **Register the same global `ValidationPipe`/exception filter as prod** or these tests miss validation. Don't re-assert business rules here — one happy path + the HTTP edges.

## Integration — Testcontainers

Run the **real** store adapter (Prisma/Drizzle/pg) against a throwaway Postgres via `@testcontainers/postgresql` — the only layer that catches SQL, migrations, unique constraints, transactions. `globalSetup` starts the container once + migrates; separate config, longer timeout, excluded from the fast loop.

## Commands

```bash
vitest run                 # CI, single pass
vitest                     # watch (inner loop)
vitest run --coverage      # v8 provider; gate the service hard, wiring softer
vitest run -c vitest.integration.config.ts
```

## Common mistakes

- Mocking call sequences instead of asserting outcomes → brittle tests.
- Testing HTTP without the prod `ValidationPipe`/filter → validation bugs slip through.
- Faking the database in the *integration* test → it no longer catches SQL/migration/constraint bugs; use Testcontainers.
- Snapshot tests as the primary assertion for logic a table would pin precisely.
- Sharing mutable fixtures across tests instead of building fresh deps in `beforeEach`.
