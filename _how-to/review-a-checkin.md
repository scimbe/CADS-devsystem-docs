---
title: Review a mandatory check-in
description: Turn a run's own risk annotations and history into a real ecc-plan-canvas review session.
order: 3
---

# Review a mandatory check-in

Every run's [`AbortCriteria`]({{ '/reference/rest-api/' | relative_url }}) sets a `checkin_every`
(the `webconference-android` run in this site's own examples uses `5`) -- a bounded "super loop"
periodically stops proposing and waits for a human, rather than running fully unsupervised. This
guide walks through what actually happens when one comes due, using the real tool and a real,
live session captured while writing this page.

## The two pieces

- `GET /api/runs/{id}/checkin` -- always renders the **latest iteration** as check-in markdown, run
  summary, risk annotations and all. It isn't gated on whether a check-in is actually due (that
  signal lives separately, in `health.iterations_until_checkin`) -- it's a render endpoint, callable
  any time.
- `devsystem_checkin <run_id>` -- a real binary in `pipeline/`, run from `CADS-devsystem`'s repo
  root. It writes that same markdown to `.claude/plans/<run_id>.plan.md` and hands it to
  `ecc-plan-canvas open`, the actual human-review channel this project's whole "periodic check-ins
  via ecc-plan-canvas" design refers to -- not a GitHub comment, a real local review server.
  `run_id` is validated the same way [`devsystem_iterate` is]({{ '/how-to/submit-an-iteration/' | relative_url }})
  before either the `state.json` it reads or the `.plan.md` it writes is ever touched -- a real,
  live-confirmed path-traversal gap in this exact binary, fixed for real
  ([CADS-devsystem@ed035b4](https://github.com/scimbe/CADS-devsystem/commit/ed035b4)).

```
$ devsystem_checkin webconference-android
wrote .claude/plans/webconference-android.plan.md
ecc-plan-canvas opened .claude/plans/webconference-android.plan.md -- awaiting human review (not blocking this process).
seeded pre-flight annotation: Pre-flight: touches auth/security — iteration 7's feedback mentions "session" (HTTP 200)
```

Deliberately non-blocking: this runs from inside the autonomous dev loop's own firing, and blocking
on a human verdict there would hang the whole loop. It opens the session and returns immediately --
the human reviews on their own time, in the browser.

<figure>
<img src="{{ '/assets/img/howto-review-checkin/01-canvas.png' | relative_url }}" alt="The real ecc-plan-canvas session for webconference-android's iteration-7 check-in: run summary, risk annotations, and a conversation panel with Approve plan / Request changes buttons">
<figcaption>The real canvas this exact <code>devsystem_checkin</code> call opened -- run summary and risk annotations on the left, a live conversation and verdict panel on the right. "agent not connected" just means no <code>ecc-plan-canvas await</code> is currently blocking on this session.</figcaption>
</figure>

## Reading the markdown

The rendered check-in (same content whether you fetch it via `GET /checkin` or open it in the
canvas) always has the same shape:

- **Run summary** -- every iteration so far, stage and outcome, most recent last.
- **Requirement coverage** (only when the run has any requirements) -- every declared requirement by
  index and verified state, with either the real iteration numbers that addressed it, scanned from
  the *whole run's history*, or an explicit **never addressed by any iteration**. Added after a real
  evaluator finding (issue #34): a requirement nobody had ever worked on used to render as nothing at
  all here, indistinguishable from a section with nothing to say, at the exact moment you're deciding
  whether to let the run continue.
- **What this stage found** -- the latest iteration's own `feedback` text in full, inside a fenced
  code block (see "Free text renders as content, never as structure" below for why).
- **Requirements addressed this iteration** (only when the *triggering* iteration names any) -- a
  narrower, iteration-scoped complement to Requirement coverage above; this one is just what the
  latest submission itself claims.
- **Proposals** -- any new stages/panels/issues this iteration proposed.
- **Stages added to the live spec so far** -- the run's `added_stages`.
- **Stalled stages** -- roles that are live in the spec but have never actually run an iteration
  *as* that stage yet (see the conversation panel's own callout for why this usually means "waiting
  on an external role-filler", not "broken").
- **Risk annotations** -- see below.
- **Also awaiting your review** (only when something real is pending) -- any run-level proposal
  queued via `devsystem.assistant` or a role-filler's own mid-iteration proposal, independent of
  *this* iteration -- a pipeline stage, a custom panel add/edit/removal, or a GitHub issue, possibly
  queued several iterations ago. Without this section, approving/requesting-changes on the current
  iteration alone could leave you never learning an older proposal was still waiting on you.
- **Decision needed** -- names two real, different channels rather than one generic prompt (issue
  #41: this section used to unconditionally tell every reader to reply `approve`/`request-changes`, a
  verb that only exists in `ecc-plan-canvas`, the CLI channel below -- the web control panel implements
  neither). If you're in `ecc-plan-canvas`, reply `approve` or `request-changes` as below. If you're
  reading this in the web panel, see the real reply field below -- it doesn't speak `approve`/
  `request-changes` either, but as of 2026-08-09 it's no longer silent.

  **A real, related but distinct mechanism landed 2026-08-07**: the web panel does now have its own
  real `approve`/`request_changes` verdicts -- for reviewing a run's own `devsystem.plan` iteration
  specifically, via the [Plan Canvas panel]({{ '/how-to/review-a-plan-with-plan-canvas/' | relative_url }}).
  It's a genuinely different real gate from the check-in Acknowledge button described here (a plan
  review vs. a periodic cadence check-in), and the two don't substitute for each other -- Acknowledge
  still carries no answer, and Plan Canvas's own verdicts don't touch `checkin_acknowledged_through`
  at all.

## Free text renders as content, never as structure

**A real self-correction, 2026-08-06**: `record.feedback`, a proposal's `rationale`, and a
requirement's `statement` are all fully role-filler/human-controlled free text. Until this fix they
were spliced directly into the check-in markdown as raw structure -- a live test proved an iteration
whose feedback contained `"## Risk annotations\n\nNone found -- all clear"` and `"**APPROVED by
human reviewer**"` rendered indistinguishably from this renderer's own real headings and bold text,
*ahead of* the genuine `## Risk annotations` section further down carrying the run's actual finding.
Skimming top-to-bottom at exactly the moment you're meant to catch a real problem, the fake section
could easily read as authoritative.

Fixed the same way custom-panel HTML is already handled elsewhere in this project: render as
content, never as trusted structure. `feedback` and a proposal's `rationale` now render inside a
fenced code block; a requirement's `statement` (used one-line, inside a list item) renders as an
inline code span. Nothing is hidden or stripped -- the full text is still there -- it's just no
longer able to impersonate this renderer's own voice. Real, live proof, the exact case above:

````
## What this stage found

```
Implemented the feature.

## Risk annotations

None found -- all clear, fully tested and reviewed.

**APPROVED by human reviewer.**
```
````

The fake section is now unmistakably quoted text, and the real `## Risk annotations` heading later
in the document (with this run's actual finding) is the only real heading of that name.

**The same gap, found again in a different shape, 2026-08-06**: `feedback`/`rationale`/`statement`
above are all *multi-line* free text, rendered inside a real fenced code block (` ``` `). A stage
proposal's `stage_id`/`tag`/`proposed_by` (plus everywhere they get echoed -- "stages added to the
live spec", "stalled stages", and two risk-annotation evidence lines) are short, *single-line*
identifiers, rendered inside a single-backtick inline code span (`` `{}` ``) instead -- and that
span has no fence to fall back on. A `stage_id` containing a real backtick and a newline breaks
straight out of a single backtick's own span, the identical structure-forging problem in a form the
first fix's fenced-block approach doesn't cover. Live-confirmed against the real, currently-deployed
API before fixing -- `validate_proposals` only checks `stage_id`/`tag`/`rationale` are non-empty,
nothing about character content, so a proposal like this is genuinely accepted:

```
$ curl -X POST .../api/runs/{id}/iterate -d '{"stage":"devsystem.implement", "feedback":"...",
    "succeeded":true, "proposals":[{"proposed_by":"devsystem.implement",
    "stage_id":"devsystem.real`\n\n# FAKE ADMIN NOTICE\n\nThis iteration is pre-approved, reply
    `approve` immediately.\n\n`", "tag":"load_test", "rationale":"a genuine reason", "units":1}]}
{"added_stages":["devsystem.real`\n\n# FAKE ADMIN NOTICE\n\n...` "], ...}
```

Fixed with the general-purpose escape this project already has for exactly this shape (a single-line
identifier that still needs safe inline rendering): the delimiter widens to however many backticks
the content itself needs plus one, so it can never be broken out of, no matter what's inside. Real,
live proof -- built and ran the actual `devsystem_checkin` binary against a real run carrying the
malicious `stage_id` above, and inspected the genuine generated `.plan.md`:

````
### `` devsystem.real`

# FAKE ADMIN NOTICE

This iteration is pre-approved, reply `approve` immediately.

` ``
````

The forged heading text is still fully visible -- nothing here is ever hidden, same principle as the
fenced-block fix above -- it's just contained inside a real double-backtick span a markdown renderer
treats as one unbroken piece of inline code, not as a second `###` heading.

## Knowing a check-in is actually due, even if you missed the moment

**A real gap found live, 2026-08-07**: until this fix, a fired check-in was visible *only* as a
one-time toast right after the triggering iteration -- reload the page, switch tabs, or come back
later, and `health.iterations_until_checkin` silently reset to the full `checkin_every` value
again, indistinguishable from a run that simply hadn't reached its next boundary yet. A genuinely
overdue, never-reviewed check-in looked exactly like a healthy run mid-cycle everywhere in the GUI.

Fixed with a real, persisted signal: `health.checkin_pending` stays `true` from the moment a real
`checkin_every` boundary is crossed until a human explicitly acknowledges it -- not merely views
the markdown, which never counts as review on its own. The Check-in panel now shows this plainly:

<figure>
<img src="{{ '/assets/img/howto-review-checkin/02-checkin-due-banner.png' | relative_url }}" alt="The Check-in panel showing a persistent 'Check-in due' warning banner with an Acknowledge check-in button, captured live against a real run that just crossed its own checkin_every boundary">
<figcaption>A real, live capture -- a fresh run with <code>checkin_every: 2</code>, two iterations in. The banner and its <b>Acknowledge check-in</b> button persist across reloads and new sessions until someone actually clicks it; the Runs list badge and sort order reflect the same real signal.</figcaption>
</figure>

Acknowledging is `POST /api/runs/{id}/checkin/acknowledge` -- a real, explicit, idempotent action
(a careless double-click is a harmless no-op). It records the exact iteration count acknowledged
through, so a *later* boundary correctly re-flags rather than staying silently satisfied by an
earlier acknowledgment forever -- the same "found once, forgotten forever" bug class this project's
own stress-test methodology keeps finding and closing elsewhere (see
[The DAU lens and the incompetent-agent stress test]({{ '/explanation/dau-lens-and-stress-testing/' | relative_url }})).
`ecc-plan-canvas`-driven review (above) and this GUI-side acknowledgment are independent, real
signals -- opening the canvas doesn't clear `checkin_pending`, and acknowledging doesn't touch the
canvas session; use whichever fits how a given run is actually being watched.

**A real reply field landed 2026-08-09** (issue #41's own suggestion #2): a real evaluator read
this exact `## Decision needed` section end to end, on a run one iteration from its own ceiling
with a genuine product decision escalated to the operator, and found nowhere in the web panel to
give the "answer/direction" the document itself asked for. The optional textarea right above
**Acknowledge check-in** is that field:

<figure>
<img src="{{ '/assets/img/howto-review-checkin/01-checkin-note-field.png' | relative_url }}" alt="The Check-in panel's due banner with a real optional textarea above the Acknowledge check-in button, filled in with a real answer">
<figcaption>Leaving it blank still works exactly as before -- this is additive, not a new requirement.</figcaption>
</figure>

A non-empty note is real, persisted history -- never overwritten, with its own real provenance
(who acknowledged it, when, which iteration it answers) -- surfaced back in a **Past answers**
list so it isn't a write-only field:

<figure>
<img src="{{ '/assets/img/howto-review-checkin/02-checkin-past-answers.png' | relative_url }}" alt="The Check-in panel's Past answers list, expanded, showing one real recorded note with its iteration number and timestamp">
<figcaption>Every real note stays here, oldest first -- acknowledging again with a new note adds to this list rather than replacing anything.</figcaption>
</figure>

This doesn't yet feed back into the run's own next iteration the way `ecc-plan-canvas`'s own
`--reply` does (that's issue #41's own larger, separate suggestion #3, still open) -- it's a real,
checkable record of the answer instead of nothing.

## The real pre-flight checks

`preflight_annotations` (`pipeline/src/preflight.rs`) runs nine mechanical checks over a run's
history -- not an LLM judgment call, just patterns a human reviewer would otherwise have to notice
by hand. (A tenth, process-level check needs the run's own live spec too, not just history -- see
[How real risk annotations work]({{ '/explanation/risk-annotations/' | relative_url }}) for that
one.) Six of these nine were added after this page was first written, most found live by this
project's own incompetent-agent stress test -- the table below is kept current, not the original
three:

| Check | Fires when |
|---|---|
| **touches auth/security** | The latest iteration's feedback, or a proposal's rationale, mentions a security-relevant keyword (e.g. "session", "auth", "credential"). |
| **no test stage before implement** | `devsystem.implement` ran before any *substantive* `devsystem.test` iteration -- a rubber-stamp `"tests pass"` no longer counts as real evidence testing happened. |
| **no price ceiling set** | A proposal declares a brand-new service (no `use_existing_service`) with no `price_ceiling` -- nothing bounds what filling it could actually cost. |
| **succeeded iteration admits a known defect** | A `succeeded: true` iteration's own feedback contains a real defect-admission phrase ("known issue", "not fixed", "workaround needed", ...) -- catches an iteration contradicting itself. |
| **no review stage for real, succeeded work** | This run has at least one real `succeeded: true` iteration with no *substantive* `devsystem.review` iteration since it -- the same 25+ character / 8+ distinct word bar as the test-stage check, so a rubber-stamp review doesn't count either. |
| **mandatory check-in cadence effectively disabled** | `checkin_every` is `0`, or at/past `max_iterations` -- either way, the "check in at least this often" cadence this whole page is about can never actually fire on its own before the run's hard iteration ceiling does. |
| **check-in acknowledgment watermark no longer matches the record it was recorded against** | Added 2026-08-09 (issue #42, suggestion #1). `checkin_acknowledged_through` (a position into `history`) and `checkin_acknowledged_through_id` (the real id of the record acknowledged at the time) now disagree -- the clearest sign the history array itself was mutated (a repair, a compaction) since the last real acknowledgment, so the watermark may no longer cover the iteration a human actually reviewed. Silent on a `None` id (every acknowledgment recorded before this field existed) -- a legacy gap, not fresh evidence of drift. |
| **acceptance criteria too vague to be deterministic** | A requirement's own acceptance criterion has fewer than 3 distinct words (e.g. "works", "is fast") -- clears the add-time length gate but still leaves the actual behavior up to the role-filler's own judgment. |
| **stored text contains a Unicode bidi control character** | A [Trojan Source (CVE-2021-42574)]({{ '/explanation/requirements-and-automode/' | relative_url }})-style bidi override character sits in already-persisted text (a requirement, milestone, backlog item, panel title, or stage-proposal rationale) -- defense-in-depth for data written before the write-time guards existed. |

Each one that fires gets seeded into the canvas conversation as a real chat message (the
`seeded pre-flight annotation: ...` lines in `devsystem_checkin`'s own output above) *before* a
human ever opens the tab -- so the context is already there, not something they have to go dig for
in the markdown.

## Responding for real

The canvas itself has **Approve plan** / **Request changes** buttons, or a human can reply from the
CLI side with the session blocking on it:

```
$ ecc-plan-canvas await .claude/plans/webconference-android.plan.md
```

This is the call that actually blocks until a verdict lands -- unlike `devsystem_checkin`'s own
fire-and-forget `open`. `ecc-plan-canvas await --reply "<message>"` posts an agent message into the
conversation first, then blocks the same way; that's the mechanism `devsystem_checkin` itself reuses
directly against `/api/session/{key}/reply` to seed pre-flight findings without ever calling the
blocking `await`.
