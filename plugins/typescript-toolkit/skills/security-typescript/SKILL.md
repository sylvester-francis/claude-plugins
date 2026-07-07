---
name: security-typescript
description: Use when reviewing or hardening a Node.js/TypeScript backend (Express or NestJS) for security — input validation, injection, authn/authz and IDOR, secrets, headers, CORS, rate limiting, dependency CVEs, prototype pollution, and SSRF.
---

# Security review — TypeScript / Node

## Overview

Review worst-blast-radius first. **Broken authorization ships more breaches than injection does** — check it on every data-fetching route. Validate every external input at the boundary (parse, don't cast), and let the tools catch the mechanical issues so review time goes to authz and business logic.

## Recon (before reading code)

```bash
npm audit --audit-level=high          # dependency CVEs (also osv-scanner, Dependabot)
npx semgrep --config=p/owasp-top-ten --config=p/typescript
npx gitleaks detect --source . --redact   # secrets already in git history
node -v                               # old Node = unpatched platform CVEs; use active LTS
```

Wire `npm audit --audit-level=high` + semgrep into CI so it doesn't rot.

## The checklist (priority order)

**1. Authorization / IDOR — the one people skip.** Auth tells you *who*; almost every endpoint also needs *may this user touch this row?* Scope every lookup to the owner/tenant:
```ts
// before: any logged-in user reads any order
const o = await db.order.findUnique({ where: { id: req.params.id } });
// after: scoped to the caller; 404 (not 403) so existence doesn't leak
const o = await db.order.findFirst({ where: { id: req.params.id, userId: req.user.id } });
if (!o) return res.sendStatus(404);
```
Grep handlers for `findUnique`/`findById` and confirm each carries an owner/tenant predicate. Multi-tenant → enforce `tenantId` centrally (repository layer / RLS), not per-handler hope.

**2. Input validation — parse at the boundary, don't cast.** `req.body as Dto` is a lie; the data is still whatever the client sent. Validate body/query/params/headers into a typed object:
```ts
const CreateUser = z.object({ email: z.string().email().max(254), age: z.number().int().min(0).max(120) }).strict();
// .strict() rejects unknown keys -> blocks mass-assignment + prototype pollution
```
NestJS analog: `new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true })`. Never accept `role`/`isAdmin`/`tenantId` from the client. Bound every string/array length (ReDoS/DoS). Cap body size (`express.json({ limit: '100kb' })`).

**3. Injection.** SQL: parameterize (Prisma tagged template `db.$queryRaw\`...\${x}\``, `pool.query('... $1', [x])`) — never `$queryRawUnsafe` with concatenation. NoSQL: operator injection (`{"password":{"$ne":null}}`) — step 2's `z.string()` rejects the object. Command: `execFile('convert', [file])`, never `exec(\`convert ${file}\`)`. Path traversal: `path.resolve(root, name)` then confirm it `startsWith(root + sep)`.

**4. Auth mechanics.** Pin the JWT algorithm (`jwt.verify(t, PUBLIC_KEY, { algorithms: ['RS256'], issuer, audience })`) — the RS256→HS256 confusion attack signs with your public key as an HMAC secret. Short access TTL + revocable refresh (jti denylist / token version). Sessions: cookies `httpOnly`+`secure`+`sameSite`, regenerate id on login (fixation), server-side store.

**5. Secrets.** Validate env at boot (`z.object({ JWT_SECRET: z.string().min(32), ... }).parse(process.env)`) so a missing secret fails loud. `.env` gitignored; anything hardcoded is already in history — rotate it.

**6. Headers + CORS.** `app.use(helmet())`; for a JSON API set a strict CSP. CORS is an **allowlist, never a reflector** — `cors({ origin: true, credentials: true })` lets any site make authenticated requests as your user.

**7. Rate limiting.** Blanket limiter + stricter on auth routes, backed by a shared store (Redis) so it works behind >1 instance. `bcrypt`/`argon2` for passwords (never fast hashes); `crypto.timingSafeEqual` for token compare.

**8. SSRF** on any server-side fetch of a user URL: allowlist the host, resolve DNS and reject private/link-local ranges (blocks `169.254.169.254`), `redirect: 'error'`, re-check after DNS (rebinding).

## Tools

`npm audit` / `osv-scanner` (CVEs) · `semgrep p/owasp-top-ten` + `eslint-plugin-security` (static) · `gitleaks` (secrets) · `tsc --strict` (kills the `as` casts that hide unvalidated input).

## Common mistakes

- Authenticated but not **authorized** — no per-object ownership check (IDOR).
- `req.body as Dto` instead of a parsed, `.strict()`/`whitelist` schema.
- `cors({ origin: true, credentials: true })` — reflects any origin with cookies.
- JWT verify without pinning `algorithms` → alg-confusion.
- Fast hash for passwords; string `==` for token compare.
