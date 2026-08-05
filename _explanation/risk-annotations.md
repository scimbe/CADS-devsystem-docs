---
title: How real risk annotations work
description: Mechanical checks over a run's own history and spec -- not an LLM guess, real patterns a human reviewer would otherwise have to spot by hand.
order: 5
---

# How real risk annotations work

`GET /api/runs/{id}`'s `risks` field, and `GET /api/runs`'s `risk_count`, aren't an LLM's opinion —
every entry is a real, mechanical check over a run's own actual history (and, for one dimension,
its own actual spec), traceable back to concrete evidence a human can verify in seconds. This
project's own convention, stated directly in the source: simple, explainable checks, never fake
LLM-judgment-in-disguise.

## Two dimensions, for a real reason

- **History-only checks** (`preflight_annotations`) — a security-relevant keyword in the latest
  iteration's feedback, `devsystem.implement` running before any `devsystem.test` iteration, a new
  service proposal with no `price_ceiling`. These only need a run's `RunState` — its iteration
  history — so they're usable everywhere a run's history is available, including
  `devsystem_checkin`'s own binary (which never loads the run's spec at all).
- **Process-level checks** (`process_annotations`, added 2026-08-05) — need the run's own live
  `PipelineSpec` too, since they're about *which roles are declared*, not just what already
  happened. The first one: a run with 3+ real successful iterations that has never declared a
  `devsystem.review` role. Kept as a genuinely separate function rather than folded into
  `preflight_annotations` — extending that one's signature would have broken every existing caller
  that only ever has a bare `RunState`.

## Real, live example — both dimensions firing on the same run

A real test run, three successful `devsystem.implement` iterations, no `devsystem.test` iteration
ever run, and no `devsystem.review` role ever declared:

```
$ curl .../api/runs/{id}
{
  "risks": [
    {
      "label": "no test stage before implement",
      "evidence": "devsystem.implement first ran at iteration 1, with no devsystem.test iteration before it"
    },
    {
      "label": "no review role declared despite real progress",
      "evidence": "3 successful iteration(s) so far, but this run has never declared a devsystem.review role -- gap #2's mandatory review gate (requirements can't be marked verified without a real review) has no teeth here at all, since it only applies once review is declared."
    }
  ]
}
```

Compare against the real `webconference-android` run: it declared `devsystem.review` back at
iteration 8 (see [the goal document](https://github.com/scimbe/CADS-devsystem/blob/main/docs/development-system-goal.md)'s
own incident log for why), so today it shows `"risks": []` — the same check, correctly silent once
the actual condition it flags no longer holds.

## Why "3+ successful iterations", not a specific stage name

The process-level check deliberately doesn't look for `devsystem.implement` by name. This
pipeline's own roles are custom-named per project — `webconference-android` itself never has a
literal `devsystem.implement` stage, only project-specific ones like
`devsystem.android_native_bridge`. Counting real *successful* iterations (`succeeded: true`) is the
honest, general signal available across every run, regardless of what its stages happen to be
called — and only successful ones count, since a string of failed attempts isn't the "real
progress" this check is actually about.
