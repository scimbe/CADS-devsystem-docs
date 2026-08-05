---
title: Requirements, verification, and automode
description: Why acceptance-criteria judgment stays human by default, and what opting in actually authorizes.
order: 1
---

# Requirements, verification, and automode

A CADS-devsystem [`Requirement`](https://github.com/scimbe/CADS-devsystem/blob/main/pipeline/src/runner.rs)
is modeled on [EARS notation](https://en.wikipedia.org/wiki/Easy_Approach_to_Requirements_Syntax):
a real trigger, a real system behavior, and a list of concrete `acceptance_criteria` — distinct from
a `Milestone` (a checkpoint) or a `BacklogItem` (a task). It gives both a human reviewer and any
role-filler agent something checkable to verify against, instead of a vague wish.

## What actually counts as a real requirement -- and a real acceptance criterion

**The `statement` field itself, first.** Until 2026-08-05 this endpoint accepted literally any
non-empty string as a "requirement" — a live test proved `{"statement":"asdf",...}` got a real
`200`, despite this whole feature being built around EARS notation. Fixed as a hard gate: a
statement must contain **"SHALL"** (case-insensitive) — the one universal, defining keyword across
every real EARS requirement type. Deliberately doesn't also require `"WHEN"`: a real *ubiquitous*
EARS requirement (no trigger clause at all, e.g. "THE SYSTEM SHALL always encrypt messages at
rest") is legitimate and would be wrongly rejected by a stricter check.

`POST /api/runs/{id}/requirements` doesn't just check that a criterion is non-empty, either. That
alone turned out to be a real gap: a live test of this exact endpoint (simulating the least
competent realistic role-filler on purpose — see [the goal document](https://github.com/scimbe/CADS-devsystem/blob/main/docs/development-system-goal.md)'s
incompetent-agent stress test) proved `"ok"`, `"."`, `"done"`, and — found along the way, not
planned — a criterion that was **only a zero-width space** (U+200B) all sailed through as real,
checkable criteria. The zero-width case is the more interesting one: `.trim()` only strips Unicode
`White_Space` characters, and U+200B's Unicode category is *Format* (Cf), not *White_Space* — so it
rendered as an apparently blank checkbox line in the GUI while technically passing "not empty".

Every acceptance criterion now needs a minimum count of **alphanumeric** characters, not just
non-empty content — one rule that catches both problems, since an invisible-character-only string
has zero alphanumeric characters under this count:

```
$ curl -X POST .../api/runs/{id}/requirements -d '{"statement": "WHEN a user submits an empty message, THE SYSTEM SHALL reject it", "acceptance_criteria": ["ok"]}'
acceptance criterion "ok" doesn't have enough real content to be checkable (minimum 5
letters/digits) -- "ok", ".", or an invisible character aren't real acceptance criteria.
HTTP 400
```

The bar is deliberately low (5 alphanumeric characters) — this filters non-answers, not real short
criteria. `"no crash"` (7 alphanumeric characters) is accepted without issue.

## Two independent, explicit signals

`verified` (the whole requirement) and `verified_criteria` (each individual criterion) are
deliberately kept separate rather than auto-derived from one another. A human confirming "yes, this
whole requirement is done" and a human ticking off individual criteria are two different judgment
calls — coupling them silently would be a real design decision made *for* the operator, not asked
of them.

## Why automode is per-requirement, not a project setting

The natural next question, once acceptance criteria exist at all, is: can `devsystem.assistant`
judge them itself, instead of a human clicking checkboxes? The honest answer is: **by default, no**
— every requirement starts, and stays, 100% human-driven.

A requirement's owner can opt a *specific* requirement into "automode" via `auto_judge`. This is
deliberately **not** a global run or account setting. Opting in is a real, considered choice on one
requirement — not something every future requirement should silently inherit just because automode
was turned on once, somewhere, for something else. The flag is real and live in the API/GUI today;
setting it only *authorizes* future judgment — it doesn't perform any judgment itself yet. Building
the actual logic (grounding the assistant in a run's real iteration history/evidence before it
touches a checkbox, and being honest when the evidence doesn't support a clean yes/no) is real,
separate, still-open work.

## The real, mandatory review gate

Every signal above (`verified`, `verified_criteria`, `auto_judge`, `proposed_by`) was, until
2026-08-05, purely advisory: nothing actually stopped anyone (human or role-filler) from marking a
requirement `verified` with zero real review behind it. This project's own
[goal document](https://github.com/scimbe/CADS-devsystem/blob/main/docs/development-system-goal.md)
states the governing principle directly: *"it is the fault of the pipeline, not the user of the
pipeline, if the process leads him not to the perfect result."* A soft annotation a role-filler can
freely ignore doesn't satisfy that — so this is now a real, hard block, not another advisory signal.

**The rule**: if a run's own `PipelineSpec` declares a `review` role (tag `"review"`, service
`Custom("devsystem.review")`), marking one of its requirements verified is rejected with a real
`409 Conflict` unless a `devsystem.review` iteration that `succeeded` and named that exact
requirement (via `requirement_indices`) already exists in the run's history. Un-verifying is always
allowed unconditionally — loosening a claim never needs a review to justify it.

**Scoping, deliberately**: a run that never declares `review` (every new run's default —
`plan_only_spec` has no such role) is never gated at all. There's nothing to hold a run accountable
to a role it never opted into; the gate only bites once a run's own spec says review is part of its
process.

**A qualifying review has to clear two real, mechanical bars, not just exist.** Found live, via this
project's own incompetent-agent stress test, not designed in up front: an early version of this gate
checked only that *a* successful `devsystem.review` iteration named the requirement, and a
one-line rubber-stamp (`"looks fine to me"`) satisfied it just as well as real scrutiny would have.
Fixed with a minimum feedback length (25 characters) -- then, once that shipped, the *next*
realistic lazy move (padded filler well past the length bar: `"looks good looks good looks good
looks good"`) satisfied that too. Both real gaps are closed the same honest way: simple, explainable,
mechanical proxies, not fake LLM-judgment-in-disguise -- **25 characters AND 8 distinct words**
(case-insensitive, punctuation-collapsing, so `"Good! good? GOOD."` still counts as one distinct
word). Neither bar claims to verify the review is actually *good* -- only that it isn't trivially
empty or trivially repetitive. A generic-but-varied review ("looks good, works fine, nothing to
flag, all clear here") would still clear both bars without being real scrutiny either -- a known,
honestly-named, still-open gap, not claimed solved.

Real, live proof against the actual deployment:

```
$ curl -X POST .../api/runs/{id}/requirements/0/toggle   # no review iteration exists yet
requirement 0 cannot be marked verified yet -- this run declares a devsystem.review
role, but no successful devsystem.review iteration addressing requirement 0 (via its
requirement_indices) exists yet. Submit one first.
HTTP 409

$ curl -X POST .../api/runs/{id}/iterate -d '{"stage":"devsystem.review","feedback":"reviewed, approved","succeeded":true,"requirement_indices":[0]}'
HTTP 200

$ curl -X POST .../api/runs/{id}/requirements/0/toggle   # a review exists, but it's too short/repetitive
requirement 0 cannot be marked verified yet -- every devsystem.review iteration addressing
it is too short or too repetitive to plausibly be real scrutiny (best is iteration 1, 18
character(s) and 2 distinct word(s); minimum 25 characters AND 8 distinct words). A
rubber-stamp or padded filler review doesn't satisfy this gate.
HTTP 409

$ curl -X POST .../api/runs/{id}/iterate -d '{"stage":"devsystem.review","feedback":"Checked the actual EditText handling directly against the acceptance criteria, confirmed the real behavior matches.","succeeded":true,"requirement_indices":[0]}'
HTTP 200

$ curl -X POST .../api/runs/{id}/requirements/0/toggle   # now a real, substantive review exists
{"requirements":[{...,"verified":true,...}]}
HTTP 200
```

The GUI surfaces the block as a real error message next to the Requirements panel, and — a related
gap found and fixed alongside the gate itself — reverts the checkbox's own visual state rather than
leaving it looking checked when the toggle was actually rejected.

## Who wrote this requirement: `proposed_by`

A requirement can come from two real places: a human typing directly into the GUI's Requirements
panel, or `devsystem.assistant` proposing one from a chat exchange (its real `AddRequirement`
action). Until 2026-08-05 there was no way to tell which was which once it landed — both looked
identical in the panel. `proposed_by` closes that gap: `null` means a human wrote it directly;
a stage tag (today, always `"devsystem.assistant"`) means an LLM proposed it and it's still that
LLM's first draft, not yet reviewed. The GUI shows a real **LLM-proposed** badge on any requirement
carrying the tag.

Real, live proof against the actual deployment (not just the hermetic test suite) — adding one of
each and reading them straight back:

```
$ curl -X POST https://devsystem-demo.bunsenbrenner.org/api/runs/{id}/requirements \
    -d '{"statement": "WHEN a human writes a requirement directly, THE SYSTEM SHALL record it with no proposed_by", "acceptance_criteria": ["checkable"]}'
{"requirements":[{"...","proposed_by":null,"statement":"WHEN a human writes a requirement directly, THE SYSTEM SHALL record it with no proposed_by",...}]}

$ curl -X POST https://devsystem-demo.bunsenbrenner.org/api/runs/{id}/requirements \
    -d '{"statement": "WHEN devsystem.assistant proposes a requirement, THE SYSTEM SHALL record it with proposed_by set", "acceptance_criteria": ["checkable"], "proposed_by": "devsystem.assistant"}'
{"requirements":[{...},{"...","proposed_by":"devsystem.assistant","statement":"WHEN devsystem.assistant proposes a requirement, THE SYSTEM SHALL record it with proposed_by set",...}]}
```

The GUI's Requirements panel renders the second kind with a real **LLM-proposed** badge next to its
statement; the first gets none.

This isn't decorative. It's the concrete answer to a real design question this project's own
[goal document](https://github.com/scimbe/CADS-devsystem/blob/main/docs/development-system-goal.md)
asks directly: an LLM is meant to fill in a spec's first draft at a high level, but the human has to
be able to tell, per detail, which parts are still that first draft waiting on their review versus
already theirs to trust. Before `proposed_by` existed, that distinction only lived in whoever's
memory happened to remember which requirement came from which conversation.

## Taking it with you: a real Markdown export

Every requirement shown in the GUI is also a real, downloadable document —
`GET /api/runs/{id}/requirements/export`, or the **⬇ Download as Markdown** link in the
Requirements panel itself. Real, live output from a run with one requirement already reviewed and
verified:

```
$ curl .../api/runs/{id}/requirements/export
# Requirements: `{id}`

1/1 verified.

## 1. ✅

WHEN a user taps send with an empty message, THE SYSTEM SHALL not attempt to send anything and SHALL leave the input focused for retry

*Human-authored.*

Acceptance criteria:

- [ ] empty/whitespace-only input never calls sendText
- [ ] the input field keeps focus after a no-op tap
- [ ] no crash or exception is thrown
```

Notice the whole requirement shows `✅` verified while its individual acceptance criteria still
show unchecked — that's not a bug in the export, it's the same real, deliberate separation §2 above
describes: `verified` (the whole requirement) and `verified_criteria` (each individual criterion)
are two independent signals, and this run's owner confirmed the requirement as a whole without
separately ticking off each criterion. The export renders reality exactly as it is, not a cleaned-up
version of it.

## Why this matters for reproducibility

Fine-grained requirements are, honestly, hard to make perfectly reproducible run to run — the exact
wording of a trigger or a criterion shapes what gets built. What *is* reproducible, and what this
whole model is built to support, is the discipline: every requirement is a real, checkable claim,
every verification is an explicit, attributable signal (a specific human, or an explicitly-authorized
assistant), and nothing about "done" is ever silently assumed.
