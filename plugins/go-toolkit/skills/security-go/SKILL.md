---
name: security-go
description: Use when reviewing or hardening a Go HTTP/JSON backend for security — server timeouts, authn/authz and IDOR, input validation, SQL injection, secrets, TLS, security headers, SSRF, rate limiting, and dependency CVEs (govulncheck/gosec).
---

# Security review — Go

## Overview

Automated triage first (let scanners find the boring 80%), then trace one request end to end — listener → middleware → handler → DB/outbound — looking for a trust boundary crossed without a check. **Authorization bugs are the ones scanners miss; read those by hand.** Put hardening in the middleware chain so a new handler is safe-by-default, not safe-if-remembered.

## Tooling pass (gate these in CI)

```bash
go vet ./...
govulncheck ./...        # official, reachability-aware CVE scan of deps + stdlib
gosec -severity medium ./...
staticcheck ./...
go mod verify            # deps untampered
gitleaks detect          # secrets in git history, not just the tree
```

`gosec` rules never to ignore: **G101** hardcoded creds · **G107** SSRF · **G201/G202** SQL via string formatting · **G204** command injection · **G401/G501** MD5/SHA1 · **G402** `InsecureSkipVerify` · **G404** `math/rand` for security.

## Checklist (P0 = exploitable now)

- **Server timeouts — the #1 Go footgun.** Bare `http.ListenAndServe` has no timeouts → one slow client holds a connection open forever (Slowloris). Always use a configured `http.Server`:
  ```go
  srv := &http.Server{
      Addr: ":8080", Handler: mux,
      ReadHeaderTimeout: 5 * time.Second, ReadTimeout: 10 * time.Second,
      WriteTimeout: 15 * time.Second, IdleTimeout: 60 * time.Second,
      MaxHeaderBytes: 1 << 20,
      TLSConfig: &tls.Config{MinVersion: tls.VersionTLS12},
  }
  ```
- **Authorization / IDOR.** Every handler taking an `id` must verify the caller *owns* the object, not just that they're logged in: `WHERE id = $1 AND owner_id = $2`. Deny-by-default; enforce authn in middleware wrapping the router (missing-by-default is the common hole).
- **SQL injection.** 100% parameterized (`db.QueryContext(ctx, "... WHERE email = $1", email)`), never `fmt.Sprintf` into a query. Identifiers can't be bound — allowlist them.
- **Bounded, strict input.** `http.MaxBytesReader` *before* decoding; `json.Decoder` with `DisallowUnknownFields()`; explicit `Validate()` (don't trust zero-values). Path traversal: `filepath.Clean` + confine to a base dir. Exec: `exec.Command("bin", arg)` with fixed argv, never `sh -c` with interpolation.
- **Crypto.** `crypto/rand` for tokens/session IDs (never `math/rand`). Constant-time compare — hash to fixed length then `subtle.ConstantTimeCompare`. JWT: verify `alg` (reject `none`), check `exp`/`aud`/`iss`.
- **Secrets.** None committed; load from env/secret manager; never logged. Strip `\n`/`\r` from user-controlled values before logging (log injection).
- **SSRF** on any fetch of a user URL: allowlist host/scheme, block link-local/metadata (`169.254.169.254`) and RFC1918, outbound `http.Client{Timeout: ...}`, never `InsecureSkipVerify`.

## P1 (defense-in-depth, in the chain)

Security-headers middleware (`X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Content-Security-Policy: default-src 'none'`, HSTS); CORS with an explicit origin allowlist (never `*` with credentials); panic-recovery middleware (500, not a leaked stack); generic client errors (full detail server-side only); per-client rate limit (`golang.org/x/time/rate`) + `netutil.LimitListener`; propagate `r.Context()` into DB/outbound calls.

```go
// hardening lives in the chain -> new handlers are safe by default
handler := secureHeaders(throttle(recoverPanic(auth(mux))))
```

## Output shape

Prioritize: **P0** (SQLi, missing authz/IDOR, `InsecureSkipVerify`, hardcoded secret, no server timeouts) → **P1** (headers, rate limit, body caps, generic errors) → **P2** (CI gating on `govulncheck`/`gosec`, dep pinning), each with `file:line` + a before/after diff and the CI change so it can't reland.

## Common mistakes

- Bare `http.ListenAndServe` with no timeouts (Slowloris).
- Authenticated but not **authorized** per object (IDOR).
- `fmt.Sprintf` into a SQL string; unbounded `json.Decode` that ignores errors + extra fields.
- `math/rand` for tokens; `==` for secret compare.
- `InsecureSkipVerify: true` "just for now."
