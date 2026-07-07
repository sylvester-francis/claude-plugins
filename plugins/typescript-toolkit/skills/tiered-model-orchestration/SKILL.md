---
name: tiered-model-orchestration
description: Use when a task is substantial enough to split across model tiers — routing planning to Opus at max effort, execution to Sonnet, and an independent cross-verification pass to Haiku or Sonnet — for multi-step features, refactors, migrations, or anything where a wrong result is costly.
---

# Tiered model orchestration

## Overview

Match the model to the phase. A hard task has three phases with different needs: **planning** rewards the most capable model thinking hard; **execution** is well-defined work a fast, strong model does cheaply; **verification** needs a *fresh, independent* checker — not the model that just did the work and shares its blind spot.

**Plan (Opus, max) → Execute (Sonnet) → Cross-verify (Haiku / Sonnet).**

The verifier is the whole point: an independent pass with fresh context catches what the executor can't see. Route it to Haiku for cheap mechanical checks, Sonnet for adversarial review.

## The pipeline

| Phase | Model | Effort | Job |
|---|---|---|---|
| **Plan** | Opus 4.8 (`opus`) | `max` | Decompose the task, design the approach, emit a concrete step list **and explicit acceptance checks** the verifier will use. |
| **Execute** | Sonnet 5 (`sonnet`) | `high` (`xhigh` for hard coding/agentic) | Carry out the plan step by step; produce the artifact **plus evidence** (diffs, test output, screenshots). |
| **Cross-verify** | Haiku 4.5 (`haiku`) cheap · Sonnet 5 (`sonnet`, `high`) adversarial | — | **Independently** check the result against the plan's acceptance checks. Try to *refute*, not confirm. Return pass/fail + reasons. |

Loop: if verify fails, feed its reasons into a fresh Execute pass. Escalate the verifier when the check is subtle.

## Running it in Claude Code

**Workflow tool (preferred — deterministic per-phase model + effort):**
```js
const plan  = await agent(planPrompt,            { model: 'opus',   effort: 'max',  schema: PLAN })
const work  = await agent(execPrompt(plan),      { model: 'sonnet', effort: 'high' })
const check = await agent(verifyPrompt(plan, work), { model: 'haiku', schema: VERDICT })
// if (!check.pass) re-run execute with check.reasons; escalate verifier to sonnet/high when hard
```
**Omit `effort` on a `haiku` verifier** — effort isn't supported on Haiku (it errors). Use `sonnet` at `high`/`xhigh` for verification that needs reasoning.

**Agent tool (lighter, manual):** dispatch three subagents in sequence with `model: "opus" | "sonnet" | "haiku"`. Per-phase effort comes from the session; use the Workflow tool when you need precise per-phase effort.

## Rules that make it work

- **The plan must emit acceptance checks.** Verification is only as good as the checklist the planner hands the verifier. No checks → the verifier invents its own bar.
- **The verifier is independent and adversarial.** Fresh context, prompted to break the result — never "confirm this looks right." A confirmer rubber-stamps.
- **Escalate on doubt.** Haiku for mechanical/format checks; Sonnet (`high`) when correctness is subtle; a 2–3 verifier vote for high-stakes.
- **Loop, don't ship-on-first-pass.** Feed failed-verification reasons back into execution.

## When to use / skip

Use for substantial, multi-step, costly-if-wrong work — features, refactors, migrations, research. **Skip** trivial one-shots: the three-phase overhead outweighs the value; just do it directly.

## Model reference (current)

- `opus` = **Opus 4.8** (`claude-opus-4-8`) — most capable; run the plan at `max`.
- `sonnet` = **Sonnet 5** (`claude-sonnet-5`) — near-Opus on coding/agentic; `low`–`max` effort (`xhigh` sweet spot for hard coding).
- `haiku` = **Haiku 4.5** (`claude-haiku-4-5`) — fastest/cheapest; **no effort parameter**.

## Common mistakes

- **Verifier = the executor's model told to "check your work"** → shares the blind spot. Use a separate, independent pass.
- **No acceptance checks from the plan** → verification is vibes.
- **Passing `effort` to Haiku** → errors; drop it (or verify with Sonnet).
- **Running the full pipeline on a trivial task** → overhead > value.
- **Planning at low effort** → the plan is where Opus/`max` pays off most; don't skimp there.
