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
  iteration's feedback, `devsystem.implement` running before any *substantive* `devsystem.test`
  iteration, a new service proposal with no `price_ceiling`, and a `succeeded: true` iteration whose
  own feedback admits a known defect. These only need a run's `RunState` — its iteration history —
  so they're usable everywhere a run's history is available, including `devsystem_checkin`'s own
  binary (which never loads the run's spec at all).

  **"Substantive" is load-bearing, not decorative** — found live by this project's own
  incompetent-agent stress test, 2026-08-05: the test-before-implement check originally only asked
  *whether* a `devsystem.test` record existed, not whether it had any real content. A rubber-stamp
  `feedback: "tests pass"` iteration silently satisfied it exactly as well as real testing would
  have, making the risk annotation vanish even though nothing real had actually been tested. Fixed
  with the same two mechanical bars the review gate uses (see
  [Requirements, verification, and automode]({{ '/explanation/requirements-and-automode/' | relative_url }})) —
  25+ characters and 8+ distinct words. A `devsystem.test` record that doesn't clear both doesn't
  count as real evidence testing happened, and the check falls through to flagging the risk as if
  no test iteration existed at all.

  **Contradicting yourself doesn't go unnoticed either** — a different flavor of the same
  discipline, this one going after a gap [the goal document's §5 quality-bar table](https://github.com/scimbe/CADS-devsystem/blob/main/docs/development-system-goal.md)
  already named directly rather than one found by simulating a lazy agent: nothing stopped an
  iteration from claiming `succeeded: true` while its own feedback admitted a real, open defect.
  `DEFECT_ADMISSION_PHRASES` -- six specific multi-word phrases ("known issue", "known bug", "not
  fixed", "not implemented", "workaround needed", "still broken") -- flags exactly that
  contradiction. Deliberately narrow: single common words like "broken" alone would false-positive
  on something like "fixes the previously broken X", so the phrases are specific enough to mean what
  they say. Only fires on `succeeded: true` — a FAILED iteration honestly admitting it's broken is
  the behavior this check wants to encourage, not flag.
- **Process-level checks** (`process_annotations`, added 2026-08-05) — need the run's own live
  `PipelineSpec` too, since they're about *which roles are declared*, not just what already
  happened. The first one: a run with 3+ real successful iterations that has never declared a
  `devsystem.review` role. Kept as a genuinely separate function rather than folded into
  `preflight_annotations` — extending that one's signature would have broken every existing caller
  that only ever has a bare `RunState`.

## Real, live example — two findings, one run

A real test run: a single `devsystem.implement` iteration, marked `succeeded: true`, whose own
feedback admits an unfixed defect, with no `devsystem.test` iteration ever run and no
`devsystem.review` role ever declared:

```
$ curl .../api/runs/{id}/iterate -d '{"stage":"devsystem.implement","feedback":"Shipped the retry-on-failure feature. Known issue: it crashes on a null message id, not fixed yet, workaround needed before real use.","succeeded":true}'

$ curl .../api/runs/{id}
{
  "risks": [
    {
      "label": "no test stage before implement",
      "evidence": "devsystem.implement first ran at iteration 1, with no devsystem.test iteration before it that's substantive enough to count as real evidence testing happened (25+ characters and 8+ distinct words of feedback, not a rubber-stamp)"
    },
    {
      "label": "succeeded iteration admits a known defect",
      "evidence": "iteration 1's own feedback contains \"known issue\" while marked succeeded:true -- goal doc §5's Vertragsgemäße/Sachmangelfreie row names this exact gap: nothing else blocks marking work \"done\" with open, known defects"
    }
  ]
}
```

(The process-level dimension -- `"no review role declared despite real progress"` -- needs 3+ real
successful iterations to fire, so a single-iteration example like this one never triggers it; see
the comparison right below for what its *absence* looks like on a run that has since declared
`review`.)

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
