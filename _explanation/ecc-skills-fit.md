---
title: Which ECC skills actually fit this pipeline
description: A real audit of the 280-skill ECC catalog against this project's own seven stages -- including honest negatives.
order: 4
---

# Which ECC skills actually fit this pipeline

This project uses exactly one surface of `ecc-universal` today: `ecc-plan-canvas`, for the human
check-in channel (see [Review a mandatory check-in]({{ '/how-to/review-a-checkin/' | relative_url }})).
The catalog ships 280 skills total. This is a real audit against them — read directly, not assumed
from names — asking one honest question per pipeline stage: does a real skill fit, or doesn't it?

## Per-stage findings

| Stage | Candidate | Fit? |
|---|---|---|
| `plan` | `architecture-decision-records` | **Yes.** The zylos envelope's `constraints` field already functions as an informal decision record threaded stage to stage — this skill formalizes exactly that pattern (context, alternatives, rationale). |
| `implement` | `android-clean-architecture` | **Yes.** Names Room/SQLDelight/Ktor and UseCase/Repository patterns directly — relevant to real, already-made decisions in `CADS-webconference-android` (e.g. choosing plain `SQLiteOpenHelper` over Room to avoid a new toolchain surface). |
| `test` | `rust-testing` / `kotlin-testing` | **Yes, split correctly.** This project genuinely has two sides — Rust (`pipeline`/`web`) and Kotlin (the Android app) — and each skill names the real toolchain that side actually uses. |
| `verify` | `verification-loop` | **No.** Its own phases are `npm run build`/`pnpm build` — this project has no JS build step anywhere in its real gate. An honest negative, not force-adopted. |
| `review` | `security-review` | **Yes.** "adding authentication, handling user input, working with secrets, creating API endpoints" describes `web/src/main.rs` almost exactly. |
| `improve` | `agent-architecture-audit` | **Yes, strongly.** Its own description — "diagnostic workflow for agent systems that hide failures behind wrapper layers, stale memory, retry loops" — names precisely the failure class a same-day deploy-race incident on this exact project turned out to be: a silent process gap, a wrong result, no visible error. |
| `remember` | `unified-memory` (ECC Memory Vault) | **No, deliberately.** The zylos envelope already exists to solve a *different* problem than ECC's generic cross-harness (Claude/Codex/Hermes/Cursor/OpenCode) context-sharing layer — this is a real, already-made design decision, re-confirmed here rather than silently revisited. |

## The one finding that isn't about any single stage

**`loop-design-check`** reviews an agent loop for "the ways loops go wrong: spinning and burning
tokens, Goodhart-gaming the verifier, running a wrong answer to completion" — decidability,
boundaries, fallback, judge independence. That isn't a fit for one of the seven stages; it's a fit
for **this project's own super-loop** (`AbortCriteria`, `should_abort`, `should_checkin`) as a
whole. Arguably the single most relevant unadopted skill in the entire catalog to what "The
Development System" actually is — not run yet, flagged here as the clear next candidate.

## What this audit is, and isn't

A fit assessment, not an adoption. None of the "yes" rows above are wired into the pipeline's actual
behavior yet — `devsystem.review` doesn't reference `security-review`'s checklist today,
`devsystem.plan`'s output isn't ADR-structured. That's real, separate follow-up work. The full audit,
including exact quoted skill descriptions, lives in
[`docs/ecc-skills-audit.md`](https://github.com/scimbe/CADS-devsystem/blob/main/docs/ecc-skills-audit.md).
