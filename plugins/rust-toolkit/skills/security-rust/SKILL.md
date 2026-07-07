---
name: security-rust
description: Use when reviewing or hardening a Rust web backend (axum/actix) for security — authz/IDOR, input validation, SQL injection, secrets, auth, TLS, headers/CORS, SSRF, auditing unsafe, and the panic/unwrap-as-DoS footguns — with cargo-audit/cargo-deny.
---

# Security review — Rust

## Overview

Rust's memory safety removes whole bug classes (buffer overflows, use-after-free, data races in safe code) — but **logic vulnerabilities remain**: broken authorization, injection, weak auth, SSRF. Review those by hand, gate the advisory scanners in CI, and treat every `unsafe` block and every `unwrap()` on user input as a finding.

## Tooling pass (gate in CI)

```bash
cargo audit                    # RustSec advisory DB — dependency CVEs
cargo deny check               # advisories + licenses + banned/duplicate deps
cargo clippy --all-targets -- -D warnings
cargo geiger                   # counts unsafe usage across the tree
cargo +nightly miri test       # UB detection in tests (for any unsafe you keep)
```

## Checklist (P0 first)

- **Authorization / IDOR.** Scope every lookup to the owner/tenant: `sqlx::query!("... WHERE id = $1 AND owner_id = $2", id, user_id)`; return 404 on miss. Auth (an extractor/middleware) proves *who*; the query must prove *whose*.
- **Input validation — parse, don't validate.** Deserialize into strong types with `serde` + the `validator` crate; axum/actix extractors reject malformed input. Use newtypes so an `OrderId` can't be a `UserId`. Bound body size (`DefaultBodyLimit` / payload limits).
- **SQL injection.** Prefer `sqlx` **compile-time-checked** queries (`query!`) or parameterized (`$1`), or `diesel`'s query builder — **never `format!` a value into SQL**. Identifiers can't bind — allowlist them.
- **Command injection.** `std::process::Command::new("bin").arg(x)` with separate args — never build a shell string.
- **Auth mechanics.** Hash passwords with the **`argon2`** crate (never a plain hash). JWT: `jsonwebtoken` with an explicit `Validation { algorithms, validate_exp, aud, iss }` — the default or an unpinned algorithm invites alg-confusion. Constant-time compare with the **`subtle`** crate (`ConstantTimeEq`), never `==`.
- **Secrets.** Load from env (`dotenvy` in dev only); wrap in **`secrecy::Secret`** so they zeroize and don't `Debug`-print; validate config at startup. Never commit or log them.
- **TLS.** Use **`rustls`** (memory-safe); never disable certificate verification.
- **Headers / CORS.** `tower-http` `SetResponseHeaderLayer` for security headers; `CorsLayer` with an **explicit origin list** — never `CorsLayer::permissive()` / `AllowOrigin::any()` together with credentials.
- **SSRF** on any fetch of a user URL: allowlist scheme/host, resolve and reject private/link-local ranges (`ipnet`), disable redirects, cap timeout/size.

## Rust-specific footguns

- **`unwrap()`/`expect()`/panic on user input = DoS.** A panic in a request handler aborts that task (and with `panic = "abort"`, the process). Return `Result` and map errors to status codes; never `unwrap` parsed request data.
- **Integer overflow** wraps silently in `--release` (panics in debug). For anything security-sensitive (sizes, indices, balances) use `checked_`/`saturating_` explicitly.
- **`unsafe` blocks** bypass the safety guarantees — audit each one, minimize, and cover with `miri`. `cargo geiger` surfaces how much unsafe (yours and deps') you're trusting.
- **`.unwrap()` in the `main`/setup path** is fine; in request handling it's an availability bug.

## Common mistakes

- Authenticated but not **authorized** per object (IDOR).
- `format!` into SQL instead of `sqlx query!`/parameters.
- `unwrap()`/`expect()` on user input → panic/DoS; unhandled overflow in release.
- `jsonwebtoken` decode without a pinned `Validation`; `==` for secret compare.
- `CorsLayer::permissive()` with credentials; secrets in plain `String`/logs (use `secrecy`).
- Unaudited `unsafe` with no `miri` coverage.
