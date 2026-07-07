# golang-standards/project-layout — annotated reference

The recommendations from <https://github.com/golang-standards/project-layout>, with a verdict for each directory. Use this as a menu, not a mandate.

## Disclaimers (from the repo's own README)

- **Not official.** "This is NOT an official standard defined by the core Go dev team."
- **Application-oriented.** It describes a general layout for applications and is basic in content; it is not tailored to libraries.
- **`/pkg` is contested.** The README itself says it "is not universally accepted and some in the Go community don't recommend it… It's ok not to use it."
- **Overkill for small projects.** The README notes that for learning, a PoC, or a simple personal project the layout is overkill — start flat and grow into it.

## Directory-by-directory verdict

| Directory | What the repo says | Verdict for your projects |
|---|---|---|
| `/cmd` | Main applications; each subdir matches an executable name; keep `main` small. | **Adopt (apps).** `cmd/<name>/main.go`, thin. Libraries omit it unless shipping a reference CLI. |
| `/internal` | Private application and library code; compiler-enforced non-importable. | **Adopt.** Put most code here for apps; impl details for libs. Your hard API/impl boundary. |
| `/pkg` | Library code OK for external apps to import. | **Avoid by default.** Breaks a library's clean import path; contested upstream. Root package + `internal/` instead. Consider only for a large app publishing shared packages — never move a *published* library into it. |
| `/vendor` | Application dependencies. | **Skip.** Use Go modules. Vendor only for hermetic/air-gapped/reproducible-offline builds. |
| `/api` | OpenAPI/Swagger, JSON schema, protocol definitions. | **Conditional.** Only if you actually have API schema/proto files. |
| `/web` | Web app components: static assets, templates, SPAs. | **Conditional.** Only for a web app. A marketing/docs site can live in `docs/` or `site/`. |
| `/configs` | Config file templates / default configs. | **Conditional.** Skip if the app is flag/env-driven; add sample configs here if you ship them. |
| `/init` | System init and process-manager/supervisor configs. | **Conditional.** Add systemd/service units only if you ship them. |
| `/scripts` | Build, install, analysis scripts. | **Skip if a `Makefile` covers it.** Otherwise a home for helper scripts. |
| `/build` | Packaging and CI configs/scripts. | **Skip.** GitHub Actions *must* live in `.github/workflows`; GoReleaser config lives at the repo root. |
| `/deployments` | IaaS/PaaS/container-orchestration configs and templates. | **Adopt.** The home for `Dockerfile`, `docker-compose.yml`, k8s/helm/terraform. (Root-level `Dockerfile` is also common and tooling-friendly — pick one and keep refs consistent.) |
| `/test` | External test apps and test data. | **Prefer `testdata/`** for fixtures (special-cased by the go tool). Use `/test` only for large external test harnesses. |
| `/docs` | Design and user documentation. | **Adopt.** Add `docs/adr/` for Architecture Decision Records. |
| `/tools` | Supporting tools for this project. | **Adopt.** Dev tooling (e.g. `tools/doccheck`, `tools/mutate`) or the project's own tool packages. |
| `/examples` | Examples for apps and public libraries. | **Adopt.** Keep runnable and CI-built; keep dependency-light. |
| `/third_party` | External helper tools, forked code, 3rd-party utilities. | **Conditional.** Only if you actually vendor forked tooling. |
| `/githooks` | Git hooks. | **Skip** unless you distribute hooks. |
| `/assets` | Images, logos, other repo assets. | **Conditional.** Only if you have them. |
| `/website` | Project website data (if not using GitHub Pages). | **Skip** if using GitHub Pages. |

## Rules of thumb

1. **Library → flat root package + `internal/`.** The root package is the API; `import "github.com/you/lib"` should give `lib.Do()`. No `/pkg`, no `/src`.
2. **Application → `cmd/` + `internal/`.** Thin `main`, everything else private and testable.
3. **A directory earns its place by having real contents.** Don't scaffold empty `/api`, `/web`, `/build` just to "match the standard."
4. **Adopt the intent (clean root, clear public/private split), not the letter.** The most valuable conventions here — `cmd/`, `internal/`, `testdata/`, `docs/`, `examples/` — are exactly the ones the Go tool and community already bless.
