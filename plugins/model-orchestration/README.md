# model-orchestration

A Claude Code plugin that routes a substantial task across model tiers.

## Install

```
/plugin marketplace add sylvester-francis/claude-plugins
/plugin install model-orchestration@sylvester-plugins
```

## What it does

The bundled skill activates on multi-step, costly-if-wrong work (features,
refactors, migrations, research) and runs it as a three-phase pipeline:

**Plan (Opus, max effort) → Execute (Sonnet) → Cross-verify (Haiku / Sonnet).**

- **Plan** — Opus 4.8 at `max` effort decomposes the task and emits explicit
  acceptance checks.
- **Execute** — Sonnet 5 carries out the plan and produces evidence.
- **Cross-verify** — an *independent, adversarial* pass (Haiku for cheap checks,
  Sonnet for subtle ones) tries to refute the result against the plan's checks;
  on failure, its reasons feed back into a fresh execution pass.

It shows how to wire the pipeline with the Workflow tool (per-phase `model` +
`effort`) or the Agent tool, and calls out the gotchas (Haiku takes no `effort`
parameter; the verifier must be independent, never the executor told to "check
your work").

## License

[Apache-2.0](../../LICENSE).
