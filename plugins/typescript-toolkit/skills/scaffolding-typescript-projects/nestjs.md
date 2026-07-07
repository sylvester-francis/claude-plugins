# NestJS — conventions reference

How to structure and scaffold a NestJS service that survives for years. Companion to `SKILL.md`.

## Project shape

Start from `nest new` (or `nest g` schematics), then organize **feature-first**:

```
src/
  main.ts                     # composition root: pipes, security, versioning, docs, listen
  app.module.ts               # root module: wires config, logging, db, feature modules
  config/
    configuration.ts          # registerAs('app'|'database', ...) typed factories
    config.types.ts           # AppConfig / DatabaseConfig interfaces
    env.validation.ts         # validate process.env once, at boot (fail fast)
  common/                     # cross-cutting: filters, guards, interceptors, pipes, shared DTOs
  database/                   # module + DataSource for the migration CLI + migrations/
  health/                     # Terminus health controller
  <feature>/                  # vertical slice, one folder per feature area
    <feature>.module.ts
    <feature>.controller.ts   # thin: HTTP mapping + validation only
    <feature>.service.ts      # business logic + persistence
    <feature>.service.spec.ts # colocated unit test
    dto/                      # create/update DTOs with class-validator decorators
    entities/                 # ORM entities
test/
  jest-e2e.json  *.e2e-spec.ts  setup-e2e.ts
```

**Rule:** a feature module owns its controller/service/DTOs/entity/tests and `exports` its service. Siblings depend on the **service**, never on another module's repository. New feature = `nest g resource <name>`, nothing else moves.

## Layering

- **Controller** — HTTP surface only: route, bind DTO, delegate. No business logic, no DB.
- **Service** — logic + owns the repository (`@InjectRepository`). Injectable, constructor-injected deps.
- **Cross-cutting** as first-class Nest constructs: **Guards** (authz), **Interceptors** (transform/logging), **Pipes** (validation/coercion), **Exception Filters** (uniform error envelope). Register global ones via `APP_GUARD`/`APP_FILTER`/etc. providers so they're DI-aware.

## Validation

- DTOs are classes with `class-validator` decorators; wire **one global `ValidationPipe`** in `main.ts`:
  ```ts
  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,            // strip unknown properties
    forbidNonWhitelisted: true, // 400 on unknown properties
    transform: true,            // instantiate DTO classes + coerce types
    transformOptions: { enableImplicitConversion: true },
  }));
  ```
- Derive update DTOs with `PartialType(CreateXDto)` so rules aren't duplicated. The same decorators feed `@nestjs/swagger`. (`nestjs-zod` is the schema-first alternative.)

## Configuration

- `@nestjs/config`, `isGlobal: true`, `cache: true`.
- **Validate `process.env` at boot** (`validate:` fn using class-validator or zod/Joi) so a missing var crashes immediately with a clear message, not at first use.
- **`registerAs('app', ...)` namespaces** returning typed objects, consumed via `configService.getOrThrow<AppConfig>('app')`. No stringly-typed `get('PORT')`, and `process.env` is read **only** in this layer.

## Persistence

- TypeORM is the most DI-native (`@InjectRepository`, decorator entities); Prisma/Drizzle trade DI-nativeness for query-result type safety.
- **`synchronize` is off** everywhere except throwaway test DBs. Use a real migration workflow: a standalone `DataSource` (`data-source.ts`) + `migration:generate|run|revert` scripts producing reviewable, versioned SQL.
- Store money as integer minor units, never floats.

## Testing

- **Unit** (colocated `*.spec.ts`): `Test.createTestingModule` with deps mocked (`getRepositoryToken(Entity)` → mock). Fast, no I/O; assert logic and error paths.
- **E2E** (`test/`): real HTTP with `supertest` against the full pipe→controller→service→DB path, using a real ephemeral Postgres via **Testcontainers** — don't mock away the database. Apply the same global pipes as production.
- Transform with `@swc/jest` for near-instant runs. NestJS is jest-native; keep jest here even if libraries use vitest.

## Production baseline (cheap on day one, painful to retrofit)

Structured logging (`nestjs-pino`), `helmet` + CORS, URI versioning under a global `/api` prefix, a Terminus `/health` with a DB ping, a global exception filter for a uniform error envelope, `app.enableShutdownHooks()`, Swagger docs, and a multi-stage non-root Dockerfile (`node:22-alpine`, `pnpm --frozen-lockfile`, run as the `node` user).

## Build & tooling

- `nest build` with the **SWC builder** and `typeCheck: true` (SWC speed, `tsc` still fails on type errors). Production runs plain `node dist/main.js`.
- `tsconfig`: `strict` with the one concession `strictPropertyInitialization: false` (entities/DTOs declare fields decorators populate), plus `emitDecoratorMetadata` + `experimentalDecorators`. Keep `noUncheckedIndexedAccess` on, but leave **`verbatimModuleSyntax` off** here — with `emitDecoratorMetadata`, a type-only import used in a constructor-parameter position can be elided and silently break DI at runtime. (This is the one universal-baseline flag NestJS drops.)
- Enforce lint/format pre-commit (husky + lint-staged) and again in CI.

## Common NestJS mistakes

- Business logic in controllers → move to services.
- Depending on another module's repository → depend on its exported service.
- `synchronize: true` against a real database → migrations only.
- Forgetting `reflect-metadata` (import once at the entry) → DI/decorators break at runtime.
- Reading `process.env` outside the config layer → centralize + validate at boot.
- No global `ValidationPipe` / missing `whitelist` → clients smuggle extra fields into persistence.
- Skanky e2e that mocks the DB → use Testcontainers so the riskiest layer is actually exercised.
