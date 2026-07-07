---
name: security-python
description: Use when reviewing or hardening a Python web backend (FastAPI, Django, or Flask) for security — input validation, injection, authz/IDOR, secrets, auth, headers/CORS, SSRF, and Python-specific footguns (pickle, yaml.load, shell=True, debug mode).
---

# Security review — Python

## Overview

Review worst-blast-radius first: **authorization bugs ship more breaches than injection**. Validate every external input at the boundary with a schema (pydantic / DRF serializer), and let the tools catch the mechanical issues. Watch the Python-specific footguns that turn benign-looking code into RCE.

## Recon (gate in CI)

```bash
pip-audit                      # dependency CVEs (or `safety scan`)
bandit -r src -ll              # security linter (also ruff's `S` / flake8-bandit rules)
semgrep --config=p/python --config=p/owasp-top-ten
gitleaks detect --redact       # secrets in git history
```

## Checklist (priority order)

**1. Authorization / IDOR.** Auth proves *who*; each data route also needs *may this user touch this row?* Scope the query to the owner/tenant — `Order.objects.filter(id=id, user=request.user)` / `select().where(Order.owner_id == user.id)` — and 404 (not 403) on miss. Grep detail/update/delete views for a lookup that trusts only the URL id. Use object-level permissions (DRF `permission_classes`, Django guardian), not view-level only.

**2. Input validation at the boundary.** Parse into a typed model, don't trust `request` dicts. FastAPI: pydantic models (rejects unknown fields; set `model_config = ConfigDict(extra="forbid")`). Django/DRF: forms/serializers. **Never accept `is_admin`/`role`/`owner_id` from the client** (mass assignment) — set them server-side. Bound string/list sizes; cap request body size at the server/proxy.

**3. Injection — Python's sharp edges.**
- **SQL:** the ORM parameterizes for free; raw SQL must use params — `cursor.execute("... WHERE e=%s", [email])`, `text("... :e").bindparams(e=email)` — never f-string/`.format` into SQL.
- **Command:** `subprocess.run(["convert", path])` with a list, **never `shell=True`** with interpolation.
- **Deserialization = RCE:** never `pickle.loads` untrusted data; `yaml.safe_load` (never `yaml.load`); avoid `eval`/`exec` on input.
- **SSTI:** keep Jinja2 autoescape on; never render a user string as a template.

**4. Auth mechanics.** Hash passwords with **argon2/bcrypt** (`passlib`/`argon2-cffi`) — never `hashlib`/`md5`/plain. JWT: pin the algorithm (`jwt.decode(t, key, algorithms=["RS256"], audience=..., issuer=...)`) — an unpinned decode allows `alg:none`/RS256→HS256 confusion. Constant-time compare with `secrets.compare_digest` (never `==`). Short token TTL + revocable refresh.

**5. Secrets.** Load from env / secret manager; validate at startup (`pydantic-settings` `BaseSettings`) so a missing secret fails loud. `.env` gitignored; rotate anything already committed (it's in history). Never log tokens/PII.

**6. Headers, CORS, rate limiting.** Security headers via middleware; `SecurityMiddleware` (Django) / `secweb`. CORS is an **allowlist** — never `allow_origins=["*"]` with `allow_credentials=True`. Rate-limit (slowapi / django-ratelimit), stricter on auth routes.

**7. SSRF** on any fetch of a user URL: allowlist scheme/host, resolve DNS and reject private/link-local ranges (blocks `169.254.169.254`), disable redirects, cap timeout/size.

## Python-specific footguns

- **`DEBUG=True` in prod** (Django/Flask) → leaks tracebacks + settings, enables the debugger. Force off in prod config.
- **`assert` for security checks** → stripped under `python -O`; use real conditionals.
- **`pickle`/`yaml.load`/`eval`/`marshal`** on untrusted input → RCE.
- **`subprocess(..., shell=True)`** with user input → command injection.
- **Flask `SECRET_KEY`/Django `SECRET_KEY`** hardcoded or weak → session forgery.

## Common mistakes

- Authenticated but not **authorized** per object (IDOR).
- Trusting `request.json`/`request.POST` instead of a schema with `extra="forbid"`.
- `yaml.load`/`pickle.loads` on untrusted data; `shell=True`.
- `hashlib` for passwords; `==` for token/secret compare.
- `allow_origins=["*"]` + credentials; `DEBUG=True` shipped.
